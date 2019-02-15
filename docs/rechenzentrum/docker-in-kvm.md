# Docker in QEMU/KVM

Some applications may require a properly isolated Docker engine where users of the API have every freedom but when they must not be able to compromise the host security. Since access to the Docker socket is equivalent to being `root` ([or worse](https://opensource.com/article/18/10/podman-more-secure-way-run-containers)) we must preferably run the engine on a seperate machine.

Long story short: virtualization with QEMU/KVM provides all the required isolation and CoreOS is easy to deploy and bundles Docker by default.

The following steps are designed for a CentOS 7.6 hypervisor.

## Prerequisites

First of all, we need to prepare our hypervisor, so install QEMU and libvirt.

    yum install qemu-kvm libvirt
    modprobe kvm
    systemctl enable --now libvirtd

!!! info
    We are going to use `virt-install` as well, however the version in EPEL is not recent enough to use the `kernel=` and`initrd=` arguments with `--location`. Thus prefer a local manager and append `--connect qemu+ssh://root@hypervisor/system` to `virsh` or `virt-install` commands.

## Boot a Virtual Machine

There is [a guide](https://coreos.com/os/docs/latest/booting-with-libvirt.html) on how to boot CoreOS with libvirt but I prefer to perform a clean installation to disk. Therefore we need to boot CoreOS to RAM and deploy using an Ignition configuration.

### Via Text Console

If you don't want to bother with VNC connections and would prefer to install via a text
console on the hypervisor itself, you can download and run the CoreOS `vmlinuz` and `cpio.gz` directly:

```sh
virt-install --name runner --memory 4096 --vcpus 4 \
  --disk size=20,bus=virtio --hvm --rng /dev/urandom \
  --autostart --nographics --console pty,target_type=virtio \
  --location "https://stable.release.core-os.net/amd64-usr/current/,kernel=coreos_production_pxe.vmlinuz,initrd=coreos_production_pxe_image.cpio.gz" \
  --extra-args "coreos.autologin console=ttyS0" \
  --os-variant virtio26
```

!!! note
    Remember to append a `--connect` string if you are connecting to a remote libvirt socket.

If you prefer to download and verify [an ISO](https://stable.release.core-os.net/amd64-usr/current/coreos_production_iso_image.iso) locally instead, you can substitute:

      --location /tmp/coreos.iso,kernel=/coreos/vmlinuz,initrd=/coreos/cpio.gz \

!!! hint
    You can find files inside an ISO with `isoinfo -Jf -i /path/to/iso`.

### Via VNC Viewer

Apart from using VNC, we are going to use [netboot.xyt](https://netboot.xyz) in this approach. This is possible with `virt-install` version 1.5.0 on CentOS 7.

First, download the netboot image:

    cd /var/lib/libvirt/boot
    curl -LO https://boot.netboot.xyz/ipxe/netboot.xyz.iso

Now create the virtual machine with `virt-install`:

    virt-install --name runner --memory 4096 --vcpus 4 \
      --disk size=20,bus=virtio --hvm --rng /dev/urandom \
      --autostart --cdrom /var/lib/libvirt/boot/netboot.xyz.iso \
      --graphics vnc,listen=0.0.0.0 --noautoconsole \
      --os-variant virtio26

This should start the installation process and enable a VNC console. You can check the port with
`virsh vncdisplay runner` and verify with `ss -tln`. In my case `:0` corresponds to port 5900 on the
host, so temporarily open that port in the firewall:

    firewall-cmd --add-port 5900/tcp

Connect with your favourite VNC client and complete the installation.

!!! hint
    You can't currently change the keyboard map on the console. Set a password with
    `sudo passwd core` and connect with `ssh` instead if you run into problems.

## Install CoreOS to Disk

### Prepare Ignition

By now you should have prepared an Ignition configuration. There is of course a lot of variation possible here but most importantly you should enable `rngd.service` and `docker.service` and make sure that you can connect with SSH public keys. Mine looks somewhat like this:

```yaml
---
# enable docker service
systemd:
  units:
    - name: rngd.service
      enabled: yes
    - name: docker.service
      enabled: yes

# ssh public keys
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - # add your keys here

# automatic updates during maintenance window
locksmith:
  reboot_strategy: reboot
  window_start: 04:00
  window_length: 3h

# enable console autologin
storage:
  filesystems:
    - name: OEM
      mount:
        device: /dev/disk/by-label/OEM
        format: ext4
  files:
    - filesystem: OEM
      path: /grub.cfg
      mode: 0644
      append: true
      contents:
        inline: |
          set linux_append="$linux_append coreos.autologin"
```

### Installation

After transpiling, I am using [surge.sh](https://surge.sh) to host small static files quickly. Download the configuration and finally install CoreOS to disk:

```sh
curl -LO "https://ks.surge.sh/coreos/docker.json"
sudo coreos-install -d /dev/vda -i docker.json
sudo udevadm settle
sudo reboot
```

## Miscellaneous

### Fixed DHCP Address

You can add a fixed address for this virtual machine by creating an IP assignment for its MAC address with `virsh`:

```sh
virsh net-dhcp-leases default   # see current leases
virsh net-update default add-last ip-dhcp-host \
  --xml "<host mac='52:54:00:e7:b6:4d' ip='192.168.122.2' />" \
  --live --config
```

### SSH Client Configuration

Add an appropriate SSH config on the hypervisor:

    Host runner
      User core
      HostName 192.168.122.2
      StrictHostKeyChecking no
      UserKnownHostsFile /dev/null
