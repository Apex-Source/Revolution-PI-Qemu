# Revolution-PI-Qemu
This repository is meant for those who want to run the Revolution PI in a Qemu environment. For whatever reason they want to.

## Prerequisites
1) Revolution PI Connect 4 or 5 / Raspberry PI 4 Compute Module
2) Windows 11 Qemu (winget install qemu-system)
3) A Shared folder between your host(Windows 11 machine in my case) and the RevPI.
4) The following packages:

```bash
sudo apt install bison flex libncurses-dev build-essentials libssl-dev
```

5) Optional: I have created a samba share to create a shared folder between my host machine and the RevPi.
   
## Steps

## Compiling kernel.

### Clone Repository
Git clone the Linux kernel for the RevPI, repo is 5G+ so it takes some time ;-)

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
Finally run
```bash
sudo make KERNELRELEASE=$(uname -r) install
```

## Collect Qemu resources.
Copy the compiled files to the shared folder.qq
###

### Mount Share
```bash
sudo mount -t cifs //192.168.XX.XX/CloudShare /cloudshare -o username=****,password=*****
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
```
### Create Image
```
qemu-img create -f raw rpi-boot.img 32G
```

### Connect NBD drive
```
qemu-nbd -c /dev/ndb0 rpi-boot.img
```

### Format Drive with RPI Imager
```
# I did use the UI here.
rpi-imager /dev/nbd0 250234-bookworm-revpi-image.img cm4
```

### Disconnect NBD
```
qemu-nbd -d /dev/nbd0
```

### Start QEMU
```bash
qemu-system-aarch64 -M virt -m 4G -cpu max -kernel Image.gz  -drive file=rpi-boot.img,format=raw,if=none,id=hd0 -serial mon:stdio -initrd initramfs8  -append "root=/dev/vda2 rootfstype=ext4 fsck.repair=yes rootwait console=ttyAMA0" -device virtio-blk-device,drive=hd0
```

### Ubuntu comparison test
I had a lot of trouble mounting the root filesystem. I compiled the kernel with VIRTIO enabled, but it didn’t work. I had to modify the initrd module configuration to get it working. I compared the Ubuntu boot output with the output from the RevPi image and immediately noticed that the VIRTIO kernel module was missing at boot time on the RevPi.
```bash
qemu-system-aarch64 -M virt -m 4G -cpu max -smp 4 -kernel Image.gz  -drive file=rpi-boot.img,format=raw,if=none,id=hd0 -initrd initramfs8  -append "root=/dev/vda2 rootfstype=ext4 fsck.repair=yes rootwait console=ttyAMA0" -device virtio-blk-device,drive=hd0 -accel tcg
```



