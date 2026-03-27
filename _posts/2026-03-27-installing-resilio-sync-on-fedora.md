---
layout: post
title: "Installing Resilio Sync on Fedora the Right Way"
date: 2026-03-27
---

Most Resilio Sync guides on Linux stop at "download the binary and run it." That works until it doesn't — no updates, no service management, nothing integrated with the system. This guide does it the Fedora way: official RPM repo, `dnf`, `systemd`. It takes five minutes longer and you'll never have to think about it again.

# The repository

Resilio publishes an official RPM repository. Drop a repo file into `/etc/yum.repos.d/` — the standard location for third-party repos on Fedora — and import the GPG key:

```bash
sudo tee /etc/yum.repos.d/resilio-sync.repo << 'EOF'
[resilio-sync]
name=Resilio Sync
baseurl=https://linux-packages.resilio.com/resilio-sync/rpm/$basearch
enabled=1
gpgcheck=1
EOF

sudo rpm --import https://linux-packages.resilio.com/resilio-sync/key.asc
```

The GPG key step isn't optional — `dnf` will refuse the install without it.

# Installation

```bash
sudo dnf install resilio-sync
```

The package puts the binary at `/usr/bin/rslsync`, a default config at `/etc/rslsync.conf`, and registers systemd unit files. Nothing lands in your home or downloads directory.

# Running it as your own user

For a desktop, I tend to run Sync under my own account. It feels more integrated for personal files and keeps permissions simple.

The upstream unit file is written for system mode, so you need to fix the `WantedBy` target before enabling the user service — otherwise it won't start correctly in a user session:

```bash
sudo systemctl disable --now resilio-sync

sudo sed -i 's/WantedBy=multi-user.target/WantedBy=default.target/' \
  /usr/lib/systemd/user/resilio-sync.service

systemctl --user enable --now resilio-sync
```

If this is a machine you SSH into rather than log into interactively, enable lingering so the service survives outside of active login sessions:

```bash
loginctl enable-linger $USER
```

> **Server or NAS?** Skip all of the above and run `sudo systemctl enable --now resilio-sync` instead. That runs Sync as the dedicated `rslsync` system user, starting at boot. If it needs access to files owned by your account, add the users to each other's groups and set group-write permissions on your sync folders.

# The firewall

Fedora uses `firewalld`. If you need the WebUI from another machine or sync traffic across subnets:

```bash
sudo firewall-cmd --permanent --add-port=8888/tcp
sudo firewall-cmd --permanent --add-port=55555/tcp
sudo firewall-cmd --permanent --add-port=55555/udp
sudo firewall-cmd --reload
```

# First run

Verify the service is up and open `http://localhost:8888`:

```bash
systemctl --user status resilio-sync
```

You'll be prompted to set a username and password and name the device. From there you can add folders, generate share keys, and connect other machines.

# Updates

Because it's in a proper `dnf` repo, Resilio updates with everything else:

```bash
sudo dnf upgrade
```

No manual downloads, no version-checking scripts. The package manager owns it from here.
