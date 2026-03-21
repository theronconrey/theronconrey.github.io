---
layout: post
title: "btrfs RAID10 drive replacement and the fstab trap"
date: 2026-03-20
---

I run a six-drive btrfs RAID10 array on a home NAS (beepboop, an AMD Ryzen
box running Fedora). Recently one of the drives started accumulating
uncorrectable read errors and it was time to pull it. What followed was a
good reminder that btrfs's live replace capability is excellent, but there's
a fstab gotcha that will ruin your day if you don't know about it.

# The array

Six 5TB Seagate HDDs in a btrfs RAID10 configuration, mounted at `/mnt/data`.
RAID10 means the array can tolerate losing one drive without data loss, which
is the scenario we're dealing with.

Check array health with:

```bash
sudo btrfs device stats /mnt/data
```

When you start seeing non-zero `read_io_errs` or `corruption_errs` on a
device, it's time to act. In my case:

```bash
[/dev/sdg].read_io_errs    4
[/dev/sdg].write_io_errs   0
[/dev/sdg].flush_io_errs   0
[/dev/sdg].corruption_errs 0
[/dev/sdg].generation_errs 0
```

Four uncorrectable reads. The drive wasn't dead yet, but the trend was clear.


# The fstab trap

Before touching anything, there's something worth understanding about btrfs
RAID arrays and fstab.

By default, btrfs won't mount a RAID array in degraded mode, meaning if a
drive is missing at boot, the mount fails. On Fedora (and most systemd
distros), a failed mount during boot drops you into emergency mode. No
warning, no helpful message, just a root shell and a bad morning.

The fix is to add `degraded` to the mount options in `/etc/fstab` before
you pull any hardware:

```
UUID=your-uuid  /mnt/data  btrfs  defaults,degraded,nofail  0  0
```

The `nofail` option is worth adding too. It tells systemd to keep booting
even if the mount fails entirely, rather than halting for manual
intervention. On a headless machine, this is the difference between a
degraded array and a box you can't reach.

Verify fstab is valid before rebooting:

```bash
sudo mount -a
```


# Live drive replacement with btrfs replace

With fstab sorted, the actual drive replacement is straightforward. btrfs
has a first-class `replace` command that handles the swap while the array
stays mounted and in use.

Identify the device to replace and the new drive:

```bash
sudo btrfs filesystem show /mnt/data
```

This lists all devices in the array with their paths. Note the path of the
failing drive.

Start the replacement, which runs in the background:

```bash
sudo btrfs replace start /dev/sdg /dev/sdnew /mnt/data
```

Where `/dev/sdg` is the drive being replaced and `/dev/sdnew` is the freshly
installed drive. The array stays fully operational during this process.


# Monitoring the replacement

On a large array with spinning rust this takes several hours. Rather than
babysitting it, I set up a cron job to check hourly and log the status:

```bash
crontab -e
```

Add the following line:

```
0 * * * * btrfs replace status /mnt/data >> /var/log/btrfs-replace.log 2>&1
```

The output from `btrfs replace status` looks like this while running:

```
Started on 14.Mar 09:15:32, position 24.34%, speed 142.3 MiB/s
```

And when complete:

```
Started on 14.Mar 09:15:32, finished on 14.Mar 14:22:10, 0 write errs,
0 uncorr. read errs
```

Once you see the finished line, the cron job can be removed.


# After the replacement

Verify the array is healthy:

```bash
sudo btrfs device stats /mnt/data
sudo btrfs filesystem show /mnt/data
```

All error counters should be zero on the new device. A scrub is worth
running afterward to verify data integrity across all devices:

```bash
sudo btrfs scrub start /mnt/data
sudo btrfs scrub status /mnt/data
```


# The short version

- Add `degraded,nofail` to your btrfs fstab entry before you ever need it
- `btrfs replace` handles live drive swaps cleanly with no downtime
- Monitor with a cron job logging `btrfs replace status`
- Verify with `btrfs device stats` and follow up with a scrub
