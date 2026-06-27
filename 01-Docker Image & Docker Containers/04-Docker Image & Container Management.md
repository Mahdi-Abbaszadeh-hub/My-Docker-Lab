# Docker Image & Container Management

> A complete reference for managing Docker images and containers — save/load, commit, attach, inspect, logging drivers, and more.

---

## Table of Contents

- [Docker Image \& Container Management](#docker-image--container-management)
  - [Table of Contents](#table-of-contents)
  - [Image Management](#image-management)
    - [save \& load](#save--load)
    - [tag](#tag)
    - [commit](#commit)
  - [Container Management](#container-management)
    - [create vs run](#create-vs-run)
    - [rename](#rename)
    - [attach \& detach](#attach--detach)
    - [top](#top)
    - [port](#port)
    - [diff](#diff)
    - [export](#export)
  - [Logging Drivers](#logging-drivers)
    - [Default behavior](#default-behavior)
    - [Configuring the log driver](#configuring-the-log-driver)
    - [Available log drivers](#available-log-drivers)
  - [Summary](#summary)

---

## Image Management

### save & load

These two commands let you **export an image to a file** and **restore it later** — without needing a registry.

Use cases:
- Transferring an image to a server with no internet access
- Archiving a specific image version
- Sharing an image with a colleague directly

```bash
# Save an image to a .tar file
docker image save -o jenkins.tar 038be1654f8f

# Load the image back from the file
docker image load -i jenkins.tar
```

> `-o` = output (write to file) | `-i` = input (read from file)

**Important:** `save`/`load` preserves the full image with all its layers, history, and metadata — use this when transferring images between systems.

> ⚠️ Do not confuse with `export`/`import`. See the [export section](#export) below for the key difference.

---

### tag

You can add or change a name and tag on an existing image **without rebuilding it**. The image ID stays the same — you're just adding a new label pointing to the same image.

```bash
# Assign a name and tag to an image that has none
docker tag 038be1654f8f jenkins:3.0.0

# You can also use the full image ID
docker tag 038be1654f8f8c28d52535835a620c37cafd7673737ae5d6e9583484ff35da12 jenkins:3.0.0
```

This is useful when you built an image without `-t` and want to name it afterward.

---

### commit

`docker commit` creates a **new image from the current state of a running or stopped container**.

Normally the flow is: `image → container`. With `commit`, you reverse it: `container → new image`.

```bash
docker commit <container_id>
```

**When would you use this?**
Imagine you ran Jenkins, configured it, installed Python and Nginx inside the container, and now you want to save that state as a reusable image. `commit` captures the container's writable layer and freezes it into a new image.

```bash
# Commit with a message and author
docker commit \
  --author "Mahdi Abbaszadeh" \
  --message "Added Python and Nginx to Jenkins" \
  eca20d186bbb \
  jenkins-custom:1.0
```

> **Note:** The original image is untouched (it's read-only). `commit` produces a brand new image with a new ID. The committed image will be larger than the original because it includes all the changes made inside the container.

> **Best practice:** In production, always prefer building from a Dockerfile over using `commit`. Dockerfiles are reproducible and version-controlled; committed images are opaque and harder to maintain.

---

## Container Management

### create vs run

```bash
# create — builds the container but does NOT start it
docker create 038be1654f8f

# run — creates AND starts the container immediately
docker run ubuntu:latest
```

| Command | Creates container | Starts container |
|---|---|---|
| `docker create` | ✅ | ❌ |
| `docker run` | ✅ | ✅ |

`create` is useful when you want to prepare a container in advance and start it later with `docker start`.

You can also explore all available flags:

```bash
docker container --help
```

---

### rename

Docker assigns a random name to every container (e.g., `pensive_jepsen`). You can rename it at any time:

```bash
docker rename e6e0b23a93c5 jenkins_1
```

You can also set a name at creation time using `--name`:

```bash
docker run -d --name jenkins_1 jenkins:3.0.0
```

---

### attach & detach

`docker attach` connects your terminal to the **main process** of a running container. You'll see its live output.

```bash
docker attach eca20d186bbb
```

⚠️ **Critical warning:** If you press `Ctrl+C` while attached, it sends a **SIGTERM/SIGINT signal to the container's main process**, which stops the container entirely.

To detach **without stopping** the container, use:

```
Ctrl + P, then Ctrl + Q
```

This key combination sends a "detach" signal instead of a stop signal — the container keeps running in the background.

**Workflow example:**

```bash
# Start a container in interactive + detached mode
docker run -dit jenkins:3.0.0

# Later, attach to see its output
docker attach fffb8d2d8688

# Detach safely without stopping it
# Press: Ctrl+P, then Ctrl+Q
```

---

### top

Shows the **running processes inside a container**, similar to the Linux `top` or `ps` command.

```bash
docker top fffb8d2d8688
```

Example output:
```
UID    PID    PPID   CMD
root   1234   1      java -jar jenkins.war
```

Useful for verifying that the expected process is actually running inside the container.

---

### port

Shows the **port mappings** for a container — which container ports are mapped to which host ports.

```bash
docker port 50e2d70259d9
```

Example output:
```
8080/tcp -> 0.0.0.0:8081
```

This tells you: port `8080` inside the container is accessible via port `8081` on your host.

---

### diff

Shows **what files have changed** in a container's filesystem compared to its original image.

```bash
docker diff 50e2d70259d9
```

Example output:
```
A /data/app/jenkins.war     ← A = Added
C /etc/hosts                ← C = Changed
D /tmp/tempfile             ← D = Deleted
```

Useful for debugging or understanding what a container has modified since it started.

---

### export

Exports the **entire filesystem** of a container to a `.tar` file.

```bash
docker export -o jenkins2.tar 50e2d70259d9
```

**`save` vs `export` — what's the difference?**

| | `docker image save` | `docker export` |
|---|---|---|
| Source | An **image** | A **container** |
| Includes image layers & history | ✅ | ❌ |
| Includes container filesystem changes | ❌ | ✅ |
| Use case | Transfer full image with metadata | Snapshot a container's current filesystem |
| Restore with | `docker image load` | `docker import` |

> ⚠️ **Important:** When you `import` an exported container, you get a flat image with no history or layers. Always prefer `save`/`load` for transferring images between systems.

---

## Logging Drivers

### Default behavior

When you run `docker logs <container_id>`, Docker reads from the container's log file stored on your host at:

```
/var/lib/docker/containers/<container_id>/<container_id>-json.log
```

> ⚠️ You cannot `cd` into this path with `sudo cd` — `cd` is a shell built-in, not an external command. The correct way:
> ```bash
> sudo ls /var/lib/docker/containers/
> # Then navigate as root or use sudo with cat/ls
> sudo cat /var/lib/docker/containers/<id>/<id>-json.log
> ```

By default, Docker uses the **`json-file`** logging driver. Every log line from the container is saved as a JSON object containing:
- `log` — the actual log message
- `stream` — `stdout` or `stderr`
- `time` — the timestamp

```json
{"log":"Jenkins is ready\n","stream":"stdout","time":"2026-01-01T00:00:00Z"}
```

---

### Configuring the log driver

The log driver is configured via the Docker daemon configuration file:

```
/etc/docker/daemon.json
```

This file does **not exist by default**. You create it manually.

**Example configuration — json-file with rotation:**

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

What this does:
- `max-size: 10m` — each log file is capped at 10 MB
- `max-file: 3` — keep a maximum of 3 rotated log files (30 MB total per container)
- When the limit is reached, the oldest log file is deleted automatically

**Why is this important?**
Without log rotation, the log file grows indefinitely. In production, this can fill up your entire disk. Always configure log rotation.

After saving `daemon.json`, restart Docker for the changes to take effect:

```bash
sudo systemctl restart docker.service
```

---

### Available log drivers

Docker supports multiple logging backends. You choose based on your infrastructure:

| Driver | Description | When to use |
|---|---|---|
| `json-file` | Default. Stores logs as JSON on the local filesystem | Development, simple setups |
| `local` | Optimized local storage with better performance and automatic rotation | Recommended for production on a single host |
| `none` | Disables all logging | When you never need logs |
| `syslog` | Sends logs to the system's syslog daemon | Traditional Linux server setups |
| `journald` | Sends logs to systemd's journal | Systemd-based systems |
| `gelf` | Sends logs to a Graylog or Logstash endpoint | Centralized logging stacks |
| `fluentd` | Sends logs to a Fluentd aggregator | Kubernetes and cloud-native setups |
| `awslogs` | Sends logs directly to AWS CloudWatch | AWS deployments |

**In practice (DevOps pipelines):**

Most production environments use a **log aggregation stack** such as:

- **ELK Stack** — Elasticsearch + Logstash + Kibana
- **EFK Stack** — Elasticsearch + Fluentd + Kibana

The typical setup:
1. Use the `local` driver to store logs on the filesystem
2. Run a log shipper (like Filebeat or Fluentd) that reads those files
3. The shipper sends logs to Elasticsearch
4. Kibana lets you search, filter, and visualize them

```
Container → local log file → Filebeat → Elasticsearch → Kibana
```

> This approach is preferred over sending logs directly to Logstash because if the network connection drops, the local file still preserves the logs until the connection is restored.

---

## Summary

| Command | What it does |
|---|---|
| `docker image save -o file.tar <image>` | Export image to a file (preserves layers) |
| `docker image load -i file.tar` | Restore image from file |
| `docker tag <id> name:tag` | Add/rename a tag on an image |
| `docker commit <container>` | Create a new image from a container's state |
| `docker create <image>` | Create a container without starting it |
| `docker rename <id> new_name` | Rename a container |
| `docker attach <id>` | Connect terminal to container (Ctrl+P+Q to detach safely) |
| `docker top <id>` | Show running processes inside a container |
| `docker port <id>` | Show port mappings |
| `docker diff <id>` | Show filesystem changes since container started |
| `docker export -o file.tar <container>` | Export container filesystem (no layers) |
| `docker logs <id>` | View container logs |