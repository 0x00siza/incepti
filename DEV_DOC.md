# DEV_DOC – Inception Developer Guide (WordPress, NGINX, MariaDB)

This guide explains how to set up the environment from scratch, build and launch the stack, manage containers and volumes, and understand where data is stored and how it persists.

## 1) Prerequisites
- Linux with `make` available
- Docker Engine and Docker Compose plugin installed and running
- `sudo` access (used by the Makefile to append an entry to `/etc/hosts`)

Verify installation:
```bash
docker --version
docker compose version    # or: docker-compose --version
make --version
```

## 2) Setup From Scratch
From the project root.

### 2.1 Clone and inspect
```bash
# if needed
git clone <repo-url>
cd ciw
ls -la
```
Key files you will use:
- `srcs/docker-compose.yml` (services, volumes, secrets)
- `srcs/.env` (environment variables consumed by services)
- `Makefile` (build/run/clean tasks)
- `./secrets/` (created by Make; stores generated passwords)

### 2.2 Configure environment variables
Open `srcs/.env` and review:
- Domain/URL: `DOMAIN_NAME`, `URL`
- Database: `DB_NAME`, `DB_USER`, `DB_HOST`, `DB_PASS_FILE`
- WordPress: `TITTLE`, `ADMIN_USER`, `ADMIN_PASS_FILE`, `ADMIN_EMAIL`
- Secondary user: `USER_TWO`, `USER_TWO_EMAIL`, `USER_PASS_FILE`

Defaults target the domain `ner-roui.42.fr` and use Docker secrets for passwords.

### 2.3 Bind‑mount paths (host persistence)
`srcs/docker-compose.yml` binds host paths:
- WordPress data → `/home/ner-roui/data/wordpress_d_volume`
- MariaDB data → `/home/ner-roui/data/mariadb_d_volume`

If you are on a different user or path, update the `device:` entries accordingly.

### 2.4 (Optional) Change the domain and TLS CN
- If you change the domain, also update `/etc/hosts` (the Makefile adds `127.0.0.1 ner-roui.42.fr` automatically; adjust if you use another domain).
- Update the CN in the NGINX image (certificate generated in `srcs/requirements/nginx/Dockerfile`).

### 2.5 Secrets generation
Passwords are generated into `./secrets` when you run `make run` or explicitly:
```bash
make generate_passwords
ls -la secrets/
```
Generated files:
- `secrets/db_password.txt`
- `secrets/admin_password.txt`
- `secrets/user_password.txt`

These are mounted into containers under `/run/secrets`.

## 3) Build and Launch
Using the Makefile (recommended):
```bash
make build   # build images
make run     # generate secrets (if needed), create host dirs, update /etc/hosts, up -d --build
```
Direct Docker Compose equivalent:
```bash
docker compose -f srcs/docker-compose.yml build
mkdir -p /home/ner-roui/data/wordpress_d_volume /home/ner-roui/data/mariadb_d_volume
# add your domain to /etc/hosts if needed
sudo sh -c 'echo "127.0.0.1 ner-roui.42.fr" >> /etc/hosts'
docker compose -f srcs/docker-compose.yml up -d --build
```
Access the site:
- Website: https://ner-roui.42.fr
- Admin:   https://ner-roui.42.fr/wp-admin

Self‑signed certificate: your browser will warn; accept for local development.

## 4) Manage Containers
Common commands (Makefile targets shown first, then Docker Compose equivalents):
```bash
make stop
# docker compose -f srcs/docker-compose.yml down

make clean
# docker compose -f srcs/docker-compose.yml down --rmi all -v --remove-orphans

make fclean   # also deletes host data and secrets (CAUTION)
# sudo rm -rf /home/ner-roui/data/wordpress_d_volume/* /home/ner-roui/data/mariadb_d_volume/*
# sudo docker system prune -af
# rm -rf ./secrets

make re       # stop, fclean, build, run
```
Day-to-day docker compose operations:
```bash
# status and logs
docker compose -f srcs/docker-compose.yml ps
docker compose -f srcs/docker-compose.yml logs -f nginx

# shell access
docker compose -f srcs/docker-compose.yml exec nginx bash
docker compose -f srcs/docker-compose.yml exec wordpress bash

# rebuild one service and restart it
docker compose -f srcs/docker-compose.yml build nginx
docker compose -f srcs/docker-compose.yml up -d nginx

# restart services
docker compose -f srcs/docker-compose.yml restart wordpress
```

## 5) Manage Volumes and Data Persistence
Host persistence (bind‑mounts configured in `srcs/docker-compose.yml`):
- WordPress files: `/home/ner-roui/data/wordpress_d_volume`
- MariaDB data:   `/home/ner-roui/data/mariadb_d_volume`

Secrets on host (created by Make):
- `./secrets/db_password.txt`
- `./secrets/admin_password.txt`
- `./secrets/user_password.txt`

Wiping data (CAUTION – destructive):
```bash
make fclean
# or manual:
sudo rm -rf /home/ner-roui/data/wordpress_d_volume/*
sudo rm -rf /home/ner-roui/data/mariadb_d_volume/*
rm -rf ./secrets
```
Data persists across container rebuilds/restarts because it lives in the host paths above. Rebuilding images or recreating containers does not remove these bind‑mounts unless you explicitly clean them (e.g., `make fclean`).

## 6) Developer Tips
- Change `.env` then recreate affected services:
  ```bash
  docker compose -f srcs/docker-compose.yml up -d --build
  ```
- Use WP‑CLI for admin tasks:
  ```bash
  docker compose -f srcs/docker-compose.yml exec wordpress wp plugin list --allow-root
  docker compose -f srcs/docker-compose.yml exec wordpress wp user list --allow-root
  ```
- Quick DB check:
  ```bash
  docker compose -f srcs/docker-compose.yml exec mariadb mysql -e "SHOW DATABASES;"
  ```
- Adjust NGINX PHP routing in `srcs/requirements/nginx/conf/nginx.conf` if needed.
- PHP-FPM listen address is set to `0.0.0.0:9000` in the WordPress image for container networking.

## 7) Troubleshooting (Dev)
- Port 443 already in use → stop conflicting service or change port mapping in compose.
- Certificate warnings → expected (self‑signed). For real domains, replace with a trusted cert.
- WordPress not configured yet → wait a few seconds; check `wordpress` logs.
- Full reset → `make fclean && make run` (destroys local data and secrets).
