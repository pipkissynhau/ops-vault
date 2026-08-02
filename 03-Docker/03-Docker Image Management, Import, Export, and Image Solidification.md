# 03-Docker Image Management, Import/Export, and Image Solidification

#Docker #Image Management #Image Import/Export #Offline Migration #dockercommit #dockerload #dockersave #Operations and Maintenance

---

## Recommended Path

03-Container Technology/03-Docker Image Management, Import/Export, and Image Solidification.md

---

## I. Document Description

This document compiles common operations and maintenance methods for Docker image management, import/export, and image solidification, focusing on:

- Viewing local images
- Pulling images
- Deleting images
- Packaging containers into images
- Adding descriptions to committed images
- Saving images as tar packages
- Importing tar packages as local images
- The differences between `save/load` and `export/import`
- Exporting container file systems
- The standard process for offline image migration

The goal is:

To be able to view images → Pull images → Delete images → Temporarily solidify container states on-site → Export image tar packages → Import images in offline environments → Distinguish between image migration and container file system migration

---

## II. Basic Docker Image Management

---

## Scenario 6: Image Management

Viewing images:

```bash
docker images
```

Pulling images:

```bash
docker pull nginx
```

Deleting images:

```bash
docker rmi imageID
```

### Explanation

`docker images` is used to view existing local images.

`docker pull` is used to pull images from an image repository.

`docker rmi` is used to delete local images.

Common usage scenarios:

- Checking if the target image exists locally
- Pulling base images
- Deleting unnecessary images
- Clearing old version images
- Verifying if an image tag exists

---

## III. Image Import/Export and Image Solidification

---

## Scenario 13: Packaging a Container into an Image (commit)

### Command

```bash
docker commit containerID my-nginx:v1
```

### Example

```bash
docker commit nginx-test my-nginx:v1
```

### Meaning

- Saves the current state of the container as a new image.
- Commonly used for:
  - Temporarily saving modified states on-site
  - Quickly solidifying environments after troubleshooting
  - Quickly retaining test environment configurations

### Explanation

- `docker commit` is suitable for temporary operations.
- For production use, it is recommended to build images using Dockerfiles.
- Long-term reliance on `commit` for image version management is not advised.

---

## Scenario 14: Adding Descriptions to a Committed Image

### Command

```bash
docker commit -m "Install curl and modify config" containerID my-nginx:v2
```

### Explanation

- `-m`: Adds descriptive information, making it easier to identify the image's origin later.

---

## Scenario 15: Saving an Image as a tar Package

### Command

```bash
docker save -o nginx.tar nginx:latest
```

### Exporting Multiple Images Together

```bash
docker save -o app-bundle.tar nginx:latest redis:7 alpine:latest
```

### Meaning

- Saves images as tar packages.
- Commonly used for:
  - Offline transfer
  - Image backup
  - Importing images in environments without internet access

---

## Scenario 16: Importing an Image Tar Package Locally

### Command

```bash
docker load -i nginx.tar
```

### Meaning

- Restores the image from a tar package into local Docker.

### Verification

```bash
docker images
```

---

## Scenario 17: The Differences between save/load and export/import

### save/load

```bash
docker save
```

```bash
docker load
```

Features:

- Saves the entire image, including layers, tags, and metadata.
- More suitable for image migration and backup.

### export/import

```bash
docker export
```

```bash
docker import
```

Features:

- Exports only the container file system.
- Does not retain image layer history or most metadata.
- Generally not used as a standard method for image migration.

---

## Scenario 18: Exporting a Container File System

### Command

```bash
docker export containerID -o container.tar
```

### Importing as an Image

```bash
cat container.tar | docker import - myimage:imported
```

### Explanation

- This is for exporting/importing the container file system, not the entire image.

---

## Scenario 19: The Standard Process for Offline Image Migration

### Export

```bash
docker save -o myapp.tar myapp:latest
```

### Transfer the tar package to the target machine

Copy `myapp.tar` to the target server using methods like `scp`, bastion host file transfer, USB drive, or internal network file servers.

### Import

```bash
docker load -## Docker Image Export

Export a single image:

```bash
docker save -o nginx.tar nginx:latest
```

Export multiple images:

```bash
docker save -o app-bundle.tar nginx:latest redis:7 alpine:latest
```

---

## Docker Image Import

Import an image from a tar package:

```bash
docker load -i nginx.tar
```

Verify after import:

```bash
docker images
```

```bash
docker images | grep myapp
```

---

## Docker Container File System Export and Import

Export the container file system:

```bash
docker export containerID -o container.tar
```

Import as an image:

```bash
cat container.tar | docker import - myimage:imported
```

---

## Offline Migration Standard Process

On a machine with internet access, pull the image:

```bash
docker pull myapp:latest
```

Export the image:

```bash
docker save -o myapp.tar myapp:latest
```

On the target machine, import the image:

```bash
docker load -i myapp.tar
```

Verify the image:

```bash
docker images | grep myapp
```

---

## VII. Summary in One Sentence

The core capabilities of Docker image management include:

- Viewing images
- Pulling images
- Deleting images
- Temporarily preserving container states
- Exporting images
- Importing images
- Performing offline image migrations

Key differences:

```text
docker commit
→ Saves the current container state as a new image, suitable for temporary preservation but not for long-term production use

docker save / docker load
→ Imports/export at the image level, ideal for migration, backup, and offline delivery

docker export / docker import
→ Imports/export at the container file system level, without retaining all image layers and metadata
```

Production recommendations:

```text
Use Dockerfile for building official images
Use docker commit for temporary troubleshooting and preservation
Use docker save / docker load for offline image migrations
Avoid using docker export / import as standard image migration methods
```