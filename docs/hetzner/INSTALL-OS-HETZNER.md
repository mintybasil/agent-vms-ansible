# Installing the host OS on Hetzner Robot

This document describes how to install the minimal host OS on a Hetzner Robot dedicated server. The process is **manual** (not automated): you use the Rescue System and the installimage script. The target setup is **Debian 13 (Trixie) Base** — current stable — with **software RAID0** across **two NVMe drives**.

**Why Debian 13 from a security perspective:** Debian 13 (Trixie) is the current stable release and receives full support from the Debian Security Team for three years, plus LTS thereafter (to mid 2030). It ships with a newer kernel (6.12 LTS) and packages that include recent security fixes and hardening. Choosing current stable gives you the longest support runway and up-to-date mitigations. If your Rescue system’s installimage does not yet offer Debian 13, **Debian 12 (Bookworm)** remains a supported and secure fallback.

---

## Prerequisites

- A Hetzner Robot dedicated server with **two NVMe drives** (e.g. `/dev/nvme0n1`, `/dev/nvme1n1`).
- Robot login and (optional) a [webservice user](https://docs.hetzner.com/robot/dedicated-server/robot-interfaces/) if you use the rescue-prepare script.
- SSH key or ability to note the Rescue System password.

---

## Step 1: Activate the Rescue System

1. In [Robot](https://robot.hetzner.com/), select your server.
2. Open the **Rescue** tab.
3. Choose **Linux** (and 64-bit if prompted).
4. (Optional) Add an SSH public key so you can log in without the rescue password.
5. Click **Activate rescue system**. Note the displayed root password (or rely on your SSH key).
6. **Reset** the server (e.g. **Reset** → **Execute an automatic hardware reset**) so it boots once into Rescue.  
   Alternatively, use the script:  
   `HETZNER_USER=... HETZNER_PASS=... ./iac/scripts/hetzner-rescue-prepare.sh <server_number>`  
   and choose to trigger the hardware reset when prompted.

Rescue activation is valid for **one boot** only. If you don’t reset within about 60 minutes, you may need to activate Rescue again.

---

## Step 2: Log in to the Rescue System

1. Wait for the server to come back up (watch the server status in Robot).
2. SSH into the server as `root` using either:
   - The Rescue password from Robot, or  
   - Your SSH key if you added it when activating Rescue.

```bash
ssh root@<server_main_ip>
```

---

## Step 3: Prepare drives (if reusing a server)

If the server already had an OS or software RAID, stop existing RAID arrays and clear partition tables **before** running installimage:

```bash
# Stop any active software RAID
mdadm --stop /dev/md* 2>/dev/null || true

# Wipe partition tables (NVMe)
wipefs -fa /dev/nvme0n1
wipefs -fa /dev/nvme1n1
```

If the server is new or you’re sure the disks are empty, you can skip this step or run it to be safe.

---

## Step 4: Confirm block devices

List block devices and note the two NVMe data devices (no partition number):

```bash
lsblk
```

You should see something like:

- `nvme0n1` — first NVMe disk  
- `nvme1n1` — second NVMe disk  

Use these as `DRIVE1` and `DRIVE2` in the installimage config (e.g. `/dev/nvme0n1`, `/dev/nvme1n1`). If your server uses different names (e.g. `sda`/`sdb`), use those instead.

---

## Step 5: Run installimage

1. Start the script:

   ```bash
   installimage
   ```

2. In the menu, select **Debian 13** (Trixie) **Base** if available; otherwise **Debian 12** (Bookworm) **Base**. The script will open the configuration in an editor (e.g. mcedit).

3. Edit the config so it matches the **RAID0 + two NVMe** layout. You can use a preset from this repo as a template:
   - **Debian 13 (recommended):** `installimage/preset-debian13-raid0-nvme.txt`
   - **Debian 12 (fallback):** `installimage/preset-debian12-raid0-nvme.txt`  
   Or manually set at least:
     - **HOSTNAME** — e.g. `agent-host`
     - **DRIVE1** — `/dev/nvme0n1` (or your first NVMe block device)
     - **DRIVE2** — `/dev/nvme1n1` (or your second NVMe block device)
     - **SWRAID** — `1` (enable software RAID)
     - **SWRAIDLEVEL** — `0` (RAID0)
     - **PART** / **LV** lines — as in the preset (e.g. `/boot` 512M, LVM for root and swap).  
   **Important:** For RAID0, Hetzner recommends that `/boot` be on **RAID1** so the system can boot. If the installimage template uses a single RAID level for all partitions, check the script’s documentation or the editor comments; some setups use a separate small RAID1 for `/boot` and RAID0 for the rest. Adjust if your installimage version offers that.

4. Save and exit the editor (e.g. **F10** in mcedit). The script will check the config and then start the installation.

5. Wait for the installation to finish (usually a few minutes).

---

## Step 6: Reboot into the new system

When installimage reports success:

```bash
reboot
```

The server will boot from disk into the newly installed Debian. Rescue is one-time only, so the next boot will be the installed OS.

---

## Step 7: First login and next steps

1. SSH again to the server’s main IP as `root`. If you used an SSH key in Rescue, the same key is in the installed system’s `/root/.ssh/authorized_keys`, so you can log in with it (no password needed). If you used the Rescue password, that password is now the root password on the installed system.
2. **Lock down root and secure SSH** — follow [Step 8: Lock down root and secure SSH](#step-8-lock-down-root-and-secure-ssh) below so you use a dedicated admin user and key-only auth.
3. Proceed with the rest of the deployment from the [project plan](../PROJECT-PLAN.md): run Ansible (or your IaC) for host hardening, KVM, Tailscale, firewall, Promtail, then create and configure the VM.

---

## Step 8: Lock down root and secure SSH

Do this **while still logged in as root** (or in a second terminal). If you only have key-based root access and no saved password, this creates a dedicated admin user with your key and then disables direct root login so you don’t get locked out.

**Important:** Keep at least one root session open until you have confirmed you can log in as the new user and use `sudo`. Only then apply the SSH hardening and close the root session.

### 8.1 Create an admin user and add your SSH key

Pick a username (e.g. `admin` or your preferred name). On the server as root:

```bash
# Create user with a dedicated group (no password yet; we use keys only)
adduser --gecos "" --disabled-password admin

# Add user to sudo (and optionally sudo group for passwordless sudo with NOPASSWD)
usermod -aG sudo admin

# Copy your authorized_keys from root to the new user so you can log in with the same key
mkdir -p /home/admin/.ssh
cp /root/.ssh/authorized_keys /home/admin/.ssh/authorized_keys
chown -R admin:admin /home/admin/.ssh
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
```

**Home directory permissions:** SSH will refuse key auth if the user’s home directory is group- or world-writable. Ensure:

```bash
chown admin:admin /home/admin
chmod 755 /home/admin
```

### 8.2 Test the new user (do not skip)

In a **second terminal** (or another machine), log in as the new user:

```bash
ssh admin@<server_main_ip>
```

Then run `sudo -v` or `sudo whoami`. The admin user was created with no password, so sudo will prompt for one and fail until you either:

- **Set a password (recommended):** As root, run `passwd admin`, choose a strong password and store it in a password manager. Use that password when sudo asks.
- **Allow passwordless sudo:** As root, run:
  ```bash
  echo 'admin ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/90-admin-nopasswd
  chmod 440 /etc/sudoers.d/90-admin-nopasswd
  ```
  (Replace `admin` with your username if different.) Then sudo will not ask for a password; only your SSH key protects the account.

If that works, you have a working admin path. Leave this session open.

### 8.3 Harden SSH (key-only, no root login)

Back in the **root** session, edit the SSH daemon config:

```bash
# Backup
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Edit (nano, vim, or your preferred editor)
nano /etc/ssh/sshd_config
```

Set or add these (remove any conflicting lines):

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Optional but recommended:

```
MaxAuthTries 3
PermitEmptyPasswords no
X11Forwarding no
```

Check that the config is valid, then restart SSH:

```bash
sshd -t && systemctl restart sshd
```

(On some systems the service is `ssh` not `sshd`: use `systemctl restart ssh` if that’s the case.)

### 8.4 Verify and close root

In the **second terminal** (where you are logged in as the admin user), run `sudo -v` again or open a new SSH connection as `admin`. If everything works, you can close the root session. Root login is now disabled; all access is via the admin user and SSH keys.

### 8.5 Optional: set root password for recovery

If you want a known root password for rescue/maintenance (e.g. via console), set it only after the admin user is working:

```bash
sudo passwd root
```

Store it in a password manager; do not use it over the network. You can also leave root without a password and rely on Rescue or single-user mode if you have console access.

---

## Preset files

The repo includes presets for **two NVMe drives and RAID0**:

- **`installimage/preset-debian13-raid0-nvme.txt`** — Debian 13 (Trixie) Base, **recommended** (current stable, longest security support).

Copy the chosen preset to the Rescue system (e.g. via scp) or paste its contents into the installimage editor. Ensure **DRIVE1** and **DRIVE2** match your `lsblk` output (e.g. `/dev/nvme0n1`, `/dev/nvme1n1`). The **IMAGE** path must match what installimage shows for your chosen Debian version in the Rescue environment; if in doubt, run `installimage`, pick the version, and note the IMAGE path in the editor before applying the preset.

---

## References

- [Debian 13 “Trixie” release information](https://www.debian.org/releases/trixie/) — current stable, support timeline
- [Hetzner Rescue System](https://docs.hetzner.com/robot/dedicated-server/troubleshooting/hetzner-rescue-system/)
- [Installimage](https://docs.hetzner.com/robot/dedicated-server/operating-systems/installimage/)
- [Linux Software RAID (Hetzner)](https://docs.hetzner.com/robot/dedicated-server/raid/linux-software-raid/) — RAID0 and /boot as RAID1
- [Standard images](https://docs.hetzner.com/robot/dedicated-server/operating-systems/standard-images/)
