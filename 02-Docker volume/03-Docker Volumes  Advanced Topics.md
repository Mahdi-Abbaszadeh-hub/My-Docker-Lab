# Docker Volumes — Advanced Topics

> Data propagation rules, anonymous volumes, the `--mount` flag, read-only volumes, and tmpfs (in-memory) volumes.

---

## Table of Contents

- [Data Propagation](#data-propagation)
- [Anonymous Volumes](#anonymous-volumes)
- [`-v` vs `--mount`](#-v-vs---mount)
- [Read-Only Volumes](#read-only-volumes)
- [tmpfs — In-Memory Volumes](#tmpfs--in-memory-volumes)
- [Summary Table](#summary-table)

---

## Data Propagation

**Data propagation** describes what happens when you mount a host path into a container — specifically, what happens to existing data on **either side**.

There are two locations involved:

```
Host path  ←──── mounted to ────→  Container path
```

### The Four Scenarios

| Host has data? | Container path has data? | What happens |
|---|---|---|
| ❌ Empty | ❌ Empty | Both stay empty. Nothing to propagate. |
| ❌ Empty | ✅ Has data | The container's existing data gets **copied to the host**. |
| ✅ Has data | ❌ Empty | The host's data gets **copied into the container**. |
| ✅ Has data | ✅ Has data | **Overshadowing** — the host's data takes priority and is what appears inside the container. |

### Why this matters

This is exactly what we saw in the Logstash example earlier: we mounted an empty named volume to the container's config directory, and the container's existing config files were automatically copied out to the host volume. That's data propagation case #2 (container has data, host doesn't).

### The "overshadow" case explained

If **both sides already have data** before mounting, the host's data wins. The container's original data at that path becomes inaccessible — not deleted, just hidden/replaced for as long as the mount is active. The host version is what the application actually sees and uses.

> **Practical takeaway:** Always be aware of what already exists on both sides before mounting. If you're not careful, you might unexpectedly overshadow data inside a container with empty (or different) data from your host.

---

## Anonymous Volumes

When you use `-v` with **only a container path** (no volume name, no host path), Docker automatically creates a volume for you — with a **random ID instead of a name**.

```bash
# Named volume (we provide the name)
docker run -v logstash-vol:/etc/logstash/conf.d logstash:8.13.0

# Anonymous volume (no name before the colon)
docker run -v /etc/logstash/conf.d logstash:8.13.0
```

In the second example, Docker:
1. Creates a new volume automatically
2. Assigns it a long random ID as its name (not a readable name)
3. Mounts it to `/etc/logstash/conf.d` inside the container

### Verifying an anonymous volume

```bash
docker volume ls
```

Output:
```
DRIVER    VOLUME NAME
local     a3f8e9c1b2d4f6...   ← random ID, this is our anonymous volume
```

### Why anonymous volumes still work

Even without a name, the volume behaves like any other volume — data propagation still applies. If the container path had existing files (like Logstash's default config), they get copied into this new anonymous volume automatically, exactly as we saw with named volumes.

### When to use anonymous volumes

- Quick testing where you don't care about referencing the volume later
- When you just want to "detach" a directory from the container's writable layer without needing to manage it explicitly

**Downside:** Since there's no name, it's harder to find and manage later — you have to look it up by ID. For anything you'll need to reference again, use a **named volume** instead.

---

## `-v` vs `--mount`

Both flags accomplish the same thing — attaching storage to a container — but with different syntax.

### `-v` (short syntax)

```bash
docker run -v logstash-vol:/usr/share/logstash/data logstash:8.13.0
```

Compact, colon-separated: `source:destination`

### `--mount` (verbose syntax)

```bash
docker run --mount type=volume,source=logstash-vol,destination=/usr/share/logstash/data logstash:8.13.0
```

Explicit key-value pairs, comma-separated:

| Key | Value |
|---|---|
| `type` | `volume` (or `bind`, or `tmpfs`) |
| `source` | the volume name (or host path for bind mounts) |
| `destination` | the path inside the container |

### Comparison

| | `-v` | `--mount` |
|---|---|---|
| Syntax | Short, colon-separated | Verbose, key=value pairs |
| Readability | Less readable for complex options | Much clearer, especially with many options |
| Extra options | Limited | Supports more advanced options (`readonly`, `tmpfs-size`, etc.) |
| Recommended for | Quick commands | Scripts, production configs, anything complex |

> **Best practice:** `--mount` is the more explicit and recommended form for production use, especially as your `docker run` commands grow more complex (multiple mounts, read-only flags, etc.).

---

## Read-Only Volumes

By default, a mounted volume is **read-write** — the container can both read and write to it. You can restrict this to **read-only**, meaning the container can only read from the volume and cannot write to it.

### Using `-v` with `:ro`

```bash
docker run -v logstash-vol:/etc/logstash/conf.d:ro logstash:8.13.0
```

Adding `:ro` after the destination path makes the mount read-only.

### Using `--mount` with `readonly`

```bash
docker run --mount type=volume,source=logstash-vol,destination=/etc/logstash/conf.d,readonly logstash:8.13.0
```

### What happens if the container tries to write?

```bash
docker exec -it logstash sh
echo "test" > /etc/logstash/conf.d/newfile.txt
# → sh: can't create newfile.txt: Read-only file system
```

Checking the logs confirms the same:

```bash
docker logs logstash
# → ... must be a writable directory ...
```

### Why use read-only volumes?

This is a **security best practice**. If a directory should never be modified by the container — configuration files, secrets, static assets — mounting it as read-only prevents:

- Accidental modification by the application
- Malicious modification if the container is compromised
- Configuration drift between what you expect and what's actually running

> **Rule of thumb:** If the container only needs to *read* something, mount it read-only. Only use read-write when the container genuinely needs to write data back.

---

## tmpfs — In-Memory Volumes

**tmpfs** is a completely different storage model. Instead of writing to disk (local filesystem or a Docker-managed volume), data is stored **in memory (RAM)**.

### Why use in-memory storage?

For **sensitive, temporary data** — passwords, secrets, session tokens, database credentials — that should:
- Never touch the disk (so it can't be recovered later by anyone with disk access)
- Disappear immediately and completely when the container stops

```
Disk-based volume:  Container removed → data still exists on disk until manually deleted
tmpfs volume:        Container removed → data is gone instantly, no trace on disk
```

### Using `--mount` with tmpfs

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  --mount type=tmpfs,destination=/etc/logstash/conf.d \
  logstash:8.13.0
```

Key difference: `type=tmpfs` instead of `type=volume`. Notice there's **no `source`** — tmpfs has no host-side location at all; it exists purely in memory.

### Important limitation

tmpfs mounts have no source to copy data from. If the application expects files to already exist at that path (like Logstash expecting a config file), and nothing is there, you'll get an error:

```bash
docker logs logstash
# → could not locate config file
```

**Solution:** Either have the application generate its own files at runtime, or place your tmpfs mount at a different, empty path:

```bash
docker run -d \
  -p 5000:5000 \
  --name logstash \
  --mount type=tmpfs,destination=/etc/logstash/conf.d/logstash.conf \
  logstash:8.13.0 \
  logstash -f /etc/logstash/conf.d/logstash.conf
```

### When NOT to use tmpfs

- For anything that needs to survive a container restart (tmpfs data dies with the container, just like the writable layer — except it's not even on disk)
- For large amounts of data (RAM is limited and expensive compared to disk)
- For Linux-only feature — tmpfs is not available on Windows containers

---

## Summary Table

| Volume Type | Storage Location | Survives Container Removal | Use Case |
|---|---|---|---|
| **Named Volume** | `/var/lib/docker/volumes/` (disk) | ✅ | Databases, persistent app data |
| **Bind Mount** | Any path you choose (disk) | ✅ | Config files, development workflows |
| **Anonymous Volume** | `/var/lib/docker/volumes/` (disk, random name) | ✅ (but hard to find) | Quick tests, throwaway storage |
| **tmpfs** | RAM (memory only) | ❌ (gone instantly) | Secrets, passwords, sensitive temp data |

### Mount flags quick reference

```bash
# Named volume
-v volume_name:/container/path
--mount type=volume,source=volume_name,destination=/container/path

# Bind mount
-v /host/path:/container/path
--mount type=bind,source=/host/path,destination=/container/path

# Anonymous volume
-v /container/path

# Read-only
-v volume_name:/container/path:ro
--mount type=volume,source=volume_name,destination=/container/path,readonly

# tmpfs (in-memory)
--mount type=tmpfs,destination=/container/path
```