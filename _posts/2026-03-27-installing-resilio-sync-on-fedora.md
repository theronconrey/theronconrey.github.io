---
layout: post
title: "Installing Resilio Sync on Fedora the Right Way"
date: 2026-03-27
---

Most Resilio Sync guides on Linux stop at "download the binary and run it." That works until it doesn't — no updates, no service management, nothing integrated with the system. This guide does it the Fedora way: official RPM repo, `dnf`, `systemd`. It takes five minutes longer and you'll never have to think about it again.

# Add the RPM Repository

Create a repo file at the standard location for third-party repositories:

```bash
sudo tee /etc/yum.repos.d/resilio-sync.repo << 'EOF'
[resilio-sync]
name=Resilio Sync
baseurl=https://linux-packages.resilio.com/resilio-sync/rpm/$basearch
enabled=1
gpgcheck=1
EOF
```

This drops a `.repo` file into `/etc/yum.repos.d/`, which is exactly where Fedora expects third-party repository definitions to live.

# Import the GPG Key

Red Hat-based systems require a valid GPG key before installing packages from a third-party repository:

```bash
sudo rpm --import https://linux-packages.resilio.com/resilio-sync/key.asc
```

Skipping this step will cause `dnf` to refuse the installation.

# Install via dnf

```bash
sudo dnf install resilio-sync
```

The package installs the binary to `/usr/bin/rslsync`, places a default config at `/etc/rslsync.conf`, and registers systemd unit files. Nothing ends up in your home or downloads directory.

# Run It as Your Own User

For a desktop setup, run Sync under your own user account — it's more natural when syncing personal files and keeps permissions simple.

First, make sure the system-level service is off:

```bash
sudo systemctl disable --now resilio-sync
```

The upstream unit file is written for system mode, so fix the `WantedBy` target before enabling the user service — otherwise it won't start correctly in a user session:

```bash
sudo sed -i 's/WantedBy=multi-user.target/WantedBy=default.target/' \
  /usr/lib/systemd/user/resilio-sync.service
```

Enable and start it for your user:

```bash
systemctl --user enable --now resilio-sync
```

If this is a machine you SSH into rather than log into interactively, enable lingering so the service starts and persists outside of active login sessions:

```bash
loginctl enable-linger $USER
```

> **Server or NAS?** Skip all of this and run `sudo systemctl enable --now resilio-sync` instead. That runs Sync as the dedicated `rslsync` system user, starting at boot before anyone logs in. If it needs access to files owned by your account, add the users to each other's groups: `sudo usermod -aG $USER rslsync && sudo usermod -aG rslsync $USER`, and set group-write permissions on your sync folders.

# Open the Firewall

Fedora uses `firewalld` by default. If you need to access the WebUI from another machine, or want sync traffic to work across subnets, open the relevant ports:

```bash
# WebUI
sudo firewall-cmd --permanent --add-port=8888/tcp

# Sync traffic (default port — change if you customized it)
sudo firewall-cmd --permanent --add-port=55555/tcp
sudo firewall-cmd --permanent --add-port=55555/udp

sudo firewall-cmd --reload
```

# Access the WebUI

Check that the service is running:

```bash
systemctl --user status resilio-sync
```

Then open a browser and navigate to `http://localhost:8888`. You'll be prompted to create a username and password and give your device a name. From there you can add sync folders, generate share keys or links, and connect other devices.

# Staying Updated

Because Resilio Sync is installed through a proper `dnf` repository, it receives updates alongside the rest of your system:

```bash
sudo dnf upgrade
```

No manual downloads, no version-checking scripts — it just works.
