# Revolution-PI-Qemu
This repository is meant for those who want to run the Revolution PI in a Qemu environment. For whatever reason they want to.

## My use case.
For software that i'm developing, it is crucial that everything works, since it is controlling civil
# My setup
```
Windows 11
WSL version: 2.4.12.0
WSL Kernel: 6.6.75.1-microsoft-standard-WSL2+
Ubuntu 24.04.2 LTS
QEMU version 9.2.2
```

## Prerequisites
1) Revolution PI Connect 4 or 5 / Raspberry PI 4 Compute Module
2) Boot image: [250234-bookworm-revpi-image.img](https://revolutionpi.com/fileadmin/downloads/images/250124-revpi-bookworm-arm64-lite.zip)
3) Windows 11 Qemu (winget install qemu-system)
4) A Shared folder between your host(Windows 11 machine in my case) and the RevPI.
5) The following packages installed on your RevPI:

```bash
sudo apt install bison flex libncurses-dev build-essentials libssl-dev
```

5) Optional: I have created a samba share to create a shared folder between my host machine and the RevPi.
   
## Steps

## Compiling kernel.

### Clone Repository
This aint probably required, but i just did. You might chose your own way to compile the kernel.
```
ssh user@revpi
cd /usr/src/linux
git clone https://github.com/RevolutionPi/linux.git
```

### Make config
This is pretty optional here, i wanted to enable some specific kernel debugging properties and Virtio drivers. Just configure how you want.
```bash
# Create a blank .config file to work against.
make revpi-v8_defconfig
# Menuconfig
make menuconfig
# In menu config, configure the modules how you want. Checkout RevPi4MenuConfig for my version of this config.
```
> [!important]
> Ensure that the kernel is configured with the virtio_blk module enabled:
```bash
CONFIG_VIRTIO_BLK=y
```
### Make
Please take not of the KERNELRELEASE argument. If you do not specify this arg, you will ending up build the general kernel version (Github version).
This might take a while...
```bash
sudo make KERNELRELEASE=$(uname -r) install
```

### Make install modules
This might take a while too...
```bash
sudo make KERNELRELEASE=$(uname -r) modules_install
```

### Make install
```bash
sudo make KERNELRELEASE=$(uname -r) install
```

### Make Image.gz
Finally run
```bash
sudo make KERNELRELEASE=$(uname -r) Image.gz
```

### Test your kernel
Reboot the RevPI and check with dmesg if everything still works. If everything works, you should be able to run
```bash
sudo modinfo virtio_blk
```
It should return something like:
```
name:           virtio_blk
filename:       (builtin)
license:        GPL
file:           drivers/block/virtio_blk
description:    Virtio block driver
parm:           num_request_queues:Limit the number of request queues to use for blk device. 0 for no limit. Values > nr_cpu_ids truncated to nr_cpu_ids. (uint)
parm:           poll_queues:The number of dedicated virtqueues for polling I/O (uint)
parm:           queue_depth:uint
```

### Install Initialize RAM File System Tools
```bash
sudo apt install initramfs-tools
```

### Create backup
```bash
cp /etc/initramfs-tools/modules /etc/initramfs-tools/modules.backup
```

### Add module 'virtio_blk' to /initramfs module.
```bash
vim /etc/initramfs-tools/modules
# or just
echo "virtio_blk" >> /etc/initramfs-tools/modules
```

### Update RAM File system
```
sudo update-initramfs -c -k $(uname -r)
```

### Copy resources to local WSL
```bash
mkdir $HOME/revpi-qemu
cd $HOME/revpi-qemu
scp user@revpi:/boot/firmware/kernel8.img $HOME/revpi-qemu
scp user@revpi:/boot/firmware/initramfs8 $HOME/revpi-qemu
```

### Create eMMC file system on WSL
```
cd $HOME/revpi-qemu
qemu-img create -f raw revolution-pi-filesystem.img 32G
```

### Connect NBD drive
This is required to use tools like fdisk or gparted, make sure nbd module is enabled!
```
sudo modprobe nbd
sudo qemu-nbd -f raw -c /dev/nbd0 revolution-pi-filesystem.img
```

### Format Drive with RPI Imager
```
sudo rpi-imager
```
1) Choose Raspberry PI Device - RevPI is based on Compute Module 4.
2) Choose OS, select downloaded bookworm revpi image (in my case 250234-bookworm-revpi-image.img)
3) Choose your NBD storage device. The version
4) Click next, and you will be asked to specify some default system settings, like networking, username and password, and more. Please modify accordingly.
5) Save the settings and procceed.
6) Close RPI Imager

### Create SWAP partition.
I have noticed that it complained about missing a swap partition, i created one with gparted.
You may also use fdsik or anything else.

### Disconnect NBD
```
sudo qemu-nbd -d /dev/nbd0
```

### Start QEMU
```bash
sudo qemu-system-aarch64 -M virt \
-m 4G \
-cpu max \
-smp 4 \
-kernel Image.gz  \
-accel tcg \
-initrd initramfs8  \
-append "root=/dev/vda2 rootfstype=ext4 fsck.repair=yes rootwait console=ttyAMA0" \
-drive file=revolution-pi-filesystem.img,format=raw,if=none,id=hd0 \
-device virtio-blk-device,drive=hd0 \
-serial mon:stdio
```
Command

### Done
When everything works, you should see something like: 


<img width="413" alt="image" src="https://github.com/user-attachments/assets/6a2dfeee-6a13-4280-8e3e-b09f5e170ab7" />




