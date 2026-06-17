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

## 4. External HDD Spindown Configuration (Optional)

This section configures automatic HDD spindown for the external 750GB WD Elements drive attached to the homelab server.

The goal is to reduce power consumption, lower operating temperatures, and minimize unnecessary mechanical wear by automatically placing the drive into standby mode after **20 minutes of inactivity**.

### 1. Install hdparm

`hdparm` is the standard Linux utility for configuring drive power management and standby timers.

Install the package:

```bash
sudo apt update && sudo apt install hdparm -y
```

### 2. Verify Drive Compatibility

Before configuring a persistent standby timer, verify that the USB enclosure properly supports ATA power management commands.

#### Step 1: Force the Drive into Standby

Assuming the external HDD currently appears as `/dev/sdc`:

```bash
sudo hdparm -y /dev/sdc
```

The drive should immediately spin down and become silent.

#### Step 2: Check the Current Power State

Verify the drive status:

```bash
sudo hdparm -C /dev/sdc
```

Expected output:

```text
/dev/sdc:
 drive state is: standby
```

If the drive is still running:

```text
/dev/sdc:
 drive state is: active/idle
```

Successful standby confirmation indicates that the USB-to-SATA bridge supports the required ATA commands and can be managed through `hdparm`.

### 3. Configure the Standby Timeout

The `-S` parameter controls how long the drive must remain completely idle before entering standby mode.

For values between `1` and `240`:

```text
timeout = value × 5 seconds
```

To configure a 20-minute timeout:

```text
20 minutes = 1200 seconds
1200 ÷ 5 = 240
```

Test the timeout interactively:

```bash
sudo hdparm -S 240 /dev/sdc
```

This setting remains active until the next reboot.

### 4. Create a Persistent Configuration

Linux block device names such as `/dev/sdb` or `/dev/sdc` are assigned dynamically during boot and may change whenever storage devices are added, removed, or reordered.

To ensure the spindown policy survives reboots and hardware changes, use a persistent identifier from `/dev/disk/by-id/`.

#### Step 1: Locate the Persistent Device Identifier

List available device identifiers:

```bash
ls -l /dev/disk/by-id/
```

Example output:

```text
usb-WD_Elements_1023_575834314136313536353731-0:0 -> ../../sdc
```

This identifier remains stable even if the kernel later assigns a different device name.

#### Step 2: Configure hdparm

Open the configuration file:

```bash
sudo nano /etc/hdparm.conf
```

Append the following block at the bottom:

```ini
/dev/disk/by-id/usb-WD_Elements_1023_575834314136313536353731-0:0 {
    spindown_time = 240
}
```

Where:

```text
240 × 5 seconds = 1200 seconds = 20 minutes
```

#### Step 3: Apply the Configuration

On Debian 13, `hdparm` is typically applied through udev rules during device initialization rather than through a dedicated systemd service.

To ensure the new configuration is loaded, reboot the server:

```bash
sudo reboot
```

### 5. Verify After Reboot

After restarting the service or rebooting the system, verify that the configuration is active.

Check the drive power state:

```bash
sudo hdparm -C /dev/sdc
```

Expected output after sufficient idle time:

```text
/dev/sdc:
 drive state is: standby
```

## 5. Laptop Lid Close Tweak (Prevent Sleep)

By default, systemd will put a laptop to sleep when the lid is closed. Since this is a dedicated server, we need it to stay awake 24/7 with the lid closed.

#### 1. Open the systemd login configuration file

```bash
sudo nano /etc/systemd/logind.conf   
```

#### 2. Locate and modify the following lines (remove the # comment sign if present)

```bash
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

#### 3. Save and exit (Ctrl+O, Enter, Ctrl+X).

#### 4. Restart the systemd-logind service to apply changes immediately:

```bash
sudo systemctl restart systemd-logind
```

Now you can safely close the laptop lid and tuck the Vaio away near your router.

## 6 Memory Optimization: Setting up zram

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

## 7. Basic System Utilities

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
