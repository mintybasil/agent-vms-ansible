I'll walk you through setting up a KVM/libvirt VM on a Debian host with a Debian cloud image, Tailscale, and proper isolation.

## 1. Install KVM/libvirt on the Host

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virtinst cloud-image-utils
sudo usermod -aG libvirt,kvm $USER
# Log out and back in, or:
newgrp libvirt
```

Verify KVM is working:
```bash
virsh list --all
kvm-ok  # from cpu-checker package if you want to confirm hardware virt
```

---

## 2. Download the Debian Cloud Image

```bash
mkdir -p ~/vms/debian-ts
cd ~/vms/debian-ts

# Debian 12 (Bookworm) cloud image
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

---

## 3. Create a Persistent Disk from the Cloud Image

Don't use the cloud image directly — create a copy that will be your VM's persistent disk:

```bash
# Create a 20GB persistent disk backed by the cloud image
qemu-img create -f qcow2 -b debian-12-genericcloud-amd64.qcow2 -F qcow2 debian-ts-disk.qcow2 20G

# Or for a fully independent copy (recommended for clarity):
qemu-img convert -f qcow2 -O qcow2 debian-12-genericcloud-amd64.qcow2 debian-ts-disk.qcow2
qemu-img resize debian-ts-disk.qcow2 20G
```

Move it somewhere permanent:
```bash
sudo mv debian-ts-disk.qcow2 /var/lib/libvirt/images/debian-ts.qcow2
sudo chown libvirt-qemu:libvirt-qemu /var/lib/libvirt/images/debian-ts.qcow2
```

---

## 4. Create a cloud-init ISO

Cloud images require `cloud-init` for first-boot configuration (SSH keys, users, packages, etc.).

**`user-data`:**
```yaml
#cloud-config
hostname: debian-ts
fqdn: debian-ts.local

users:
  - name: debian
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...yourkey...  # <-- paste your public key here

package_update: true
package_upgrade: true

packages:
  - curl
  - ca-certificates
  - iptables

runcmd:
  # Install Tailscale
  - curl -fsSL https://tailscale.com/install.sh | sh
  # Enable IP forwarding (needed if using as exit node or subnet router)
  - echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/99-tailscale.conf
  - sysctl -p /etc/sysctl.d/99-tailscale.conf
  # Start and authenticate Tailscale (replace with your auth key)
  - tailscale up --authkey=tskey-auth-XXXXX --ssh
  # Enable tailscaled on boot
  - systemctl enable tailscaled

power_state:
  mode: reboot
  delay: now
  message: "Rebooting after cloud-init"
```

**`meta-data`:**
```yaml
instance-id: debian-ts-01
local-hostname: debian-ts
```

**`network-config`** (optional, DHCP by default):
```yaml
version: 2
ethernets:
  enp1s0:
    dhcp4: true
```

Build the ISO:
```bash
sudo apt install -y cloud-image-utils
cloud-localds ~/vms/debian-ts/cloud-init.iso user-data meta-data --network-config network-config
sudo mv ~/vms/debian-ts/cloud-init.iso /var/lib/libvirt/images/debian-ts-cloudinit.iso
```

---

## 5. Define and Start the VM

```bash
sudo virt-install \
  --name debian-ts \
  --ram 1024 \
  --vcpus 2 \
  --os-variant debian12 \
  --disk path=/var/lib/libvirt/images/debian-ts.qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/debian-ts-cloudinit.iso,device=cdrom \
  --network network=default,model=virtio \
  --graphics none \
  --console pty,target_type=serial \
  --noautoconsole \
  --import \
  --boot hd,cdrom
```

Watch cloud-init progress:
```bash
sudo virsh console debian-ts
# Ctrl+] to exit the console
```

---

## 6. Host Isolation (Minimal Host Access)

**Network isolation** — use libvirt's default NAT network (already applied above). The VM gets internet access but is firewalled from the host. To be more explicit:

```bash
# Verify the VM is on the NAT network, not bridged to host
virsh domiflist debian-ts
```

**No shared filesystems** — avoid `--filesystem` or `--shared-memory` flags in virt-install.

**Lock down QEMU** with a security profile. Edit `/etc/libvirt/qemu/debian-ts.xml` (after creation):

```bash
sudo virsh edit debian-ts
```

Add inside `<domain>`:
```xml
<seclabel type="dynamic" model="apparmor" relabel="yes"/>
```

**Restrict the virtual network further** (optional — block VM→host traffic):
```bash
# Drop traffic from VM network to host (192.168.122.0/24 is libvirt's default)
sudo iptables -I FORWARD -i virbr0 -o virbr0 -j ACCEPT
sudo iptables -I INPUT -i virbr0 -s 192.168.122.0/24 -d 192.168.122.1 -j DROP
# Make persistent:
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

## 7. Data Persistence & VM Lifecycle

```bash
# Stop/start cleanly
sudo virsh shutdown debian-ts
sudo virsh start debian-ts

# Autostart on host boot
sudo virsh autostart debian-ts

# Snapshot before risky changes
sudo virsh snapshot-create-as debian-ts snap1 "before update"

# View disk usage / confirm persistence
sudo qemu-img info /var/lib/libvirt/images/debian-ts.qcow2
```

---

## 8. Access the VM

Once Tailscale is up inside the VM, you can SSH directly via Tailscale IP or MagicDNS:

```bash
ssh debian@debian-ts  # via Tailscale MagicDNS
# or
ssh debian@100.x.x.x  # via Tailscale IP
```

You can also check the VM's local IP for initial access:
```bash
sudo virsh domifaddr debian-ts
ssh debian@<local-ip>
```

---

## Summary of Key Isolation Properties

| Property | How it's achieved |
|---|---|
| Network isolation | libvirt NAT (virbr0), no bridge to host |
| No host filesystem access | No `--filesystem` mounts |
| AppArmor confinement | `seclabel` in VM XML |
| Data persistence | Independent qcow2 disk in `/var/lib/libvirt/images/` |
| Remote access | Tailscale SSH (no open ports on host needed) |

The Tailscale `--ssh` flag is particularly useful here — it means you don't need to manage SSH keys long-term and can access the VM from anywhere on your tailnet without exposing any host ports.