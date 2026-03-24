# Testing VM Setup — Fedora / libvirt / KVM

Local Ubuntu 24.04 LTS test VM on Fedora using libvirt/KVM and `virt-install`. Used as the target for developing and testing Ansible roles before deploying to Proxmox.

---

## Prerequisites

### Packages

```bash
sudo dnf install -y qemu-kvm libvirt-daemon libvirt-client virt-install genisoimage
```

### User groups

Your user must be in the `kvm` and `libvirt` groups:

```bash
sudo usermod -aG kvm,libvirt $USER
```

> Log out and back in for group changes to take effect.

---

## Step 1 — Start the libvirt modular daemons

Fedora 38+ uses a modular libvirt architecture (`virtqemud` instead of the old `libvirtd`).

```bash
sudo systemctl start virtqemud virtnetworkd virtstoraged virtnodedevd virtlogd
sudo systemctl enable virtqemud virtnetworkd virtstoraged virtnodedevd virtlogd
```

Verify:

```bash
systemctl is-active virtqemud virtnetworkd virtstoraged
```

---

## Step 2 — Set up the default NAT network

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-start default
sudo virsh net-autostart default
```

Verify:

```bash
sudo virsh net-list --all
# Expected: default   active   yes   yes
```

> **Note:** if `net-start` fails with "network is already in use by interface virbr0",
> the network is already active — skip that step and just run `net-autostart`.

---

## Step 3 — Fix the libvirt default URI

By default `virsh` connects to `qemu:///session` (user daemon). Networking and system-level VM management require `qemu:///system`.

Add to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
echo 'export LIBVIRT_DEFAULT_URI="qemu:///system"' >> ~/.bashrc && source ~/.bashrc
```

Verify:

```bash
virsh uri
# Expected: qemu:///system

virsh net-list --all
# Expected: default listed as active
```

> **Note:** scripts and tools that spawn their own shell (e.g. Claude Code's Bash tool)
> won't inherit this variable. Pass `--connect qemu:///system` explicitly in those contexts.

---

## Step 4 — Download the Ubuntu 24.04 LTS cloud image

Using the cloud image avoids a full ISO install — it boots directly and accepts cloud-init for first-boot configuration.

```bash
mkdir -p ~/libvirt/images ~/libvirt/cloud-init

curl -L --progress-bar -o ~/libvirt/images/noble-server-cloudimg-amd64.img https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

> **Important:** pass the URL as a single unbroken line to avoid shell parsing errors.

Verify (~600 MB):

```bash
ls -lh ~/libvirt/images/noble-server-cloudimg-amd64.img
```

---

## Step 5 — Create the cloud-init seed ISO

The cloud image requires a seed ISO on first boot to configure the user and SSH keys.

Create `meta-data`:

```bash
cat > ~/libvirt/cloud-init/meta-data <<EOF
instance-id: ansible-test-01
local-hostname: ansible-test-01
EOF
```

Create `user-data` (uses your homelab SSH key):

```bash
cat > ~/libvirt/cloud-init/user-data <<EOF
#cloud-config
users:
  - name: ansible
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_ed25519_homelab.pub)

ssh_pwauth: false
chpasswd:
  expire: false

package_update: true
packages:
  - qemu-guest-agent
EOF
```

Generate the ISO:

```bash
genisoimage \
  -output ~/libvirt/images/ansible-test-01-seed.iso \
  -volid cidata -joliet -rock \
  ~/libvirt/cloud-init/user-data \
  ~/libvirt/cloud-init/meta-data
```

---

## Step 6 — Prepare the VM disk

Copy the cloud image (keep the original as a base for future VMs) and resize it:

```bash
cp ~/libvirt/images/noble-server-cloudimg-amd64.img \
   ~/libvirt/images/ansible-test-01.img

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
  --disk path=$HOME/libvirt/images/ansible-test-01.img,format=qcow2 \
  --disk path=$HOME/libvirt/images/ansible-test-01-seed.iso,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default \
  --graphics none \
  --import \
  --noautoconsole
```

> `--import` boots directly from the existing disk image, skipping any installer.
> `$HOME` is used instead of `~` to avoid path expansion issues with `virt-install`.

---

## Step 8 — Get the VM IP address

Wait ~30 seconds for the VM to boot and cloud-init to complete, then:

```bash
virsh net-dhcp-leases default
```

Or:

```bash
virsh domifaddr ansible-test-01
```

---

## Step 9 — Verify SSH access

```bash
ssh -i ~/.ssh/id_ed25519_homelab ansible@<VM_IP>
```

You should get a shell without a password prompt.

---

## Step 10 — Add the VM to the testing inventory

Edit `inventories/testing/hosts.yml`:

```yaml
all:
  hosts:
    ansible-test-01:
      ansible_host: <VM_IP>
      ansible_user: ansible
```

Test Ansible connectivity:

```bash
ansible -i inventories/testing/hosts.yml all -m ping \
  --private-key ~/.ssh/id_ed25519_homelab
```

Expected:

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

# Open serial console (Ctrl+] to exit)
virsh console ansible-test-01

# Delete VM and its disk
virsh undefine ansible-test-01 --remove-all-storage
```

---

## Troubleshooting

### `virsh net-start` fails: "network already in use by virbr0"
The bridge interface exists but libvirt hasn't claimed it. Skip `net-start` and just run:
```bash
sudo virsh net-autostart default
```
The network will be fully active after the next `virtnetworkd` restart.

### `virsh` shows network as inactive but `sudo virsh` shows it as active
Your shell is connecting to `qemu:///session` instead of `qemu:///system`.
Fix: see Step 3.

### VM has no IP address
- Confirm the VM is running: `virsh list --all`
- Check cloud-init completed: `virsh console ansible-test-01`
- Confirm the seed ISO was attached and `user-data` is valid YAML

### SSH: permission denied
- Confirm you're using the right key: `ssh -i ~/.ssh/id_ed25519_homelab ansible@<VM_IP>`
- Check cloud-init injected the key: connect via console and inspect `~ansible/.ssh/authorized_keys`
