# Docker Volumes — Introduction & Concepts

> Why containers lose data, what Docker volumes are, and how they solve the persistence problem.

---

## Table of Contents

- [Docker Volumes — Introduction \& Concepts](#docker-volumes--introduction--concepts)
  - [Table of Contents](#table-of-contents)
  - [The Problem — Containers Are Temporary](#the-problem--containers-are-temporary)
  - [The Wrong Solution — docker commit](#the-wrong-solution--docker-commit)
  - [The Right Solution — Docker Volumes](#the-right-solution--docker-volumes)
  - [Two Types of Volumes](#two-types-of-volumes)
    - [1. Docker Managed Volumes (Named Volumes)](#1-docker-managed-volumes-named-volumes)
    - [2. Bind Mounts](#2-bind-mounts)
  - [The Real-World Scenario — ELK + Monitoring Stack](#the-real-world-scenario--elk--monitoring-stack)
    - [Log Aggregation — ELK Stack](#log-aggregation--elk-stack)
    - [Monitoring — Prometheus + Grafana](#monitoring--prometheus--grafana)
    - [Caching — Redis + RedisInsight](#caching--redis--redisinsight)
    - [The Challenges This Stack Introduces](#the-challenges-this-stack-introduces)

---

## The Problem — Containers Are Temporary

As we learned earlier, a container has a **writable layer** on top of the read-only image layers. Any changes made inside a running container — new files, database records, configuration changes — live in that writable layer.

When the container is **removed**, the writable layer is gone too. Everything inside it disappears.

**Example:**

Imagine you run a PostgreSQL container. A user creates a table, inserts records, and then the container crashes or gets removed. When you start a new container from the same image:

```
New container → fresh image → empty database → all data gone ❌
```

This is a fundamental problem for any stateful application — databases, log files, configuration, uploaded files.

---

## The Wrong Solution — docker commit

One naive approach is to use `docker commit` to snapshot the container's state into a new image before removing it.

**Why this is bad:**

- The image grows larger with every commit — it carries all the data inside it
- You have to remember to commit manually before every removal
- Images are not designed to store runtime data
- Moving data between systems means moving the entire image
- In CI/CD pipelines, this approach breaks down completely

This is not a real solution — it's a workaround that creates more problems than it solves.

---

## The Right Solution — Docker Volumes

A **Docker volume** is a dedicated storage location managed outside the container's lifecycle. Data written to a volume **persists independently** of any container.

```
Container A → writes data → Volume
Container A removed        ↓
Container B created   → reads same Volume → data still there ✅
```

The new container picks up exactly where the old one left off. From the application's perspective, nothing changed.

**Key properties of volumes:**

- Exist independently of containers — removing a container does NOT remove its volume
- Can be attached to multiple containers simultaneously
- Can be backed up, moved, and restored like regular files
- Managed by Docker — no manual path management needed

---

## Two Types of Volumes

Docker provides two ways to persist data:

### 1. Docker Managed Volumes (Named Volumes)

Docker creates and manages the volume in its own directory:

```
/var/lib/docker/volumes/<volume_name>/_data
```

You create it with a name, attach it to containers, and Docker handles the rest. You never need to know or care about the exact path on the host.

```bash
# Create
docker volume create my-volume

# Attach when running a container
docker run -v my-volume:/data/app myimage
```

**Best for:** Databases, persistent application data — anything where you want Docker to manage storage.

### 2. Bind Mounts

You choose an **exact path on your local machine** and mount it directly into the container. Docker doesn't manage this — you control the path.

```bash
docker run -v /home/mahdi/config:/etc/logstash/conf.d myimage
```

What happens:
- The directory `/home/mahdi/config` on your machine becomes visible at `/etc/logstash/conf.d` inside the container
- Files you create or edit on your machine instantly appear inside the container
- Files the container writes appear on your machine

**Best for:** Configuration files, development workflows — anything where you want to edit files on your machine and have them reflected inside the container immediately.

---

## The Real-World Scenario — ELK + Monitoring Stack

Starting from Chapter 4, we work toward building a **production-ready infrastructure** inside Docker. The full stack includes:

### Log Aggregation — ELK Stack

| Tool | Role |
|---|---|
| **Elasticsearch** | Database that stores and indexes logs |
| **Logstash** | Pipeline that reads logs from files and sends them to Elasticsearch |
| **Kibana** | Web UI dashboard for searching and visualizing logs |

### Monitoring — Prometheus + Grafana

| Tool | Role |
|---|---|
| **Prometheus** | Collects metrics from services |
| **Grafana** | Visualizes metrics with dashboards |

### Caching — Redis + RedisInsight

| Tool | Role |
|---|---|
| **Redis** | High-performance in-memory cache |
| **RedisInsight** | Web UI for managing and inspecting Redis |

### The Challenges This Stack Introduces

Every challenge maps to a Docker concept we learn:

| Challenge | Solution |
|---|---|
| Elasticsearch data must survive container restarts | **Named volumes** |
| Logstash needs a custom config file from our machine | **Bind mounts** |
| All services need to communicate with each other | **Docker networks** (next chapter) |
| Logstash might need multiple instances under load | **Scaling** |
| All of this should start with one command | **Docker Compose** (final chapter) |

This is the goal: a full production stack that starts with a single command, with all networking, volumes, and configuration applied automatically.

We start by running Logstash — first without a volume, to see the problem firsthand.