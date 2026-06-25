# Building and Running Your Own Docker Image

> In this section, we move beyond pulling pre-built images and learn how to build our own Docker image using a `Dockerfile` — then run, manage, and connect to a container from it.

---

## Table of Contents

- [Building and Running Your Own Docker Image](#building-and-running-your-own-docker-image)
  - [Table of Contents](#table-of-contents)
  - [What Is a Dockerfile?](#what-is-a-dockerfile)
  - [Our Example Dockerfile](#our-example-dockerfile)
  - [Dockerfile Instructions Explained](#dockerfile-instructions-explained)
    - [`FROM`](#from)
    - [`MAINTAINER`](#maintainer)
    - [`RUN`](#run)
    - [`EXPOSE`](#expose)
    - [`CMD`](#cmd)
  - [Building the Image](#building-the-image)
  - [Running the Container](#running-the-container)
    - [Basic run (attached mode)](#basic-run-attached-mode)
    - [Detached mode](#detached-mode)
  - [Managing Containers](#managing-containers)
    - [Stop a container](#stop-a-container)
    - [Start a stopped container](#start-a-stopped-container)
    - [List all containers (including stopped ones)](#list-all-containers-including-stopped-ones)
  - [Port Mapping — Connecting to the Container](#port-mapping--connecting-to-the-container)
  - [Cleaning Up](#cleaning-up)
    - [Remove a container](#remove-a-container)
    - [Remove an image](#remove-an-image)
  - [Summary of Commands](#summary-of-commands)

---

## What Is a Dockerfile?

A **Dockerfile** is a plain text file named exactly `Dockerfile` (no extension). Inside it, you write a series of instructions — one per line — that Docker executes top to bottom in order to build a Docker image.

Key points:
- Keywords written in **UPPERCASE** (`FROM`, `RUN`, `EXPOSE`, etc.) are Docker's own reserved instructions
- Each instruction adds a new **layer** to the resulting image
- The Dockerfile is **read-only** — you don't modify it at runtime; you build a new image if you need changes

---

## Our Example Dockerfile

```dockerfile
FROM ubuntu:latest
MAINTAINER Mahdi Abbaszadeh
RUN apt-get update && apt-get install -y apache2 && apt-get clean
RUN echo "Hello, World!" > /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

This Dockerfile builds an image that:
1. Starts from an Ubuntu base
2. Installs the Apache web server
3. Creates a simple HTML page
4. Exposes port 80 for HTTP traffic
5. Starts Apache automatically when a container is launched

---

## Dockerfile Instructions Explained

### `FROM`

```dockerfile
FROM ubuntu:latest
```

Specifies the **base image**. Docker pulls this image from a registry and uses it as the starting point. Every subsequent instruction builds on top of it.

You can use any valid image as a base — `ubuntu`, `nginx`, `node`, `python`, etc.

> Think of it like choosing which cookbook you'll adapt. You take an existing recipe and add your own modifications on top.

---

### `MAINTAINER`

```dockerfile
MAINTAINER Mahdi Abbaszadeh
```

Metadata only — records who created this image. Useful when sharing images with a team or publishing them publicly.

---

### `RUN`

```dockerfile
RUN apt-get update && apt-get install -y apache2 && apt-get clean
```

Executes a shell command **during the build process**. You can write any valid Linux command after `RUN`.

In this case:
- `apt-get update` — refreshes the Ubuntu package repository
- `apt-get install -y apache2` — installs the Apache web server
- `apt-get clean` — removes cached package files to keep the image size small

> **Best practice:** Always clean up after installing packages. Smaller images are faster to pull and deploy.

```dockerfile
RUN echo "Hello, World!" > /var/www/html/index.html
```

Creates a simple HTML file at Apache's default web root. When Apache serves this file, the browser displays "Hello, World!".

---

### `EXPOSE`

```dockerfile
EXPOSE 80
```

Declares that the container will listen on port **80** (the default HTTP port). Without this, no external traffic can reach the container's web server.

> `EXPOSE` alone doesn't make the port accessible from your local machine — you still need to **map** it when running the container (covered below).

---

### `CMD`

```dockerfile
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

Defines the **default command** that runs when a container is started from this image. Here it launches Apache in the foreground, keeping the container alive.

> Every image has a `CMD`. When you ran `docker run hello-world` earlier, that image's `CMD` was what printed the welcome message.

---

## Building the Image

Navigate to the directory containing your `Dockerfile`, then run:

```bash
docker build .
```

The `.` tells Docker to look for a `Dockerfile` in the current directory (`pwd`).

To give the image a **name and tag** at build time:

```bash
docker build -t apache:1.0.0 .
```

- `-t` sets the name and tag in the format `name:tag`
- `apache` is the image name
- `1.0.0` is the version tag

Verify the image was created:

```bash
docker images
```

If you built without `-t`, the image will have no name or tag but will still have a unique **Image ID**.

To build using a Dockerfile with a different name or in a different path:

```bash
docker build -f /path/to/MyDockerfile .
```

---

## Running the Container

### Basic run (attached mode)

```bash
docker run apache:1.0.0
```

This starts the container and **attaches** your terminal to its logs. You'll see Apache output but can't do anything else in that terminal window.

### Detached mode

```bash
docker run -d apache:1.0.0
```

The `-d` flag runs the container in the **background** (detached). Docker prints the container ID and returns control to your terminal immediately.

Check running containers:

```bash
docker ps
```

Example output:

```
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS     NAMES
a3f2c1d9e8b7   apache:1.0.0  "/usr/sbin/apache2ct…"   5 seconds ago   Up 4 seconds   80/tcp    hopeful_morse
```

---

## Managing Containers

### Stop a container

```bash
docker stop <container_id_or_name>
```

### Start a stopped container

```bash
docker start <container_id_or_name>
```

### List all containers (including stopped ones)

```bash
docker ps -a
```

> Stopped containers still exist on disk. Use `docker start` to bring them back without creating a new one.

---

## Port Mapping — Connecting to the Container

Even though we used `EXPOSE 80` in the Dockerfile, the port is only accessible **inside** Docker's internal network. To reach it from your local machine (or a browser), you must **map** a port on your host to the container's port.

This is done with the `-p` flag:

```bash
docker run -d -p 8000:80 apache:1.0.0
```

Format: `-p <host_port>:<container_port>`

| Side | Value | Meaning |
|---|---|---|
| Left | `8000` | Port on your local machine |
| Right | `80` | Port inside the container |

Now open your browser and visit:

```
http://127.0.0.1:8000
```

You should see the **"Hello, World!"** page served by Apache running inside the container.

> You can use any available port on the left side — `8080`, `8888`, `9000`, etc. The right side must match what the container `EXPOSE`s.

---

## Cleaning Up

### Remove a container

```bash
docker rm <container_id>
```

> You must **stop** a container before removing it.

### Remove an image

```bash
docker image rm <image_id_or_name>
```

> You cannot remove an image if a container (even a stopped one) was built from it. Remove the container first.

---

## Summary of Commands

| Command | Description |
|---|---|
| `docker build .` | Build an image from the Dockerfile in the current directory |
| `docker build -t name:tag .` | Build and assign a name and tag to the image |
| `docker run <image>` | Create and start a container (attached mode) |
| `docker run -d <image>` | Run container in background (detached) |
| `docker run -d -p 8000:80 <image>` | Run detached with port mapping |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (running + stopped) |
| `docker stop <id>` | Stop a running container |
| `docker start <id>` | Restart a stopped container |
| `docker rm <id>` | Delete a stopped container |
| `docker image rm <id>` | Delete an image |