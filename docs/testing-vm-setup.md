# Testing VM Setup — Fedora / libvirt / KVM

This document covers the full process of setting up a local Ubuntu 24.04 LTS test VM on Fedora using libvirt/KVM and `virt-install`. This VM is used as the target for developing and testing Ansible roles before deploying to Proxmox.

---

## Prerequisites

### Required packages

The following packages must be installed:

```bash
sudo dnf install -y qemu-kvm libvirt-daemon libvirt-client virt-install
```

### User groups

Your user must be in the `kvm` and `libvirt` groups:

```bash
sudo usermod -aG kvm,libvirt $USER
```

> Log out and back in after adding groups for changes to take effect.

---

## Step 1 — Start the libvirt modular daemons

Fedora 38+ uses a modular libvirt architecture. Start the required daemons:

```bash
sudo systemctl start virtqemud virtnetworkd virtstoraged virtnodedevd virtlogd
```

To persist across reboots:

```bash
sudo systemctl enable virtqemud virtnetworkd virtstoraged virtnodedevd virtlogd
```

Verify they are running:

```bash
systemctl is-active virtqemud virtnetworkd virtstoraged
```

---

## Step 2 — Set up the default NAT network

Define and start the default NAT network:

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

Verify:

```bash
sudo virsh net-list --all
```

Expected output:

```
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

---

## Step 3 — Fix the libvirt default URI

By default, `virsh` connects to `qemu:///session` (user daemon). For networking and system-level VM management, it must use `qemu:///system`.

Add to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
export LIBVIRT_DEFAULT_URI="qemu:///system"
```

Reload:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

Verify:

```bash
virsh uri
# Expected: qemu:///system

virsh net-list --all
# Expected: default network listed as active
```

---

## Step 4 — Download the Ubuntu 24.04 LTS cloud image

Using the cloud image (`.img`) is faster than a full ISO install. It boots directly and accepts cloud-init configuration.

```bash
mkdir -p ~/libvirt/images
cd ~/libvirt/images

curl -LO https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

Verify the download:

```bash
ls -lh noble-server-cloudimg-amd64.img
```

---

## Step 5 — Create a cloud-init seed ISO

The cloud image requires a `cloud-init` seed to set credentials and SSH keys on first boot.

```bash
mkdir -p ~/libvirt/cloud-init
cd ~/libvirt/cloud-init
```

Create `meta-data`:

```bash
cat > meta-data <<EOF
instance-id: ansible-test-01
local-hostname: ansible-test-01
EOF
```

Create `user-data` (replace the SSH public key with your own):

```bash
cat > user-data <<EOF
#cloud-config
users:
  - name: ansible
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_ed25519.pub)

ssh_pwauth: false
chpasswd:
  expire: false

package_update: true
packages:
  - qemu-guest-agent
EOF
```

Generate the seed ISO:

```bash
sudo dnf install -y genisoimage   # install if not present

genisoimage \
  -output ~/libvirt/images/ansible-test-01-seed.iso \
  -volid cidata \
  -joliet \
  -rock \
  ~/libvirt/cloud-init/user-data \
  ~/libvirt/cloud-init/meta-data
```

---

## Step 6 — Create a writable disk from the cloud image

The cloud image is read-only by default. Create a copy to use as the VM disk:

```bash
cp ~/libvirt/images/noble-server-cloudimg-amd64.img \
   ~/libvirt/images/ansible-test-01.img

# Resize to 20GB
qemu-img resize ~/libvirt/images/ansible-test-01.img 20G
```

---

## Step 7 — Create the VM with virt-install

```bash
virt-install \
  --connect qemu:///system \
  --name ansible-test-01 \
  --memory 2048 \
  --vcpus 2 \
  --disk path=~/libvirt/images/ansible-test-01.img,format=qcow2 \
  --disk path=~/libvirt/images/ansible-test-01-seed.iso,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default \
  --graphics none \
  --console pty,target_type=serial \
  --import \
  --noautoconsole
```

> `--import` skips the installer and boots directly from the existing disk image.

---

## Step 8 — Get the VM IP address

Wait ~30 seconds for the VM to boot, then:

```bash
virsh domifaddr ansible-test-01
```

Or via the DHCP leases:

```bash
virsh net-dhcp-leases default
```

---

## Step 9 — Verify SSH access

```bash
ssh ansible@<VM_IP>
```

You should land in a shell without a password prompt.

---

## Step 10 — Add the VM to the testing inventory

Edit `inventories/testing/hosts.yml` and add the VM:

```yaml
all:
  hosts:
    ansible-test-01:
      ansible_host: <VM_IP>
      ansible_user: ansible
```

Test Ansible connectivity:

```bash
ansible -i inventories/testing/hosts.yml all -m ping
```

Expected output:

```
ansible-test-01 | SUCCESS => {
    "ping": "pong"
}
```

---

## VM Management Reference

```bash
# List all VMs
virsh list --all

# Start VM
virsh start ansible-test-01

# Graceful shutdown
virsh shutdown ansible-test-01

# Force off
virsh destroy ansible-test-01

# Delete VM and its disk
virsh undefine ansible-test-01 --remove-all-storage

# Open a console (Ctrl+] to exit)
virsh console ansible-test-01
```

---

## Troubleshooting

### `virsh` shows network as inactive but `sudo virsh` shows it as active
Your shell is connecting to `qemu:///session` instead of `qemu:///system`.
Fix: set `export LIBVIRT_DEFAULT_URI="qemu:///system"` in your shell profile (see Step 3).

### VM has no IP address
- Check the VM booted: `virsh list --all`
- Check cloud-init ran: `virsh console ansible-test-01` and look for cloud-init logs
- Ensure the seed ISO was correctly attached and the `user-data` syntax is valid YAML

### Permission denied on disk image
```bash
sudo chown $USER:$USER ~/libvirt/images/*.img
sudo chmod 660 ~/libvirt/images/*.img
```
