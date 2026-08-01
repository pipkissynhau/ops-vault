# 08-Docker Compose Common Commands and Service Orchestration Basics

#Docker #DockerCompose #ContainerSchedule #OrganizationOfServices #YAML #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/08-Docker Compose Common Commands and Service Orchestration Basics.md

---

## I. Document Description

This document organizes common Docker Compose commands and basic service orchestration capabilities, with focus on:

- Checking Docker Compose version
- Starting services in foreground
- Starting services in background
- Stopping services
- Stopping and removing services
- Removing services and cleaning volume
- Restarting services
- Checking service status
- Viewing logs
- Viewing logs of specific service
- Entering Compose-managed container
- Building images
- Building and starting services
- Force-rebuilding services
- Checking Compose configuration parsing results
- Pulling images
- Scaling services
- Compose environment variable passing
- `environment`
- `env_file`

The goal is:

Can use Docker Compose to start multi-container services

→ Can check service status

→ Can view logs

→ Can enter container for troubleshooting

→ Can build and rebuild services

→ Can check Compose configuration

→ Can understand Compose environment variable management

---

## II. Basic Understanding of Docker Compose

Docker Compose is mainly used to manage a group of related containers.

Single container can use:

```bash
docker run
```

Multi-container services are better suited with:

```bash
docker compose
```

For example, a simple business system may include:

- Web service
- Redis
- MySQL
- Nginx
- Background task service

If all are started manually with `docker run`, the commands will be numerous and maintenance cost high.

After using Docker Compose, these services can be unified written in `compose.yaml` or `docker-compose.yml`, and managed with a set of commands for unified start, stop, check and management.

---

## III. Common Docker Compose Commands

---

## Scenario 55: Check Version

### Command

```bash
docker compose version
```

### Description

- New versions recommend using `docker compose`, with space in the middle
- Old versions are `docker-compose`, with hyphen in the middle

### Operations Understanding

Now it's recommended to use:

```bash
docker compose
```

Old version common writing is:

```bash
docker-compose
```

Difference between them:

```text
docker compose
→ Docker CLI Plugin format, more recent version recommended

docker-compose
→ The old independent binary command
```

If the following command fails:

```bash
docker compose version
```

Try:

```bash
docker-compose version
```

---

## Scenario 56: Start Services

### Foreground Start

```bash
docker compose up
```

### Background Start

```bash
docker compose up -d
```

### Description

Foreground start:

```bash
docker compose up
```

Features:

- Logs are directly output to current terminal
- Suitable for debugging
- Terminal interruption may affect service

Background start:

```bash
docker compose up -d
```

Features:

- Service runs in background
- Terminal can be released
- More suitable for daily operation and test environment

---

## Scenario 57: Stop and Remove

### Stop Only

```bash
docker compose stop
```

### Stop and Remove

```bash
docker compose down
```

### Remove Including Volume

```bash
docker compose down -v
```

### Description

`stop` stops container only, does not delete container:

```bash
docker compose stop
```

`down` stops and removes Compose-created containers, networks, etc. resources:

```bash
docker compose down
```

`down -v` additionally deletes volume:

```bash
docker compose down -v
```

### Note

`docker compose down -v` has data risk.

If volume contains database data, uploaded files or other business data, execution may lead to data loss.

---

## Scenario 58: Restart and Status Check

### Restart

```bash
docker compose restart
```

### Check Status

```bash
docker compose ps
```

### Description

Restart all Compose services:

```bash
docker compose restart
```

Check service status:

```bash
docker compose ps
```

Common status includes:

```text
running
exited
restarting
created
```

If service is not running normally, prioritize check:

```bash
docker compose ps
```

Then check logs:

```bash
docker compose logs -f
```

---

## Scenario 59: View Logs

### View All Logs

```bash
docker compose logs
```

### Real-time Log View

```bash
docker compose logs -f
```

### View Specific Service Logs

```bash
docker compose logs -f nginx
```

### Description

View all service logs:

```bash
docker compose logs
```

Real-time log tracking:

```bash
docker compose logs -f
```

View logs of specific service:

```bash
docker compose logs -f nginx
```

Common uses:

- Troubleshoot service startup failure
- Troubleshoot application errors
- Check if dependencies are normal
- View logs of Nginx, MySQL, Redis, etc.
- Observe service startup order issues

---

## Scenario 60: Enter Container

### Command

```bash
docker compose exec nginx /bin/bash
```

### Description

This command represents entering the container of Compose service named `nginx`.

If container has no bash, can use:

```bash
docker compose exec nginx /bin/sh
```

### Operations Understanding

`docker compose exec` and `docker exec` are similar.

Difference is:

```text
docker exec
→ Through the container ID or the name of the container enters the container

docker compose exec
→ Pass. Compose Service name in the container
```

For example Compose service name is `web`, can execute:

```bash
docker compose exec web /bin/sh
```

---

## Scenario 61: Build and Rebuild

### Build Only

```bash
docker compose build
```

### Build and Start

```bash
docker compose up -d --build
```

### Force Rebuild

```bash
docker compose up -d --force-recreate
```

### Description

Build image only, no start service:

```bash
docker compose build
```

Build image and start service in background:

```bash
docker compose up -d --build
```

Force recreate container:

```bash
docker compose up -d --force-recreate
```

### Operations Understanding

Common use scenarios:

```text
Modify Dockerfile
→ docker compose build

Restart after changing code or mirror construction logic
→ docker compose up -d --build

Suspected that the container is in an abnormal state and needs to be forcibly rebuilt
→ docker compose up -d --force-recreate
```

---

## Scenario 62: View Configuration Parsing Results

### Command

```bash
docker compose config
```

### Function

- Check if yml is correct
- View final merged configuration

### Description

`docker compose config` is a very useful check command.

Common uses:

- Check YAML syntax
- View final result after environment variable substitution
- View merged configuration from multiple Compose files
- Check if service, network, volume configurations meet expectations

If Compose file has syntax errors, this command will usually report errors directly.

---

## Scenario 63: Pull Images and Scale

### Pull Image

```bash
docker compose pull
```

### Scale Service

```bash
docker compose up -d --scale web=3
```

### Description

Pull images defined in Compose file:

```bash
docker compose pull
```

Scale `web` service to 3 replicas:

```bash
docker compose up -d --scale web=3
```

### Operations Understanding

`--scale` is suitable for simple multi-replica testing scenarios.

For example:

```bash
docker compose up -d --scale web=3
```

Means start 3 containers of `web` service.

Note:

- If a service is bound to a fixed host port, scaling may cause port conflicts
- More complex multi-replica and service discovery capabilities are typically handled by platforms like Kubernetes
- Compose scaling is suitable for single-machine testing and simple scenarios

---

## Scenario 64: Passing Environment Variables to Compose

### Compose File Example

```yaml
services:
  web:
    image: nginx
    environment:
      - APP_ENV=prod
      - TZ=Asia/Shanghai
```

### Or use env_file

```yaml
services:
  web:
    image: nginx
    env_file:
      - .env
```

### Explanation

There are two common ways to pass environment variables in Compose:

The first is to write them directly in `environment`:

```yaml
services:
  web:
    image: nginx
    environment:
      - APP_ENV=prod
      - TZ=Asia/Shanghai
```

The second is to use `env_file`:

```yaml
services:
  web:
    image: nginx
    env_file:
      - .env
```

`.env` File Example:

```text
APP_ENV=prod
TZ=Asia/Shanghai
```

---

## IV. Basic Compose File Examples

---

## 1. Minimal Nginx Compose Example

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
```

Start:

```bash
docker compose up -d
```

Check status:

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

## 2. Compose Example with Environment Variables

```yaml
services:
  web:
    image: nginx:latest
    environment:
      - APP_ENV=prod
      - TZ=Asia/Shanghai
```

Check configuration:

```bash
docker compose config
```

Start service:

```bash
docker compose up -d
```

---

## 3. Compose Example Using env_file

`compose.yaml` Example:

```yaml
services:
  web:
    image: nginx:latest
    env_file:
      - .env
```

`.env` Example:

```text
APP_ENV=prod
TZ=Asia/Shanghai
```

Start:

```bash
docker compose up -d
```

---

## 4. Compose Example with Ports and Mounts

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
```

Start:

```bash
docker compose up -d
```

Check if mounts are effective:

```bash
docker compose exec nginx /bin/sh
```

Check inside the container:

```bash
ls -lh /usr/share/nginx/html
```

---

## V. Common Troubleshooting Approaches

---

## 1. Compose Service Startup Failure

First check service status:

```bash
docker compose ps
```

Check logs:

```bash
docker compose logs
```

Real-time log viewing:

```bash
docker compose logs -f
```

Check logs for a specific service:

```bash
docker compose logs -f nginx
```

Check Compose configuration:

```bash
docker compose config
```

Common causes:

- YAML indentation errors
- Image pull failures
- Port conflicts
- Volume mount path errors
- Missing environment variables
- Service startup command errors
- Dependent services not started
- Application errors inside the container

---

## 2. Compose YAML Format Errors

Prioritize executing:

```bash
docker compose config
```

If there are YAML format errors, they will typically be exposed here directly.

Common issues:

- Indentation errors
- Missing space after colon
- Mismatched quotation marks
- `services` level written incorrectly
- `ports`, `volumes` list format errors

Error example:

```yaml
services:
 nginx:
 image: nginx
```

More standardized writing:

```yaml
services:
  nginx:
    image: nginx
```

---

## 3. Port Conflict Troubleshooting

Check Compose service status:

```bash
docker compose ps
```

Check host port listening:

```bash
ss -tunlp
```

If the port is already occupied, modify the port mapping in the Compose file.

For example, originally:

```yaml
ports:
  - "8080:80"
```

Can be changed to:

```yaml
ports:
  - "8081:80"
```

Then restart:

```bash
docker compose up -d
```

---

## 4. Troubleshooting Image Pull Failures

First pull the image:

```bash
docker compose pull
```

Check specific errors:

```bash
docker compose up
```

Common causes:

- Incorrect image name
- Non-existent tag
- Network unable to access image registry
- Private registry not logged in
- HTTP Harbor not configured with trust
- containerd / Docker registry configuration inconsistency

If it's a Docker private registry, you may need to log in first:

```bash
docker login Warehouse Address
```

---

## 5. Troubleshooting Service Repeated Restarts

Check status:

```bash
docker compose ps
```

Check logs:

§

Check logs:

```bash
docker compose logs -f Service Name
```

Enter the container for troubleshooting:

```bash
docker compose exec Service Name /bin/sh
```

Common causes:

- Application configuration errors
- Startup command errors
- Dependency service connection failures
- Incorrect environment variables
- Mounted files overriding critical directories in the container
- Port listening failures

---

## 6. Troubleshooting Ineffective Environment Variables

Check the final Compose configuration:

```bash
docker compose config
```

Check environment variables inside the container:

```bash
docker compose exec web env
```

If using `env_file`, check if `.env` file exists:

```bash
ls -lh .env
```

Check contents of `.env`:

```bash
cat .env
```

Common causes:

- `.env` file path is incorrect
- Environment variable name written incorrectly
- Compose file indentation errors
- Application reads variable names inconsistent with inputs
- No container recreation after modifying environment variables

Recreate the service:

```bash
docker compose up -d --force-recreate
```

---

## VI. Production Considerations

---

## 1. Do not casually execute down -v

High-risk command:

```bash
docker compose down -v
```

Reasons:

- Deletes Compose-created volumes
- May result in database data loss
- May result in uploaded file loss
- May result in business state data loss

Before execution, it's recommended to first check volumes:

```bash
docker volume ls
```

Confirm whether Compose project-related volumes can be deleted.

---

## 2. After modifying Compose files, it's recommended to first check the configuration

After modification, first execute:

```bash
docker compose config
```

Confirm there are no syntax errors before starting:

```bash
docker compose up -d
```

This can avoid service startup failures caused by YAML errors.

---

## 3. Avoid port conflicts in port mapping

Compose port mapping example:

```yaml
ports:
  - "8080:80"
```

Meaning:

```text
Host 8080 -> Containers 80
```

If multiple services bind to the same host port, it will cause conflicts.

Check host listening:

```bash
ss -tunlp
```

---

## 4. Pay attention to port binding when scaling services

Scaling command:

```bash
docker compose up -d --scale web=3
```

If `web` service has configured fixed host ports, for example:

```yaml
ports:
  - "8080:80"
```

Scaling may fail because multiple replicas cannot bind to the same host port simultaneously.

In such cases, it's more suitable to:

- Not expose each replica's port directly
- Use a reverse proxy
- Use Docker internal network
- Or delegate to Kubernetes Service handling

---

## 5. Do not put sensitive information arbitrarily in env_file

`.env` File example:

```text
DB_PASSWORD=123456
```

Risks:

- Easy to be mistakenly committed to Git
- Easy to be viewed by unauthorized personnel
- Easy to remain in plaintext files on servers

Recommendations:

- Control file permissions
- Avoid committing to code repositories
- Use CI/CD key variables
- Use more secure Secret management methods in production environments

---

## 6. Compose is more suitable for single-machine or small-scale orchestration

Docker Compose is suitable for:

- Local development environment
- Single-machine testing environment
- Small-scale service composition
- Rapid setup of dependency environment
- Learning and validating container service orchestration

Complex production environments are typically more suitable for:

- Kubernetes
- Swarm
- Nomad
- Cloud provider container services

---

## Seven, Common Commands Summary

---

## Version and Configuration Check

Check version:

```bash
docker compose version
```

Old version command:

```bash
docker-compose version
```

Check Compose configuration:

```bash
docker compose config
```

---

## Start and Stop

Foreground start:

```bash
docker compose up
```

Background start:

```bash
docker compose up -d
```

Stop only:

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

## Restart and Status Check

Restart service:

```bash
docker compose restart
```

Check status:

```bash
docker compose ps
```

---

## Log Viewing

View all logs:

```bash
docker compose logs
```

Real-time view all logs:

```bash
docker compose logs -f
```

View logs for specific service:

```bash
docker compose logs -f nginx
```

---

## Enter Container

Enter nginx service container:

```bash
docker compose exec nginx /bin/bash
```

If no bash:

```bash
docker compose exec nginx /bin/sh
```

Enter web service container:

```bash
docker compose exec web /bin/sh
```

---

## Build and Rebuild

Build only:

```bash
docker compose build
```

Build and start:

```bash
docker compose up -d --build
```

Force rebuild:

```bash
docker compose up -d --force-recreate
```

---

## Image Pull and Scaling

Pull image:

```bash
docker compose pull
```

Scale service:

```bash
docker compose up -d --scale web=3
```

---

## Troubleshooting Assistance

Check host port listening:

```bash
ss -tunlp
```

Check Docker volume:

```bash
docker volume ls
```

Check `.env` file:

```bash
ls -lh .env
```

Check `.env` content:

```bash
cat .env
```

Enter container to check environment variables:

```bash
docker compose exec web env
```

---

## Eight, One-Sentence Summary

The core value of Docker Compose is:

Converging multiple `docker run` commands

→ Into a single Compose configuration file

→ Then using unified commands to start, stop, check, and troubleshoot services

Common operation flow:

```text
docker compose config
→ docker compose up -d
→ docker compose ps
→ docker compose logs -f
→ docker compose exec Service Name /bin/sh
```

Service update flow:

```text
Modify compose.yaml
→ docker compose config
→ docker compose up -d
→ docker compose ps
→ docker compose logs -f Service Name
```

Build update flow:

```text
Modify Dockerfile or build context
→ docker compose build
→ docker compose up -d --build
→ docker compose logs -f
```

Troubleshooting flow:

```text
docker compose ps
→ docker compose logs -f Service Name
→ docker compose config
→ docker compose exec Service Name /bin/sh
→ Check port, mount, environment variable, service dependence
```

Production recommendations:

```text
Modify Compose File First Execute docker compose config
Don't just do it. docker compose down -v
Fixed port mapping affects service expansion.
Environmental variable files need to be sensitive to information protection
Compose Suitable for single- or small-scale service organization and more suitable for a complex production environment Kubernetes
```