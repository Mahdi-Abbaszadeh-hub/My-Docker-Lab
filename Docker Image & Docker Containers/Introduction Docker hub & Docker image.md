# Docker Images & Containers

> A beginner-friendly introduction to Docker images and containers — what they are, how they relate, and how to run your first container.

---

## Table of Contents

- [What Is a Docker Image?](#what-is-a-docker-image)
- [The Cookbook Analogy](#the-cookbook-analogy)
- [Where Are Images Stored? (Registries)](#where-are-images-stored-registries)
- [Pulling and Pushing Images](#pulling-and-pushing-images)
- [Image Layers](#image-layers)
- [What Is a Container?](#what-is-a-container)
- [Container Lifecycle](#container-lifecycle)
- [Your First Container](#your-first-container)

---

## What Is a Docker Image?

A **Docker image** is a single, read-only file that contains everything needed to run an application — the code, runtime, libraries, and configuration. Images are stored either:

- **Locally** on your own machine
- **In a public registry** like [Docker Hub](https://hub.docker.com)
- **In a private registry** hosted on your own server or cloud infrastructure

Docker images are the **blueprint** for containers. You don't run an image directly — you use it to create and run containers.

---

## The Cookbook Analogy

Think of it this way:

| Concept | Real-world Analogy |
|---|---|
| Docker Image | A cookbook (read-only, fixed recipe) |
| Container | The actual dish you cook from that cookbook |
| Registry | A library where cookbooks are stored |
| `docker pull` | Borrowing a cookbook from the library |
| `docker push` | Donating your own cookbook to the library |

You cook a meal **from** the cookbook, and you can adjust the dish (add more salt, less pepper) — but the cookbook itself stays unchanged. If you want a different recipe, you write a new cookbook — meaning you create a new Docker image.

Multiple people can cook from the same cookbook independently. Each person's dish is separate and unique — just like multiple containers from the same image.

---

## Where Are Images Stored? (Registries)

A **registry** is a storage system for Docker images. There are two types:

### Public Registries

The most well-known public registry is **Docker Hub** ([hub.docker.com](https://hub.docker.com)). It hosts thousands of ready-to-use images from both official sources and the community.

Other popular registries include:

- **GCR** — Google Container Registry
- **ECR** — Amazon Elastic Container Registry
- **Quay.io** — Red Hat's container registry

### Private Registries

A private registry is set up and hosted by a team or company on their own server. Only authorized users can access the images stored there.

> **Tip:** The more downloads an image has on Docker Hub, the more trusted and widely used it generally is.

---

## Pulling and Pushing Images

### Pulling an Image

To download an image from a registry to your local machine:

```bash
docker pull nginx
```

If no tag (version) is specified, Docker automatically pulls the `latest` tag.

### Pushing an Image

To upload your own image to a registry so others can use it:

```bash
docker push your-username/your-image-name
```

### Viewing Local Images

To see all images currently stored on your machine:

```bash
docker images
```

Example output:

```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         latest    a6bd71f48f68   2 days ago     187MB
ubuntu        latest    bf3dc08bfed0   3 weeks ago    77.8MB
hello-world   latest    d2c94e258dcb   1 year ago     13.3kB
```

> Each image has a unique **Image ID**. Docker shows only the first few characters of the full ID for readability.

---

## Image Layers

A Docker image is not a single monolithic file — it is made up of **multiple layers**, each representing a specific instruction (e.g., installing the OS, adding a web server, copying application files).

**Example layers for an Nginx image:**

```
Layer 1: Ubuntu base OS
Layer 2: Install Nginx
Layer 3: Copy application files
```

This layered design has a major benefit: **if you only change one layer, the other layers stay cached and don't need to be re-downloaded or rebuilt.** For example, updating the Nginx version only rebuilds Layer 2 — not the entire image.

---

## What Is a Container?

A **container** is a **running instance** of a Docker image. While an image is static and read-only, a container is live — it has a process running inside it.

Key characteristics of a container:

- Created **from** a Docker image
- Has its own **unique ID** (different every time you create one from the same image)
- Has a **writable layer** on top of the image's read-only layers (changes are temporary)
- Is **isolated** from other containers

Multiple containers can be created from the same image. Each one is completely independent.

---

## Container Lifecycle

Unlike images, containers have a **lifecycle** — they can be started, paused, stopped, and restarted:

```
Created → Running → Paused → Running → Stopped → Removed
```

| State | Description |
|---|---|
| **Running** | The container is actively executing |
| **Paused** | Execution is temporarily frozen |
| **Stopped** | The container has exited but still exists |
| **Removed** | The container is permanently deleted |

> When a container is removed, any data written inside it (in the writable layer) is lost — unless you use **volumes** to persist data.

---

## Your First Container

### Step 1 — Pull an image

```bash
docker pull hello-world
```

### Step 2 — Run the container

```bash
docker run hello-world
```

If the image is already pulled, `docker run` will use the local copy. If not, it automatically pulls it first and then runs it.

**Expected output:**

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

This confirms that Docker successfully pulled the image and ran a container from it. 

### Step 3 — View your images

```bash
docker images
```

You should now see `hello-world` listed with its tag, image ID, and size.

---

## Summary

| Concept | Key Point |
|---|---|
| **Image** | Read-only blueprint; stored on disk; built from layers |
| **Container** | Running instance of an image; has a lifecycle; has a unique ID |
| **Registry** | Storage system for images (public or private) |
| **`docker pull`** | Downloads an image from a registry |
| **`docker push`** | Uploads an image to a registry |
| **`docker run`** | Creates and starts a container from an image |
| **`docker images`** | Lists all locally stored images |