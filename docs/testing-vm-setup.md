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

> **Note:** scripts and tools that spawn their own shell won't inherit this variable.
> Pass `--connect qemu:///system` explicitly in those contexts.

---

## Step 4 — Fix firewalld: assign virbr0 to the libvirt zone

libvirt should assign `virbr0` to the `libvirt` firewalld zone automatically, but on
Fedora this sometimes doesn't happen until the network is restarted. Without this,
DHCP is blocked and VMs will never get an IP address.

Check active zones:

```bash
sudo firewall-cmd --get-active-zones
```

If `virbr0` is missing from the output, restart the libvirt network to trigger the assignment:

```bash
sudo virsh net-destroy default && sudo virsh net-start default
```

Verify `virbr0` now appears under the `libvirt` zone:

```bash
sudo firewall-cmd --get-active-zones
# Expected:
# libvirt
#   interfaces: virbr0
```

---

## Step 5 — Download the Ubuntu 24.04 LTS cloud image

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

## Step 6 — Create the cloud-init seed ISO

The cloud image requires a seed ISO on first boot to configure users and SSH keys.

### Generate a hashed password for console access

```bash
openssl passwd -6 "your-password"
```

### Create meta-data

```bash
cat > ~/libvirt/cloud-init/meta-data <<EOF
instance-id: ansible-test-01
local-hostname: ansible-test-01
EOF
```

### Create user-data

Replace `YOUR_SSH_PUBLIC_KEY` with the output of `cat ~/.ssh/id_ed25519_homelab.pub`
and `YOUR_HASHED_PASSWORD` with the output of the `openssl passwd` command above.

```bash
cat > ~/libvirt/cloud-init/user-data <<EOF
#cloud-config
users:
  - name: ansible
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - YOUR_SSH_PUBLIC_KEY
  - name: ubuntu
    groups: sudo
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    hashed_passwd: "YOUR_HASHED_PASSWORD"

ssh_pwauth: true
chpasswd:
  expire: false

package_update: false
packages:
  - qemu-guest-agent
EOF
```

> The `ubuntu` user with a console password is useful for debugging via `virsh console`
> when SSH is not yet reachable. `ssh_pwauth: true` enables password login on the console
> only — harden this after initial setup if needed.

### Generate the seed ISO

```bash
sudo genisoimage \
  -output ~/libvirt/images/ansible-test-01-seed.iso \
  -volid cidata -joliet -rock \
  ~/libvirt/cloud-init/user-data \
  ~/libvirt/cloud-init/meta-data
```

> `sudo` is required here because libvirt may have taken ownership of the images directory.

---

## Step 7 — Prepare the VM disk

Copy the cloud image (keep the original as a clean base for future VMs) and resize it:

```bash
cp ~/libvirt/images/noble-server-cloudimg-amd64.img \
   ~/libvirt/images/ansible-test-01.img

qemu-img resize ~/libvirt/images/ansible-test-01.img 20G
```

---

## Step 8 — Create the VM with virt-install

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

## Step 9 — Get the VM IP address

Wait ~30 seconds for the VM to boot and cloud-init to complete, then:

```bash
virsh net-dhcp-leases default
```

Or check from inside the VM via console (see below).

---

## Step 10 — Verify via console (optional)

Connect to the serial console:

```bash
virsh console ansible-test-01
```

Login with `ubuntu` / `<your-password>`. Check the network interface:

```bash
ip addr show
```

Exit the console with **`Ctrl+]`**.

---

## Step 11 — Verify SSH access

```bash
ssh -i ~/.ssh/id_ed25519_homelab ansible@<VM_IP>
```

You should get a shell without a password prompt.

---

## Step 12 — Add the VM to the testing inventory

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

## Rebuilding the VM from scratch

If cloud-init has already run on a disk, it won't run again on reboot. To fully reset:

```bash
virsh destroy ansible-test-01 && virsh undefine ansible-test-01 && \
sudo genisoimage -output ~/libvirt/images/ansible-test-01-seed.iso -volid cidata -joliet -rock ~/libvirt/cloud-init/user-data ~/libvirt/cloud-init/meta-data && \
cp ~/libvirt/images/noble-server-cloudimg-amd64.img ~/libvirt/images/ansible-test-01.img && \
qemu-img resize ~/libvirt/images/ansible-test-01.img 20G && \
virt-install --connect qemu:///system --name ansible-test-01 --memory 2048 --vcpus 2 --disk path=$HOME/libvirt/images/ansible-test-01.img,format=qcow2 --disk path=$HOME/libvirt/images/ansible-test-01-seed.iso,device=cdrom --os-variant ubuntu24.04 --network network=default --graphics none --import --noautoconsole
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

# Open serial console (exit with Ctrl+])
virsh console ansible-test-01

# Delete VM and its disk
virsh undefine ansible-test-01 --remove-all-storage
```

---

## Troubleshooting

### VM gets no IP address
Most likely cause: `virbr0` is not in the `libvirt` firewalld zone, so DHCP is blocked.
Fix: restart the libvirt network (see Step 4).

### `virsh net-start` fails: "network already in use by virbr0"
The bridge interface already exists. Skip `net-start` and run `net-autostart` only.
The network will be fully managed after the next `virtnetworkd` restart.

### `virsh` shows network as inactive but `sudo virsh` shows it as active
Your shell is connecting to `qemu:///session` instead of `qemu:///system`.
Fix: see Step 3.

### Console login fails
The default cloud image `ubuntu` user has no password. Rebuild the VM with a
`hashed_passwd` in `user-data` (see Step 6).

### `genisoimage` permission denied on the seed ISO
libvirt took ownership of the file after the VM ran. Use `sudo genisoimage` to overwrite it,
or destroy the VM first.

### SSH: permission denied
Confirm you are using the right key: `ssh -i ~/.ssh/id_ed25519_homelab ansible@<VM_IP>`
