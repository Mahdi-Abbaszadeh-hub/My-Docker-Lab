# Docker Networking — Custom Networks & Container DNS

> Why the default bridge network isn't enough, how to create your own networks, and how container name resolution (DNS) actually works.

---

## Table of Contents

- [Docker Networking — Custom Networks \& Container DNS](#docker-networking--custom-networks--container-dns)
  - [Table of Contents](#table-of-contents)
  - [The Problem with the Default Bridge](#the-problem-with-the-default-bridge)
  - [Network Tools — busybox, dnsutils, curl](#network-tools--busybox-dnsutils-curl)
  - [Testing Connectivity on the Default Bridge](#testing-connectivity-on-the-default-bridge)
    - [Step 1 — Run two busybox containers](#step-1--run-two-busybox-containers)
    - [Step 2 — Check their assigned IPs](#step-2--check-their-assigned-ips)
    - [Step 3 — Enter busybox1 and test connectivity](#step-3--enter-busybox1-and-test-connectivity)
    - [Step 4 — Try pinging by container name](#step-4--try-pinging-by-container-name)
  - [Why You Shouldn't Put Everything on One Network](#why-you-shouldnt-put-everything-on-one-network)
    - [The problem](#the-problem)
    - [The principle](#the-principle)
  - [Creating a Custom (User-Defined) Network](#creating-a-custom-user-defined-network)
  - [Connecting Containers with an Alias](#connecting-containers-with-an-alias)
    - [Verify both containers joined](#verify-both-containers-joined)
    - [Test name resolution on the new network](#test-name-resolution-on-the-new-network)
  - [Disconnecting from the Default Bridge](#disconnecting-from-the-default-bridge)
    - [Verify they're gone from the bridge](#verify-theyre-gone-from-the-bridge)
    - [Verify they're still reachable on backend](#verify-theyre-still-reachable-on-backend)
  - [Why Name Resolution Matters More Than IPs](#why-name-resolution-matters-more-than-ips)
    - [The wrong way](#the-wrong-way)
    - [The right way](#the-right-way)
  - [Command Summary](#command-summary)
    - [What we've covered in this chapter](#what-weve-covered-in-this-chapter)

---

## The Problem with the Default Bridge

When a container joins the default `bridge` network, it gets an automatically assigned IP address — but **no name resolution**. This means containers on the default bridge can reach each other **only by IP address**, not by container name.

This is a real limitation: IP addresses change every time a container is removed and recreated. Hardcoding an IP into another service's configuration is fragile and will eventually break.

This chapter solves that problem with **user-defined networks**, which provide automatic DNS resolution between containers by name.

---

## Network Tools — busybox, dnsutils, curl

Official Docker images (like `logstash`, `nginx`, `postgres`) are intentionally lightweight — to keep image size small, they strip out most networking tools (`ping`, `curl`, `nslookup`, etc.). This is by design for production use, but it makes hands-on network debugging difficult.

For learning and testing network concepts, the Docker community maintains small utility images built specifically for this purpose:

| Image | What it provides |
|---|---|
| `busybox` | A tiny Linux distro with basic tools like `ping`, `wget`, `sh` |
| `tutum/dnsutils` (or similar) | DNS tools like `nslookup`, `dig` |
| `curlimages/curl` | Just `curl`, for testing HTTP connectivity |

We'll use **busybox** throughout this section since it includes `ping`, which is the simplest way to verify two containers can reach each other.

---

## Testing Connectivity on the Default Bridge

### Step 1 — Run two busybox containers

`busybox` has no long-running default process (no `ENTRYPOINT`/`CMD` that keeps it alive), so we override the command to keep it running indefinitely:

```bash
docker run -dit --name busybox1 busybox
docker run -dit --name busybox2 busybox
```

- `-d` — detached mode
- `-i` — keep STDIN open
- `-t` — allocate a pseudo-TTY

Together, `-dit` keeps the container alive and ready to accept an interactive shell later, even though it's running in the background.

Neither container was given a `--network` flag, so both automatically join the default `bridge` network.

### Step 2 — Check their assigned IPs

```bash
docker network inspect bridge
```

You'll see both containers listed under `"Containers"`, each with its own IP — typically something like `172.17.0.2` and `172.17.0.3`.

### Step 3 — Enter busybox1 and test connectivity

```bash
docker exec -it busybox1 sh
```

Inside the container, ping busybox2's IP directly:

```bash
ping 172.17.0.3
```

**Result:** This works. Both containers are on the same bridge network, so they can reach each other by IP.

### Step 4 — Try pinging by container name

```bash
ping busybox2
```

**Result:** This fails — `bad address 'busybox2'`.

**Why?** The default `bridge` network does **not** provide DNS resolution between containers. It only handles routing by IP. This is the core limitation we need to solve.

```bash
exit
```

---

## Why You Shouldn't Put Everything on One Network

Imagine the full observability stack from our scenario: Kibana, Logstash, Elasticsearch, Zipkin, Redis, RedisInsight, Grafana, Prometheus.

It might seem convenient to put all of these containers on the same default bridge network so they can all reach each other. **This is a security mistake.**

### The problem

If every container shares one network:
- Grafana can directly reach Elasticsearch — even though Grafana has no legitimate reason to talk to Elasticsearch
- Prometheus can directly reach Redis — same issue
- A compromised container has a much larger attack surface, since it can potentially reach every other service

### The principle

> **Only containers that actually need to communicate should share a network.**

This is a basic network segmentation practice, common in real production environments:

| Containers | Should they share a network? |
|---|---|
| Logstash ↔ Elasticsearch | ✅ Yes — Logstash sends logs to Elasticsearch |
| Kibana ↔ Elasticsearch | ✅ Yes — Kibana queries Elasticsearch for dashboards |
| Grafana ↔ Prometheus | ✅ Yes — Grafana queries Prometheus for metrics |
| Grafana ↔ Elasticsearch | ❌ No — unrelated, no legitimate communication path |
| Prometheus ↔ Redis | ❌ No — unrelated |

A container can belong to **multiple networks simultaneously**, so you design your network topology around actual communication needs — not convenience.

---

## Creating a Custom (User-Defined) Network

```bash
docker network create --driver bridge backend
```

Breaking this down:
- `docker network create` — create a new network
- `--driver bridge` — use the bridge driver (same underlying technology as the default network, but isolated from it)
- `backend` — the name we're giving this network

Verify it was created:

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
f7ab26d8f4d7   bridge    bridge    local
a92e1c3d8f12   backend   bridge    local
...
```

> **Key difference from the default bridge:** Custom user-defined networks **do** provide automatic DNS resolution between containers — this is the whole reason to create one.

---

## Connecting Containers with an Alias

Now we connect our two busybox containers to the `backend` network, giving each one an **alias** — the name other containers on this network will use to reach it.

```bash
# Connect busybox1 to backend, with alias "busybox-1"
docker network connect --alias busybox-1 backend busybox1

# Connect busybox2 to backend, with alias "busybox-2"
docker network connect --alias busybox-2 backend busybox2
```

Syntax breakdown:

```
docker network connect --alias <alias_name> <network_name> <container_name_or_id>
```

| Part | Meaning |
|---|---|
| `--alias busybox-1` | The DNS name other containers will use to reach this one |
| `backend` | The network to connect to |
| `busybox1` | The container being connected |

> The alias doesn't have to match the container's actual name — you can choose any name you want. This is especially useful when migrating services or running multiple versions side by side.

### Verify both containers joined

```bash
docker network inspect backend
```

```json
"Containers": {
  "abc123...": {
    "Name": "busybox1",
    "IPv4Address": "172.20.0.2/16"
  },
  "def456...": {
    "Name": "busybox2",
    "IPv4Address": "172.20.0.3/16"
  }
}
```

Both containers are now connected to **two networks simultaneously**: the default `bridge` (their original network) and `backend` (the one we just created).

### Test name resolution on the new network

```bash
docker exec -it busybox1 sh
ping busybox-2
```

**Result:** ✅ This works! On a user-defined network, Docker's built-in DNS server resolves container names (and aliases) to their current IP automatically.

```bash
exit
```

---

## Disconnecting from the Default Bridge

To enforce proper network segmentation, we now remove these containers from the default `bridge` network — they should only be reachable through `backend`.

```bash
docker network disconnect bridge busybox1
docker network disconnect bridge busybox2
```

Syntax:

```bash
docker network disconnect <network_name> <container_name_or_id>
```

### Verify they're gone from the bridge

```bash
docker network inspect bridge
```

```json
"Containers": {}
```

Empty — neither container is connected to the default bridge anymore.

### Verify they're still reachable on backend

```bash
docker network inspect backend
```

Both containers are still listed here, each with their assigned IP.

```bash
docker exec -it busybox1 sh
ping 172.20.0.3
# ✅ works — direct IP still reachable

ping busybox-2
# ✅ works — alias resolution still works

exit
```

Both IP-based and name-based connectivity work correctly — but now **only** through the `backend` network. The containers are no longer reachable via the default bridge at all.

---

## Why Name Resolution Matters More Than IPs

This is one of the most important practical lessons in Docker networking.

### The wrong way

```yaml
# Logstash config pointing directly at an IP
output {
  elasticsearch {
    hosts => ["172.17.0.5:9200"]
  }
}
```

**Problem:** If the Elasticsearch container is ever removed and recreated — due to a crash, an update, a redeploy — Docker will likely assign it a **different IP address**. The moment that happens, Logstash can no longer find Elasticsearch, and the connection breaks silently until someone notices and manually updates the config.

### The right way

```yaml
# Logstash config pointing at a container name / alias
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
  }
}
```

**Why this works:** As long as both containers are on the same user-defined network, Docker's internal DNS resolves `elasticsearch` to whatever the current IP happens to be — automatically, every time. The name stays constant even when the underlying IP changes.

> **Rule of thumb for production:** Never hardcode container IPs into configuration files. Always reference other services by container name or network alias. This is exactly the kind of practice that makes a setup resilient to container restarts, redeployments, and scaling.

---

## Command Summary

```bash
# List all networks
docker network ls

# Create a new user-defined network
docker network create --driver bridge <network_name>

# Connect a running container to a network, with an alias
docker network connect --alias <alias_name> <network_name> <container>

# Disconnect a container from a network
docker network disconnect <network_name> <container>

# Inspect a network (see connected containers, IPs, subnet, etc.)
docker network inspect <network_name>

# Remove a specific network
docker network rm <network_name>

# Remove all unused networks
docker network prune
```

### What we've covered in this chapter

- The default `bridge` network connects containers by IP only — no name resolution
- User-defined networks (`docker network create`) provide automatic DNS resolution between containers
- Containers can belong to multiple networks simultaneously
- Network segmentation is a security best practice — only containers that need to talk to each other should share a network
- Always reference other services by container name/alias in configuration files, never by hardcoded IP

This sets up everything we need for the next step: connecting our full ELK + monitoring stack together, with each service only able to reach the services it actually depends on.