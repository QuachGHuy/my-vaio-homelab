# Journal: Fixing Ubuntu Freezes Caused by an Offline NFS Server

**Date:** June 24, 2026
**Status:** Resolved ✅
**Category:** Networking & Infrastructure

---

## Overview

While optimizing my Homelab storage network, I noticed an annoying issue on the Ubuntu client machine.

Whenever the NFS server (`192.168.100.199`) became unavailable, the desktop would start freezing or lagging, even though I was only trying to access local files.

This journal documents the root cause and the final solution.

---

## The Problem

The NFS share was configured using Systemd automount and mounted inside my home directory:

```text
/home/<username>/NAS
```

At first glance, this seemed perfectly reasonable. However, whenever the server went offline, several problems appeared:

* Nautilus (Files) became slow to open
* The Home directory took a long time to load
* Loading spinners appeared frequently
* Some applications temporarily froze while accessing files

The issue only occurred when the NFS server was unreachable.

---

## Root Cause Analysis

Although the share was configured with `x-systemd.automount`, the mount point itself was located inside the Home directory.

Modern file managers such as Nautilus automatically scan folders to gather information like:

* File counts
* Folder metadata
* Mounted devices
* Icons and thumbnails

Every time I opened my Home directory, Nautilus attempted to inspect everything inside it, including the NFS mount point.

When the NFS server was offline, the operating system had to wait for a network response before completing those filesystem requests. During that wait, Nautilus could become blocked, causing the entire desktop experience to feel frozen.

In short:

```text
Open Home Folder
        ↓
Nautilus scans ~/NAS
        ↓
NFS server is offline
        ↓
Filesystem waits for network response
        ↓
Nautilus hangs
        ↓
Desktop feels frozen
```

The issue was not caused by a software bug. It was the result of how Nautilus, the Linux Virtual File System (VFS), and NFS timeouts interact when a network-mounted directory becomes unavailable.

---

## The Solution

The fix was straightforward: move the NFS mount point outside the Home directory.

Instead of:

```text
/home/<username>/NAS
```

I created a dedicated mount location:

```text
/media/<username>/NAS
```

Create the directory:

```bash
sudo mkdir -p /media/<username>/NAS
sudo chown -R <username>:<username> /media/<username>/NAS
```

---

## Updating the NFS Configuration

Edit the client's `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Update the NFS entry:

```fstab
192.168.100.199:/mnt/nas_storage/<username>  /media/<username>/NAS  nfs  rw,soft,timeo=3,retrans=1,rsize=1048576,wsize=1048576,noatime,_netdev,x-systemd.automount,x-systemd.device-timeout=10,x-systemd.idle-timeout=5min  0  0
```

### Important Options

| Option                        | Purpose                                         |
| ----------------------------- | ----------------------------------------------- |
| `soft`                        | Return an error instead of hanging indefinitely |
| `timeo=3`                     | Reduce NFS timeout to 0.3 seconds               |
| `retrans=1`                   | Retry only once before failing                  |
| `_netdev`                     | Wait until networking is available              |
| `x-systemd.automount`         | Mount only when accessed                        |
| `x-systemd.device-timeout=10` | Limit wait time during mount attempts           |
| `x-systemd.idle-timeout=5min` | Automatically unmount after inactivity          |

These settings ensure the system fails quickly when the server is unavailable instead of blocking the desktop.

---

## Applying the Changes

Unmount any existing mounts:

```bash
sudo umount -f -l /home/<username>/NAS 2>/dev/null
sudo umount -f -l /media/<username>/NAS 2>/dev/null
```

Reload Systemd:

```bash
sudo systemctl daemon-reload
```

Start the automount service:

```bash
sudo systemctl start media-<username>-NAS.automount
```

Verify the status:

```bash
sudo systemctl status media-<username>-NAS.automount
```

Expected result:

```text
Loaded: loaded (/etc/fstab; generated)
Active: active (waiting)
```

---

## Results

After moving the NFS mount point outside the Home directory and reducing the NFS timeout values:

✅ Nautilus opens instantly, even when the server is offline.

✅ The Home directory remains fully responsive.

✅ Desktop navigation is smooth and unaffected by NFS availability.

✅ The NFS share is mounted only when explicitly accessed.

The key lesson is simple:

> Network mounts should generally not live inside directories that desktop applications constantly scan, such as the user's Home directory. Placing them in a dedicated mount location like `/media` helps isolate network-related delays from the normal desktop experience.

---

## Final Architecture

```text
Ubuntu Client
│
├── /home/<username>
│   └── Local files only
│
└── /media/<username>/NAS
        │
        └── NFS Automount
                │
                └── 192.168.100.199
```

With this layout, the desktop remains responsive during normal use, even when the NFS server is offline.
