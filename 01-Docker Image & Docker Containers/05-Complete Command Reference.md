# Docker CLI — Complete Command Reference

> A comprehensive reference for all essential Docker commands — registry operations, image management, and container lifecycle.

---

## Table of Contents

- [Docker CLI — Complete Command Reference](#docker-cli--complete-command-reference)
  - [Table of Contents](#table-of-contents)
  - [Registry Commands](#registry-commands)
    - [login \& logout](#login--logout)
    - [search](#search)
    - [pull \& push](#pull--push)
  - [Image Commands](#image-commands)
    - [images \& images -q](#images--images--q)
    - [image history](#image-history)
    - [image inspect](#image-inspect)
    - [image tag](#image-tag)
    - [image prune](#image-prune)
    - [image rm](#image-rm)
  - [Container Commands](#container-commands)
    - [create vs run](#create-vs-run)
    - [ps \& ps -a](#ps--ps--a)
    - [rename](#rename)
    - [stop \& kill](#stop--kill)
    - [start \& restart](#start--restart)
    - [pause \& unpause](#pause--unpause)
    - [logs](#logs)
    - [cp](#cp)
    - [exec](#exec)
    - [wait](#wait)
  - [Quick Reference Summary](#quick-reference-summary)
    - [Registry](#registry)
    - [Images](#images)
    - [Containers](#containers)

---

## Registry Commands

### login & logout

To push your own images to Docker Hub, you must log in first.

```bash
# Log in to Docker Hub
docker login

# Log out (clears saved credentials)
docker logout
```

When you log in, Docker saves your credentials locally so you don't need to re-enter them every time. `docker logout` clears those saved credentials.

> ⚠️ If you are in Iran, you need to bypass sanctions restrictions before `docker login` or `docker pull` will work.

---

### search

Search for images on Docker Hub directly from the terminal — useful when you're working on a server without a browser.

```bash
docker search nginx
```

The output shows:
- Image name
- Description
- Star count (higher = more popular and trusted)
- Whether it's an official image

---

### pull & push

```bash
# Download an image from a registry
docker pull nginx

# Upload your own image to a registry
docker push yourusername/myapp:1.0.0
```

`push` requires you to be logged in and the image name must include your Docker Hub username as a prefix.

---

## Image Commands

### images & images -q

```bash
# List all local images with full details
docker images

# List only image IDs (quiet mode)
docker images -q
```

`-q` (quiet) is extremely useful in shell scripts. For example, to delete all images at once:

```bash
# Delete all images in one command
docker image rm $(docker images -q)
```

---

### image history

Shows the **build history** of an image — every instruction that was executed to create it, when it was created, and how much each layer added to the image size.

```bash
docker image history <image_id>
```

---

### image inspect

Returns a large JSON object with **all metadata about an image**:

```bash
docker image inspect <image_id>
```

What you can find inside:
- `RepoTags` — the name and tag of the image
- `Author` — who built it (from `MAINTAINER` or `LABEL`)
- `Config.Cmd` — the default `CMD`
- `Config.ExposedPorts` — which ports are exposed
- `Config.Labels` — all labels
- `Architecture` — the CPU architecture
- `Size` — exact size in bytes

> **When is this useful?** When you have an image with no documentation and need to understand what it does, how it's configured, and what ports it uses — `inspect` tells you everything.

---

### image tag

Adds a new name and tag to an existing image **without modifying it**. The image ID stays the same — `tag` just creates a new pointer to the same image.

```bash
# Syntax: docker image tag <source_image:tag> <new_name:new_tag>
docker image tag apache:1.0.0 apache2:1.0.2
```

**Important — how deletion works with tags:**

When you tag an image, the new tag becomes a **reference** to the original image. This means:

- You cannot delete the original image while the tag still references it
- You must delete the tag (the referencing image) first, then delete the original

```bash
# This will FAIL if apache2:1.0.2 still references apache:1.0.0
docker image rm apache:1.0.0

# Correct order:
docker image rm apache2:1.0.2   # remove the tag first
docker image rm apache:1.0.0   # now you can remove the original
```

---

### image prune

Removes all **unused images** — images that have no container (running or stopped) built from them.

```bash
# Remove unused images (asks for confirmation)
docker image prune

# Remove ALL unused images without confirmation
docker image prune -a
```

> Use with care. `-a` will remove every image that no container is currently using.

---

### image rm

Remove a specific image by ID or name:

```bash
docker image rm <image_id>
# or
docker image rm apache:1.0.0
```

> You cannot remove an image if a container (even a stopped one) was built from it. Remove the container first with `docker rm`.

---

## Container Commands

### create vs run

```bash
# Create a container WITHOUT starting it
docker create <image_id>

# Create AND start a container immediately
docker run <image>
```

| Command | Creates | Starts |
|---|---|---|
| `docker create` | ✅ | ❌ |
| `docker run` | ✅ | ✅ |

Use `docker create` when you want to prepare a container in advance and start it later with `docker start`.

---

### ps & ps -a

```bash
# List currently RUNNING containers
docker ps

# List ALL containers (running + stopped)
docker ps -a
```

Stopped containers still exist on disk until you explicitly remove them with `docker rm`.

---

### rename

```bash
docker rename <container_id> <new_name>
```

Docker assigns random names (e.g., `pensive_jepsen`) by default. You can rename at any time, or set a name at creation:

```bash
docker run -d --name my_jenkins jenkins:3.0.0
```

---

### stop & kill

Both commands stop a container, but they work very differently:

```bash
# Graceful stop (recommended)
docker stop <container_id>

# Force stop — use only as a last resort
docker kill <container_id>
```

**`docker stop` — graceful shutdown:**
Sends a `SIGTERM` signal to the container's main process. The process has time to finish what it's doing — close open files, release ports, flush data — then exits cleanly. This is the correct way.

**`docker kill` — force stop:**
Sends a `SIGKILL` signal, which immediately terminates the process with no cleanup. Data may be lost or left in an inconsistent state.

> In normal operation, you should never need `docker kill`. If `docker stop` is taking too long, investigate why the process isn't shutting down cleanly rather than force-killing it.

---

### start & restart

```bash
# Start a stopped container
docker start <container_id>

# Restart a running container (stop + start in one command)
docker restart <container_id>
```

`docker restart` is useful when a container is misbehaving and needs a fresh start without you having to manually stop and start it.

---

### pause & unpause

```bash
# Freeze all processes inside the container
docker pause <container_id>

# Resume the container from where it was paused
docker unpause <container_id>
```

`pause` freezes the container completely — no CPU is used, no processes advance. The container is still "running" from Docker's perspective (it shows as `Paused` in `docker ps`).

`unpause` resumes exactly from the point it was paused — like unpausing a video.

---

### logs

View the output (stdout/stderr) from a container's main process:

```bash
# View all logs
docker logs <container_id>

# Follow logs in real time (like tail -f)
docker logs -f <container_id>

# Show last N lines
docker logs --tail 100 <container_id>
```

> If your container is running in detached mode (`-d`) and you want to see what it's doing, `docker logs` is how you check.

---

### cp

Copy files between your local machine and a container — in either direction.

```bash
# LOCAL → CONTAINER
docker cp ./myfile.txt <container_id>:/data/app/

# CONTAINER → LOCAL
docker cp <container_id>:/data/app/jenkins.war .
```

**Syntax pattern:**

```
docker cp <source> <destination>

# source or destination can be:
#   - a local path:              ./file.txt  or  .
#   - a container path:          <container_id>:/path/to/file
```

The `.` at the end means "copy here, into the current directory".

> The container does **not** need to be running for `docker cp` to work — it works on stopped containers too.

---

### exec

Run a command inside a **running** container.

```bash
# Open an interactive shell inside the container
docker exec -it <container_id> sh

# Run a single command and return
docker exec <container_id> ls /data/app
```

Common flags:
- `-i` — interactive (keeps stdin open)
- `-t` — allocates a pseudo-TTY (makes it look like a real terminal)

Combined `-it` is what you use when you want to "enter" a container and work inside it interactively.

Once inside, type `exit` to leave (this does NOT stop the container — only the shell session ends).

---

### wait

Blocks your terminal until one or more containers stop, then prints their exit codes.

```bash
docker wait <container_id>
```

Useful in scripts where you need to wait for a container to finish before doing something else:

```bash
docker run --name job my-batch-image
docker wait job
echo "Job finished with exit code: $?"
```

---

## Quick Reference Summary

### Registry

| Command | Description |
|---|---|
| `docker login` | Log in to Docker Hub |
| `docker logout` | Log out and clear credentials |
| `docker search <term>` | Search for images on Docker Hub |
| `docker pull <image>` | Download an image |
| `docker push <image>` | Upload an image to a registry |

### Images

| Command | Description |
|---|---|
| `docker images` | List all local images |
| `docker images -q` | List image IDs only |
| `docker image history <id>` | Show build history of an image |
| `docker image inspect <id>` | Show full metadata of an image |
| `docker image tag <src> <dest>` | Add a new name/tag to an image |
| `docker image prune` | Remove unused images |
| `docker image rm <id>` | Remove a specific image |

### Containers

| Command | Description |
|---|---|
| `docker create <image>` | Create container without starting |
| `docker run <image>` | Create and start container |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker rename <id> <name>` | Rename a container |
| `docker stop <id>` | Graceful shutdown |
| `docker kill <id>` | Force stop (last resort only) |
| `docker start <id>` | Start a stopped container |
| `docker restart <id>` | Restart a running container |
| `docker pause <id>` | Freeze container processes |
| `docker unpause <id>` | Resume a paused container |
| `docker logs <id>` | View container output |
| `docker logs -f <id>` | Follow logs in real time |
| `docker cp <src> <dest>` | Copy files to/from container |
| `docker exec -it <id> sh` | Open shell inside container |
| `docker wait <id>` | Block until container stops |