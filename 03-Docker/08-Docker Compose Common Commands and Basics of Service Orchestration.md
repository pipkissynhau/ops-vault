# 08-Docker Compose Common Commands and Basics of Service Orchestration

#Docker #DockerCompose #Container Orchestration #Service Orchestration #YAML #Operations #Troubleshooting

---

## Recommended Path

03-Container Technology/08-Docker Compose Common Commands and Basics of Service Orchestration.md

---

## I. Document Description

This document outlines the common commands and basic service orchestration capabilities of Docker Compose, focusing on:

- Checking the Docker Compose version
- Starting services in foreground mode
- Starting services in background mode
- Stopping services
- Stopping and deleting services
- Deleting services and cleaning up volumes
- Restarting services
- Checking service status
- Viewing logs
- Viewing logs for a specific service
- Entering containers managed by Compose
- Building images
- Building and starting services
- Forcing the reconstruction of services
- Checking the parsed results of Compose configuration
- Pulling images
- Scaling services
- Passing environment variables in Compose
- `environment`
- `env_file`

The goal is to:

- Be able to start multi-container services using Docker Compose
- Be able to check service status
- Be able to view logs
- Be able to enter containers for troubleshooting
- Be able to build and rebuild services
- Be able to check Compose configuration
- Understand how Compose manages environment variables

---

## II. Basic Understanding of Docker Compose

Docker Compose is primarily used to manage a group of related containers.

A single container can be started using:

```bash
docker run
```

For multi-container services, it is more suitable to use:

```bash
docker compose
```

For example, a simple business system might include:

- A web service
- Redis
- MySQL
- Nginx
- A background task service

If all these are started manually using `docker run`, there would be many commands and high maintenance costs.

With Docker Compose, these services can be defined in a file like `compose.yaml` or `docker-compose.yml`, and then started, stopped, monitored, and managed with just one set of commands.

---

## III. Common Commands of Docker Compose

---

## Scenario 55: Checking the Version

### Command

```bash
docker compose version
```

### Description

- For new versions, it is recommended to use `docker compose` with a space in between.
- For older versions, use `docker-compose` with a hyphen in between.

### Operations Understanding

Nowadays, it is more common to use:

```bash
docker compose
```

An older version might be written as:

```bash
docker-compose
```

The difference between the two is:

```text
docker compose
→ A Docker CLI plugin form, recommended for newer versions

docker-compose
→ An independent binary command for older versions
```

If the following command fails:

```bash
docker compose version
```

you can try:

```bash
docker compose version
```

---

## Scenario 56: Starting Services

### Starting in Foreground Mode

```bash
docker compose up
```

### Starting in Background Mode

```bash
docker compose up -d
```

### Description

Starting in foreground mode:

```bash
docker compose up
```

Features:

- Logs are directly output in the current terminal.
- Suitable for debugging.
- Terminating the terminal may affect the service.

Starting in background mode:

```bash
docker compose up -d
```

Features:

- The service runs in the background.
- The terminal can be closed without affecting the service.
- More suitable for daily operations and testing environments.

---

## Scenario 57: Stopping and Deleting Services

### Only Stopping

```bash
docker compose stop
```

### Stopping and Deleting

```bash
docker compose down
```

### Deleting Including Volumes

```bash
docker compose down -v
```

### Description

`stop` only stops the container without deleting it:

```bash
docker compose stop
```

`down` stops and deletes all resources created by Compose, such as containers and networks:

```bash
docker compose down
```

`down -v` also deletes volumes:

```bash
docker compose down -v
```

### Note

`docker compose down -v` carries the risk of data loss.

If volumes contain database data, uploaded files, or other important business data, executing this command may result in data loss.

---

## Scenario 58: Restarting and Checking Status

### Restarting

```bash
docker compose restart
```

### Checking Status

```bash
docker compose ps
```

### Description

Restart all Compose services:

```bash
docker compose restart
```

Check the status of services:

```bash
docker compose ps
```

Common status values include:

```text
running
exited
restarting
services:
  web:
    image: nginx
    env_file:
      - .env### Background Execution

To start all services in the Docker Compose stack:

```bash
docker compose up -d
```

To stop all services without deleting them:

```bash
docker compose stop
```

To stop all services, delete them, and clean up any associated volumes:

```bash
docker compose down
```

To stop all services, delete them, and clean up volumes along with any data in them:

```bash
docker compose down -v
```

---

### Restarting and Checking Status

To restart all services:

```bash
docker compose restart
```

To list all currently running services:

```bash
docker compose ps
```

---

### Viewing Logs

To view all logs generated by the Docker Compose stack:

```bash
docker compose logs
```

To view all logs in real time:

```bash
docker compose logs -f
```

To view the logs specific to a particular service:

```bash
docker compose logs -f nginx
```

---

### Entering Containers

To enter the `nginx` service container and execute commands within it:

```bash
docker compose exec nginx /bin/bash
```

If `bash` is not available in the container, use:

```bash
docker compose exec nginx /bin/sh
```

To enter the `web` service container and execute commands within it:

```bash
docker compose exec web /bin/sh
```

---

### Building and Rebuilding Services

To build all services but not start them:

```bash
docker compose build
```

To build all services and start them immediately:

```bash
docker compose up -d --build
```

To force a rebuild of all services:

```bash
docker compose up -d --force-recreate
```

---

### Pulling Images and Scaling Services

To pull images from Docker repositories:

```bash
docker compose pull
```

To scale up a service by increasing the number of instances:

```bash
docker compose up -d --scale web=3
```

---

### Auxiliary Tools for Troubleshooting

To check which ports on the host machine are being listened on by Docker services:

```bash
ss -tunlp
```

To list all Docker volumes currently in use:

```bash
docker volume ls
```

To view the contents of the `.env` file:

```bash
ls -lh .env
```

To read the entire `.env` file:

```bash
cat .env
```

To check the environment variables inside a container:

```bash
docker compose exec web env
```

---

## Summary

The core value of Docker Compose is that it allows you to consolidate multiple `docker run` commands into a single configuration file, making it much easier to manage and automate your Docker applications. Common operations include configuring services, starting and stopping them, checking their status, and viewing logs. For more advanced tasks, you can build and rebuild services, pull images, or scale them accordingly.

Here’s a summary of common workflows:

- **Configuration and Execution**: `docker compose config` → `docker compose up -d`
- **Status Monitoring**: `docker compose ps`
- **Log Viewing**: `docker compose logs -f`
- **Container Interaction**: `docker compose exec service_name /bin/sh`
- **Service Updates**: Modify the `compose.yaml` file, then reconfigure and execute services.
- **Build and Rebuild**: `docker compose build` → `docker compose up -d --build`
- **Troubleshooting**: Use `docker compose ps`, logs, and other tools to identify and resolve issues.

For production environments, it’s recommended to carefully manage environment variables in `.env` files and avoid unnecessary downgrades or force-recreates of services. Docker Compose is well-suited for small-scale deployments but may be more complex to manage in larger production setups, where Kubernetes might be a better choice.