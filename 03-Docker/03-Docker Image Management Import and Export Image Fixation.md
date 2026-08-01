# 03-Docker Image Management, Import/Export and Image Solidification

#Docker #MirrorManagement #MirrorImportExport #OfflineMigration #dockercommit #dockerload #dockersave #Transport

---

## Recommended Path

03-Container Technology/03-Docker Image Management, Import/Export and Image Solidification.md

---

## One, Document Explanation

This document organizes common operations for Docker image management, image import/export and image solidification, with focus on:

- View local images
- Pull images
- Delete images
- Package container as image
- Add commit notes to image
- Save image as tar package
- Import tar package as local image
- Difference between `save / load` and `export / import`
- Export container filesystem
- Standard process for offline image migration

Goals:

Can view images

→ Can pull images

→ Can delete images

→ Can temporarily solidify container state

→ Can export image tar package

→ Can import image in offline environment

→ Can distinguish between image migration and container filesystem migration

---

## Two, Docker Image Basic Management

---

## Scenario 6: Image Management

View images:

```bash
docker images
```

Pull images:

```bash
docker pull nginx
```

Delete images:

```bash
docker rmi MirrorID
```

### Explanation

`docker images` is used to view existing local images.

`docker pull` is used to pull images from image repository.

`docker rmi` is used to delete local images.

Common usage scenarios:

- Check if target image exists locally
- Pull base image
- Delete unused images
- Clean up old version images
- Verify if image tag exists

---

## Three, Image Import/Export and Image Solidification

---

## Scenario 13: Package Container as Image (commit)

### Command

```bash
docker commit ContainersID my-nginx:v1
```

### Example

```bash
docker commit nginx-test my-nginx:v1
```

### Meaning

- Save current container state as new image
- Commonly used for:
  - Save state after temporary modification
  - Quickly solidify environment after troubleshooting
  - Rapidly preserve test environment

### Explanation

- `docker commit` is suitable for temporary operations
- Formal production recommends using Dockerfile to build images
- Not recommended to rely on commit for long-term image maintenance

---

## Scenario 14: Add Notes to Commit Image

### Command

```bash
docker commit -m "install curl and modify config" ContainersID my-nginx:v2
```

### Explanation

- `-m`: Add description information
- Facilitate subsequent identification of image source

---

## Scenario 15: Save Image as tar Package

### Command

```bash
docker save -o nginx.tar nginx:latest
```

### Export Multiple Images

```bash
docker save -o app-bundle.tar nginx:latest redis:7 alpine:latest
```

### Meaning

- Export image as tar package
- Commonly used for:
  - Transfer in offline environment
  - Backup images
  - Import images in no-external-network environment

---

## Scenario 16: Import Image tar Package to Local

### Command

```bash
docker load -i nginx.tar
```

### Meaning

- Restore image from tar package to local Docker

### Verification

```bash
docker images
```

---

## Scenario 17: Difference between save/load and export/import

### save/load

```bash
docker save
```

```bash
docker load
```

Features:

- Save image
- Preserve image layers, tags, metadata
- More suitable for image migration and backup

### export/import

```bash
docker export
```

```bash
docker import
```

Features:

- Export container filesystem
- Do not preserve image layer history and many metadata
- Generally not used as standard image migration method

---

## Scenario 18: Export Container Filesystem

### Command

```bash
docker export ContainersID -o container.tar
```

### Import as Image

```bash
cat container.tar | docker import - myimage:imported
```

### Explanation

- This is "container filesystem export/import"
- It's different from `save/load`

---

## Scenario 19: Standard Process for Offline Image Migration

### Export

```bash
docker save -o myapp.tar myapp:latest
```

### Transfer tar package to target machine

Copy `myapp.tar` to target server, for example through `scp`, file transfer via bastion host, USB drive, internal network file server, etc.

### Import

```bash
docker load -i myapp.tar
```

### Verification

```bash
docker images | grep myapp
```

---

## Four, Operation Scenario Understanding

---

## 1. Why Not Recommend Long-term Dependency on docker commit

`docker commit` can quickly save current container state as image, but it's not suitable for long-term image building.

Reasons:

- Irreversible operation process
- Intransparent modification steps
- Not convenient for code review
- Not convenient for CI/CD automatic building
- Not conducive to team collaboration
- Not conducive to version management

Suitable scenarios:

- Temporarily save troubleshooting state
- Rapidly solidify test environment
- Verify feasibility of temporary modification
- Preserve current state from abnormal container

Unsuitable scenarios:

- Formal production image building
- Long-term version maintenance
- Standardized delivery
- Automated pipeline building

Production environment recommends using Dockerfile:

```bash
docker build -t myapp:latest .
```

---

## 2. save/load is more suitable for image migration

If the goal is to migrate images, prioritize using:

```bash
docker save -o myapp.tar myapp:latest
```

Then import on target machine:

```bash
docker load -i myapp.tar
```

Reasons:

- Preserve image layers
- Preserve tags
- Preserve image metadata
- Suitable for offline environment
- Suitable for image backup and migration

---

## 3. export/import is more like container filesystem migration

`docker export` exports not the complete image, but a snapshot of a container's filesystem.

Example:

```bash
docker export ContainersID -o container.tar
```

Import:

```bash
cat container.tar | docker import - myimage:imported
```

This method does not preserve complete image layer history and many metadata, so it's not suitable as standard image migration method.

---

## 4. Common Offline Environment Image Migration Process

Common process:

```text
Cyber machine. pull Mirror
→ docker save Export tar Package
→ Transfer tar Packed offline.
→ docker load Import Mirror
→ docker images Authentication
→ docker run or K8s Use this mirror
```

Example:

```bash
docker pull nginx:latest
```

```bash
docker save -o nginx.tar nginx:latest
```

```bash
docker load -i nginx.tar
```

```bash
docker images | grep nginx
```

---

## Five, Common Notes

---

## 1. Image tag should be clear

Not recommended to use long-term:

```bash
nginx:latest
```

Better to name according to environment and version, for example:

```bash
myapp:v1.0.0
```

```bash
myapp:20260425
```

```bash
myapp:prod-20260425
```

In CI/CD scenarios, you can also use branch name, commitID, pipelineID to generate image tag.

---

## 2. Confirm if image is used by containers before deletion

Delete image:

```bash
docker rmi MirrorID
```

If image is used by containers, deletion may fail.

First check containers:

```bash
docker ps -a
```

Confirm it's no longer in use before deletion.

---

## 3. Multiple images can be exported together for offline migration

If offline environment needs multiple images, export them together:

```bash
docker save -o app-bundle.tar nginx:latest redis:7 alpine:latest
```

Import:

```bash
docker load -i app-bundle.tar
```

This reduces transfer times.

---

## 4. Do not confuse image and container filesystem

Image migration should prioritize:

```bash
docker save
```

```bash
docker load
```

Container filesystem export use: /think

```bash
docker export
```

```bash
docker import
```

One-sentence understanding:

```text
save/load = Mirror Level Migration
export/import = Container file system level migration
```

---

## Six. Summary of Common Commands in This Article

---

## Docker Image Basics

View images:

```bash
docker images
```

Pull images:

```bash
docker pull nginx
```

Delete images:

```bash
docker rmi MirrorID
```

---

## Docker Image Commit

Package container as image:

```bash
docker commit ContainersID my-nginx:v1
```

Add commit message to image:

```bash
docker commit -m "install curl and modify config" ContainersID my-nginx:v2
```

---

## Docker Image Export

Export single image:

```bash
docker save -o nginx.tar nginx:latest
```

Export multiple images:

```bash
docker save -o app-bundle.tar nginx:latest redis:7 alpine:latest
```

---

## Docker Image Import

Import image tar package:

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

## Docker Container Filesystem Export and Import

Export container filesystem:

```bash
docker export ContainersID -o container.tar
```

Import as image:

```bash
cat container.tar | docker import - myimage:imported
```

---

## Offline Migration Standard Process

Pull images from machine with network:

```bash
docker pull myapp:latest
```

Export images:

```bash
docker save -o myapp.tar myapp:latest
```

Import images on target machine:

```bash
docker load -i myapp.tar
```

Verify images:

```bash
docker images | grep myapp
```

---

## Seven. One-sentence Summary

The core capability of Docker image management is:

Can view images

→ Can pull images

→ Can delete images

→ Can temporarily commit container state

→ Can export images

→ Can import images

→ Can complete image migration in offline environment

Core difference:

```text
docker commit
→ Maintaining the current state of the container into a new mirror, suitable for temporary solidification and suitable for long-term production construction

docker save / docker load
→ Mirror level import export suitable for mirror migration, backup, offline environment delivery

docker export / docker import
→ Container file system level import export, not leaving complete mirror layer and metadata
```

Production recommendation:

```text
Priority for official mirror construction Dockerfile
Temporary containment can be used. docker commit
Offline mirror migration priority docker save / docker load
Don't. docker export / import As a standard mirror migration.
```