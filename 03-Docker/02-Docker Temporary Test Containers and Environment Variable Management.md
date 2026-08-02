# 02-Docker Temporary Test Containers and Environment Variable Management

#Docker #Containers #Temporary Containers #Environment Variables #Ops #Troubleshooting

---

## Recommended Path

03-Container Technology/02-Docker Temporary Test Containers and Environment Variable Management.md

---

## I. Document Description

This document outlines common practices for managing Docker temporary test containers and environment variables, focusing on:

- Temporary test containers
- Automatic deletion after container exit
- Background temporary test containers
- Passing environment variables using `-e`
- Passing environment variables using `--env-file`
- Viewing container environment variables
- Temporary BusyBox/Alpine test containers

The goal is to:

- Quickly launch temporary containers
- Verify network, DNS, and command environments
- Control container behavior through environment variables
- Prevent temporary troubleshooting containers from remaining permanently

---

## II. Temporary Test Containers and Environment Variables

---

## Scenario 8: Temporary test container that automatically deletes after exit

### Command

```bash
docker run --rm -it nginx /bin/sh
```

### Meaning

- `--rm`: The container is automatically deleted after it exits.
- `-it`: Enter the container interactively.

### Common Scenarios

- Start a temporary BusyBox for network testing:

```bash
docker run --rm -it busybox /bin/sh
```

- Start a temporary Alpine for DNS testing:

```bash
docker run --rm -it alpine /bin/sh
```

### Note

- Very suitable for temporary testing.
- Does not leave any stopped containers behind.
- Commonly used in production troubleshooting.

---

## Scenario 9: Background temporary test that is deleted after completion

### Command

```bash
docker run --rm -d --name test-nginx nginx
```

### Note

- After use, simply `docker stop test-nginx` to delete the container.
- The container is automatically deleted once it stops.

To stop a background temporary container:

```bash
docker stop test-nginx
```

---

## Scenario 10: Passing environment variables using `-e`

### Command

```bash
docker run -d -e APP_ENV=prod nginx
```

### Multiple environment variables

```bash
docker run -d \
  -e APP_ENV=prod \
  -e APP_PORT=8080 \
  -e TZ=Asia/Shanghai \
  nginx
```

### Meaning

- `-e`: Passes environment variables to the container.

Common uses:

- Differentiating between production and testing environments.
- Configuring ports.
- Setting database connection parameters.
- Adjusting time zones.

---

## Scenario 11: Passing environment variables using an env file

### Command

```bash
docker run -d --env-file .env nginx
```

### Note

- Suitable for scenarios with many environment variables.
- More readable than writing multiple `-e` commands.

---

## Scenario 12: Viewing container environment variables

### Command

```bash
docker inspect containerID | grep -A 20 Env
```

Or, to view inside the container:

```bash
docker exec -it containerID env
```

---

## III. Operational Understanding

---

## 1. Why `--rm` is commonly used for temporary test containers

During troubleshooting, tools like BusyBox, Alpine, Nginx, and Curl are often used in container form to verify issues.

Without `--rm`, these containers will remain on the host after they exit, potentially leading to a large number of stopped containers over time.

Example usage:

```bash
docker run --rm -it busybox /bin/sh
```

This ensures that containers are automatically deleted, making them ideal for temporary testing and troubleshooting.

---

## 2. The role of `-it`

`-it` is often used together with `--rm`:

```bash
docker run --rm -it alpine /bin/sh
```

### Meaning

- `-i`: Keeps the standard input open.
- `-t`: Allocates a pseudo-terminal.

Common uses:

- Entering the shell inside the container.
- Manually executing commands.
- Conducting temporary DNS or network tests.
- Checking if tools are available within the container.

---

## 3. The difference between `-d` and `-it`

### Background execution

```bash
docker run --rm -d --name test-nginx nginx
```

### Interactive execution

```bash
docker run --rm -it busybox /bin/sh
```

**Difference:**

- `-d`: Runs the container in the background, suitable for long-term or temporary service verification.
- `-it`: Runs the container interactively, allowing for manual troubleshooting inside the container.

---

## 4. Why environment variables are suitable for configuration passing

It is not recommended to hardcode all configurations in a Docker image.

A better approach is:

- Keep the image generic.
- Pass configurations via environment variables at runtime.
- Different environments can use different parameters.

Example:

```