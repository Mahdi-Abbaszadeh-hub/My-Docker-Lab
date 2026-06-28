# Docker Volumes — Practical Guide

> Step-by-step walkthrough of named volumes and bind mounts — with Logstash as the real example.

---

## Table of Contents

- [Part 1 — The Problem in Action](#part-1--the-problem-in-action)
- [Part 2 — Named Volumes](#part-2--named-volumes)
- [Part 3 — Bind Mounts](#part-3--bind-mounts)
- [Volume Commands Reference](#volume-commands-reference)

---

## Part 1 — The Problem in Action

Before using volumes, we first demonstrate the problem — data that disappears when a container is removed.

### Step 1 — Run Logstash without a volume

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  logstash:8.13.0
```

Flag breakdown:
- `-d` — detached mode (runs in background)
- `-p 5000:5000` — maps port 5000 on your machine to port 5000 inside the container
- `--name logstash` — gives the container a readable name
- `logstash:8.13.0` — image name and version tag

### Step 2 — Check the logs

```bash
# View all logs
docker logs logstash

# Follow logs in real time
docker logs -f logstash
```

Wait until Logstash finishes starting up. You'll see it's running and the port is open.

### Step 3 — Enter the container and explore

```bash
docker exec -it logstash sh
```

Now you're inside the container. Let's look at the default config:

```bash
pwd
# → /usr/share/logstash

ls
# → you see Logstash's directories: bin, config, data, lib, etc.

cd config
ls
# → logstash.yml  pipelines.yml  and others

cat logstash.yml
# → shows the default configuration Logstash uses when no custom config is provided
```

### Step 4 — Create a file inside the container

Let's simulate "making a change" by creating a new config file:

```bash
echo "hello" > /usr/share/logstash/config/logstash2.yml
ls
# → logstash.yml  logstash2.yml  ← our new file is here
```

Exit the container:

```bash
exit
```

### Step 5 — Remove the container and prove data is gone

```bash
# Force remove (stops and removes in one command)
docker rm -f logstash
```

> The `-f` flag force-removes even a running container — equivalent to `docker stop` + `docker rm` in one step.

### Step 6 — Create a new container from the same image

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  logstash:8.13.0
```

Enter it and check for our file:

```bash
docker exec -it logstash sh
ls /usr/share/logstash/config/
# → logstash.yml  ← logstash2.yml is GONE ❌
exit
```

**This proves the problem.** Any change made inside a container is lost when the container is removed. Now let's fix it.

---

## Part 2 — Named Volumes

A **named volume** is created and managed by Docker. Docker stores it at:

```
/var/lib/docker/volumes/<volume_name>/_data
```

You never need to interact with this path directly — Docker manages it. You just give the volume a name and attach it to containers.

### Step 1 — Volume commands overview

```bash
docker volume --help
```

The main subcommands:

| Subcommand | Description |
|---|---|
| `create` | Create a new volume |
| `ls` | List all volumes |
| `inspect` | Show detailed info about a volume |
| `rm` | Remove a specific volume |
| `prune` | Remove all unused volumes |

### Step 2 — Create a named volume

```bash
docker volume create logstash-vol
```

Verify it was created:

```bash
docker volume ls
```

Output:
```
DRIVER    VOLUME NAME
local     logstash-vol
```

### Step 3 — Inspect the volume

```bash
docker volume inspect logstash-vol
```

Output:
```json
[
  {
    "CreatedAt": "2026-01-01T00:00:00Z",
    "Driver": "local",
    "Labels": {},
    "Mountpoint": "/var/lib/docker/volumes/logstash-vol/_data",
    "Name": "logstash-vol",
    "Options": {},
    "Scope": "local"
  }
]
```

Key fields:
- `Driver: local` — Docker manages this volume locally on your machine
- `Mountpoint` — the actual path on your host where data is stored
- `Scope: local` — this volume exists on this machine only (not shared across a cluster)

### Step 4 — Navigate to the volume on your host

```bash
cd /var/lib/docker/volumes/
ls
# → logstash-vol/

cd logstash-vol
ls
# → _data/

cd _data
ls
# → empty for now — data will appear here when the container writes to it
```

### Step 5 — Run a container with the named volume attached

Use `-v <volume_name>:<path_inside_container>`:

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  -v logstash-vol:/usr/share/logstash/data \
  logstash:8.13.0
```

What this means:
- `logstash-vol` — the named volume we created
- `/usr/share/logstash/data` — the path inside the container that gets mapped to the volume

Any file written to `/usr/share/logstash/data` inside the container will actually be saved in `/var/lib/docker/volumes/logstash-vol/_data` on your host.

### Step 6 — Verify data persists across container removal

```bash
# Enter the container and create a file
docker exec -it logstash sh
echo "persistent data" > /usr/share/logstash/data/testfile.txt
exit

# Remove the container
docker rm -f logstash

# Check the volume on your host — data is still there
ls /var/lib/docker/volumes/logstash-vol/_data
# → testfile.txt ✅

# Create a brand new container with the same volume
docker run -d \
  -p 5000:5000 \
  --name logstash-new \
  -v logstash-vol:/usr/share/logstash/data \
  logstash:8.13.0

# Enter it and check
docker exec -it logstash-new sh
ls /usr/share/logstash/data/
# → testfile.txt ✅ — data survived!
cat /usr/share/logstash/data/testfile.txt
# → persistent data
exit
```

**The data survived the container removal.** This is what Docker volumes are for.

---

## Part 3 — Bind Mounts

A **bind mount** maps a specific path on your local machine directly into the container. Unlike named volumes, you choose the exact path — Docker doesn't manage it.

### When to use bind mounts

Bind mounts are perfect for **configuration files**. You write the config on your machine, bind mount it into the container, and the container reads it — without you having to copy files manually or rebuild the image.

### Our goal

We have a custom Logstash pipeline config file (`logstash.conf`) on our local machine. We want to:

1. Mount that file into the container at Logstash's config path
2. Tell Logstash to use our config instead of its default

### Step 1 — The custom config file

Here's the `logstash.conf` we created locally:

```
# logstash.conf
input {
  tcp {
    port => 5000
  }
}

filter {
  mutate {
    add_tag => ["test"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
  }
}
```

What this pipeline does:
- **Input:** listens on TCP port 5000 — developers send logs here
- **Filter:** adds a tag `test` to every log entry
- **Output:** forwards logs to Elasticsearch on port 9200

### Step 2 — Clean up previous containers and volumes

```bash
# Remove all stopped containers
docker rm -f logstash logstash-new

# Remove the volume we created (clean slate)
docker volume rm logstash-vol

# Verify
docker volume ls
docker ps -a
```

### Step 3 — Run Logstash with a bind mount AND a custom CMD

This is the full command — read it carefully:

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  -v $(pwd)/logstash.conf:/etc/logstash/conf.d \
  logstash:8.13.0 \
  logstash -f /etc/logstash/conf.d
```

**Breaking this down piece by piece:**

| Part | Meaning |
|---|---|
| `docker run -d` | Run in detached (background) mode |
| `-p 5000:5000` | Map host port 5000 → container port 5000 |
| `--name logstash` | Name the container |
| `-v $(pwd)/logstash.conf:/etc/logstash/conf.d` | **Bind mount** — see below |
| `logstash:8.13.0` | The image to use |
| `logstash -f /etc/logstash/conf.d` | **CMD override** — see below |

**The bind mount `-v $(pwd)/logstash.conf:/etc/logstash/conf.d`:**

```
$(pwd)/logstash.conf        :       /etc/logstash/conf.d
        ↑                                   ↑
  path on YOUR machine            path INSIDE the container
  (current directory)             (where Logstash looks for config)
```

`$(pwd)` is a Linux command that prints your current directory. So if you're in `/home/mahdi/project`, the full path becomes `/home/mahdi/project/logstash.conf`.

**The CMD override `logstash -f /etc/logstash/conf.d`:**

Every Docker image has a default `CMD`. The Logstash image runs Logstash with its default pipeline config.

We want to **override that CMD** and tell Logstash to use our custom config file instead. Any command written **after the image name** in `docker run` replaces the image's default CMD:

```bash
docker run ... logstash:8.13.0 logstash -f /etc/logstash/conf.d
#                    ↑                      ↑
#               image name          overrides the default CMD
```

`logstash -f /etc/logstash/conf.d` tells Logstash:
> "Use the config file(s) at this path"

Since we bind mounted our `logstash.conf` to exactly that path, Logstash reads our custom pipeline.

### Step 4 — Verify the bind mount worked

Check the logs:

```bash
docker logs -f logstash
```

You should see Logstash starting up and loading your pipeline config — no errors.

Enter the container and verify the file is there:

```bash
docker exec -it logstash sh
ls /etc/logstash/conf.d
# → logstash.conf ✅ — our file is here

cat /etc/logstash/conf.d/logstash.conf
# → shows our custom pipeline config ✅

exit
```

**The bind mount worked.** Logstash is now using our custom pipeline config from our local machine — without rebuilding the image, without copying files manually.

---

## Named Volume vs Bind Mount — Summary

| | Named Volume | Bind Mount |
|---|---|---|
| **Who manages the path** | Docker | You |
| **Where data lives** | `/var/lib/docker/volumes/` | Anywhere you choose |
| **Best for** | Database data, persistent app data | Config files, development |
| **Syntax** | `-v volume_name:/path/in/container` | `-v /your/local/path:/path/in/container` |
| **Survives container removal** | ✅ | ✅ |
| **Edit files from host** | ❌ (managed by Docker) | ✅ (that's the point) |

---

## Volume Commands Reference

```bash
# Create a named volume
docker volume create <name>

# List all volumes
docker volume ls

# Inspect a volume
docker volume inspect <name>

# Remove a specific volume
docker volume rm <name>

# Remove all unused volumes (not attached to any container)
docker volume prune

# Run with a named volume
docker run -v <volume_name>:<container_path> <image>

# Run with a bind mount
docker run -v <host_path>:<container_path> <image>

# Force remove a running container (stop + rm)
docker rm -f <container_id>
```