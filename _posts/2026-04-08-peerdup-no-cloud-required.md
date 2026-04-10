---
layout: post
title: "Introducing peerdup: BitTorrent-backed private file replication"
date: 2026-04-08
---

I've used similar products for years. I've even written blog posts about [them](https://www.resilio.com/blog/sync-hacks-how-to-use-bittorrent-sync-as-geo-replication-for-storage). The reality is Resilio's Sync is awesome, but overkill for my usecase. For both home use and datacenter level geo-replication, it just wasn't a clean fit.

# The problem it solves

What I wanted was cli driven, scriptable, and open source for bare metal servers. Get the files back to a NAS, and then let it handle backups, snapshots or whatever via BTRFS or ZFS. For my desktop, I wanted something I can bake into GNOME natively to sync directories with ease. For servers, I wanted something I could push with ansible.

I wanted something that keeps files in sync across my machines — fast, encrypted, and without routing everything through someone else's servers. Every major solution I looked at either required a cloud account, imposed storage limits, or made me nervous about what was happening to my data in transit. So I built one.

It's called [peerdup](https://github.com/theronconrey/peerdup). It's open source, self-hosted, and built around a simple idea: your files should travel directly between your devices whereever they are. It shouldn't be difficult.

# How it works

The architecture has three components: a registry, a relay, and a daemon that runs on each of your machines.

```
┌─────────────────┐
│    Registry     │  gRPC / TLS — peer discovery, ACL, presence
└────────┬────────┘
         │
  ┌──────┴──────┐
  │             │
┌─┴──────┐  ┌──┴─────┐
│Daemon A│◄─►Daemon B│  direct P2P via libtorrent
└────────┘  └────────┘
               ╲
            ┌───┴───┐
            │ Relay │   NAT fallback (optional)
            └───────┘
```

The registry is a lightweight service you run once on an always-on machine (or a small VPS). It handles peer discovery and access control — it knows which machines exist and who is allowed to sync what — but it never sees the actual file contents. When two daemons need to talk, the registry introduces them and steps aside.

The relay is there for the hard cases: when one of your devices is behind a symmetric NAT that prevents a direct connection. The daemon tries both a direct path and a relay-bridged path simultaneously and uses whichever connects first. Most of the time, you won't even notice it's there.

## Two sync modes

[peerdup](https://github.com/theronconrey/peerdup) ships with two modes, depending on your setup:

| Mode | Registry needed? | Discovery | Access control |
|------|-----------------|-----------|----------------|
| registry | Yes | Registry + LAN multicast | ACL enforced by registry |
| local | No | LAN multicast only | Anyone on LAN with the share ID |

The local mode is great if all your devices are on the same network and you want zero infrastructure. Just start the daemon, create a share with `--local`, share the share ID with another machine on the same LAN, and you're syncing.

# Key features

**Transfer** — Direct P2P via libtorrent. All file data moves device to device. No relay unless NAT requires it.

**Security** — TLS on all registry communication. libtorrent encryption on transfers.

**Access** — Per-share ACL. Grant and revoke peers per share. Registry enforces it at the discovery layer.

**Conflicts** — Three resolution strategies: last-write-wins, rename-on-conflict, or manual review — configurable per share.

**Limits** — Bandwidth throttling. Set per-share upload and download limits so [peerdup](https://github.com/theronconrey/peerdup) doesn't saturate your connection.

**Ops** — Docker + Caddy stack. One script gets the registry and relay running with automatic Let's Encrypt TLS.

# Getting started

Installation is a single curl command on each machine:

```bash
curl -fsSL https://raw.githubusercontent.com/theronconrey/peerdup/main/install.sh | sh
```

The installer walks you through whether to also set up a registry on that machine (say yes on your server, no on laptops). From there, `peerdup-setup` handles daemon configuration, and you're ready to create your first share.

For those who prefer containers, the Docker Compose stack brings up the registry, relay, and Caddy reverse proxy with automatic TLS, you just need a domain and a public-facing host. Run `./start.sh` and it handles the rest. Transparently, this is not working without some level of tinkering. This is where I'm currently focused on making the installation smoother.

<p style="text-align: center; margin-top: 1em;"><a href="https://github.com/theronconrey/peerdup">View on GitHub</a></p>

# Where it stands today

[peerdup](https://github.com/theronconrey/peerdup) is a pet project in active development. The core sync loop works, the CLI is functional, and the Docker deployment path is solid. There's still a lot of surface area to improve — better observability, broader platform testing, documentation depth. If this is something that is interesting to you, reach out and say hello!
