# Run ubuntu cloud under firecracker

## Install

```
wget https://github.com/firecracker-microvm/firecracker/releases/download/v0.24.4/firecracker-v0.24.4-x86_64.tgz
tar xvzf firecracker-v0.24.4-x86_64.tgz
mv release-v0.24.4/firecracker-v0.24.4-x86_64 ~/.local/bin/firecracker
mv release-v0.24.4/jailer-v0.24.4-x86_64 ~/.local/bin/jailer
```

## Get kernel and initrd

```
wget -O initrd https://cloud-images.ubuntu.com/focal/20210603/unpacked/focal-server-cloudimg-amd64-initrd-generic
wget https://cloud-images.ubuntu.com/focal/20210603/unpacked/focal-server-cloudimg-amd64-vmlinuz-generic
wget https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux
chmod +x extract-vmlinux
./extract-vmlinux focal-server-cloudimg-amd64-vmlinuz-generic > vmlinux # firecracker requires an uncompressed kernel
```

## Build rootfs

```
dd if=/dev/zero of=rootfs.ext4 bs=1M count=3000
mkfs.ext4 rootfs.ext4
e2label rootfs.ext4 cloudimg-rootfs
mkdir /tmp/my-rootfs
sudo mount rootfs.ext4 /tmp/my-rootfs
cd /tmp/my-rootfs/
sudo tar xvJfp ~/Downloads/focal-server-cloudimg-amd64-root.tar.xz # TODO: check why some file don't have the right permissions
sudo umount /tmp/my-rootfs
```

## Prepare a tap interface

```
sudo ip tuntap add tap0 mode tap
sudo ip addr add 172.16.0.1/24 dev tap0
sudo ip link set tap0 up
```

## Cloud-init configuration

```
cloud-localds -v --network-config=network-config seed.iso user-data meta-data
```

## Start the VM

```
firecracker --api-sock /tmp/firecracker.socket --config-file config.json
```

## Stop the VM

From the guest:

```
reboot
```

From the host:

```
curl --unix-socket /tmp/firecracker.socket -i     -X PUT "http://localhost/actions"     -H  "accept: application/json"     -H  "Content-Type: application/json"     -d "{
             \"action_type\": \"SendCtrlAltDel\"
    }"
```

Note, you may have to delete `/tmp/firecracker.socket` before restarting the VM.
