# 13-Docker Compose Advanced Topics: depends_on, healthcheck, and Multi-Service Orchestration

#Docker #DockerCompose #ComposeAdvanced #depends_on #healthcheck #Multi-ServiceOrchestration #profiles #environmentVariables #ops #troubleshooting

---

## Recommended Reading Path

03-ContainerTechnology/13-Docker Compose Advanced Topics: depends_on, healthcheck, and Multi-Service Orchestration.md

---

## I. Document Overview

This article covers advanced orchestration capabilities of Docker Compose, focusing on the following topics:

- Structure of Compose files
- `services`
- `networks`
- `volumes`
- `depends_on`
- `depends_on.condition`
- `service_started`
- `service_healthy`
- `service_completed_successfully`
- `healthcheck`
- Service startup order
- Why `depends_on` does not guarantee application availability
- MySQL + Redis + Web multi-service example
- Accessing Compose networks and service names
- Persistence of Compose volumes
- `.env` files
- `environment`
- `env_file`
- `profiles`
- Managing multiple environments with Compose
- Advanced troubleshooting strategies for Compose
- Production considerations

The goal is to enable users to:

- Write multi-service Compose files
- Understand service dependencies
- Use healthchecks to monitor service health
- Avoid issues where containers start but applications are not ready
- Differentiate between development, debugging, and auxiliary components using profiles
- Troubleshoot abnormal startup of multi-services in Compose

---

## II. Advanced Understanding of Compose

Article 08 already covers common commands for Docker Compose.

Basic usage includes:

```bash
docker compose up -d
```

```bash
docker compose ps
```

```bash
docker compose logs -f
```

```bash
docker compose down
```

The focus of this article is not on these commands themselves, but rather on how to:

- Organize multiple services together
- Define service dependencies
- Determine whether a service is truly ready for use
- Ensure data persistence
- Enable network communication between services
- Manage environment variables effectively
- Decide which services to enable in different scenarios

Docker Compose can be considered as:

```text
A tool for orchestrating multiple containers on a single machine
```

It is suitable for:

```text
Local development environments
Testing environments
Small-scale single-machine service combinations
Rapid deployment of middleware
CI testing environments
Ops experimentation environments
```

For complex production environments, Kubernetes is usually a better choice.

---

## III. Basic Structure of Compose Files

A typical Compose file includes the following sections:

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "8080:80"
networks:
  app-net:

volumes:
  app-data:
```

Core components include:

```text
services
→ Defines service containers
networks
→ Defines networks
volumes
→ Defines data volumes
```

Common file names include:

```text
compose.yaml
compose.yml
docker-compose.yml
```

Currently, `compose.yaml` is the recommended format, but `docker-compose.yml` is still commonly used in production and older projects.

---

## IV. services: Service Definition

---

## Scenario 1: Minimum Service Definition

```yaml
services:
  nginx:
    image: nginx:1.27
    ports:
      - "8080:80"
```

To start the service:

```bash
docker compose up -d
```

To view services:

```bash
docker compose ps
```

To view logs:

```bash
docker compose logs -f nginx
```

To stop and delete the service:

```bash
docker compose down
```

---

## Scenario 2: Common Fields in Services

Common fields include:

```yaml
services:
  web:
    image: nginx:1.27
    container_name: demo-nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      APP_ENV: prod
      TZ: Asia/Shanghai
    networks:
      - app-net
    restart: unless-stopped
```

Field explanations:

```text
image
→ Specifies the image to use
container_name
→ Sets a unique name for the container
ports
→ Defines port mappings
volumes
→ Specifies directories or data volumes to mount
environment
→ Sets environment variables for the container
networks
→ Adds the service to a specified network
restart
→ Defines how the container should be restarted
```

---

## Scenario 3: Difference between `image` and `build`

Using an existing image:

```yaml
services:
  web:
    image: nginx:1.27
```

Building## Scenario 9: The Difference Between `down` and `down -v`

To stop and delete a container and its associated network:

```bash
docker compose down
```

To stop, delete a container, its associated network, and volume:

```bash
docker compose down -v
```

Note:

```text
Running `docker compose down -v` will delete the volumes created by Compose.
If these volumes contain database data, it may result in data loss.
Do not execute this command casually in a production environment:
```bash
docker compose down -v
```
``````markdown
MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
MYSQL_DATABASE: ${MYSQL DATABASE}
MYSQL_USER: ${MYSQL_USER}
MYSQL_PASSWORD: ${MYSQL_PASSWORD}
TZ: Asia/Shanghai
volumes:
  - mysql-data:/var/lib/mysql
networks:
  - app-net
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-p${MYSQL_ROOT_PASSWORD}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s
restart: unless-stopped

redis:
  image: redis:7
  command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]
volumes:
  - redis-data:/data
networks:
  - app-net
healthcheck:
  test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
  interval: 10s
  timeout: 3s
  retries: 5
  start_period: 10s
restart: unless-stopped

networks:
  app-net:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

---

## Scenario 24: Starting Multiple Services

Check the configuration:

```bash
docker compose config
```

Start the services:

```bash
docker compose up -d
```

View the status:

```bash
docker compose ps
```

View all logs:

```bash
docker compose logs -f
```

View MySQL logs:

```bash
docker compose logs -f mysql
```

View Redis logs:

```bash
docker compose logs -f redis
```

View Web logs:

```bash
docker compose logs -f web
```

---

## Scenario 25: Verifying Service Name Resolution

Enter the web container:

```bash
docker compose exec web /bin/sh
```

Resolve the MySQL service name:

```bash
getent hosts mysql
```

Resolve the Redis service name:

```bash
getent hosts redis
```

Explanation:

```text
Services within the same network in Compose can access each other by their service names.
```

---

## Chapter 10: Advanced Environment Variables

---

## Scenario 26: Using environment variables

```yaml
services:
  web:
    image: nginx:1.27
    environment:
      APP_ENV: prod
      TZ: Asia/Shanghai
```

It can also be written in a list format:

```yaml
services:
  web:
    image: nginx:1.27
    environment:
      - APP_ENV=prod
      - TZ=Asia/Shanghai
```

Recommendation:

```text
Simple variables can be directly specified in the environment section.
```

---

## Scenario 27: Using an env_file

```yaml
services:
  web:
    image: nginx:1.27
    env_file:
      - ./web.env
```

Example for `web.env`:

```text
APP_ENV=prod
TZ=Asia/Shanghai
API_BASE_URL=http://api:8080
```

This is suitable for:

```text
When there are many variables
When variables need to be managed separately for different services
To prevent the compose.yaml file from becoming too long
```

---

## Scenario 28: Interpolation in .env files

Example for `.env`:

```text
WEB_PORT=8080
NGINX_VERSION=1.27
```

Example for `compose.yaml`:

```yaml
services:
  web:
    image: nginx:${NGINX_VERSION}
    ports:
      - "${WEB_PORT}:80"
```

Check the final configuration:

```bash
docker compose config
```

Explanation:

```text
.env files are often used to replace variables in Compose files.
ENV_files are commonly used to inject environment variables into containers.
```

---

## Scenario 29: Common Confusions About Environment Variables

Things that are easily confused:

```text
.env
→ Used for variable interpolation in Compose files
env_file
→ Used to inject environment variables into containers
environment
→ Used to directly set environment variables for containers within Compose files
```

It is recommended to understand:

```text
Compose reads .env files itself.
Containers read from the environment or env_file during runtime.
```

Check the final configuration:

```bash
docker compose config
```

Enter the container to view variables:

```bash
docker compose exec web env
```

---

## Chapter 11: Profiles: Enabling Services on Demand

---

## Scenario 30: What are profiles?

Profiles are used to selectively enable certain services based on the environment or purpose.

Suitable for:

```text
Debugging tools
Monitoring components
Management backends
One-time tasks
Local development assistance services
```

Services that```markdown
restart: unless stopped
```

Explanation:

```text
Base files define general services.
Environment files override differences such as ports, variables, mounts, and restart policies.
```

---

## Thirteen Common Troubleshooting Approaches

---

## Issue 1: Service does not start as expected

Check status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f
```

Inspect configuration:

```bash
docker compose config
``

Common causes:

```text
YAML indentation errors
Incorrect service name
Failed image pull
Port conflicts
Missing environment variables
Profile not enabled
depends_on dependencies not met
Healthcheck failures persisting
```

---

## Issue 2: depends_on is configured, but the service still fails to connect

Common reasons:

```text
depends_on primarily controls startup order by default.
The dependency container starting does not mean the application is ready.
Database initialization takes a long time.
The web application starts too quickly, failing to connect to the database.
```

Suggestions:

```text
Configure a healthcheck for the database.
For web applications, use depends_on(condition: service_healthy).
The application itself should also support retry connections.
```

Example:

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

---

## Issue 3: Healthchecks consistently show "unhealthy"

Check status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs -f service_name
```

Inspect container details:

```bash
docker inspect container_id
```

Common causes:

```text
The healthcheck command is not present.
Incorrect healthcheck port configuration.
Service takes too long to start.
Too short start_period setting.
Incorrect authentication parameters.
localhost reference is incorrect.
The application's health API returns a non-2xx status code.
```

For example, if the image does not contain curl:

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost/ || exit 1"]
```

This will result in failure.

---

## Issue 4: Environment variables are not taking effect

Check final configuration:

```bash
docker compose config
```

Enter the container:

```bash
docker compose exec web env
```

Inspect `.env` file:

```bash
cat .env
```

Check `env_file`:

```bash
cat web.env
```

Common causes:

```text
Confusing between `.env` and `env_file`.
Incorrect variable names.
Variables are not being referenced.
Container is not recreated after changing variables.
Shell environment variables override the `.env` file.
``

To resolve, recreate the container:

```bash
docker compose up -d --force-recreate
```

---

## Issue 5: Profiles for services are not starting

Check the Compose file:

```bash
cat compose.yaml
```

 Verify if there are profiles defined:

```yaml
profiles:
  - debug
```

By default, starting with:

```bash
docker compose up -d
```

will not start profile services.

To enable a specific profile:

```bash
docker compose --profile debug up -d
```

Or:

```bash
export COMPOSE_PROFILES=debug
```

```bash
docker compose up -d
```

---

## Issue 6: Services cannot access each other

Check the network configuration:

```bash
docker network ls
```

View network details:

```bash
docker network inspect network_name
```

Check service status:

```bash
docker compose ps
```

Enter the container to test service name resolution:

```bash
docker compose exec web /bin/sh
```

```bash
getent hosts mysql
```

Common causes:

```text
Services are not in the same network.
Incorrect service names.
Target service is not running.
Wrong listening address for the target service.
The application is connecting to 127.0.0.1, which refers to the container itself.
```

Note:

```text
127.0.0.1 inside a container refers to the container itself,
not another service container.
```

To access MySQL, use:

```text
mysql:3306
```

To access Redis, use:

```text
redis:6379
```

---

## Issue 7: Port conflicts

Check host machine listening ports:

```bash
ss -tunlp
```

View Compose status:

```bash
docker compose ps
```

Example:

```yaml
ports:
  - "8080:80"
```

If port 8080 is already in use on the host, change it to:

```yaml
ports:
  - "8081:80"
```

Restart the service:

```bash
docker compose up -d
```

---

## Issue 8: Volume data is accidentally deleted

High-risk command:

```## Volume Troubleshooting

To view volumes:

```bash
docker volume ls
```

To check volume details:

```bash
docker volume inspect volume_name
```

---

## Healthcheck Troubleshooting

To check the status:

```bash
docker compose ps
```

To view container details:

```bash
docker inspect container_id
```

To view service logs:

```bash
docker compose logs -f service_name
```

---

## Profiles

To enable the debug profile:

```bash
docker compose --profile debug up -d
```

To set environment variables to enable the profile:

```bash
export COMPOSE_PROFILES=debug
```

To start the application:

```bash
docker compose up -d
```

To disable the profile:

```bash
unset COMPOSE_PROFILES
```

---

## Multi-Environment Compose

For development:

```bash
docker compose \
  -f compose.yaml \
  -f compose.dev.yaml \
  up -d
```

For production:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d
```

To view the final merged configuration:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  config
```

---

## Sixteen, One-Sentence Summary

The core of advanced Docker Compose usage lies not in creating additional services but in ensuring:

- Services have dependencies.
- Dependent services are healthy.
- Services communicate using service names.
- Data is persisted using volumes.
- Environment variables are managed clearly.
- Debugging services is enabled as needed using profiles.

Understanding `depends_on`:

- It controls the startup order by default but does not mean the dependent service is fully available.
- A safer approach combines `depends_on`, healthchecks, and the service’s own retry mechanism.

Understanding healthchecks:

- A running container simply means its process is active.
- A healthy container has passed all health checks, indicating it is functional.

Understanding environment variables:

- `.env` files allow variable interpolation in Compose configurations.
- `env_file` files can be used to inject specific environment variables into a container.
- Environment variables can also be set directly within the Compose file.

Understanding profiles:

- Default services start without configuring profiles.
- Optional services can be started on demand by configuring their profiles.

Production recommendations:

- Always execute `docker compose config` after changing the Compose file.
- Configure healthchecks for critical dependencies like databases and Redis.
- Ensure business services support retries in addition to depends_on.
- Never share real production passwords in Git repos.
- Avoid executing `docker compose down -v` unless necessary.
- Use service names for inter-service communication instead of hard-coding container IPs.
- For complex production scenarios, Kubernetes is still the preferred choice over Compose.