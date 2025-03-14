# Revolution-PI-Qemu
This repository is meant for those who want to run the Revolution PI in a Qemu environment. For whatever reason they want to.

# Prerequisites
1) Revolution PI Connect 4 or 5
2) Windows 11 Qemu (winget install qemu-system)
3) A Shared folder between your host(Windows 11 machine in my case) and the RevPI.

4) The following packages:
```bash
sudo apt install bison flex libncurses-dev build-essentials libssl-dev
```

3) Optional: I have created a samba share to create a shared folder between my host machine and the RevPi.
   
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
sudo mount -t cifs //192.168.50.54/CloudShare /cloudshare -o username=****,password=*****
```


```bash
sudo qemu-system-aarch64 \
-M virt \
-m 4G \
-cpu cortex-a72 \
-kernel kernel8.img \
-initrd initramfs8 \
-append "root=/dev/mmcblk0p1 rootfstype=ext4 fsck.repair=yes rootwait console=ttyAMA0" \
-drive file=revpi.img,if=sd,media=disk,format=raw
```





