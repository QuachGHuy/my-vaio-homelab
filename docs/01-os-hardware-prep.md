# OS Installation & Hardware Preparation

This document guides you through the initial phase of turning the legacy Sony Vaio laptop into a dedicated, stable headless server. It covers OS installation flavor, laptop-specific power tweaks, LVM verification, and external mass-storage setup.

---

## Target Hardware

This guide is written for my Sony Vaio SVE14136CVB:

- Intel Core i5-3230M
- 4GB DDR3 RAM
- Samsung 860 EVO 250GB SSD
- WD 750GB HDD
- Debian 13 Headless

The same steps should work on most x86 laptops with minor adjustments.

## 1. Debian 13 Headless Installation

To minimize resource overhead, we install a clean, minimal version of Debian.

### Steps

1. Download the official **Debian 13 (Trixie) Netinst ISO**.
2. Flash it to a USB drive using Rufus or BalenaEtcher.
3. Boot the Sony Vaio into the BIOS/UEFI (Press `F2` or the `ASSIST` button during startup) and change the boot order to prioritize the USB drive.
4. Select **Graphical Install** or **Install** (Standard text mode).
5. During **Software Selection**, uncheck any Desktop Environments (GNOME, XFCE, etc.) to keep it strictly headless.
6. **Mandatory selections:**
   - `[x] SSH server` (For remote management)
   - `[x] Standard system utilities`

---

## 2. Storage Setup & LVM Verification

During the Debian installation partition phase, we use LVM (Logical Volume Manager) on the internal SSD (sda) to optimize system responsiveness.

### Verify LVM Structure

Run the following command to verify your internal storage allocation layout:

```bash
lsblk -f
```

Example layout used in this homelab:

- /boot (~1GB): Standard ext4 partition for system kernel boot files.

- LVM Volume Group (homelab-vg):

  - vg_root (~38GB formatted to ext4): Mounted at / for system OS runtime.
  - vg_docker (~210GB formatted to ext4): Mounted at /var/lib/docker to host all high-IOPS container databases and volumes.

## 3. External HDD Mounting (Mass-Storage Partition)

The external 750GB HDD (sdb) acts as our NAS data vault. We mount it permanently using its unique UUID to avoid mounting issues if the USB cable is replugged into a different port.

1. Create a permanent directory for the mount point:

    ```bash
    sudo mkdir -p /mnt/nas_storage
    ```

2. Identify the UUID of your external partition (sdb1):

    ```bash
    sudo lsblk -f
    ```

    Copy the UUID string (e.g., UUID="xxxx-xxxx-xxxx").

3. Open the file system table configuration:

    ```bash
    sudo nano /etc/fstab
    ```

4. Add the following line at the bottom of the file (replace with your actual UUID and file system type, e.g., ext4):

    ```text
    UUID=your-actual-uuid-here /mnt/nas_storage auto defaults,nofail,x-systemd.device-timeout=5 0 0
    ```

    With `nofail` and `x-systemd.device-timeout=5`, the homelab:
    - Waits up to 5 seconds for the drive to appear.
    - If not found, logs an error but continues booting.
    - The server runs normally, except that partition is not mounted.

5. Mount the drive instantly without rebooting:

    ```bash
    sudo mount -a
    ```

6. Verify the mount status:

    ```bash
    df -h /mnt/nas_storage
    ```

### External HDD Spindown Configuration

This document provides a standalone guide to configuring automated power management for the external 750GB mechanical HDD attached via USB to the Debian 13 headless server.

By enforcing an automated spindown (sleep) policy after **20 minutes** of absolute idle time, the homelab minimizes electricity consumption, reduces overall thermal output, and extends the mechanical lifespan of the drive.

---

#### 1. Prerequisites & Installation

The standard tool used to manage Advanced Power Management (APM) and spindown timeouts on Linux storage drives is `hdparm`.

Install the utility directly onto the host OS:

```bash
sudo apt update && sudo apt install hdparm -y
```

#### 2. Manual Spindown Verification

Before writing persistent configuration profiles, verify that your external USB-to-SATA enclosure bridge chip properly supports and passes down ATA power management commands.
Step 1: Force Standby Mode

Execute the command below to immediately force the drive (assumed as /dev/sdb) to spin down:

##### Step 1: Force Standby Mode

Execute the command below to immediately force the drive (assumed as /dev/sdb) to spin down:

```bash
sudo hdparm -y /dev/sdb
```

Listen closely to the physical drive enclosure. You should distinctively hear the drive platters spin down and go completely silent.

##### Step 2: Check Power Status Safely

To verify the drive state without sending a command that inadvertently wakes it back up, run:

```bash
sudo hdparm -C /dev/sdb
```

**Expected Output:**

```text
/dev/sdb:
 drive state is:  standby
```

(If the drive is running, it will output: drive state is: active/idle)

#### 3. Idle Timeout Calculation

The hdparm -S flag dictates the idle timeout threshold. The value parsing logic follows strict rules defined by the Linux kernel storage subsystem:

- Values from 1 to 240 are multiplied by 5 seconds.
- Values from 241 to 251 are multiplied by 30 minutes.

To calculate a precise 20-minute standby window:

- Convert minutes to seconds: 20 minutes = 1200 seconds.
- Divide by the multiplier: 1200 seconds / 5 seconds = 240.

Test the timeout threshold live on the device block:

```bash
sudo hdparm -S 240 /dev/sdb
```

#### 4. Persistent Configuration via UUID Mapping

External USB drives are prone to shifting block letters (e.g., swapping from /dev/sdb to /dev/sdc) during reboots or USB bus resets. To prevent configuration failures, lock the spindown configuration permanently using the disk's unique UUID.

##### Step 1: Locate the Drive UUID

Run lsblk to fetch the UUID of your mass storage partition:

```bash
sudo lsblk -f
```

Copy the exact UUID string mapping to your external partition.

##### Step 2: Append Rules to hdparm Configuration

Open the system daemon configuration file:

```bash
sudo nano /etc/hdparm.conf
```

Scroll to the absolute bottom of the file and append the following block, substituting your actual block UUID:

```ini
/dev/disk/by-uuid/your-actual-uuid-here {
    spindown_time = 240
}
```

##### Step 3: Restart the Service

Save changes and exit the text editor (Ctrl+O, Enter, Ctrl+X), then restart the hdparm daemon to instantly apply the rule:

```bash
sudo systemctl restart hdparm
```

## 4. Laptop Lid Close Tweak (Prevent Sleep)

By default, systemd will put a laptop to sleep when the lid is closed. Since this is a dedicated server, we need it to stay awake 24/7 with the lid closed.

1. SSH into your newly installed Debian server.
2. Open the systemd login configuration file:

   ```bash
   sudo nano /etc/systemd/logind.conf   
   ```

3. Locate and modify the following lines (remove the # comment sign if present):

    ```bash
    HandleLidSwitch=ignore
    HandleLidSwitchExternalPower=ignore
    HandleLidSwitchDocked=ignore
    ```

4. Save and exit (Ctrl+O, Enter, Ctrl+X).

5. Restart the systemd-logind service to apply changes immediately:

   ```bash
   sudo systemctl restart systemd-logind
   ```

Now you can safely close the laptop lid and tuck the Vaio away near your router.

## 5 Memory Optimization: Setting up zram

To expand our tight 4GB RAM capacity and protect the SSD from heavy SWAP wear, we utilize zram with `zstd` compression.

1. Install the zram-tools package:

   ```bash
   sudo apt update && sudo apt install zram-tools -y
   ```

2. Configure the zram parameters:

   ```bash
   sudo nano /etc/default/zramswap
   ```

3. Uncomment and edit the configuration to allocate 2GB of compressed SWAP:

   ```text
   ALGORITHM=zstd
   SIZE=2048
   PRIORITY=100
   ```

4. Restart the zramswap service:

   ```bash
   sudo systemctl restart zramswap
   ```

5. Verify that your system is successfully running the compressed in-memory SWAP layer:

   ```bash
   swapon --show
   ```

   You should see /dev/zram0 listed with a size of 2G.

## 6. Basic System Utilities

To keep the system lightweight, only a small set of essential administration and monitoring tools are installed directly on the host.

```bash
sudo apt install ufw htop sysstat -y
```

Most day-to-day administration is performed through Cockpit's web interface. Only a minimal collection of command-line tools is installed on the host.

### * UFW (Uncomplicated Firewall)

UFW is used as the primary host firewall to restrict unnecessary network access and expose only the services required by the homelab.

Verify firewall status:

```bash
sudo ufw status verbose
```

### * htop

Provides a real-time interactive view of:

- CPU utilization
- Memory usage
- Running processes
- System load

Launch:

```bash
htop
```

### * sysstat

Provides historical and real-time performance monitoring tools that are especially useful on resource-constrained hardware.

Common commands:

```bash
# Disk I/O statistics
iostat -x

# Per-CPU statistics
mpstat -P ALL
```

These tools help identify performance bottlenecks, monitor resource usage trends, and validate optimization efforts on aging hardware.
