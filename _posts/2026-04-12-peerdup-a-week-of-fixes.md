---
layout: post
title: "peerdup: a week of fixes"
date: 2026-04-12
---

When I wrote the [introductory post](https://theronconrey.github.io/2026/04/08/peerdup-no-cloud-required/) last week, peerdup worked — but "worked" was carrying a lot of heavy lifting in that sentance. The sync loop ran. Files moved between machines. The CLI did what you asked. But if you actually beat on it, you'd find a pile of edge cases that ranged from annoying to genuinely broken. So this weekend I wanted to get it sorted out.

# Sync engine

The biggest category of bugs. File sync sounds simple until you actually implement it, and then it turns out there are a lot of ways to get it wrong.

**Efficient renames and moves.** When a folder gets reorganized — files renamed, moved into subdirectories, whatever — peerdup would previously just re-transfer everything. That's wasteful and slow. The fix: when a peer receives a new torrent layout, peerdup now moves existing files into place *before* libtorrent checks pieces. If the data is already there under a different path, it gets repositioned, not re-downloaded. This works by matching files by name and size against the incoming layout and pre-positioning any matches. Stale files and empty directories from the prior layout get cleaned up automatically in the same pass.

**Ping-pong.** This one took a while to diagnose. Two peers would sync, both see the resulting filesystem changes, both react, and the two daemons would end up in an endless loop of overwriting each other. The fix was to pause the filesystem watcher during the SYNCING state so the daemon doesn't react to its own incoming changes. Queued events are drained cleanly when the transition to SEEDING completes.

**Bidirectional sync.** An earlier version had "owner immunity" — the share owner's version always won. That sounds reasonable until you think about what it means in practice: if the non-owner machine changed a file, that change could silently disappear. Reverted that. Both sides now accept remote changes, with sequence numbers on the LAN wire format determining which version wins when there's a genuine conflict.

**Bootstrap on empty shares.** Joining a share that had no files yet crashed the daemon. Fixed.

**Stale torrent cache.** If you deleted files from a share and then rebuilt it, the old `.torrent` cache could cause the deleted files to reappear like ghosts. The fix is simple: delete the cache before rebuilding.

# LAN discovery

The LAN multicast wire format went through three revisions this weekend.

v2 added peer names to multicast packets, so you can actually tell which machine is which when you run `peerdup share peers`. v3 added per-share `info_hash` to the packets — the daemon now detects when it's looking at a stale hash and automatically switches to the remote torrent. v4 added per-share sequence numbers, which is what makes conflict resolution work without a registry.

One subtle fix: the daemon was only sending multicast to the multicast group address, which works fine between devices on the same WiFi AP, but doesn't reach wired machines when the AP doesn't forward multicast. Added a subnet broadcast fallback to `255.255.255.255` so wired and wireless peers can find each other reliably.

Interface auto-detection also landed. Previously you had to know which network interface to use. Now the daemon picks the interface associated with the default gateway and `peerdup-setup` asks you to confirm it.

# Security

this is still a work in progress. not for prod use. yada yada yada.

**Announce rate limiting.** Each `(peer_id, share_id)` pair gets a token bucket. If a peer announces too aggressively, it gets a `RESOURCE_EXHAUSTED` response with a `Retry-After` header. The limit is configurable in `config.toml`.

**Audit logging.** Every authenticated RPC is now written to a structured JSON log via a rotating file handler (10 MB × 5 files). If something goes wrong, there's a record.

**TLS defaults.** Registry `tls.enabled` now defaults to `true` on new installs. Previously it defaulted to off, which was the wrong default.

# Observability

`peerdup status` now shows real health information: whether the database is reachable, whether the TTL sweep is running, an overall `ok`/`degraded`/`error` status, and peer and share counts. Previously it returned basically nothing useful.

`peerdup registry health` and `peerdup registry status` were added to let you inspect the registry connection directly — version, uptime, TLS config, token validity.

For those running their own infrastructure, there's also an optional Prometheus endpoint now. Counters for RPCs, announces, and peers online; histograms for latency and TTL sweep duration. Disabled by default, configurable in `config.toml`.

# Bandwidth policies

Share owners can now publish advisory rate limits from the registry. All members receive the policy via the live peer event stream and apply it to their libtorrent handle immediately. Local per-share caps always win — if you've set a local limit, the registry can't override it. `0` means unlimited.

# Operator tooling

`peerdup-registry-admin` is a new standalone CLI that connects directly to the registry gRPC without needing the daemon to be running. Useful for server-side administration: health checks, listing and removing peers, listing and purging shares, tailing the audit log.

`peerdup-setup` now walks through mTLS configuration if you're connecting to a registry that requires it — prompts for CA cert, client cert, and key, and writes them to `config.toml`.

# GNOME extension

A few quality-of-life improvements on the desktop side. The top bar now shows live upload/download rates inline while a share is actively syncing (`↑ 1.2M ↓ 4.8M`). The popup menu gained a colored registry health dot — green when connected and healthy, yellow when degraded, red when unreachable.

New share and join share dialogs are available via the popup menu when `zenity` is installed. Nothing fancy, just enough to avoid needing a terminal for common operations.

Fixed a PATH issue that caused the extension to fail to find the `peerdup` binary on some systems, since GNOME Shell's subprocess environment doesn't include `~/.local/bin`. Added GNOME Shell 49 to the supported versions list. The extension is now pre-enabled via `gsettings` during install so it activates on next login without a manual `gnome-extensions enable` step.

# Refactor

The sync coordinator had grown into a single file that was doing too many things. Split it into three focused modules: `announce.py` for registry heartbeats, `torrent_mgr.py` for the libtorrent handle lifecycle, and `peer_handler.py` for registry stream events, LAN peer handling, conflict dispatch, and policy application. Same behavior, easier to reason about.

# Tech debt

There was a gnarly `registry.proto` descriptor pool clash that was causing the integration test suite to fail intermittently. The daemon's registry stubs were replaced with static shim files that re-export from the registry package, and the registry's own `registry_pb2_grpc.py` was fixed to use an explicit relative import. The `Makefile` was updated to apply the fix automatically on `make proto`. All 21 integration tests pass now.

---

There's still a lot left to do. The Docker deployment path needs work — I mentioned that in the first post and it's still true. Broader platform testing. Documentation could go deeper. But the sync engine is noticeably more solid than it was Friday, and the observability and bits means it's something I can actually run on real machines with a better understanding of what is going on.

If you're using peerdup or just curious about any of this, feel free to open an issue or reach out directly.

<p style="text-align: center; margin-top: 1em;"><a href="https://github.com/theronconrey/peerdup">View on GitHub</a></p>
