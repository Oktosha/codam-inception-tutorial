# Inception — A Hands‑On Tutorial (Codam)

> ℹ️ This document is configured for intra login **`dkolodze`** → domain `dkolodze.42.fr`, host path `/home/dkolodze/data`.

This tutorial is built in **four stages**, matching how you asked to learn:

1. **Docker basics** — refresh by playing with a single image.
2. **Docker Compose** — what it is, then build the *whole* Inception network using **ready‑made images** (fast, so you see the architecture work end‑to‑end).
3. **Custom images** — replace every ready‑made image with your own `Dockerfile`s, because the subject **forbids** ready‑made images.
4. **Compliance & defense** — the checklist of subject rules, the README/doc files, and the questions you'll be asked.

Stages 2 and 3 are deliberately separate. Stage 2 is a *throwaway sandbox* to understand Compose. Stage 3 is the *real project*. **Only stage 3 goes into your repo.**

---

## Table of contents

- [Stage 0 — Mental model & what the subject demands](#stage-0)
- [Environment setup (Apple Silicon / M2 Mac)](#env-m2)
- [Stage 1 — Docker basics with one image](#stage-1)
- [Stage 2 — Compose, with ready‑made images (sandbox)](#stage-2)
- [Stage 3 — Custom images (the real project)](#stage-3)
  - [3.1 Project layout](#stage-3-1)
  - [3.2 Secrets and .env](#stage-3-2)
  - [3.3 docker-compose.yml (the orchestrator)](#stage-3-3)
  - [3.4 MariaDB image (+ isolation test)](#stage-3-4)
  - [3.5 WordPress + php‑fpm image (+ isolation test)](#stage-3-5)
  - [3.6 NGINX image (+ isolation test)](#stage-3-6)
  - [3.7 Makefile](#stage-3-7)
- [Stage 4 — Rules checklist, docs & defense prep](#stage-4)
- [Bonus](#bonus)

---

<a name="stage-0"></a>
## Stage 0 — Mental model & what the subject demands

### The architecture (from the subject diagram)

```
                                   WWW (your browser)
                                        |
                                        | 443 (HTTPS, TLSv1.2/1.3 ONLY)
                                        v
  Computer HOST  ┌───────────────────────────────────────────────┐
                 │  Docker network "inception"                    │
                 │                                                 │
  /home/dkolodze/   │   ┌────────┐  3306   ┌──────────────┐  9000  ┌──────────┐
   data/         │   │ MariaDB│<------->│ WordPress    │<------>│  NGINX   │
   ├─ db    <────────│ (db)   │         │ + php-fpm    │        │ (nginx)  │──> 443
   └─ wordpress<─────┐         │         │ (wordpress)  │        │          │
                 │   └────────┘         └──────────────┘        └──────────┘
                 └───────────────────────────────────────────────┘
   named volumes:  db  -> /home/dkolodze/data/db    (MariaDB data)
                   wp  -> /home/dkolodze/data/wordpress  (WP site files)
```

Three services, three containers, three custom images:

| Service     | Image name  | Contains                          | Talks to        | Exposes |
|-------------|-------------|-----------------------------------|-----------------|---------|
| `nginx`     | `nginx`     | NGINX + TLS, **the only entrypoint** | wordpress:9000 | 443 → host |
| `wordpress` | `wordpress` | WordPress + php‑fpm (no nginx)    | mariadb:3306    | 9000 (internal only) |
| `mariadb`   | `mariadb`   | MariaDB only (no nginx)           | —               | 3306 (internal only) |

> **Image name == service name.** The subject says: *"Each Docker image must have the same name as its corresponding service."*

### Hard rules you must not break (memorize these)

- ✅ Done **in a Virtual Machine** (Docker on macOS/Windows desktop is fine for *learning*, but the graded project must run in a Linux VM).
- ✅ Use **`docker compose`**.
- ✅ **Build your own images** from **Dockerfiles** (one per service). Pulling ready‑made app images (WordPress/NGINX/MariaDB from DockerHub) is **forbidden**. Only the **Alpine** or **Debian** base image is allowed.
- ✅ Base = **penultimate stable** Alpine or Debian. *Penultimate = second‑newest stable release.* (See note below.)
- ❌ The **`latest`** tag is forbidden — always pin a version.
- ❌ **No passwords in Dockerfiles.** Use environment variables + a **`.env`** file. Docker **secrets** are strongly recommended for credentials.
- ❌ **No `network: host`, no `--link`, no `links:`.** You must declare your own network; the `networks` line **must be present** in `docker-compose.yml`.
- ❌ **No infinite‑loop hacks** as the main process: no `tail -f`, no `sleep infinity`, no `while true`, no `bash` as PID 1 just to keep the container alive. Each container runs its service as a **foreground daemon (PID 1)**.
- ✅ **Named volumes only** (no bind mounts) for the two persistent stores, and they must physically live in `/home/dkolodze/data`.
- ✅ Containers must **restart on crash**.
- ✅ NGINX is the **only entrypoint**, **port 443 only**, **TLSv1.2 or TLSv1.3 only**.
- ✅ WordPress DB must have **two users**, one is the admin; the admin username **must not** contain `admin`/`administrator` (case‑insensitive). E.g. don't use `admin`, `Admin`, `administrator`, `admin-123`. Pick something like `boss`, `wp_owner`, `sitemanager`.
- ✅ Domain `dkolodze.42.fr` points to your local IP.

> **Penultimate stable — how to pick the tag.** Check the official release pages, then pick the *second newest*:
> - **Alpine:** look at https://alpinelinux.org/releases/ — if the newest stable branch is `3.23`, the penultimate is `3.22`. Pin e.g. `FROM alpine:3.22`.
> - **Debian:** if newest stable is `trixie` (13), penultimate is `bookworm` (12). Pin `FROM debian:bookworm`.
>
> This tutorial uses **Alpine** (your choice). **Verify the current numbers yourself** on the day you build — I'm writing this with `3.21`/`3.22`-era numbers, and you must use whatever is penultimate when you submit. The evaluator *will* ask why you chose that tag.

---

<a name="env-m2"></a>
## Environment setup (Apple Silicon / M2 Mac)

The subject requires the project to run **inside a Virtual Machine**. At school you have an
**x86_64** Ubuntu VM in VirtualBox. On an **M2 Mac (ARM64 / aarch64)** that exact setup can't be
reproduced — but you don't need it to be. Here's why, and how to set up an equivalent.

### The architecture reality (read this first)

- School VM = **x86_64**, M2 Mac = **ARM64**. Different CPU architectures.
- ❌ You **cannot** import your school `.vdi`/`.ova` onto the M2. Stable **VirtualBox is x86-only**;
  the Apple-Silicon build is an experimental ARM-only preview. **Don't try to replicate VirtualBox.**
- ❌ Running an x86_64 VM on the M2 = full CPU **emulation** = correct but painfully slow. Avoid.
- ✅ You **can** run the **same Ubuntu version as an ARM64 build**, natively and fast. Same distro,
  same tools, same workflow — only the arch under the hood differs. That's as close to "the same VM"
  as is physically possible.

### Why your Inception code needs (almost) no changes

Docker base images are **multi-arch**: `FROM alpine:3.22` automatically pulls the **arm64** variant
on the M2 and the **x86_64** variant at school — *same Dockerfile, no edits*. Every package you
install (`nginx`, `mariadb`, `php82-fpm`, `openssl`) exists in the Alpine arm64 repos, and **WP-CLI
is a PHP `.phar`, which is architecture-independent.**

Because your `Makefile` rebuilds images from the Dockerfiles (`docker compose up -d --build`), moving
to the school x86_64 VM simply **rebuilds from the same source** for x86_64. The project is
reproducible on both machines; you never copy built images between them — you rebuild.

**Cross-arch do's and don'ts:**

- ❌ Don't add `platform: linux/amd64` in `docker-compose.yml`, or `FROM --platform=...` in a
  Dockerfile — that forces slow emulation.
- ❌ Don't download arch-specific **binaries** (a prebuilt amd64 static binary, an arch-pinned
  `.deb`, etc.). Stick to `apk` packages + the WP-CLI phar — the approach in this tutorial is
  already arch-clean. ✅
- ❌ Don't copy the named-volume data in `/home/dkolodze/data` between home and school. Let `make`
  create it fresh on each machine.
- ℹ️ "Penultimate stable Alpine" is a **version** question, not an arch question — verify the current
  number the same way on either machine.

### Which hypervisor to use on the M2

| Option | Cost | Notes |
|--------|------|-------|
| **UTM** ⭐ | Free (open source) | QEMU + Apple Virtualization. Runs Ubuntu ARM64 natively, fast. Best free GUI option; closest "VirtualBox replacement" feel. |
| **VMware Fusion** | Free (personal use) | Full-featured, runs Ubuntu ARM well. |
| **Parallels Desktop** | Paid (trial) | Smoothest Ubuntu-ARM experience; overkill if budget matters. |
| **Multipass** | Free (Canonical) | Headless Ubuntu VMs via CLI (`multipass launch`). Great if you live in the terminal and don't need a desktop GUI. |

**Recommended:** **UTM + Ubuntu 24.04 ARM64** (Server if you're fine in the terminal, Desktop if you
want a browser inside the VM). Free, native-speed, and it gives you the genuine "I provisioned and
administer a Linux VM" experience the subject wants — your defense is cleaner showing a real VM than
Docker Desktop's hidden one.

> A note on Docker Desktop / Colima / Lima directly on macOS: they spin up a hidden Linux VM and
> Inception would *run*, but you lose the "set up a VM" part of the exercise. Use a proper VM.

### Setup steps (UTM)

1. Download **UTM** (mac.getutm.app) and the **Ubuntu 24.04 ARM64** ISO (ubuntu.com/download/server
   — the ARM/`arm64` build, *not* amd64).
2. In UTM: **Create a New VM → Virtualize → Linux**, attach the ARM64 ISO, give it ~2 CPUs, 4 GB RAM,
   25+ GB disk. Install Ubuntu as usual.
3. Inside the VM, install the toolchain:
   ```bash
   sudo apt update && sudo apt install -y make git ca-certificates curl
   # Docker Engine + compose plugin (official convenience script):
   curl -fsSL https://get.docker.com | sudo sh
   sudo usermod -aG docker $USER     # log out/in so docker works without sudo
   docker compose version            # confirm the compose plugin is present
   ```
4. Clone your repo *inside the VM*, develop and test exactly as the tutorial describes.
5. `git push` from the VM → at school, `git pull` into the x86_64 VM and run `make`. It rebuilds for
   x86_64 with no edits.

### Accessing the site from a browser

The subject says the domain must point to your **local IP**. Inside the VM that's `127.0.0.1`. So the
*canonical* setup is a `/etc/hosts` entry **inside the VM** and a browser **inside the VM** — but on a
headless Server VM (or if you just prefer your Mac's browser) you reach it from the host instead.
Three scenarios:

**A) Browser inside the VM (any Desktop VM — simplest).**
```bash
# inside the VM:
echo "127.0.0.1 dkolodze.42.fr" | sudo tee -a /etc/hosts
```
Open `https://dkolodze.42.fr` in the VM's browser and accept the self-signed cert warning. Done.

**B) UTM Server on the M2 → view from the Mac's browser.**
1. UTM VM network = **Shared Network** (the default). Inside the VM, find its IP:
   ```bash
   ip -4 addr show | grep inet      # e.g. 192.168.64.x
   ```
2. On **macOS**, map the domain to that VM IP:
   ```bash
   sudo sh -c 'echo "192.168.64.x  dkolodze.42.fr" >> /etc/hosts'   # use your real VM IP
   ```
3. Open `https://dkolodze.42.fr` in Safari/Chrome on the Mac, accept the cert warning.
   *(The VM's own `/etc/hosts` should still point the domain at `127.0.0.1` for in-VM `curl` tests.)*

**C) VirtualBox at school → view from the host's browser.**
Easiest with **NAT + port forwarding** (no network reconfiguration needed):
1. VM **powered off** → *Settings → Network → Adapter 1* (leave it **NAT**) → *Advanced → Port Forwarding*.
2. Add a rule:

   | Name  | Protocol | Host IP   | Host Port | Guest IP | Guest Port |
   |-------|----------|-----------|-----------|----------|------------|
   | https | TCP      | 127.0.0.1 | 443       | *(blank)*| 443        |

3. On the **host**, map the domain to localhost:
   ```bash
   sudo sh -c 'echo "127.0.0.1  dkolodze.42.fr" >> /etc/hosts'
   ```
4. Start the VM, `make`, then open `https://dkolodze.42.fr` in the host browser.

> ⚠️ Binding **host** port 443 can require admin rights. If the rule fails or the port is taken, use
> **Host Port 8443 → Guest Port 443** instead and browse `https://dkolodze.42.fr:8443`. This only
> changes the *host-side test URL* — nginx still listens on 443 inside the VM, so the subject's
> "443 only" rule is unaffected.
>
> Alternative: set Adapter 1 to **Bridged Adapter**. The VM then gets a real LAN IP (`ip -4 addr`),
> and you map `dkolodze.42.fr → that LAN IP` in the host's `/etc/hosts` — no port forwarding needed.
>
> Simplest of all at school: if the VirtualBox VM has a **desktop**, just use scenario **A** and
> browse inside the VM.

---

<a name="stage-1"></a>
## Stage 1 — Docker basics with one image

Goal: shake the rust off. Do this in a scratch directory, **not** in your repo.

```bash
mkdir -p ~/docker-playground && cd ~/docker-playground
```

### 1.1 Run a container, look inside

```bash
# Pull + run an interactive Alpine shell. --rm deletes the container on exit.
docker run --rm -it alpine:3.22 sh

# now you're INSIDE the container:
cat /etc/os-release      # tiny Alpine Linux
apk add --no-cache curl  # install something
exit                     # container is destroyed (--rm)
```

Key idea: a **container** is a running instance of an **image**. The image is the read‑only template; the container is the live, writable process.

### 1.2 The core verbs

```bash
docker images                 # list local images
docker ps                     # running containers
docker ps -a                  # all containers (incl. stopped)
docker run -d --name web nginx:1.27-alpine   # run detached (background)
docker logs web               # see its output
docker exec -it web sh        # get a shell in a RUNNING container
docker stop web && docker rm web             # stop + remove
docker rmi nginx:1.27-alpine  # remove the image
```

> ℹ️ `nginx:1.27-alpine` here is just for *practice* in stage 1. You will **not** use ready‑made nginx in the final project.

### 1.3 Ports — getting traffic in

A container is isolated. To reach a service from your host you **publish** a port with `-p HOST:CONTAINER`:

```bash
docker run -d --name web -p 8080:80 nginx:1.27-alpine
# open http://localhost:8080  -> you see the nginx welcome page
docker rm -f web
```

`-p 8080:80` = "forward host port 8080 to container port 80."

### 1.4 Build your *first* image from a Dockerfile

This is the muscle you'll use all project long. Create `Dockerfile`:

```dockerfile
# ~/docker-playground/Dockerfile
FROM alpine:3.22
RUN apk add --no-cache curl
# A container needs a foreground process to stay alive.
# This one just prints and exits — that's fine for a demo.
CMD ["echo", "Hello from my own image"]
```

```bash
docker build -t myfirst .     # -t = tag/name ; . = build context (this dir)
docker run --rm myfirst       # prints the message, then exits
```

**Dockerfile instructions you'll actually use:**

| Instruction | Meaning |
|-------------|---------|
| `FROM`      | Base image (yours: `alpine:3.22`) |
| `RUN`       | Run a command **at build time** (install packages, etc.) |
| `COPY`      | Copy files from build context into the image |
| `EXPOSE`    | Document a port (does **not** publish it) |
| `ENTRYPOINT`| The fixed executable that runs when the container starts |
| `CMD`       | Default args (or default command). Combined with ENTRYPOINT. |

### 1.5 Volumes — persisting data

Containers are ephemeral; their writable layer dies with them. **Volumes** outlive containers.

```bash
docker volume create mydata
docker run --rm -v mydata:/data alpine:3.22 sh -c "echo persisted > /data/file.txt"
docker run --rm -v mydata:/data alpine:3.22 cat /data/file.txt   # still there!
docker volume rm mydata
```

That `-v name:/path` is a **named volume** (good, project‑legal).
`-v /host/path:/path` would be a **bind mount** (forbidden for your two stores).

### 1.6 The PID 1 idea (critical for this project)

When a container starts, your command becomes **PID 1** — the init process. When PID 1 exits, the container stops. That's *why* services must run in the **foreground**:

- `nginx` → run with `daemon off;` so it doesn't fork into the background and exit.
- `php-fpm` → run with `-F` (foreground).
- `mariadbd`/`mysqld` → it stays in the foreground by default.

❌ The forbidden hacks (`tail -f /dev/null`, `sleep infinity`, `while true`) are people *faking* a foreground process because their real daemon backgrounded itself and the container died. **Don't do that** — run the real daemon in the foreground instead.

> ✅ **You now have the basics.** Clean up: `docker system prune -af` (removes unused images/containers — careful, it's aggressive).

---

<a name="stage-2"></a>
## Stage 2 — Compose, with ready‑made images (sandbox)

> 🧪 **This stage is a learning sandbox.** Do it in `~/docker-playground/compose-demo/`, **NOT in your repo.** Here we *break* the rules (ready‑made images, bind mounts) on purpose so you can see the whole network stand up in 2 minutes and understand Compose before writing Dockerfiles.

### 2.1 What Compose is

Running 3 containers by hand (`docker run ... ; docker run ...`) with the right ports, network, env, and volumes is painful. **Docker Compose** declares the whole stack in one YAML file and brings it up with one command. Concepts map 1:1 to `docker run` flags:

| `docker run` flag        | Compose key   |
|--------------------------|---------------|
| image name               | `image:`      |
| `--name`                 | `container_name:` |
| `-p 443:443`             | `ports:`      |
| `-e KEY=val` / `--env-file` | `environment:` / `env_file:` |
| `-v vol:/path`           | `volumes:`    |
| `--network`              | `networks:`   |
| `--restart`              | `restart:`    |
| `build .`                | `build:`      |

### 2.2 A throwaway compose file (ready‑made images)

`~/docker-playground/compose-demo/docker-compose.yml`:

```yaml
# SANDBOX ONLY — uses ready-made images, which are FORBIDDEN in the real project.
services:
  mariadb:
    image: mariadb:11.4          # ready-made (forbidden in real project)
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db:/var/lib/mysql
    networks:
      - inception
    restart: always

  wordpress:
    image: wordpress:6-php8.2-fpm  # ready-made (forbidden in real project)
    depends_on:
      - mariadb
    environment:
      WORDPRESS_DB_HOST: mariadb:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
    volumes:
      - wp:/var/www/html
    networks:
      - inception
    restart: always

  nginx:
    image: nginx:1.27-alpine       # ready-made (forbidden in real project)
    depends_on:
      - wordpress
    ports:
      - "8443:443"                 # we'll do real TLS on 443 in stage 3
    volumes:
      - wp:/var/www/html
    networks:
      - inception
    restart: always

volumes:
  db:
  wp:

networks:
  inception:
```

> This sandbox won't fully serve WordPress over TLS (nginx here has no PHP wiring/cert) — that's intentional. The point is to *feel* Compose: services, one network, named volumes, `depends_on`, env vars.

### 2.3 Drive Compose

```bash
cd ~/docker-playground/compose-demo
docker compose up -d            # build/pull + create network + volumes + start all
docker compose ps               # status of each service
docker compose logs -f mariadb  # follow one service's logs
docker compose exec mariadb sh  # shell into a service
docker compose down             # stop + remove containers + network (keeps volumes)
docker compose down -v          # also delete the named volumes
```

**What `up` did for you automatically:**
- Created a **user‑defined bridge network** `composedemo_inception`. On it, containers reach each other **by service name** — `wordpress` connects to `mariadb` simply via the host `mariadb`. (This is why `--link` is obsolete and forbidden.)
- Created named **volumes** `composedemo_db`, `composedemo_wp`.
- Started everything with `restart: always`.

Play with it:
```bash
docker compose exec mariadb mariadb -uwpuser -pwppass -e "SHOW DATABASES;"
docker network inspect composedemo_inception   # see both containers attached
```

> ✅ **Checkpoint:** You now understand Compose: one file, one network, service‑name DNS, named volumes, restart policy. Tear it down (`docker compose down -v`) and **forget this folder** — none of it goes in your repo.

---

<a name="stage-3"></a>
## Stage 3 — Custom images (the real project)

Now we rebuild that exact topology, but **every image is yours**, built from Alpine. **This is what goes into `codam-inception/`.**

<a name="stage-3-1"></a>
### 3.1 Project layout

The subject's expected structure (adapted):

```
codam-inception/                 <- repo root
├── Makefile
├── README.md
├── USER_DOC.md
├── DEV_DOC.md
├── secrets/                     <- git-ignored! real credentials (one secret per file)
│   ├── db_root_password.txt     <- MariaDB root password
│   ├── db_password.txt          <- WP DB user password
│   ├── wp_admin_password.txt    <- WordPress admin user password
│   └── wp_user_password.txt     <- WordPress second (non-admin) user password
└── srcs/
    ├── .env                     <- git-ignored! non-secret + referenced vars
    ├── docker-compose.yml
    └── requirements/
        ├── mariadb/
        │   ├── Dockerfile
        │   ├── .dockerignore
        │   ├── conf/            <- my.cnf etc.
        │   └── tools/           <- init/entrypoint script
        ├── nginx/
        │   ├── Dockerfile
        │   ├── .dockerignore
        │   ├── conf/            <- nginx.conf
        │   └── tools/
        └── wordpress/
            ├── Dockerfile
            ├── .dockerignore
            ├── conf/            <- www.conf (php-fpm) etc.
            └── tools/           <- wp setup entrypoint script
```

**`.gitignore` at repo root (do this first!):**
```gitignore
secrets/
srcs/.env
**/*.crt
**/*.key
```
> 🔒 The subject fails you outright if credentials are committed. `.env` and `secrets/` must **never** be pushed. (You'll commit *example* files instead — see DEV_DOC below.)

<a name="stage-3-2"></a>
### 3.2 Secrets and `.env`

**`srcs/.env`** (referenced by compose; non‑secret config + the names of things):
```dotenv
# Domain
DOMAIN_NAME=dkolodze.42.fr

# MySQL / MariaDB
MYSQL_DATABASE=wordpress
MYSQL_USER=wp_user
# NOTE: passwords are NOT here — they come from secrets files (see below)

# WordPress
WP_TITLE=Inception
WP_ADMIN_USER=bossman           # MUST NOT contain admin/administrator
WP_ADMIN_EMAIL=bossman@example.com
WP_USER=editor
WP_USER_EMAIL=editor@example.com
WP_URL=https://dkolodze.42.fr
```

**`secrets/` files** — **one secret per file**, each containing only the raw value, git‑ignored:
```
secrets/db_root_password.txt    ->  a strong MariaDB root password
secrets/db_password.txt         ->  the wp_user DB password
secrets/wp_admin_password.txt   ->  the WordPress admin user password
secrets/wp_user_password.txt    ->  the WordPress second (non-admin) user password
```

Each file holds **just the password**, nothing else — no `KEY=`, no quotes, no label. This is the Docker‑secrets ideal: *one secret = one file = one value*, read directly with `cat`. No parsing, nothing to misquote.

> 📝 **Trailing‑newline gotcha.** If your editor adds a trailing newline to every file (a good habit — POSIX text files should end with `\n`), **keep it**: every read in this tutorial uses command substitution `DB_PASS="$(cat /run/secrets/db_password)"`, and `$(...)` strips *all* trailing newlines automatically. So the password is clean before it's ever used. The only way a trailing `\n` bites you is if you feed the secret *file path* to a tool that reads its raw bytes without trimming (some programs take a `..._FILE` argument and read it literally) — we never do that here. **Rule of thumb: always read secrets via `$(cat ...)`, then trailing newlines never matter.**

**How secrets reach the container.** Docker Compose `secrets:` mounts each file read‑only at `/run/secrets/<name>` inside the container. Your entrypoint scripts read the password from that file — so **no password is ever baked into an image or printed in compose**. We wire this in 3.3 (the compose file, next).

> 💡 **Secrets vs environment variables** (a defense question): env vars are visible via `docker inspect` and leak into logs/`/proc`; file‑based secrets are mounted only into the containers that declare them and aren't in the image history. Use **secrets for passwords**, env vars for non‑sensitive config like the DB name and domain.

<a name="stage-3-3"></a>
### 3.3 docker-compose.yml (the orchestrator)

We write the compose file **first**, before the Dockerfiles, for one practical reason: it lets you **build and test each container in isolation as you write it**. `docker compose up <service>` builds and starts only that one service (plus anything it `depends_on`) — so you can bring MariaDB up alone, verify it, then add WordPress, then NGINX, instead of debugging all three at once at the end.

The file lists all three services now, but the build contexts (`./requirements/mariadb`, etc.) are just *referenced* — Compose only touches the one you name on the command line, so the others not existing yet is fine.

**`srcs/docker-compose.yml`:**
```yaml
services:
  mariadb:
    build: ./requirements/mariadb
    image: mariadb                 # image name == service name
    container_name: mariadb
    env_file: .env
    secrets:
      - db_root_password
      - db_password
    volumes:
      - db:/var/lib/mysql
    networks:
      - inception
    restart: always

  wordpress:
    build: ./requirements/wordpress
    image: wordpress
    container_name: wordpress
    depends_on:
      - mariadb
    env_file: .env
    environment:
      WORDPRESS_DB_HOST: mariadb:3306
    secrets:
      - db_password
      - wp_admin_password
      - wp_user_password
    volumes:
      - wp:/var/www/html
    networks:
      - inception
    restart: always

  nginx:
    build: ./requirements/nginx
    image: nginx
    container_name: nginx
    depends_on:
      - wordpress
    ports:
      - "443:443"                  # the ONLY published port
    volumes:
      - wp:/var/www/html           # nginx serves the WP files
    networks:
      - inception
    restart: always

volumes:
  db:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/dkolodze/data/db        # named volume backed by host path
  wp:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /home/dkolodze/data/wordpress

networks:
  inception:
    driver: bridge

secrets:
  db_root_password:
    file: ../secrets/db_root_password.txt
  db_password:
    file: ../secrets/db_password.txt
  wp_admin_password:
    file: ../secrets/wp_admin_password.txt
  wp_user_password:
    file: ../secrets/wp_user_password.txt
```

> 🧠 **"But that volume uses `o: bind` — isn't bind‑mount forbidden?!"**
> No — this is the standard, accepted Inception pattern and it's a **named volume**, not a service‑level bind mount. The subject forbids writing `- /host/path:/container/path` directly under a service's `volumes:`. Here you declare a *named* volume `db`/`wp` (services reference it by name), and you tell Docker's `local` driver to store that named volume's data at a host path under `/home/dkolodze/data` — which the subject *explicitly requires* ("both named volumes must store their data inside `/home/dkolodze/data`"). Be ready to explain this distinction at defense.

> 🔁 **Optional, cleaner dependency handling:** instead of the wait‑loop in WordPress, add a `healthcheck` to `mariadb` and use `depends_on: { mariadb: { condition: service_healthy } }` on `wordpress`. Either approach is fine; the loop is simpler to reason about for a first submission.

**Prerequisite for any test or run** — the named volumes are backed by host paths that must exist first (the Makefile does this for you later):
```bash
mkdir -p /home/dkolodze/data/db /home/dkolodze/data/wordpress
```

#### ✅ Verify the compose file before building anything

You can sanity-check the compose file the moment you've written it — **no images or Dockerfiles required**:

```bash
docker compose -f srcs/docker-compose.yml config
```

This parses and fully resolves the file — merging `.env`, expanding variables, and validating structure — then prints the final effective config (or fails with a precise line number). It's the fastest way to catch a YAML indentation slip, a typo'd key, an unresolved `${VAR}`, or a misreferenced secret/volume **before** you spend time on a build. Things to eyeball in the output:

- every service has the `image:` name matching its service name (`mariadb`/`wordpress`/`nginx`);
- the `networks:` block is present and no service uses `network_mode: host`;
- the two named volumes resolve to `device: /home/dkolodze/data/...`;
- the `secrets:` resolve to your `../secrets/*.txt` files (paths shown relative to `srcs/`).

> 💡 `config` reads `.env` and the secret *file paths*, but it does **not** print secret contents — so it's safe to run and share. If a `${VAR}` is missing you'll see it as empty or get a warning, which is exactly the early signal you want.

> 🧪 **How the per-service isolation tests below work.** Each container section (3.4–3.6) ends with a "Verify in isolation" block you run **as soon as you've written that service** — using `docker compose -f srcs/docker-compose.yml up -d --build <service>`. Because of `depends_on`, MariaDB comes up alone, WordPress brings up MariaDB+itself, and NGINX brings up the whole stack. If your Compose version objects to the not‑yet‑written build contexts, you can instead build that one image directly with `docker build` (shown for MariaDB in 3.4).

<a name="stage-3-4"></a>
### 3.4 MariaDB image (+ isolation test)

**`srcs/requirements/mariadb/Dockerfile`:**
```dockerfile
FROM alpine:3.22

RUN apk add --no-cache mariadb mariadb-client \
    && mkdir -p /var/lib/mysql /run/mysqld \
    && chown -R mysql:mysql /var/lib/mysql /run/mysqld

COPY conf/my.cnf /etc/my.cnf.d/inception.cnf
# Alpine's mariadb package ships /etc/my.cnf.d/mariadb-server.cnf with
# `skip-networking` enabled (no TCP listener). Files in /etc/my.cnf.d/ are read
# in alphabetical order, so mariadb-server.cnf loads AFTER inception.cnf and its
# skip-networking wins — leaving the DB reachable only over its unix socket.
# Neutralize it at the source so the wordpress container can connect over TCP.
RUN sed -i 's/^skip-networking/#skip-networking/' /etc/my.cnf.d/mariadb-server.cnf
COPY tools/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

EXPOSE 3306
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

> ⚠️ **Why the `sed` line matters (a real gotcha).** Without it, MariaDB starts but reports `port: 0` / `skip_networking = ON`, and the only TCP listener in the container is Docker's internal DNS — so WordPress's `mariadb -hmariadb ...` connection silently fails forever in the wait‑loop. The Alpine default config simply overrides your `inception.cnf` because it loads later alphabetically. Setting `skip-networking=0` in your own `my.cnf` (below) makes the intent explicit, but the `sed` is what actually wins the ordering — keep both.

**`srcs/requirements/mariadb/conf/my.cnf`:**
```ini
[mysqld]
skip-networking = 0          # make intent explicit: DO accept TCP connections
skip-host-cache
skip-name-resolve
bind-address = 0.0.0.0        # accept connections from the wordpress container
port = 3306
datadir = /var/lib/mysql
socket = /run/mysqld/mysqld.sock
```

**`srcs/requirements/mariadb/tools/entrypoint.sh`:**
```sh
#!/bin/sh
set -e

DB_ROOT_PASS="$(cat /run/secrets/db_root_password)"
DB_PASS="$(cat /run/secrets/db_password)"

# Initialize the data dir only on first run (volume is empty).
if [ ! -d "/var/lib/mysql/mysql" ]; then
    echo "Initializing MariaDB data directory..."
    mariadb-install-db --user=mysql --datadir=/var/lib/mysql --skip-test-db > /dev/null

    # Bootstrap: create DB, users, grants. --bootstrap runs SQL then exits.
    mariadbd --user=mysql --bootstrap <<EOF
USE mysql;
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '${DB_ROOT_PASS}';
CREATE DATABASE IF NOT EXISTS \`${MYSQL_DATABASE}\`;
CREATE USER IF NOT EXISTS '${MYSQL_USER}'@'%' IDENTIFIED BY '${DB_PASS}';
GRANT ALL PRIVILEGES ON \`${MYSQL_DATABASE}\`.* TO '${MYSQL_USER}'@'%';
FLUSH PRIVILEGES;
EOF
    echo "MariaDB initialized."
fi

# Hand over to the real daemon as PID 1 (foreground). 'exec' replaces the shell.
exec mariadbd --user=mysql --datadir=/var/lib/mysql
```

Why this is rule‑compliant:
- `exec mariadbd ...` runs the daemon as **PID 1 in the foreground** — no `tail -f`, no loops.
- Passwords are **read from `/run/secrets/...`**, never hardcoded.
- The init block is **idempotent** (only runs when the volume is fresh), so restarts/crashes don't wipe data.

#### ✅ Verify MariaDB in isolation

Run this the moment the three MariaDB files exist — debugging one service is far easier than the whole stack.

```bash
# from the repo root (~/codam-inception); the volume dir must exist (see 3.3):
mkdir -p /home/dkolodze/data/db
docker compose -f srcs/docker-compose.yml up -d --build mariadb   # builds & starts ONLY mariadb

# 1. Up, and init ran cleanly? (look for "ready for connections", no errors)
docker compose -f srcs/docker-compose.yml ps
docker compose -f srcs/docker-compose.yml logs mariadb

# 2. PID 1 is the real daemon — no tail -f/sleep hacks (subject rule)
docker compose -f srcs/docker-compose.yml exec mariadb ps aux   # mariadbd should be PID 1

# 3. Root password works + DB and both accounts exist
docker compose -f srcs/docker-compose.yml exec mariadb \
  mariadb -uroot -p"$(cat secrets/db_root_password.txt)" \
  -e "SHOW DATABASES; SELECT user,host FROM mysql.user;"
#    -> 'wordpress' DB present; 'root'@'localhost' and 'wp_user'@'%' present

# 4. The WordPress account can connect with ITS password, into ITS db
docker compose -f srcs/docker-compose.yml exec mariadb \
  mariadb -uwp_user -p"$(cat secrets/db_password.txt)" wordpress -e "SELECT 'ok' AS connect_test;"

# 5. Persistence: a full recreate keeps the data (init guard SKIPS the 2nd time)
docker compose -f srcs/docker-compose.yml down
docker compose -f srcs/docker-compose.yml up -d mariadb
docker compose -f srcs/docker-compose.yml logs mariadb   # should NOT re-initialize
```

| Check | Proves |
|-------|--------|
| 1 | Image builds, entrypoint runs, no fatal SQL errors |
| 2 | Daemon is PID 1 in the foreground (no forbidden keep‑alive hack) |
| 3 | `ALTER USER` set the root password; DB + accounts created right |
| 4 | `wp_user@'%'` + grants work → WordPress *will* be able to connect over the network |
| 5 | Named volume persists; the `if [ ! -d .../mysql ]` guard is idempotent |

> `-p"$(cat secrets/...txt)"` strips the file's trailing newline, so logins work despite your editor's trailing `\n`. A "password on the command line is insecure" warning is normal for a local test.
>
> **Reset between attempts** (to force a fresh init after editing the entrypoint):
> ```bash
> docker compose -f srcs/docker-compose.yml down
> sudo rm -rf /home/dkolodze/data/db/*
> ```
>
> **Compose-free fallback** (if your Compose version objects to the unwritten WordPress/NGINX contexts):
> ```bash
> cd srcs/requirements/mariadb && docker build -t mariadb-test .
> docker run --rm --name mariadb-test --env-file ../../.env \
>   -v ~/codam-inception/secrets/db_root_password.txt:/run/secrets/db_root_password:ro \
>   -v ~/codam-inception/secrets/db_password.txt:/run/secrets/db_password:ro \
>   -v mariadb_test:/var/lib/mysql mariadb-test
> ```
> (Bind‑mounting secret files is fine for a *test* — the bind‑mount ban only applies to your two persistent project volumes.)

<a name="stage-3-5"></a>
### 3.5 WordPress + php‑fpm image (+ isolation test)

**`srcs/requirements/wordpress/Dockerfile`:**
```dockerfile
FROM alpine:3.22

# php-fpm + the PHP extensions WordPress needs, plus tools to install WP.
RUN apk add --no-cache \
        php82 php82-fpm php82-mysqli php82-phar php82-json php82-curl \
        php82-dom php82-exif php82-fileinfo php82-mbstring php82-openssl \
        php82-xml php82-zip php82-gd php82-session php82-tokenizer \
        curl bash mariadb-client \
    && ln -sf /usr/bin/php82 /usr/bin/php \
    && mkdir -p /var/www/html /run/php \
    && adduser -D -H -G www-data -u 82 -s /sbin/nologin www-data

# Install WP-CLI (this is a TOOL, not a ready-made app image — allowed).
RUN curl -o /usr/local/bin/wp \
        https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x /usr/local/bin/wp

# php-cli defaults to memory_limit=128M, which WP-CLI's tarball extraction
# (`wp core download`) blows past with a fatal "Allowed memory size exhausted".
# Raise it for the CLI (the scan dir is read by both php-cli and php-fpm).
RUN echo "memory_limit = 512M" > /etc/php82/conf.d/99-memory.ini

COPY conf/www.conf /etc/php82/php-fpm.d/www.conf
COPY tools/entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

WORKDIR /var/www/html
EXPOSE 9000
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
```

> 📝 PHP package names depend on the Alpine version (`php82`, `php83`, …). On your chosen Alpine, run `apk search php8` to find the right ones, and **match the php‑fpm config path** (`/etc/phpXX/php-fpm.d/www.conf`).

> ⚠️ **Two Alpine‑specific gotchas baked into the Dockerfile above — don't drop them:**
> 1. **`www-data` user.** Alpine's `php82-fpm` package already creates the **group** `www-data` (GID 82) but **no user**. A naive `adduser -u 82 ... www-data` then fails because it tries to create a *new* group at GID 82 that already exists — and if you hide that with `2>/dev/null || true`, the failure passes silently and php‑fpm later dies with `cannot get uid for user 'www-data'` / `FPM initialization failed` (crash loop). The fix is `adduser ... -G www-data -u 82 ...` (reuse the existing group) **without** the `|| true`, so a real failure surfaces at build time.
> 2. **php‑cli `memory_limit`.** The default 128M is too low for `wp core download` to unzip WordPress; it dies with `Allowed memory size of 134217728 bytes exhausted`. Note that `WP_CLI_PHP_ARGS` does **not** help here — that env var is read only by wp‑cli's bash *wrapper*, while you installed the raw `.phar` directly, so it runs under the default ini. Raising `memory_limit` via an ini in the scan dir (as above) is the reliable fix.

**`srcs/requirements/wordpress/conf/www.conf`** — make php‑fpm listen on the network (so nginx can reach it on `wordpress:9000`):
```ini
[www]
user = www-data
group = www-data
listen = 9000              ; listen on all interfaces, port 9000 (NOT a unix socket)
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
clear_env = no             ; let env vars through to PHP
```

**`srcs/requirements/wordpress/tools/entrypoint.sh`:**
```sh
#!/bin/sh
set -e

DB_PASS="$(cat /run/secrets/db_password)"
WP_ADMIN_PASS="$(cat /run/secrets/wp_admin_password)"
WP_USER_PASS="$(cat /run/secrets/wp_user_password)"

cd /var/www/html

# Wait for MariaDB to accept connections.
until mariadb -h"${WORDPRESS_DB_HOST%%:*}" -u"${MYSQL_USER}" -p"${DB_PASS}" -e "SELECT 1;" >/dev/null 2>&1; do
    echo "Waiting for MariaDB..."
    sleep 2
done

# Install WordPress only once (volume empty).
if [ ! -f wp-config.php ]; then
    wp core download --allow-root
    wp config create --allow-root \
        --dbname="${MYSQL_DATABASE}" \
        --dbuser="${MYSQL_USER}" \
        --dbpass="${DB_PASS}" \
        --dbhost="${WORDPRESS_DB_HOST}"
    wp core install --allow-root \
        --url="${WP_URL}" \
        --title="${WP_TITLE}" \
        --admin_user="${WP_ADMIN_USER}" \
        --admin_password="${WP_ADMIN_PASS}" \
        --admin_email="${WP_ADMIN_EMAIL}" \
        --skip-email
    # Second, non-admin user (required: two users total).
    wp user create --allow-root \
        "${WP_USER}" "${WP_USER_EMAIL}" \
        --role=author --user_pass="${WP_USER_PASS}"
    chown -R www-data:www-data /var/www/html
fi

# Foreground php-fpm as PID 1. -F = don't daemonize.
exec php-fpm82 -F
```

> 📝 **Notes:**
> - `WORDPRESS_DB_HOST` will be `mariadb:3306`; `${WORDPRESS_DB_HOST%%:*}` strips the port for the wait‑loop.
> - The `sleep 2` here is a *bounded wait for a dependency to become ready*, then it exits the loop — this is **not** the forbidden `sleep infinity`/`while true` keep‑alive hack. The container's real PID 1 is `php-fpm -F`. (If your evaluator is strict, you can replace this with `healthcheck`/`depends_on: condition: service_healthy` and remove the loop — cleaner, see 3.3.)
> - Each credential is its **own** secret file: the DB user, the WP admin, and the WP second user all have distinct passwords (`db_password`, `wp_admin_password`, `wp_user_password`). Don't reuse one password for several roles.
> - The admin username (`WP_ADMIN_USER` in `.env`) **must not** contain `admin`/`administrator` — keep it as something like `bossman`.

#### ✅ Verify WordPress in isolation

WordPress `depends_on` MariaDB, so this command brings up **both** (MariaDB first, then WordPress) — no NGINX yet.

```bash
mkdir -p /home/dkolodze/data/db /home/dkolodze/data/wordpress
docker compose -f srcs/docker-compose.yml up -d --build wordpress

# 1. Both containers up; watch the one-time install (waits for DB, then installs WP)
docker compose -f srcs/docker-compose.yml ps
docker compose -f srcs/docker-compose.yml logs wordpress

# 2. php-fpm is PID 1 in the foreground (no tail -f/sleep hacks)
docker compose -f srcs/docker-compose.yml exec wordpress ps aux   # php-fpm master = PID 1

# 3. WordPress installed AND talking to MariaDB over the network
docker compose -f srcs/docker-compose.yml exec wordpress wp core is-installed --allow-root \
  && echo "WP installed & DB-connected ✅"

# 4. Exactly TWO users, admin name clean, roles correct (subject rule)
docker compose -f srcs/docker-compose.yml exec wordpress wp user list --allow-root

# 5. php-fpm is listening on 9000, and site files landed in the named volume
docker compose -f srcs/docker-compose.yml exec wordpress sh -c "netstat -ltn 2>/dev/null | grep 9000"
docker compose -f srcs/docker-compose.yml exec wordpress ls /var/www/html   # wp-config.php, wp-content, ...
ls /home/dkolodze/data/wordpress                                            # same files, on the host path
```

| Check | Proves |
|-------|--------|
| 1 | Image builds; entrypoint waits for the DB then installs WP once |
| 2 | `php-fpm -F` is PID 1 in the foreground (compliant) |
| 3 | DB connectivity over the Docker network *and* a completed WP install |
| 4 | The required **two users** exist; admin username has no `admin`/`administrator` |
| 5 | FastCGI is up on **9000**; the WP files persist in the `wp` named volume |

> ℹ️ **You can't browse the site yet** — that's expected. php-fpm speaks **FastCGI on 9000**, not HTTP, and there's no NGINX in front of it. The actual `https://dkolodze.42.fr` page test comes once NGINX exists (3.6 / 3.8).

<a name="stage-3-6"></a>
### 3.6 NGINX image (+ isolation test)

**`srcs/requirements/nginx/Dockerfile`:**
```dockerfile
FROM alpine:3.22

RUN apk add --no-cache nginx openssl \
    && mkdir -p /etc/nginx/ssl /var/www/html /run/nginx

# Self-signed TLS cert (the project doesn't require a real CA).
RUN openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/nginx/ssl/inception.key \
        -out /etc/nginx/ssl/inception.crt \
        -subj "/C=NL/ST=NH/L=Amsterdam/O=42/OU=Codam/CN=dkolodze.42.fr"

COPY conf/nginx.conf /etc/nginx/nginx.conf

EXPOSE 443
# nginx must run in the foreground (daemon off) to stay PID 1.
ENTRYPOINT ["nginx", "-g", "daemon off;"]
```
> Replace `CN=dkolodze.42.fr` with your domain.

**`srcs/requirements/nginx/conf/nginx.conf`:**
```nginx
events {}

http {
    include /etc/nginx/mime.types;

    server {
        listen 443 ssl;
        server_name dkolodze.42.fr;

        ssl_certificate     /etc/nginx/ssl/inception.crt;
        ssl_certificate_key /etc/nginx/ssl/inception.key;

        # SUBJECT RULE: TLSv1.2 or TLSv1.3 ONLY. No TLSv1/1.1, no SSLv3.
        ssl_protocols TLSv1.2 TLSv1.3;

        root /var/www/html;
        index index.php index.html;

        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        # Hand PHP requests to the wordpress container on the docker network.
        location ~ \.php$ {
            include /etc/nginx/fastcgi.conf;
            fastcgi_pass wordpress:9000;     # service-name DNS over the network
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
    }
}
```
Key compliance points: **only port 443**, **only TLSv1.2/1.3**, and PHP is proxied to `wordpress:9000` over the Docker network (no `--link`).

#### ✅ Verify NGINX (and the full stack) in isolation

NGINX `depends_on` WordPress, which `depends_on` MariaDB — so bringing up NGINX brings up the **whole stack**. This is your first true end‑to‑end test.

```bash
mkdir -p /home/dkolodze/data/db /home/dkolodze/data/wordpress
docker compose -f srcs/docker-compose.yml up -d --build       # all three

# 1. All three up; nginx master is PID 1
docker compose -f srcs/docker-compose.yml ps
docker compose -f srcs/docker-compose.yml exec nginx ps aux   # "nginx: master process" = PID 1

# 2. TLS protocol enforcement (the core NGINX rule) — use openssl for a reliable server-side test:
echo | openssl s_client -connect localhost:443 -tls1_2 >/dev/null 2>&1 && echo "TLS1.2 OK ✅"
echo | openssl s_client -connect localhost:443 -tls1_3 >/dev/null 2>&1 && echo "TLS1.3 OK ✅"
echo | openssl s_client -connect localhost:443 -tls1_1 >/dev/null 2>&1 && echo "TLS1.1 ACCEPTED ❌" || echo "TLS1.1 refused ✅"

# 3. End-to-end: nginx -> php-fpm -> mariadb returns the WordPress HTML
curl -k -H "Host: dkolodze.42.fr" https://localhost:443 | head -n 20

# 4. 443 is the ONLY published port (no other host port mappings)
docker compose -f srcs/docker-compose.yml ps   # only nginx shows a ...->443 mapping
```

| Check | Proves |
|-------|--------|
| 1 | NGINX builds and runs as PID 1 in the foreground (`daemon off;`) |
| 2 | Only **TLSv1.2/1.3** accepted; **TLSv1.1 refused** (subject rule) |
| 3 | Full chain works: TLS termination → FastCGI to `wordpress:9000` → DB |
| 4 | NGINX is the **only entrypoint**, on **443 only** |

> The `curl --tlsv1.1` client flag is unreliable for this — modern OpenSSL often disables TLS 1.1 *client‑side*, so a failure wouldn't prove the *server* rejected it. `openssl s_client -tls1_1` (above) or `nmap --script ssl-enum-ciphers -p 443 localhost` test the **server's** accepted protocols directly.

<a name="stage-3-7"></a>
### 3.7 Makefile

The subject requires a Makefile at the repo root that builds everything via `docker-compose.yml`.

**`Makefile`:**
```makefile
NAME    = inception
COMPOSE = srcs/docker-compose.yml
DATA    = /home/dkolodze/data

all: up

# Create host dirs that back the named volumes, then build + start.
up:
	@mkdir -p $(DATA)/db $(DATA)/wordpress
	docker compose -f $(COMPOSE) up -d --build

down:
	docker compose -f $(COMPOSE) down

stop:
	docker compose -f $(COMPOSE) stop

start:
	docker compose -f $(COMPOSE) start

# Rebuild from scratch.
re: down up

# Remove containers + network (keeps volumes).
clean:
	docker compose -f $(COMPOSE) down

# Full wipe: containers, volumes, and host data. DESTRUCTIVE.
fclean: down
	docker system prune -af
	@sudo rm -rf $(DATA)/db $(DATA)/wordpress

.PHONY: all up down stop start re clean fclean
```
> Adjust `clean`/`fclean` semantics to your taste, but keep `all`, `up`/`down`, `re`, `clean`, `fclean` — evaluators expect them.

### 3.8 Bring it up & verify

```bash
# In your VM, from the repo root:
make
docker compose -f srcs/docker-compose.yml ps     # all three "Up"
docker compose -f srcs/docker-compose.yml logs -f wordpress
```

Point the domain at localhost (on the VM):
```bash
echo "127.0.0.1 dkolodze.42.fr" | sudo tee -a /etc/hosts
```

Then in the VM browser open **https://dkolodze.42.fr** (accept the self‑signed cert warning). You should see your WordPress site. Log in at **https://dkolodze.42.fr/wp-admin** with `WP_ADMIN_USER`.

**Quick compliance probes:**
```bash
# TLS: 1.2/1.3 must work, older must FAIL.
curl -kI --tlsv1.2 https://dkolodze.42.fr        # OK
curl -kI --tlsv1.3 https://dkolodze.42.fr        # OK
curl -kI --tlsv1.1 https://dkolodze.42.fr        # must FAIL/refuse
nmap --script ssl-enum-ciphers -p 443 dkolodze.42.fr   # lists only TLS1.2/1.3

# Persistence: data survives a full restart.
docker compose -f srcs/docker-compose.yml down
make
# -> your posts/users are still there (volumes persisted in /home/dkolodze/data)

# Two WP users, admin name is clean:
docker compose -f srcs/docker-compose.yml exec wordpress wp user list --allow-root
```

---

<a name="stage-4"></a>
## Stage 4 — Rules checklist, docs & defense prep

### Final compliance checklist

- [ ] Runs in a **VM**.
- [ ] One **Makefile** at root builds everything via `docker compose`.
- [ ] All config under **`srcs/`**.
- [ ] **One Dockerfile per service**, built (not pulled). Images named `nginx`/`wordpress`/`mariadb`.
- [ ] Base = **penultimate stable Alpine** (verify the number the day you submit). **No `latest`.**
- [ ] 3 services, **each in its own container**, each a foreground daemon (no `tail -f`/`sleep inf`/`while true`/`bash` PID‑1 hacks).
- [ ] **NGINX only entrypoint**, **443 only**, **TLSv1.2/1.3 only**.
- [ ] WordPress + php‑fpm, **no nginx** in that container.
- [ ] MariaDB, **no nginx** in that container.
- [ ] **Two named volumes** (DB + WP files) physically in **`/home/dkolodze/data`**, **no bind mounts** at service level.
- [ ] **Custom network** declared; the `networks:` line is present; **no `host`/`--link`/`links`**.
- [ ] **`restart`** policy on every service.
- [ ] **No passwords in Dockerfiles**; **`.env`** used; **secrets** for credentials; `secrets/` and `.env` **git‑ignored**.
- [ ] WP DB has **2 users**, admin name has **no** `admin`/`administrator`.
- [ ] Domain **`dkolodze.42.fr` → local IP**.

### Required documentation files

The subject demands these `.md` files at repo root:

- **`README.md`** — first line italicized: `*This project has been created as part of the 42 curriculum by dkolodze.*` Then **Description**, **Instructions**, **Resources** (incl. how you used AI), and a **Project description** section that explains your use of Docker and compares: **VM vs Docker**, **Secrets vs Environment Variables**, **Docker Network vs Host Network**, **Docker Volumes vs Bind Mounts**. English only.
- **`USER_DOC.md`** — for an end user/admin: what services the stack provides, how to start/stop it, how to reach the site + admin panel, where credentials live, how to check services are healthy.
- **`DEV_DOC.md`** — for a developer: set up from scratch (prerequisites, config files, secrets), build/launch via Makefile + Compose, useful commands to manage containers/volumes, where data is stored and how it persists.

> Since `secrets/` and `.env` are git‑ignored, commit **example** files (`secrets/*.txt.example`, `srcs/.env.example`) and document in `DEV_DOC.md` how to create the real ones. That way a developer can reproduce the setup without you leaking credentials.

### Defense questions you should be able to answer

- **VM vs Docker:** VM virtualizes hardware + a full guest OS (heavy, slow boot, strong isolation); a container shares the host kernel and isolates only the process/filesystem (light, fast, less isolation). 
- **Why no `latest`:** it's a moving target — builds become non‑reproducible and can break silently. Pin a version.
- **Secrets vs env vars:** env vars leak via `docker inspect`/logs/`/proc`; secret files mount read‑only only into declaring containers and aren't in image history.
- **Docker network vs host network:** the bridge network isolates the stack and gives service‑name DNS; `host` network removes isolation and is forbidden here.
- **Volumes vs bind mounts:** named volumes are managed by Docker and portable; bind mounts couple you to a host path/layout. (Know why your `o: bind` named volume is still a *named volume*, per 3.3.)
- **PID 1 / daemons:** why each service runs in the foreground and why the keep‑alive hacks are wrong.
- **TLS:** show that only TLSv1.2/1.3 are accepted.

---

<a name="bonus"></a>
## Bonus (only attempt after the mandatory part is flawless)

Each bonus service = its own Dockerfile + container (+ volume if needed). Same rules apply.

- **redis cache** for WordPress (install `redis`, add the WP redis plugin + `WP_REDIS_HOST`).
- **FTP server** pointing at the WordPress volume (e.g. `vsftpd`/`pure-ftpd` built from Alpine).
- A **static website** in any language **except PHP** (e.g. a small Go/Node/static‑HTML site), its own container + port.
- **Adminer** (DB admin UI) — built from source, not pulled.
- A **service of your choice** you can justify at defense.

> ⚠️ Bonus is graded **only if the mandatory part is perfect**. Don't start it until everything in the checklist passes.

---

## How to use this tutorial with me

- Do **Stage 1 and Stage 2 in `~/docker-playground`** (sandbox) — don't commit them.
- Build **Stage 3 in `codam-inception/`** yourself, file by file.
- When you want me to **verify** what you wrote, just say so — I'll read the repo (read‑only) and check it against the checklist above. I won't write to the repo.

This document is configured for login **`dkolodze`** (domain `dkolodze.42.fr`, host path `/home/dkolodze/data`).
