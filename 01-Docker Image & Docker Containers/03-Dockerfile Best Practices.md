# Advanced Dockerfile Instructions & Best Practices

> In this section we explore advanced Dockerfile instructions, build a real Jenkins image, learn the difference between `ENTRYPOINT` and `CMD`, and cover Docker image best practices used in production.

---

## Table of Contents

- [New Instructions Overview](#new-instructions-overview)
- [The Jenkins Dockerfile](#the-jenkins-dockerfile)
- [Instructions Explained](#instructions-explained)
- [ENTRYPOINT vs CMD](#entrypoint-vs-cmd)
- [Going Inside a Container](#going-inside-a-container)
- [docker logs](#docker-logs)
- [Multi-Stage Builds](#multi-stage-builds)
- [Best Practices](#best-practices)

---

## New Instructions Overview

| Instruction | Purpose |
|---|---|
| `FROM` | Set the base image |
| `LABEL` | Add metadata (replaces deprecated `MAINTAINER`) |
| `ENV` | Define environment variables |
| `ARG` | Define build-time variables |
| `RUN` | Execute a shell command during build |
| `ADD` | Copy files (supports URLs and auto-extract archives) |
| `COPY` | Copy files (simple, local only) |
| `WORKDIR` | Set the default working directory inside the container |
| `EXPOSE` | Declare the port the container listens on |
| `USER` | Set which user runs the container (security best practice) |
| `ENTRYPOINT` | Define the executable that always runs |
| `CMD` | Provide default arguments to `ENTRYPOINT` |

---

## The Jenkins Dockerfile

```dockerfile
FROM openjdk:17-jdk-slim
LABEL maintainer="Mahdi Abbaszadeh" env="production"
ENV APPAREA=/data/app
RUN mkdir -p $APPAREA
ADD jenkins.war $APPAREA
WORKDIR $APPAREA
EXPOSE 8080
ENTRYPOINT ["java", "-jar"]
CMD ["jenkins.war"]
```

> Note: `MAINTAINER` is deprecated. Use `LABEL maintainer="..."` instead.

Build and run:

```bash
docker build -f Dockerfile -t jenkins:2.3.0 .
docker run -d -p 8081:8080 jenkins:2.3.0
```

---

## Instructions Explained

### `LABEL`

```dockerfile
LABEL maintainer="Mahdi Abbaszadeh" env="production"
```

Adds metadata to the image. Anyone who inspects the image later can read this information. Labels are optional but considered a best practice for organization.

---

### `ENV`

```dockerfile
ENV APPAREA=/data/app
```

Defines an **environment variable** that is available both during the build process and at runtime inside the container. You can reference it anywhere in the Dockerfile using `$APPAREA`.

---

### `ARG`

```dockerfile
ARG APPAREA=/data/app
```

Similar to `ENV`, but only available **during the build process** — not at runtime. Useful for values that only matter while building the image (e.g., version numbers, build flags).

| | `ENV` | `ARG` |
|---|---|---|
| Available at build time | ✅ | ✅ |
| Available at runtime | ✅ | ❌ |

---

### `ADD` vs `COPY`

Both copy files into the image, but they differ in capability:

| Feature | `COPY` | `ADD` |
|---|---|---|
| Copy local files | ✅ | ✅ |
| Auto-extract `.tar` archives | ❌ | ✅ |
| Download from a URL | ❌ | ✅ |

**Rule of thumb:** Use `COPY` for simple file copying. Use `ADD` only when you need the extra features (extract or remote URL).

```dockerfile
ADD jenkins.war $APPAREA
```

This copies `jenkins.war` from the local directory (where the Dockerfile is) into `$APPAREA` inside the image.

---

### `WORKDIR`

```dockerfile
WORKDIR $APPAREA
```

Sets the default working directory inside the container. All subsequent commands (`RUN`, `CMD`, `ENTRYPOINT`) will execute from this path. It also becomes the directory you land in when you enter the container interactively.

> Think of it like running `cd /data/app` — but permanently set for the container.

---

### `USER`

```dockerfile
USER mahdi
```

By default, containers run as `root`, which is a security risk. Using `USER` switches to a less privileged user. Many companies **require** this in production environments.

```dockerfile
RUN useradd -m mahdi
USER mahdi
```

---

## ENTRYPOINT vs CMD

Both define what runs when a container starts — but they work differently:

| | `ENTRYPOINT` | `CMD` |
|---|---|---|
| Role | Defines the **executable** | Provides **default arguments** |
| Overridable at runtime | Harder to override | Easily overridden |
| Used together | Sets the command | Sets the arguments |

### Using them together (recommended)

```dockerfile
ENTRYPOINT ["java", "-jar"]
CMD ["jenkins.war"]
```

When the container starts, Docker combines them:

```bash
java -jar jenkins.war
```

This pattern is clean and flexible — you can override just the argument (`CMD`) without changing the core executable (`ENTRYPOINT`).

### Without ENTRYPOINT

If you only use `CMD`:

```dockerfile
CMD ["java", "-jar", "jenkins.war"]
```

Docker uses a default `ENTRYPOINT` of `/bin/sh -c` and passes the `CMD` as arguments. This works but is less explicit and flexible.

---

## Going Inside a Container

To enter a running container and explore its filesystem:

```bash
docker exec -it <container_id> bash
```

- `-i` — interactive mode
- `-t` — allocate a terminal
- `bash` — the shell to open

Once inside, you can navigate the filesystem normally. Your starting directory will be whatever `WORKDIR` was set to.

To exit:

```bash
exit
```

---

## docker logs

When running a container in detached mode (`-d`), you don't see its output. To view the logs:

```bash
docker logs <container_id_or_name>
```

This is especially useful for apps like Jenkins that print important information (like the initial admin password) to stdout on first run.

---

## Multi-Stage Builds

A **multi-stage build** uses multiple `FROM` instructions in a single Dockerfile. Each `FROM` starts a new **stage**.

### Why use it?

Some tools are only needed during the build process (compilers, build tools) but not in the final image. Multi-stage builds let you:
- Build in one stage (with all the heavy tools)
- Copy only the final artifact into a clean, lightweight final stage

### Example (Go application)

```dockerfile
# Stage 1: Build
FROM golang:latest AS builder
WORKDIR /src
RUN echo 'package main\nimport "fmt"\nfunc main() { fmt.Println("Hello!") }' > main.go
RUN go build -o app main.go

# Stage 2: Run
FROM alpine:latest
COPY --from=builder /src/app /app
CMD ["/app"]
```

- **Stage 1** (`builder`): Uses the full Go image to compile the code
- **Stage 2**: Uses a tiny Alpine image and copies only the compiled binary from Stage 1

The final image contains no Go compiler — just the binary. This dramatically reduces image size.

---

## Best Practices

### 1. Use a `.dockerignore` file

Like `.gitignore`, this file tells Docker which files to **exclude** from the build context. This improves build speed and avoids accidentally copying sensitive files into the image.

```
# .dockerignore
*.log
.git
docs/
node_modules/
```

### 2. Use official base images

Always prefer official, well-maintained images from Docker Hub. They are more secure, regularly updated, and widely trusted.

✅ `FROM openjdk:17-jdk-slim`
❌ `FROM some-random-user/java`

### 3. Minimize the number of layers

Each `RUN` instruction creates a new layer. Combine related commands into a single `RUN` to reduce image size:

```dockerfile
# Good
RUN apt-get update && apt-get install -y apache2 && apt-get clean

# Bad
RUN apt-get update
RUN apt-get install -y apache2
RUN apt-get clean
```

### 4. Always clean up after installing packages

```dockerfile
RUN apt-get update && apt-get install -y apache2 && apt-get clean
```

Leaving cached package files inside the image unnecessarily increases its size.

### 5. Never use `latest` as a tag in production

```bash
# Bad
docker build -t myapp:latest .

# Good
docker build -t myapp:1.2.0 .
```

`latest` is unpredictable — it changes every time a new version is released. Always pin a specific version.

### 6. Don't run as root

Create a dedicated user and switch to it before the container starts:

```dockerfile
RUN useradd -m appuser
USER appuser
```

### 7. One process per container

Each container should run a single process. Running multiple processes inside one container makes it harder to manage, monitor, and scale.

### 8. Put `EXPOSE` and `ENV` at the end

These instructions rarely change. Placing them at the bottom of the Dockerfile keeps frequently-changing instructions (like `RUN` or `ADD`) higher up, which helps Docker's layer caching work more efficiently.

### 9. Use a Dockerfile linter

Tools like **Hadolint** analyze your Dockerfile and flag issues (bad base images, too many `RUN` layers, missing `apt-get clean`, etc.).

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
```

### 10. Use multi-stage builds for compiled applications

Avoid shipping build tools and compilers in your final image. Use multi-stage builds to keep the final image as small and clean as possible.