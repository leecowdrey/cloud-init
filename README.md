# RHEL 9.x cloud-init example on KVM

## 1. Prepare cloud-init metadata

```bash
mkdir -p /var/lib/libvirt/images/example
cat <<EOF > /var/lib/libvirt/images/example/meta-data
#cloud-config
instance-id: example
hostname: example
local-hostname: example
EOF
cat <<EOF > /var/lib/libvirt/images/example/user-data
#cloud-config
timezone: UK/London
users:
  - name: root
    ssh_pwauth: False
    ssh_authorized_keys:
      - ssh-ed25519 ...
chpasswd:
  list: |
    root:pass1234
  expire: False
disable_root: false
growpart:
  mode: auto
  devices: ['/']
  ignore_growroot_disabled: false
write_files:
- encoding: b64
  content: ...==
  owner: root:root
  path: /root/.ssh/example
  permissions: '0400'
- encoding: b64
  content: ...==
  owner: root:root
  path: /root/.ssh/example.pub
  permissions: '0444'
- encoding: b64
  content: ...==
  owner: root:root
  path: /root/.ssh/config
  permissions: '0400'
runcmd:
 - "subscription-manager register --username <email> --password <password>"
 - "dnf -y update"
 - "dnf install -y git git-lfs qemu-guest-agent"
 - "ssh -o StrictHostKeyChecking=accept-new -T -p 22 github.com"
 - "git clone git@github.com:<org>/<repo>.git /root/<repo>"
 - "cd /root/<repo> && git lfs pull"
 - "chmod +x /root/<repo>/bootstrap/*.sh /root/<repo>/kickstart/*.sh"
 - "/root/<repo>/bootstrap/install.sh|tee /var/log/bootstrap.log"
final_message: "The system is finally up, after \$UPTIME seconds"
EOF
```

## 2. package cloud-init metadata as mountable ISO

```bash
pushd /var/lib/libvirt/images/example
genisoimage -output ../example.iso \
            -volid cidata \
            -joliet \
            -rock \
            user-data \
            meta-data
popd
```

## 3. Instantiate KVM guest with OS import and cloud-init metadata ISO

```bash
cp  /home/<id>/rhel-9.3-x86_64-kvm.qcow2 /var/lib/libvirt/images/example.qcow2 && \
qemu-img resize /var/lib/libvirt/images/example.qcow2 120G && \
virt-install --memory 8192 \
             --vcpus 8 \
             --name example \
             --disk /var/lib/libvirt/images/example.qcow2,device=disk,bus=virtio,format=qcow2 \
             --disk /var/lib/libvirt/images/example.iso,device=cdrom \
             --os-variant rhel9.1 \
             --virt-type kvm \
             --graphics none \
             --network bridge=br0,model=virtio \
             --import
```
