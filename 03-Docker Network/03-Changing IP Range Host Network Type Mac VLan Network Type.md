# Docker Networking — Custom Subnets, Host Network & Macvlan

> Advanced network configuration: defining your own IP ranges, understanding the host network mode, and the macvlan driver for giving containers their own identity on a physical network.

---

## Table of Contents

- [Custom Networks with Defined Subnets](#custom-networks-with-defined-subnets)
- [Specifying `--network` at Container Creation Time](#specifying---network-at-container-creation-time)
- [Host Network Mode](#host-network-mode)
- [Macvlan — Containers as Physical Network Devices](#macvlan--containers-as-physical-network-devices)
- [Network Driver Comparison](#network-driver-comparison)
- [Best Practices Summary](#best-practices-summary)
- [Command Reference](#command-reference)

---

## Custom Networks with Defined Subnets

When you create a user-defined network without specifying an IP range, Docker assigns one automatically (typically in the `172.x.x.x` range). But in real company environments, IP ranges are often tightly controlled — your infrastructure team may allocate a specific range and tell you "only use IPs from this subnet."

For this reason, Docker lets you specify your own subnet and gateway when creating a network.

### Creating a network with a custom subnet

```bash
docker network create \
  --driver bridge \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  custom
```

Breaking this down:

| Flag | Value | Meaning |
|---|---|---|
| `--driver bridge` | `bridge` | Use the bridge driver (isolated virtual network) |
| `--subnet` | `192.168.1.0/24` | The IP range for this network — containers get IPs from here |
| `--gateway` | `192.168.1.1` | The gateway IP — the router address containers use to send traffic out of this network |
| (name) | `custom` | The network name |

### What `/24` means

The `/24` is CIDR notation. It means the first 24 bits of the address are fixed (the network address), and the remaining 8 bits are available for hosts — giving you 254 usable IP addresses (`192.168.1.1` through `192.168.1.254`).

Common CIDR values and their host capacity:

| CIDR | Available host IPs | Use case |
|---|---|---|
| `/24` | 254 | Small networks, most common for dev/test |
| `/16` | 65,534 | Large networks |
| `/28` | 14 | Very small, tightly scoped segments |

### Verify the custom range was applied

```bash
docker network inspect custom
```

```json
"IPAM": {
  "Config": [
    {
      "Subnet": "192.168.1.0/24",
      "Gateway": "192.168.1.1"
    }
  ]
}
```

Any container you connect to this network will receive an IP in the `192.168.1.x` range, starting from `.2` (since `.1` is the gateway).

### Testing with dnsutils

Attach a networking utility container to verify the IP assignment:

```bash
docker run -d \
  -it \
  --name custom-utils \
  --network custom \
  tutum/dnsutils
```

Enter it and check:

```bash
docker exec -it custom-utils sh
ifconfig
# → eth0: inet addr: 192.168.1.2 ✅
```

The container received an address from your defined range exactly as expected.

> **Real-world note:** When working at a company, always check with your network team before choosing a subnet. Using a range that conflicts with your company's internal IP scheme will cause routing problems — traffic meant for internal servers may be routed to your containers instead.

---

## Specifying `--network` at Container Creation Time

So far we've been:
1. Running a container (it auto-joins the default bridge)
2. Then disconnecting it from the default bridge
3. Then connecting it to a custom network

This three-step process is unnecessary. You can specify the target network **at run time** with `--network`:

```bash
docker run -dit \
  --name mycontainer \
  --network backend \
  busybox
```

When you use `--network`, the container:
- **Only** joins the specified network
- **Does not** join the default `bridge` network at all
- No `disconnect` step needed

This is the **best practice**. Always include `--network` in your `docker run` command so the container ends up exactly where it belongs from the start. Every other flag you've learned (`-d`, `--name`, `-p`, `-v`) should be accompanied by `--network` in real usage.

---

## Host Network Mode

When you run a container with `--network host`, the container **does not get its own isolated network stack**. Instead, it shares the host machine's network interfaces directly.

```bash
docker run -dit \
  --name busybox3 \
  --network host \
  busybox
```

### What this looks like from inside

```bash
docker exec -it busybox3 sh
ifconfig
```

Instead of seeing only a container-private IP (like `172.x.x.x`), you see **all the host's interfaces**: the physical card, the VM adapter, the `docker0` bridge, everything. The container and the host are sharing the same network stack completely.

### Implications of host networking

| Aspect | Effect |
|---|---|
| Port mapping (`-p`) | **Not needed** and has no effect — the container directly uses the host's ports |
| Network isolation | **None** — the container can see and access everything the host can |
| Performance | Slightly better — no virtual interface overhead |
| Security | Weaker — container is fully exposed to the host's network |

### When host networking is appropriate

Host networking is a **bad practice** for most workloads and should be avoided by default. The exception cases:
- **Monitoring agents** that need to see the host's actual network statistics (e.g., Prometheus `node_exporter`)
- **Performance-critical networking** where virtual interface overhead is measurable and unacceptable
- **Debugging** where you need to rule out the virtual network as the cause of a problem

> **Rule:** Unless you have a specific, justified reason to use `--network host`, always use a user-defined bridge network instead.

---

## Macvlan — Containers as Physical Network Devices

**Macvlan** is a completely different model from bridge networking. It's not about virtual networks inside Docker — it's about making a container appear as a **real, independent device on your physical network**.

### The mental model

Think about the devices on a typical office network:
- Your laptop → has its own MAC address, its own IP
- Your colleague's laptop → its own MAC address, its own IP
- A printer → its own MAC address, its own IP
- A server → its own MAC address, its own IP

Each device is independently visible on the network. With macvlan, you can add a container to this list — it gets its own MAC address and its own IP, and **from the physical network's perspective, it looks like a completely separate physical device**, not something running inside Docker.

### How macvlan differs from bridge networking

| | Bridge | Macvlan |
|---|---|---|
| Container IP source | Docker's internal IP pool | Your actual physical network's IP range |
| Visibility on physical network | Hidden behind the host's IP | Directly visible as its own device |
| MAC address | Shared with host (through bridge) | Unique per container |
| Dependency on Docker's virtual switch | Yes | No |
| Complexity | Low | Higher |

### When macvlan is used

Macvlan is used when you need a container to be a **fully independent network entity** — for example:
- Legacy applications that expect to be at a fixed, known IP on the physical network
- Network appliances (firewalls, routers) running in containers
- Environments where the container must receive broadcast traffic from the physical network
- Situations where the container needs to be reachable by external devices without any port mapping

### Creating a macvlan network

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.3.0/24 \
  --gateway 192.168.3.1 \
  -o parent=ens33 \
  pub-net
```

The key new flag here is `-o parent=ens33`:

| Flag | Value | Meaning |
|---|---|---|
| `--driver macvlan` | `macvlan` | Use the macvlan driver |
| `--subnet` | `192.168.3.0/24` | Must match your physical network's IP range |
| `--gateway` | `192.168.3.1` | Your physical network's actual gateway (router) |
| `-o parent` | `ens33` | The **physical** network interface on the host to attach to |

The `parent` interface is critical — it tells Docker which of the host's physical network interfaces to "extend" with this macvlan network. Containers on this network will appear to the physical switch as if they're directly plugged into it through `ens33`.

### Running a container on macvlan

```bash
docker run -dit \
  --network pub-net \
  --name busybox5 \
  busybox
```

Inspect the container's network:

```bash
docker exec -it busybox5 sh
ifconfig
# → eth0: inet addr 192.168.3.2 — an address from your physical network's range ✅
```

The container has an address from your physical network. Other devices on that network can reach it directly at `192.168.3.2` — no port mapping, no bridge translation, nothing.

### Important limitations of macvlan

**1. The host cannot communicate with macvlan containers directly.**
This is a fundamental macvlan limitation. Because macvlan works by assigning sub-interfaces to the parent interface, the Linux kernel prevents direct host-to-container communication over the same physical interface. The container can talk to the outside network, but the host machine itself cannot reach the container. (Workaround: create a separate macvlan interface on the host — but this adds significant complexity.)

**2. Your physical switch must support promiscuous mode.**
Macvlan creates multiple MAC addresses on a single physical port. By default, most switches discard frames destined for an unknown MAC. For macvlan to work, the switch port (or the hypervisor's virtual switch) must be set to accept traffic for multiple MACs.

**3. Not suitable for general-purpose application containers.**
Macvlan is a specialized tool. For most web services, APIs, databases, and standard workloads, a user-defined bridge network is the correct choice.

---

## Network Driver Comparison

| Driver | Container has own IP? | DNS between containers? | Visible on physical network? | Use case |
|---|---|---|---|---|
| `bridge` (default) | ✅ (private) | ❌ No | ❌ No | Default; not recommended for production |
| `bridge` (user-defined) | ✅ (private) | ✅ Yes | ❌ No | **Standard best practice for most workloads** |
| `host` | ❌ (shares host) | N/A | Same as host | Monitoring agents, performance-critical |
| `none` | ❌ | N/A | ❌ No | Fully isolated containers, security |
| `macvlan` | ✅ (from physical range) | ❌ (no Docker DNS) | ✅ Yes | Legacy apps, network appliances |

---

## Best Practices Summary

**1. Always specify `--network` when running containers.**
Never rely on auto-joining the default bridge. Define your network topology explicitly.

**2. Create separate networks for separate concerns.**
Group containers that need to communicate — and keep unrelated containers apart. This is network segmentation and is a basic security principle.

**3. Define your own subnets when working in a company environment.**
Know your company's IP allocation. Avoid using ranges that conflict with internal infrastructure.

**4. Reference other containers by name, never by IP.**
Container IPs change when containers are recreated. Names and aliases on a user-defined bridge network persist as long as the network exists.

**5. Avoid `--network host` unless you have a specific reason.**
The isolation breach is rarely worth it. Most cases that seem to need host networking can be solved more cleanly with proper port mapping and user-defined bridges.

**6. Use macvlan only for specialized requirements.**
Standard containerized applications don't need to appear as physical devices. Macvlan adds complexity with limited benefit for general workloads.

---

## Command Reference

```bash
# Create a network with custom subnet and gateway
docker network create \
  --driver bridge \
  --subnet <subnet/cidr> \
  --gateway <gateway_ip> \
  <network_name>

# Create a macvlan network
docker network create \
  --driver macvlan \
  --subnet <physical_subnet/cidr> \
  --gateway <physical_gateway> \
  -o parent=<host_interface> \
  <network_name>

# Run a container and attach it to a specific network (best practice)
docker run -dit --network <network_name> --name <container_name> <image>

# Run a container on the host network
docker run -dit --network host --name <container_name> <image>

# Connect a running container to an additional network with an alias
docker network connect --alias <alias> <network_name> <container>

# Disconnect a container from a network
docker network disconnect <network_name> <container>

# List all networks
docker network ls

# Inspect a network
docker network inspect <network_name>

# Remove a network
docker network rm <network_name>

# Remove all unused networks
docker network prune
```