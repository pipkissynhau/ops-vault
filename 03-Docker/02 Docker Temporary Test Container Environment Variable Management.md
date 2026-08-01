# 02-Docker Temporary Test Containers and Environment Variable Management

#Docker #Containers #TemporaryContainers #EnvironmentalVariables #Transport #TheBarrier.

---

## Recommended Path

03-Container Technology/02-Docker Temporary Test Containers and Environment Variable Management.md

---

## I. Document Description

This document organizes common usage patterns for Docker temporary test containers and environment variable management, with focus on:

- Temporary test containers
- Automatic deletion after container exit
- Background temporary test containers
- Passing environment variables using `-e`
- Passing environment variables using `--env-file`
- Viewing container environment variables
- Temporary BusyBox / Alpine test containers

The goal is:

To be able to quickly start temporary containers

→ To verify network, DNS, and command environment

→ To control container behavior through environment variables

→ To avoid temporary troubleshooting containers lingering long-term

---

## II. Temporary Test Containers and Environment Variables

---

## Scenario 8: Temporary Test Container, Automatically Deleted After Exit

### Command

```bash
docker run --rm -it nginx /bin/sh
```

### Meaning

- `--rm`: Automatically delete container after exit
- `-it`: Interactive mode to enter container
- Suitable for temporary testing, command verification, and environment troubleshooting

### Common Scenarios

Starting a temporary BusyBox to test network:

```bash
docker run --rm -it busybox /bin/sh
```

Starting a temporary Alpine to test DNS:

```bash
docker run --rm -it alpine /bin/sh
```

### Notes

- Very suitable for temporary testing
- Will not leave stopped state containers
- Frequently used in production troubleshooting

---

## Scenario 9: Background Temporary Test, Automatically Deleted After Completion

### Command

```bash
docker run --rm -d --name test-nginx nginx
```

### Notes

- Directly use `docker stop test-nginx` after use
- Container will be automatically deleted after stopping

Stopping background temporary container:

```bash
docker stop test-nginx
```

---

## Scenario 10: Passing Environment Variables Using -e

### Command

```bash
docker run -d -e APP_ENV=prod nginx
```

### Multiple Environment Variables

```bash
docker run -d \
  -e APP_ENV=prod \
  -e APP_PORT=8080 \
  -e TZ=Asia/Shanghai \
  nginx
```

### Meaning

- `-e`: Pass environment variables to container

Commonly used for:

- Environment differentiation
- Port configuration
- Database connection parameters
- Time zone settings

---

## Scenario 11: Passing Environment Variables Using an env File

### Command

```bash
docker run -d --env-file .env nginx
```

### Notes

- Suitable for scenarios with many environment variables
- More readable than writing many `-e`

---

## Scenario 12: Viewing Container Environment Variables

### Command

```bash
docker inspect ContainersID | grep -A 20 Env
```

Or enter container to view:

```bash
docker exec -it ContainersID env
```

---

## III. Operations and Maintenance Scenario Understanding

---

## 1. Why Temporary Test Containers Frequently Use `--rm`

When temporarily troubleshooting, BusyBox, Alpine, Nginx, and Curl tool containers are often started to verify issues.

Without adding `--rm`, containers will remain on the host after exit, leading to accumulation of stopped state containers over time.

Using:

```bash
docker run --rm -it busybox /bin/sh
```

Containers will be automatically deleted after exit, better suited for temporary verification and troubleshooting.

---

## 2. The Purpose of `-it`

`-it` is typically used together:

```bash
docker run --rm -it alpine /bin/sh
```

Meaning:

- `-i`: Keep standard input open
- `-t`: Allocate pseudo-terminal

Common uses:

- Enter shell
- Manually execute commands
- Temporary DNS testing
- Temporary network testing
- Temporary tool availability testing

---

## 3. Difference Between `-d` and `-it`

Background running:

```bash
docker run --rm -d --name test-nginx nginx
```

Interactive running:

```bash
docker run --rm -it busybox /bin/sh
```

Difference:

- `-d`: Background running, suitable for long-term or temporary service verification
- `-it`: Interactive running, suitable for manual troubleshooting inside container

---

## 4. Why Environment Variables Are Suitable for Configuration Passing

Containers typically do not recommend hardcoding all configurations in the image.

More common approach:

- Keep image generic
- Pass configurations via environment variables at runtime
- Pass different parameters for different environments

Example:

```bash
docker run -d \
  -e APP_ENV=prod \
  -e APP_PORT=8080 \
  -e TZ=Asia/Shanghai \
  nginx
```

This allows the same image to run in test, staging, and production environments by adjusting environment variables.

---

## 5. `--env-file` Is Suitable for Scenarios with Many Environment Variables

When there are many environment variables, it's not recommended to write many `-e` in command line as readability decreases.

Better to use:

```bash
docker run -d --env-file .env nginx
```

Example `.env` file content:

```text
APP_ENV=prod
APP_PORT=8080
TZ=Asia/Shanghai
```

Notes:

- `.env` file is clearer
- Easier for unified maintenance
- Suitable for services with many variables
- In production environments, sensitive information protection is needed

---

## IV. Common Notes

---

## 1. Do Not Store Sensitive Passwords Long-Term in Command History

Example:

```bash
docker run -d -e DB_PASSWORD=123456 nginx
```

Such commands may be recorded in shell history.

In production environments, sensitive information should be managed more securely, such as:

- Environment variable file permission control
- Secret management
- CI/CD variables
- Kubernetes Secret

---

## 2. Do Not Mistake Temporary Containers for Formal Services

Example:

```bash
docker run --rm -d --name test-nginx nginx
```

This container will be automatically deleted after stopping.

Therefore it's more suitable for:

- Temporary verification
- Network testing
- DNS testing
- Port testing
- Tool testing

Not suitable as a formal long-running service.

---

## 3. BusyBox and Alpine Are Commonly Used for Lightweight Testing

BusyBox:

```bash
docker run --rm -it busybox /bin/sh
```

Alpine:

```bash
docker run --rm -it alpine /bin/sh
```

Common uses:

- DNS testing
- Network connectivity testing
- Basic container environment testing
- Verifying image pull functionality

---

## 4. Note Container Status When Viewing Environment Variables

If the container is running, you can enter the container to view:

```bash
docker exec -it ContainersID env
```

If you only want to view from Docker metadata, use:

```bash
docker inspect ContainersID | grep -A 20 Env
```

---

## V. Common Commands Summary of This Article

---

## Docker Temporary Test and Environment Variables

Temporary test container:

```bash
docker run --rm -it busybox /bin/sh
```

Start a temporary Alpine to test DNS:

```bash
docker run --rm -it alpine /bin/sh
```

Start a temporary Nginx container and enter shell:

```bash
docker run --rm -it nginx /bin/sh
```

Background temporary test, automatically deleted after completion:

```bash
docker run --rm -d --name test-nginx nginx
```

Stop background temporary container:

```bash
docker stop test-nginx
```

Pass environment variables:

```bash
docker run -d -e APP_ENV=prod nginx
```

Pass multiple environment variables:

```bash
docker run -d \
  -e APP_ENV=prod \
  -e APP_PORT=8080 \
  -e TZ=Asia/Shanghai \
  nginx
```

Use env file:

```bash
docker run -d --env-file .env nginx
```

View container environment variables:

```bash
docker inspect ContainersID | grep -A 20 Env
```

Enter container to view environment variables:

```bash
docker exec -it ContainersID env
```

---

## VI. One-Sentence Summary

The core value of Docker temporary test containers is:

Quick start

→ Quick verification

→ Quick exit

→ No leftover containers

The core value of environment variables is:

Keep image generic

→ Inject configuration at runtime

→ Reuse the same image across different environments

Common combination:

```text
docker run --rm -it busybox /bin/sh
→ Temporary Network / DNS / Command Test

docker run --rm -d --name test-nginx nginx
→ Backstage temporary service testing

docker run -d -e APP_ENV=prod nginx
→ Simple environment variable injection

docker run -d --env-file .env nginx
→ Harmonized management of multi-environment variables
```