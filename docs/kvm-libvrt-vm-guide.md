# Running VMs on Debian 13 with KVM and libvirt

A concise guide for setting up and launching a virtual machine from scratch on a fresh Debian 13 install.

---

## 1. Prerequisites: Verify KVM Support

Before installing anything, confirm your CPU supports hardware virtualisation:

```bash
grep -Eoc '(vmx|svm)' /proc/cpuinfo
```

A non-zero number means you're good. `vmx` = Intel VT-x, `svm` = AMD-V.

Also confirm KVM kernel modules are loaded:

```bash
lsmod | grep kvm
```

You should see `kvm_intel` or `kvm_amd` (plus `kvm`) in the output. If not, they may need enabling in your BIOS/UEFI.

---

## 2. Install KVM, libvirt, and Supporting Tools

```bash
sudo apt update
sudo apt install --no-install-recommends \
    qemu-system-x86 \
    libvirt-daemon-system \
    libvirt-clients \
    virtinst \
    bridge-utils \
    qemu-utils
```

| Package | Purpose |
|---|---|
| `qemu-system-x86` | The actual hypervisor/emulator |
| `libvirt-daemon-system` | libvirt daemon + default network |
| `libvirt-clients` | `virsh` CLI tool |
| `virtinst` | `virt-install` helper (optional but useful) |
| `bridge-utils` | Network bridge management |
| qemu-utils | |

Enable and start the daemon:

```bash
sudo systemctl enable --now libvirtd
```

Add your user to the `libvirt` group so you can manage VMs without `sudo`:

```bash
sudo usermod -aG libvirt $USER
newgrp libvirt   # Apply without logging out
```

---

## 3. Set Up the Default Network

libvirt ships with a NAT-based virtual network. Activate it and set it to auto-start:

```bash
sudo virsh net-start default
sudo virsh net-autostart default
```

Verify it's active:

```bash
virsh net-list --all
```

You should see `default` listed as **active**.

---

## 4. Create a VM Disk Image

VM disks live (by default) in `/var/lib/libvirt/images/`. Create a 20 GB disk for your VM using the `qcow2` format, which supports snapshots and thin provisioning:

```bash
sudo qemu-img create -f qcow2 /var/lib/libvirt/images/myvm.qcow2 20G
```

Verify it was created:

```bash
sudo qemu-img info /var/lib/libvirt/images/myvm.qcow2
```

---

## 5. Prepare an Install ISO

Place your OS installation ISO somewhere accessible, e.g.:

```bash
sudo cp ~/Downloads/debian-13-amd64-netinst.iso /var/lib/libvirt/images/
```

Or download directly:

```bash
sudo wget -O /var/lib/libvirt/images/debian-13-amd64-netinst.iso \
    https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-13.0-amd64-netinst.iso
```

---

## 6. Define the VM Using an XML Definition

libvirt manages VMs as XML domain definitions. Create the file below, adjusting the name, memory, vCPUs, and paths to suit your needs.

```bash
sudo nano /etc/libvirt/qemu/myvm.xml
```

Paste in:

```xml
<domain type="kvm">
  <name>myvm</name>
  <uuid><!-- leave blank; libvirt will generate one on define --></uuid>

  <memory unit="MiB">2048</memory>
  <currentMemory unit="MiB">2048</currentMemory>

  <vcpu placement="static">2</vcpu>

  <os>
    <type arch="x86_64" machine="pc-q35-8.2">hvm</type>
    <boot dev="cdrom"/>
    <boot dev="hd"/>
  </os>

  <features>
    <acpi/>
    <apic/>
  </features>

  <cpu mode="host-passthrough" check="none" migratable="off"/>

  <clock offset="utc">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
  </clock>

  <devices>
    <!-- Primary disk -->
    <disk type="file" device="disk">
      <driver name="qemu" type="qcow2"/>
      <source file="/var/lib/libvirt/images/myvm.qcow2"/>
      <target dev="vda" bus="virtio"/>
    </disk>

    <!-- Install ISO (remove after installation) -->
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/var/lib/libvirt/images/debian-13-amd64-netinst.iso"/>
      <target dev="sda" bus="sata"/>
      <readonly/>
    </disk>

    <!-- Network interface on the default NAT network -->
    <interface type="network">
      <source network="default"/>
      <model type="virtio"/>
    </interface>

    <!-- VirtIO serial console -->
    <console type="pty">
      <target type="virtio" port="0"/>
    </console>

    <!-- Graphics: VNC on localhost, auto-assigned port -->
    <graphics type="vnc" port="-1" listen="127.0.0.1"/>
    <video>
      <model type="virtio"/>
    </video>

    <!-- Recommended: memory balloon for dynamic memory -->
    <memballoon model="virtio"/>

    <!-- Recommended: virtio RNG for better entropy in guest -->
    <rng model="virtio">
      <backend model="random">/dev/urandom</backend>
    </rng>
  </devices>
</domain>
```

> **Note on the UUID field:** Remove the `<!-- comment -->` and leave `<uuid></uuid>` or simply delete the UUID line entirely — libvirt will assign a UUID automatically when you define the domain.

---

## 7. Register and Start the VM

**Define** the VM (registers it with libvirt without starting it):

```bash
sudo virsh define /etc/libvirt/qemu/myvm.xml
```

**Start** it:

```bash
sudo virsh start myvm
```

Check it's running:

```bash
virsh list --all
```

---

## 8. Connect to the VM

### Via VNC

The XML configured VNC on `127.0.0.1` with an auto-assigned port. Find the port:

```bash
virsh vncdisplay myvm
# Output example: 127.0.0.1:0  (port 5900)
```

Connect with a VNC client:

```bash
vncviewer 127.0.0.1:5900
```

### Via Serial Console (text-only, no VNC client needed)

```bash
virsh console myvm
```

Press `Ctrl+]` to detach.

---

## 9. Post-Installation Cleanup

Once the OS is installed, remove the CDROM from the VM definition so it doesn't try to boot the ISO again. Edit the live XML:

```bash
sudo virsh edit myvm
```

Delete the entire `<disk type="file" device="cdrom">...</disk>` block, save, and reboot the VM:

```bash
virsh reboot myvm
```

---

## 10. Essential virsh Commands

| Task | Command |
|---|---|
| List all VMs | `virsh list --all` |
| Start a VM | `virsh start myvm` |
| Graceful shutdown | `virsh shutdown myvm` |
| Force off | `virsh destroy myvm` |
| Delete VM (keep disk) | `virsh undefine myvm` |
| Delete VM + disk | `virsh undefine myvm --remove-all-storage` |
| View VM config | `virsh dumpxml myvm` |
| Edit VM config | `virsh edit myvm` |
| Take a snapshot | `virsh snapshot-create-as myvm snap1` |
| Autostart on boot | `virsh autostart myvm` |

---

## Troubleshooting

**`error: failed to connect to the hypervisor`** — ensure `libvirtd` is running (`systemctl status libvirtd`) and your user is in the `libvirt` group (log out and back in after adding).

**`KVM not available`** — check BIOS/UEFI has virtualisation enabled, and run `kvm-ok` (from the `cpu-checker` package) for a clearer diagnostic.

**Network not reachable from guest** — verify the `default` network is active (`virsh net-list`) and that IP forwarding is enabled on the host (`sysctl net.ipv4.ip_forward`; set to `1` in `/etc/sysctl.conf` to persist).