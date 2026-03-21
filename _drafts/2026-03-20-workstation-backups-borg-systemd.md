---
layout: post
title: "Workstation backups with Borg and systemd"
date: 2026-03-20
---

I've been running Fedora on my primary workstation (borealis.home) for a
while and finally put together a backup setup I'm happy with. The goals
were straightforward: encrypted backups, two destinations, automatic
daily runs, and no babysitting.

The result is BorgBackup driven by a systemd timer, writing to a
LUKS-encrypted local drive and a secondary NFS share on a home NAS.

---

**Stack:** BorgBackup, systemd, LUKS, NFS

---

## Why Borg

Borg does three things well: deduplication, encryption, and compression.
Deduplication means each daily backup only stores what actually changed.
Encryption means the backup repo is useless without the passphrase. And
lz4 compression keeps things small without much CPU overhead. For a
workstation backup that runs at 2am every night, that combination is
hard to beat.

## Destinations

Two repos, one script:

- **Primary:** `/mnt/backup-local/borg` on a 500GB LUKS-encrypted SSHD
  mounted at boot. This is the fast local copy.
- **Secondary:** `/mnt/beepboop/borg` on a NAS via NFS. Geographic
  separation within the LAN, useful if the primary drive fails.

The secondary is non-fatal. If the NFS mount isn't present when the
backup runs, the script logs a warning and exits cleanly. The primary
result stands regardless.

## What Gets Backed Up

Rather than backing up the entire filesystem, I keep a tight source list
focused on what would be painful to lose or rebuild:

```bash
SOURCES=(
    /home/theron/.openclaw/workspace   # primary data and notes
    /usr/local/sbin                    # local scripts, including this one
    /etc/systemd/system                # unit files
    /etc/borg                          # Borg passphrase and recovery key
    /home/theron/.local/bin            # user scripts
    /home/theron/.ssh                  # SSH keys
)
```

These paths are sufficient to fully restore the machine's automation,
keys, and data from scratch. System packages are easy to reinstall;
this is the stuff that isn't.

## The Script

`/usr/local/sbin/borg-backup.sh` handles both destinations in sequence.
Each run creates a new archive, prunes old ones, and compacts the repo.
The pruning policy keeps 7 daily, 4 weekly, and 6 monthly archives.

Mountpoint checks run before each destination. If a mount is missing,
that destination is skipped and logged rather than failing the entire job.

## systemd Timer

A oneshot service and daily timer keep things running without cron:

```bash
# Check timer status
systemctl list-timers borgbackup.timer

# Run manually
systemctl start borgbackup.service

# Watch the log
journalctl -u borgbackup.service -f
```

The timer fires at 2am with a randomized 10-minute delay to avoid
thundering herd issues if multiple machines ever run on the same schedule.
`Persistent=true` means a missed run (machine was off) catches up on
next boot.

## Verifying It Works

```bash
# List archives in the primary repo
sudo BORG_PASSPHRASE=$(sudo cat /etc/borg/passphrase) \
  BORG_REPO=/mnt/backup-local/borg \
  borg list

# Check repo integrity
sudo BORG_PASSPHRASE=$(sudo cat /etc/borg/passphrase) \
  BORG_REPO=/mnt/backup-local/borg \
  borg check

# Browse a specific archive
sudo BORG_PASSPHRASE=$(sudo cat /etc/borg/passphrase) \
  BORG_REPO=/mnt/backup-local/borg \
  borg list ::archive-name
```

The log at `/var/log/borg-backup.log` captures every run with timestamps
and exit codes, so it's easy to spot if something went wrong overnight.
