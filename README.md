*This project has been created as part of the 42 curriculum by ner-roui.*

# Inception

## Description

**Inception** is a system-administration project that deploys a fully Dockerized WordPress website running behind an NGINX reverse proxy with HTTPS (TLSv1.2/1.3) and backed by a MariaDB database. All services run in isolated containers orchestrated via Docker Compose, demonstrating infrastructure-as-code best practices.

### Goal

Set up a small infrastructure composed of different services under specific rules:
- Each service runs in a dedicated container built from a custom Dockerfile (Debian Bookworm base).
- NGINX is the only entry point and exposes port **443** (HTTPS with self-signed certificate).
- WordPress uses **PHP-FPM** (no NGINX inside the container) and communicates with NGINX via FastCGI.
- MariaDB stores site data; credentials are managed through **Docker secrets**.
- Data persists on the host via **bind mounts**.

### Brief Overview

```
[ Client ] ⇄ https:443 ⇄ [ NGINX ] ⇄ fastcgi:9000 ⇄ [ WordPress/PHP-FPM ] ⇄ [ MariaDB:3306 ]
```

| Service   | Base Image        | Exposed Port | Role                          |
|-----------|-------------------|--------------|-------------------------------|
| nginx     | debian:bookworm   | 443 (host)   | TLS termination, reverse proxy|
| wordpress | debian:bookworm   | 9000 (internal) | PHP-FPM app server          |
| mariadb   | debian:bookworm   | 3306 (internal) | Database                    |

---

## Instructions

### Prerequisites

- Linux (tested on Debian/Ubuntu)
- `make`
- Docker Engine + Docker Compose plugin
- `sudo` access (for `/etc/hosts` and volume cleanup)

### Installation & Execution

```bash
# 1. Clone the repository
git clone <repo-url> inception && cd inception

# 2. Build and run (secrets generated automatically)
make
```

This will:
1. Generate random passwords in `./secrets/`.
2. Create host directories for persistent data.
3. Add `127.0.0.1 ner-roui.42.fr` to `/etc/hosts` if not present.
4. Build all images and start containers.

Access the site at **https://ner-roui.42.fr** (accept the self-signed certificate warning).

### Useful Commands

| Command              | Description                                         |
|----------------------|-----------------------------------------------------|
| `make build`         | Build images only                                   |
| `make run`           | Generate secrets, prepare volumes, build & start    |
| `make stop`          | Stop and remove containers                          |
| `make clean`         | Stop + remove images, volumes, orphans              |
| `make fclean`        | Full clean including host data and secrets          |
| `make re`            | Rebuild from scratch                                |
| `make generate_passwords` | Regenerate secrets (if missing)               |

---

## Project Description

### Use of Docker

Each service is containerized to ensure:
- **Isolation**: services cannot interfere with each other or the host.
- **Reproducibility**: images are built from Dockerfiles with pinned base (`debian:bookworm`).
- **Portability**: the stack runs identically on any Docker-capable host.

### Sources Included

| Path                                      | Purpose                                      |
|-------------------------------------------|----------------------------------------------|
| `srcs/docker-compose.yml`                 | Service definitions, networks, volumes, secrets |
| `srcs/.env`                               | Environment variables (non-sensitive)        |
| `srcs/requirements/nginx/`                | NGINX Dockerfile + TLS config                |
| `srcs/requirements/wordpress/`            | WordPress/PHP-FPM Dockerfile + setup script  |
| `srcs/requirements/mariadb/`              | MariaDB Dockerfile + init script             |
| `./secrets/`                              | Auto-generated password files (gitignored)   |

### Design Choices

1. **Custom images** instead of official ones — gives full control and satisfies project rules.
2. **Secrets for passwords** — avoids leaking credentials in environment or logs.
3. **Bind mounts** for persistence — data survives container recreation and is easy to back up.
4. **Bridge network** — containers communicate by service name; only NGINX exposes a port.

---

## Technical Comparisons

### Virtual Machines vs Docker

| Aspect            | Virtual Machine                         | Docker Container                        |
|-------------------|----------------------------------------|----------------------------------------|
| Isolation level   | Full (hypervisor + guest OS)           | Process-level (shared kernel)          |
| Startup time      | Minutes                                | Seconds                                |
| Resource usage    | High (each VM has its own OS)          | Low (shares host kernel)               |
| Portability       | Requires compatible hypervisor         | Runs anywhere Docker is installed      |
| Use case          | Strong isolation, different OS         | Microservices, CI/CD, dev environments |

**Choice**: Docker — lightweight, fast, sufficient isolation for this web stack.

### Secrets vs Environment Variables

| Aspect            | Environment Variables                  | Docker Secrets                          |
|-------------------|----------------------------------------|----------------------------------------|
| Visibility        | Visible in `docker inspect`, logs      | Mounted as files; not in inspect output|
| Security          | Easily leaked                          | More secure (tmpfs, restricted perms)  |
| Rotation          | Requires container restart             | Can update secret and recreate service |
| Complexity        | Simple                                 | Slightly more setup                    |

**Choice**: Secrets for passwords (`db_password`, `admin_password`, `user_password`); environment variables for non-sensitive config (`DB_NAME`, `URL`).

### Docker Network vs Host Network

| Aspect            | Bridge Network (default)               | Host Network                            |
|-------------------|----------------------------------------|----------------------------------------|
| Isolation         | Containers have private IPs            | Containers share host's network stack  |
| Port mapping      | Explicit (`-p 443:443`)                | No mapping; container binds directly   |
| Security          | Better (services hidden behind proxy)  | Weaker (all ports exposed)             |
| Use case          | Multi-container apps, controlled exposure | Performance-critical, single service |

**Choice**: Bridge network `inception` — isolates internal traffic; only NGINX publishes port 443.

### Docker Volumes vs Bind Mounts

| Aspect            | Named Volume                           | Bind Mount                              |
|-------------------|----------------------------------------|----------------------------------------|
| Management        | Managed by Docker                      | Managed by user (host path)            |
| Portability       | Easier to back up via Docker CLI       | Requires knowing host path             |
| Performance       | Optimized on some platforms            | Native filesystem speed                |
| Flexibility       | Less (opaque location)                 | More (direct host access)              |

**Choice**: Bind mounts (`/home/ner-roui/data/...`) — easy inspection, backup, and satisfies project requirement for host persistence.

---

## Resources

### Official Documentation

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [NGINX Documentation](https://nginx.org/en/docs/)
- [WordPress Developer Resources](https://developer.wordpress.org/)
- [MariaDB Knowledge Base](https://mariadb.com/kb/en/)
- [WP-CLI Handbook](https://make.wordpress.org/cli/handbook/)

### Articles & Tutorials

- [Docker Secrets Management](https://docs.docker.com/engine/swarm/secrets/)
- [Self-Signed Certificates with OpenSSL](https://www.openssl.org/docs/man1.1.1/man1/req.html)
- [PHP-FPM + NGINX Setup](https://www.php.net/manual/en/install.fpm.php)


### how AI was used
AI helped me understand new concepts in the Inception project by breaking down complex topics such as Docker, containers, networking, and databases into simple, step-by-step explanations. It guided me through choosing correct configurations instead of blindly copying commands. AI also helped me connect theory with practice by explaining why certain tools and commands are used, which improved my problem-solving skills and overall understanding of the project.
---

## Additional Documentation

| File           | Purpose                                           |
|----------------|---------------------------------------------------|
| [USER_DOC.md](USER_DOC.md)   | End-user guide: access, credentials, health checks |
| [DEV_DOC.md](DEV_DOC.md)     | Developer guide: setup, build, manage, persist   |

---


