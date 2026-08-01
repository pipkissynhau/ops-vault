# 13-Docker Compose Advanced: depends_on, healthcheck, and Multi-Service Orchestration

#Docker #DockerCompose #ComposeStep #depends_on #healthcheck #Multi-serviceOrganization #profiles #EnvironmentalVariables #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/13-Docker Compose Advanced: depends_on, healthcheck, and Multi-Service Orchestration.md

---

## I. Document Description

This document organizes advanced orchestration capabilities of Docker Compose, focusing on:

- Compose File Structure
- `services`
- `networks`
- `volumes`
- `depends_on`
- `depends_on.condition`
- `service_started`
- `service_healthy`
- `service_completed_successfully`
- `healthcheck`
- Service Startup Order
- Why `depends_on` does not equal the application being definitely available
- MySQL + Redis + Web Multi-Service Example
- Compose Networking and Service Name Access
- Compose Volume Persistence
- `.env` File
- `environment`
- `env_file`
- `profiles`
- Multi-Environment Compose Management
- Compose Advanced Troubleshooting Approach
- Production Considerations

The goal is:

Can write multi-service Compose files

→ Can understand service dependencies

→ Can use healthcheck to determine service health status

→ Can avoid "container started but application not ready" issues

→ Can use profiles to distinguish development, debugging, and auxiliary components

→ Can troubleshoot Compose multi-service startup anomalies

---

## II. Advanced Understanding of Compose

The 08th article already organized common Docker Compose commands.

Basic usage is:

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

This article focuses not on the commands themselves, but on:

```text
How multiple services are organized
How services depend on each other
Is the service really ready? Okay.
How the data lasts
How the network works.
How environmental variables are managed
How to enable different services in different settings
```

Docker Compose can be understood as:

```text
Single-carrier multi-container service organization tool
```

It is suitable for:

```text
Local development environment
Test Environment
Single small service combination
Middle Quick Pull
CI Test Environment
Transport Experiment Environment
```

Complex production environments are typically better suited for Kubernetes.

---

## III. Compose File Basic Structure

A typical Compose file includes:

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

Core structure:

```text
services
→ Define service containers

networks
→ Define Network

volumes
→ Define Data Volume
```

Common file names:

```text
compose.yaml
compose.yml
docker-compose.yml
```

Now it's more recommended to use:

```text
compose.yaml
```

But in actual production and legacy projects, it's still commonly seen:

```text
docker-compose.yml
```

---

## IV. services: Service Definition

---

## Scenario 1: Minimal Service Definition

```yaml
services:
  nginx:
    image: nginx:1.27
    ports:
      - "8080:80"
```

Start:

```bash
docker compose up -d
```

Check service:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f nginx
```

Stop and remove:

```bash
docker compose down
```

---

## Scenario 2: Common Service Fields

Common fields:

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

networks:
  app-net:
```

Field understanding:

```text
image
→ Which mirror?

container_name
→ Specify container name

ports
→ Port Map

volumes
→ Mount directory or data volume

environment
→ Set container environment variable

networks
→ Add specified network

restart
→ Container Restart Policy
```

---

## Scenario 3: image vs build Difference

Use existing image:

```yaml
services:
  web:
    image: nginx:1.27
```

Build from Dockerfile:

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    image: my-web:v1
```

Understanding:

```text
image
→ Directly use existing mirrors

build
→ Based on Dockerfile Build mirrors
```

Build and start:

```bash
docker compose up -d --build
```

Only build:

```bash
docker compose build
```

---

## V. networks: Compose Service Networking

---

## Scenario 4: Default Network

Even without explicitly defining a network, Compose will create a default network for the project.

For example:

```yaml
services:
  web:
    image: nginx:1.27

  redis:
    image: redis:7
```

After startup, `web` and `redis` will be in the same Compose default network.

Services can access each other via service names:

```text
redis
```

Instead of container IP.

---

## Scenario 5: Service Name Access

Example:

```yaml
services:
  web:
    image: nginx:1.27
    networks:
      - app-net

  redis:
    image: redis:7
    networks:
      - app-net

networks:
  app-net:
```

To access Redis in the `web` container, you can use:

```text
redis:6379
```

Instead of hardcoding the Redis container IP.

Enter web container:

```bash
docker compose exec web /bin/sh
```

Test DNS resolution:

```bash
getent hosts redis
```

Or:

```bash
ping redis
```

Note:

```text
Is there a container inside? ping The order depends on the mirror itself.
```

---

## Scenario 6: Custom Network

```yaml
services:
  web:
    image: nginx:1.27
    networks:
      - app-net

  redis:
    image: redis:7
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
```

Check network:

```bash
docker network ls
```

Check network details:

```bash
docker network inspect Project name_app-net
```

Explanation:

```text
Compose Created network name usually with project name prefix
```

For example, if the directory name is `demo`, the network might be called:

```text
demo_app-net
```

---

## VI. volumes: Data Persistence

---

## Scenario 7: Using Named Volume

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: appdb
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

Start:

```bash
docker compose up -d
```

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect Project name_mysql-data
```

Explanation:

```text
mysql-data
→ Compose Data volume managed

/var/lib/mysql
→ MySQL Directory of data in containers
```

Volumes are retained by default even after container deletion.

---

## Scenario 8: Using Bind Mount

```yaml
services:
  nginx:
    image: nginx:1.27
    volumes:
      - ./html:/usr/share/nginx/html
    ports:
      - "8080:80"
```

Explanation:

```text
./html
→ Host 's current directory html

/usr/share/nginx/html
→ Inside the container Nginx Static Page Directory
```

Suitable for:

```text
Local development
Configure Mount
Debug static files
Quick Verify
```

Production environments should pay attention to host directory permissions.

---

## Scenario 9: Difference between down and down -v

Stop and remove containers, networks:

```bash
docker compose down
```

Stop and remove containers, networks, volumes:

```bash
docker compose down -v
```

Note:

```text
docker compose down -v Will be deleted Compose Created volume
```

If the volume contains database data, this may lead to data loss.

Do not casually execute this in production environments:

```bash
docker compose down -v
```

---

## VII. depends_on: Service Dependencies

---

## Scenario 10: Basic depends_on

```yaml
services:
  web:
    image: nginx:1.27
    depends_on:
      - mysql
      - redis

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123

  redis:
    image: redis:7
```

Meaning:

```text
Start web First, start. mysql and redis
Stop when you're done. webStop. mysql and redis
```

---

## Scenario 11: Common Misunderstanding of depends_on

Many people mistakenly believe:

```text
depends_on
→ Start the current service once the service is fully available
```

This understanding is incomplete.

More accurately:

```text
depends_on Default Primary Control Start Order
It doesn't necessarily mean you're ready to process the request relying on the application.
```

For example:

```text
MySQL The container is activated.
→ But... MySQL The service may still be in initial stages.

Redis The container is activated.
→ But... Redis Maybe not yet. ready

Web The container is activated.
→ But the connection to the database could fail.
```

So a more reliable approach is:

```text
depends_on + healthcheck
```

---

## Scenario 12: depends_on condition

Compose supports controlling dependencies through conditions.

Common conditions:

```text
service_started
service_healthy
service_completed_successfully
```

Meaning:

```text
service_started
→ Reliance on service started

service_healthy
→ Service-dependent health check passed

service_completed_successfully
→ Reliance on successful completion of mission to start
```

---

## Scenario 13: depends_on + service_healthy

```yaml
services:
  web:
    image: nginx:1.27
    depends_on:
      mysql:
        condition: service_healthy

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Meaning:

```text
web Wait. mysql Yes. healthcheck ♪ Turn into ♪ healthy Start later.
```

Check status:

```bash
docker compose ps
```

Check MySQL logs:

```bash
docker compose logs -f mysql
```

Check container health check details:

```bash
docker inspect ContainersID
```

---

## VIII. healthcheck: Health Check

---

## Scenario 14: healthcheck Basic Structure

```yaml
services:
  web:
    image: nginx:1.27
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 10s
```

Field understanding: /think

```text
test
→ Health check orders

interval
→ Check interval

timeout
→ Single check timeout

retries
→ Number of consecutive failures marked as unhealthy

start_period
→ Start protection period. Service is allowed to start for an initial period.
```

---

## Scenario 15: CMD Format healthcheck

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 10s
  timeout: 3s
  retries: 5
```

Suitable for:

```text
Clear command parameters
No need. shell Feature
```

---

## Scenario 16: CMD-SHELL Format healthcheck

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
  interval: 10s
  timeout: 3s
  retries: 5
```

Suitable for:

```text
Yes. shell Pipes
Variable required
Yes. || && Wait. shell Syntax:
```

---

## Scenario 17: Redis healthcheck

```yaml
services:
  redis:
    image: redis:7
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
```

When Redis is normal:

```text
PONG
```

---

## Scenario 18: MySQL healthcheck

```yaml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root123
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot123"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

Description:

```text
start_period
→ Here. MySQL Initialization set-up time
```

---

## Scenario 19: HTTP Service healthcheck

```yaml
services:
  api:
    image: my-api:v1
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
```

Note:

```text
There must be a mirror inside. curl
```

If the image lacks curl, the health check will fail.

---

## Scenario 20: Common healthcheck States

Check:

```bash
docker compose ps
```

Common states:

```text
starting
healthy
unhealthy
```

Check detailed healthcheck information:

```bash
docker inspect ContainersID
```

Focus on:

```text
State.Health.Status
State.Health.Log
```

---

## IX. Complete Multi-Service Orchestration Example: Web + MySQL + Redis

---

## Scenario 21: Directory Structure

```text
compose-demo/
├── compose.yaml
├── .env
└── web/
    └── Dockerfile
```

---

## Scenario 22: .env Example

```text
MYSQL_ROOT_PASSWORD=root123
MYSQL_DATABASE=appdb
MYSQL_USER=appuser
MYSQL_PASSWORD=app123
REDIS_PASSWORD=redis123
WEB_PORT=8080
```

Note:

```text
.env Example:
In the production environment, do not submit the real password. Git Warehouse
```

---

## Scenario 23: compose.yaml Example

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "${WEB_PORT}:80"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-net
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
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

## Scenario 24: Start Multiple Services

Check configuration:

```bash
docker compose config
```

Start:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Check all logs:

```bash
docker compose logs -f
```

Check MySQL logs:

```bash
docker compose logs -f mysql
```

Check Redis logs:

```bash
docker compose logs -f redis
```

Check Web logs:

```bash
docker compose logs -f web
```

---

## Scenario 25: Verify Service Name Resolution

Enter web container:

```bash
docker compose exec web /bin/sh
```

Resolve MySQL service name:

```bash
getent hosts mysql
```

Resolve Redis service name:

```bash
getent hosts redis
```

Description:

```text
Compose The same network service can be accessed through a service name.
```

---

## X. Advanced Environment Variables

---

## Scenario 26: environment

```yaml
services:
  web:
    image: nginx:1.27
    environment:
      APP_ENV: prod
      TZ: Asia/Shanghai
```

Can also be written in list format:

```yaml
services:
  web:
    image: nginx:1.27
    environment:
      - APP_ENV=prod
      - TZ=Asia/Shanghai
```

Recommended:

```text
Simple variables can be written directly environment
```

---

## Scenario 27: env_file

```yaml
services:
  web:
    image: nginx:1.27
    env_file:
      - ./web.env
```

`web.env` Example:

```text
APP_ENV=prod
TZ=Asia/Shanghai
API_BASE_URL=http://api:8080
```

Suitable for:

```text
More variables
Separately manage different service variables
Avoid compose.yaml Too long.
```

---

## Scenario 28: .env File Interpolation

`.env` Example:

```text
WEB_PORT=8080
NGINX_VERSION=1.27
```

`compose.yaml` Example:

```yaml
services:
  web:
    image: nginx:${NGINX_VERSION}
    ports:
      - "${WEB_PORT}:80"
```

Check final configuration:

```bash
docker compose config
```

Description:

```text
.env Used often Compose File Variable Replacement
env_file Often used to inject environmental variables into containers
```

---

## Scenario 29: Common Environment Variable Confusions

Easy to confuse:

```text
.env
→ Here. Compose File for Variable Plugin

env_file
→ Injecting containers with environmental variables

environment
→ Right there. Compose File sets the environment variable for the container
```

Recommended understanding:

```text
Compose Read it yourself. .env
Container Run Read environment / env_file
```

Check final configuration:

```bash
docker compose config
```

Enter container to view variables:

```bash
docker compose exec web env
```

---

## XI. profiles: On-demand Service Enablement

---

## Scenario 30: What are profiles

profiles are used to selectively enable certain services based on environment or purpose.

Suitable for:

```text
Debug Tool
Monitoring component
Manage backstage
One-time assignment
Local development support services
```

Services without specified profiles start by default.

Services with specified profiles only start when the corresponding profile is activated.

---

## Scenario 31: profiles Example

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "8080:80"

  adminer:
    image: adminer:latest
    ports:
      - "8081:8080"
    profiles:
      - debug
```

Default startup:

```bash
docker compose up -d
```

Will only start:

```text
web
```

Enable debug profile:

```bash
docker compose --profile debug up -d
```

Will start:

```text
web
adminer
```

---

## Scenario 32: Multiple profiles

```yaml
services:
  web:
    image: nginx:1.27

  adminer:
    image: adminer:latest
    profiles:
      - debug
      - dev

  redis-insight:
    image: redis/redisinsight:latest
    profiles:
      - debug
```

Enable debug:

```bash
docker compose --profile debug up -d
```

Enable dev:

```bash
docker compose --profile dev up -d
```

---

## Scenario 33: COMPOSE_PROFILES Environment Variable

Set profile:

```bash
export COMPOSE_PROFILES=debug
```

Start:

```bash
docker compose up -d
```

Cancel:

```bash
unset COMPOSE_PROFILES
```

---

## XII. Multi-environment Compose Management

---

## Scenario 34: Using Multiple Compose Files

Base file:

```text
compose.yaml
```

Development override file:

```text
compose.dev.yaml
```

Production override file:

```text
compose.prod.yaml
```

Start development environment:

```bash
docker compose \
  -f compose.yaml \
  -f compose.dev.yaml \
  up -d
```

Start production environment:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d
```

Check final merged configuration:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  config
```

---

## Scenario 35: compose.dev.yaml Example

```yaml
services:
  web:
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    environment:
      APP_ENV: dev
```

---

## Scenario 36: compose.prod.yaml Example

```yaml
services:
  web:
    ports:
      - "80:80"
    environment:
      APP_ENV: prod
    restart: unless-stopped
```

Description:

```text
Basic documents define common services
Environmental file covers port, variable, mount, restart policy, etc.
```

---

## XIII. Common Troubleshooting Approaches

---

## Problem 1: Services Not Starting as Expected

Check status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f
```

Check configuration:

```bash
docker compose config
```

Common causes:

```text
YAML Indentation Error
Wrong service name
Mirror pull failed
Port Conflict
Environmental variables are missing
profile Not enabled
depends_on Dependency not satisfied
healthcheck Always fail.
```

---

## Problem 2: depends_on Configured but Service Still Fails to Connect

Common causes:

```text
depends_on Default Primary Control Start Order
Reliance on container start does not mean application already ready
Long initialization of the database
Web Application started too fast to connect database
```

Recommendation:

```text
Configure Databases healthcheck
web Use depends_on.condition: service_healthy
Apply yourself to support retrying connections
```

Example:

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

---

## Problem 3: healthcheck Always Unhealthy

Check status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs -f Service Name
```

Check inspect:

```bash
docker inspect ContainersID
```

Common causes:

```text
There's no health check order.
Health check port error
It's too long to start.
start_period Too short.
Authentication parameter error
localhost Pointing at not meeting expectations.
Apply health interface back to Africa 2xx
```

For example, if the image lacks curl:

```yaml
healthcheck:
  test: ["CMD-SHELL", "curl -f http://localhost/ || exit 1"]
```

It will fail.

---

## Problem 4: Environment Variables Not Taking Effect

Check final configuration:

```bash
docker compose config
```

Enter container:

```bash
docker compose exec web env
```

Check `.env`:

```bash
cat .env
```

Check env_file:

```bash
cat web.env
```

Common causes:

```text
.env and env_file Confusion
Variable Name Error
Variables are not cited
The container was not recreated after modifying the variable
shell Environmental variables are covered. .env
```

Recreate:

```bash
docker compose up -d --force-recreate
```

---

## Problem 5: Profiles Services Not Starting

Check Compose file:

```bash
cat compose.yaml
```

Check if service has profiles:

```yaml
profiles:
  - debug
```

Default startup:

```bash
docker compose up -d
```

Profile services will not start.

Need:

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

## Problem 6: Services Cannot Access Each Other

Check network:

```bash
docker network ls
```

Check Compose network details:

```bash
docker network inspect Network Name
```

Check service status:

```bash
docker compose ps
```

Resolving service name resolution inside the container:

```bash
docker compose exec web /bin/sh
```

```bash
getent hosts mysql
```

Common causes:

```text
It's not the same network.
Wrong service name
Target service not activated.
Target service listening address error
Apply Connection Address As 127.0.0.1
```

Note:

```text
Inside the container 127.0.0.1 The container itself.
Not another service container.
```

Accessing MySQL should use:

```text
mysql:3306
```

Accessing Redis should use:

```text
redis:6379
```

---

## Problem 7: Port Conflict

Check host listening:

```bash
ss -tunlp
```

Check Compose status:

```bash
docker compose ps
```

Example:

```yaml
ports:
  - "8080:80"
```

If port 8080 on the host is already occupied, change to:

```yaml
ports:
  - "8081:80"
```

Restart:

```bash
docker compose up -d
```

---

## Problem 8: Volume Data Accidentally Deleted

High-risk command:

```bash
docker compose down -v
```

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect volumeName
```

Production note:

```text
Database volume Don't just delete it.
Check for backup before deleting
```

---

## Fourteen. Production Notes

---

## 1. depends_on Cannot Replace Application Retry Mechanism

Even if using:

```yaml
depends_on:
  mysql:
    condition: service_healthy
```

The application itself should support:

```text
Retry database connection
Redis Connect Retry
Retry after startup failed
Timeout Control
Error Log Output
```

Compose can only control container orchestration, and cannot fully replace application-layer fault tolerance.

---

## 2. healthcheck Commands Must Be Genuine and Valid

Not recommended to write formal health checks:

```yaml
healthcheck:
  test: ["CMD", "true"]
```

This only indicates the command succeeded, not business availability.

Recommended:

```text
Database use mysqladmin ping
Redis Use redis-cli ping
HTTP Service /health Interface
Core business application inspection dependency
```

---

## 3. Do Not Hardcode Production Passwords in Compose

Not recommended:

```yaml
environment:
  MYSQL_ROOT_PASSWORD: root123
```

Recommended:

```text
.env
env_file
CI/CD Secret
Special Secret Management tools
```

But even if using `.env`, do not commit real production passwords to Git.

---

## 4. Volumes Must Clearly Define Data Value

For example:

```yaml
volumes:
  - mysql-data:/var/lib/mysql
```

This volume stores database data.

Be cautious before execution:

```bash
docker compose down -v
```

Production environments should have:

```text
Backup Policy
Restore Authentication
Data retention policy
Permission Control
```

---

## 5. Compose Is Suitable for Single-Host Orchestration, Not Equal to Kubernetes

Compose is suitable for:

```text
Local development
Test Environment
Single services
Intermediate Experiment
CI Temporary environment
```

Kubernetes is more suitable for:

```text
Multinodes Schedule
Scroll Release
Service Discovery
Flexible stretch
Self-healing.
Configure and Key Management
Large-scale production environment
```

---

## 6. Multi-Environment Files Should First Undergo config Checks

Before starting, execute:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  config
```

Confirm the final configuration matches expectations before starting:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d
```

---

## Fifteen. Summary of Common Commands in This Article

---

## Compose Basic Checks

Check version:

```bash
docker compose version
```

Check configuration:

```bash
docker compose config
```

Start:

```bash
docker compose up -d
```

Rebuild and start:

```bash
docker compose up -d --build
```

Force recreate:

```bash
docker compose up -d --force-recreate
```

Stop:

```bash
docker compose stop
```

Stop and delete:

```bash
docker compose down
```

Stop, delete, and clean volume:

```bash
docker compose down -v
```

---

## Status and Logs

Check service status:

```bash
docker compose ps
```

Check all logs:

```bash
docker compose logs -f
```

Check specific service logs:

```bash
docker compose logs -f web
```

Enter service container:

```bash
docker compose exec web /bin/sh
```

---

## Network Troubleshooting

Check Docker network:

```bash
docker network ls
```

Check network details:

```bash
docker network inspect Network Name
```

Enter container to test service name resolution:

```bash
docker compose exec web /bin/sh
```

```bash
getent hosts mysql
```

```bash
getent hosts redis
```

---

## Volume Troubleshooting

Check volume:

```bash
docker volume ls
```

Check volume details:

```bash
docker volume inspect volumeName
```

---

## healthcheck Troubleshooting

Check status:

```bash
docker compose ps
```

Check container details:

```bash
docker inspect ContainersID
```

Check service logs:

```bash
docker compose logs -f Service Name
```

---

## profiles

Enable debug profile:

```bash
docker compose --profile debug up -d
```

Set environment variable to enable profile:

```bash
export COMPOSE_PROFILES=debug
```

Start:

```bash
docker compose up -d
```

Cancel:

```bash
unset COMPOSE_PROFILES
```

---

## Multi-Environment Compose

Development environment:

```bash
docker compose \
  -f compose.yaml \
  -f compose.dev.yaml \
  up -d
```

Production environment:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d
```

Check final merged configuration:

```bash
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  config
```

---

## Sixteen. One-Sentence Summary

The core of Docker Compose advancement is not adding more services, but:

```text
There's a dependency between services.
→ Reliance on services is healthy.
→ Communications between services through service name
→ Data to use volume Enduring
→ Environmental variables to be clearly managed
→ Debug Service profiles Enable on demand
```

Understanding depends_on:

```text
depends_on Default Control Start Order
It's not like relying on applications that are fully operational.

More secure:
depends_on + healthcheck + Apply its own retest mechanism
```

Understanding healthcheck:

```text
container running
→ It's just that the container process is running.

container healthy
→ Health check passed, closer to service available
```

Understanding environment variables:

```text
.env
→ Compose File Variable Plugin

env_file
→ Injecting containers with environmental variables

environment
→ Yes. Compose File directly set the packaging environment variable
```

Understanding profiles:

```text
Default service
→ Do Not Configure profiles, Default Start

Optional Services
→ Configure profiles, start as required
```

Production recommendations:

```text
Compose Execute after document change docker compose config
Databases,Redis Waiting for service to be configured healthcheck
Business services should not depend solely on depends_on, support retrying
Don't submit the real production code. Git
Don't just do it. docker compose down -v
Interservice visits use service names. Do not write dead containers. IP
Compose Appropriate for single and small-scale configuration, complex production still recommended Kubernetes
```