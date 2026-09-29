# my-NUC-homelab

A single-node Proxmox homelab running on an Intel NUC in a college apartment, a network environment I don't control, with client isolation on the Wi-Fi and a dead ethernet jack. Most of what I learned here came from working around those constraints.

> Public IPs, Tailscale addresses, and apartment network details are intentionally left out of this repo.

## Hardware

| Component | Details |
|---|---|
| Host | Intel NUC10i5FNH (Core i5-10210U) |
| Memory | ~12 GB |
| Storage | 256 GB NVMe (external USB storage planned for media) |
| Network | USB Wi-Fi adapter. The onboard NIC is unusable because the apartment's wall jack is dead |
| Hypervisor | Proxmox VE |

## Network design

```mermaid
flowchart LR
    Internet((Internet)) --- AP["Apartment Wi-Fi<br/>(client isolation on)"]
    AP --- WIFI["USB Wi-Fi adapter"]

    subgraph NUC["Intel NUC · Proxmox VE host"]
        WIFI -- "NAT / MASQUERADE" --- BR["vmbr0<br/>10.10.10.1/24<br/>(standalone bridge)"]
        BR --- LXC["LXC 100 · Debian 12<br/>10.10.10.151"]

        subgraph DOCKER["Docker (inside LXC 100)"]
            PH[Pi-hole]
            PT[Portainer]
            UK[Uptime Kuma]
            HP[Homepage]
            subgraph ARR["arrstack network"]
                JF[Jellyfin]
                JS[Jellyseerr]
            end
        end
        LXC --- DOCKER
    end

    LAPTOP["Laptop"] -. "Tailscale (WireGuard)" .- LXC
    LAPTOP -. "Tailscale" .- NUC
```

**Key points:**

- **Proxmox's bridge has no physical port.** Because the host's only working uplink is Wi-Fi, which can't be bridged directly, `vmbr0` is a standalone internal bridge (`10.10.10.0/24`). Containers get static IPs on it, and the host NATs their traffic out through the Wi-Fi adapter.
- **Tailscale handles all remote access.** The apartment Wi-Fi uses client isolation, so my laptop can't reach the NUC directly even on the same network. Tailscale builds a WireGuard mesh that routes around it, so the Proxmox UI and every service are reachable from my laptop anywhere.
- **Pi-hole is the DNS server for the whole tailnet.** It's set as a global nameserver in the Tailscale admin console with DNS override on, so every device on my tailnet gets ad and tracker blocking wherever it is.

## Services

All services run as Docker containers inside a single Debian 12 LXC, managed through Portainer.

| Service | Purpose |
|---|---|
| **Pi-hole** | Network-wide DNS filtering for every device on the tailnet |
| **Portainer** | Web UI for managing Docker containers |
| **Uptime Kuma** | Uptime and health monitoring for all services |
| **Homepage** | Dashboard with a tile for every service |
| **Jellyfin** | Self-hosted media server |
| **Jellyseerr** | Media request front end for Jellyfin |

**Why one LXC instead of one per service?** With about 12 GB of RAM on a single node, one container running Docker keeps overhead low and lets services share a Docker network. The tradeoff is less isolation: if that LXC goes down, everything does. That's acceptable for a lab, and something I'd revisit with more hardware.

## Troubleshooting log

The problems I hit taught me more than the setup itself.

### 1. Containers had no internet, with no obvious error
**Symptom:** The LXC could reach the host but nothing beyond it.
**Cause:** During install, Proxmox bridged `vmbr0` to the onboard NIC, which is plugged into a dead wall jack. The bridge was "up" but connected to nothing.
**Fix:** Rebuilt `vmbr0` as a standalone bridge (`bridge-ports none`, `10.10.10.1/24`), added NAT/MASQUERADE rules to route container traffic out through the Wi-Fi adapter, and gave the container a static IP and gateway on the new subnet.
**Lesson:** "Interface is up" doesn't mean "traffic can flow." Trace the path hop by hop.

### 2. Couldn't reach the NUC from my laptop on the same Wi-Fi
**Cause:** The apartment network enforces client (AP) isolation, so wireless clients can't talk to each other.
**Fix:** Installed Tailscale on the host, the LXC, and my laptop to create an overlay network on top of the one I can't control.

### 3. Pi-hole ignored queries coming over Tailscale
**Cause:** By default, Pi-hole only answers queries from its local network, and tailnet traffic arrives on a different interface.
**Fix:** Changed Pi-hole's interface setting to "Permit all origins." This is normally risky because it can create an open resolver, but here the container sits behind NAT on an internal bridge, so the only inbound path is the authenticated tailnet.

### 4. Services started crashing and the container froze
**Cause:** Two separate resource limits. The LXC's root disk was only 7.8 GB and filled up with Docker images. After I added Jellyfin, the 512 MB RAM allocation triggered out-of-memory freezes.
**Fix:** Resized the disk to 38 GB and raised RAM to 2 GB in Proxmox.
**Lesson:** Default container allocations are sized for a single lightweight service, not a Docker host.

### 5. Jellyseerr couldn't reliably reach Jellyfin
**Cause:** Container-to-container calls using bridge or Tailscale IPs were unreliable in this nested setup (Docker inside an LXC).
**Fix:** Created a user-defined Docker network (`arrstack`) so the containers resolve each other by name through Docker's internal DNS.

## Roadmap

- [x] Proxmox on the NUC, first LXC with Docker
- [x] Fix bridge/NAT networking
- [x] Tailscale remote access around client isolation
- [x] Pi-hole as tailnet-wide DNS
- [x] Uptime Kuma monitoring and Homepage dashboard
- [x] Jellyfin and Jellyseerr on a dedicated Docker network
- [ ] Nginx Proxy Manager for internal hostnames and SSL
- [ ] Move media storage to an external USB drive
- [ ] Expand the media stack
