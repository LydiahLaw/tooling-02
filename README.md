# tooling-02

Containerized deployment of the StegHub Tooling web application — a PHP app backed by MySQL — migrated from VM-based infrastructure to Docker containers.

## overview

This project demonstrates containerizing a PHP/MySQL application using Docker, Docker Compose, and a Jenkins CI/CD pipeline that builds, tests, and pushes images to Docker Hub automatically on every commit.

## architecture

```
browser → Apache (port 5000) → PHP app container → MySQL container
                                     ↕
                              tooling_app_network
```

Both containers run on a shared Docker bridge network (`tooling_app_network`), allowing the PHP app to reach MySQL by hostname rather than IP address.

## running with docker compose

```bash
# Start the full stack
docker compose -f tooling.yaml up -d

# Load the database schema (first run only)
docker compose -f tooling.yaml exec -T db mysql -uwebaccess -pYourPassword toolingdb < html/tooling_db_schema.sql

# Stop the stack
docker compose -f tooling.yaml down
```

Access the app at `http://localhost:5000`. Default login: `test@gmail.com` / `12345`.

## docker compose file — field documentation

```yaml
version: "3.9"
```
Specifies the Docker Compose file format version. From Compose v1.27.0 onwards this field is optional and can be omitted — newer versions will show a warning if included.

---

```yaml
services:
```
The top-level key that defines all containers in the stack. Each child key is a service name that becomes the container's hostname on the shared network.

---

```yaml
  tooling_frontend:
    build: .
```
Defines the PHP application service. `build: .` tells Compose to build the image from the `Dockerfile` in the current directory rather than pulling a pre-built image from a registry. Every time you run `docker compose up --build`, Docker rebuilds this image from scratch.

---

```yaml
    ports:
      - "5000:80"
```
Maps port 80 inside the container (where Apache listens) to port 5000 on your host machine. Format is `host_port:container_port`. Port 80 on the host is typically already in use, so 5000 is used as the external-facing port.

---

```yaml
    volumes:
      - tooling_frontend:/var/www/html
```
Mounts a named volume (`tooling_frontend`) to the app directory inside the container. Named volumes persist data between container restarts — if the container stops and restarts, the files are not lost. The volume is managed by Docker and stored at Docker's internal storage location.

---

```yaml
    environment:
      - MYSQL_IP=db
      - MYSQL_USER=webaccess
      - MYSQL_PASS=YourPassword
      - MYSQL_DBNAME=toolingdb
```
Passes environment variables into the container at runtime. The PHP app reads these via `$_ENV` to establish the database connection. Note that `MYSQL_IP=db` uses the service name `db` as the hostname — Docker's internal DNS automatically resolves service names to container IPs on the same network.

---

```yaml
    links:
      - db
```
Explicitly declares that `tooling_frontend` depends on and can communicate with the `db` service. While `depends_on` handles startup order, `links` creates a named alias for network communication. This is a legacy feature — on custom networks, containers can reach each other by service name without `links`.

---

```yaml
    depends_on:
      - db
```
Ensures the `db` container starts before `tooling_frontend`. Without this, the PHP app might try to connect to MySQL before it is ready, causing a connection error on startup.

---

```yaml
  db:
    image: mysql:5.7
```
Defines the MySQL database service. Unlike `tooling_frontend` which builds from a Dockerfile, this service uses the official `mysql:5.7` image pulled directly from Docker Hub. MySQL 5.7 is specified for compatibility with the Tooling app's SQL schema.

---

```yaml
    restart: always
```
Tells Docker to automatically restart this container if it crashes or if the Docker daemon restarts. Options are `no`, `always`, `on-failure`, and `unless-stopped`. `always` is appropriate for a database that must stay available.

---

```yaml
    environment:
      MYSQL_DATABASE: toolingdb
      MYSQL_USER: webaccess
      MYSQL_PASSWORD: YourPassword
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
```
Environment variables consumed by the MySQL Docker image on first startup:
- `MYSQL_DATABASE` — creates this database automatically on initialization
- `MYSQL_USER` / `MYSQL_PASSWORD` — creates a non-root user with access to the above database
- `MYSQL_RANDOM_ROOT_PASSWORD` — generates a random root password since direct root access is not needed. This is more secure than setting a known root password.

---

```yaml
    volumes:
      - db:/var/lib/mysql
```
Mounts a named volume to MySQL's data directory. This persists your database data across container restarts and recreations. Without this, all data would be lost every time the container stops.

---

```yaml
volumes:
  tooling_frontend:
  db:
```
Declares the named volumes used by the services above. Docker creates and manages these volumes. Declaring them here makes them available to any service in the Compose file. Both are empty declarations — no special configuration needed, Docker handles the rest.

## ci/cd pipeline

The Jenkins multibranch pipeline at `.jenkins/Jenkinsfile` runs on every push to any branch:

| stage | what it does |
|---|---|
| Initial Cleanup | wipes the Jenkins workspace for a clean build |
| Checkout | pulls the latest code from GitHub |
| Build Docker Image | builds `lydiahlaw/tooling:<branch>-0.0.1` |
| Test | starts the container, curls it from a sibling container, asserts HTTP 200/302 |
| Push to Docker Hub | authenticates and pushes the tagged image |
| Cleanup Images | removes the image from the Jenkins server |

Branch-based image tagging:
- `main` branch → `lydiahlaw/tooling:main-0.0.1`
- `feature/docker-pipeline` branch → `lydiahlaw/tooling:feature-docker-pipeline-0.0.1`

## related repositories

- **php-todo-docker** — [https://github.com/LydiahLaw/php-todo-docker](https://github.com/LydiahLaw/php-todo-docker) — Laravel todo app containerized with a similar pipeline
- **Project 20 documentation** — [https://github.com/LydiahLaw/Steghub-Devops-Cloud-Engineer](https://github.com/LydiahLaw/Steghub-Devops-Cloud-Engineer) — full project writeup
