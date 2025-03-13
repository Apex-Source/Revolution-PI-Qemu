# Revolution-PI-Qemu
This repository is meant for those who want to run the Revolution PI in a Qemu environment. For whatever reason they want to.

# Prerequisites
1) Revolution PI Connect 4 or 5

2) The following packages:
```bash
sudo apt install bison flex libncurses-dev build-essentials libssl-dev
```

3) Optional: I have created a samba share to create a shared folder between my host machine and the RevPi.
   
## Steps

### Clone Repository
Git clone the Linux kernel for the RevPI, repo is 5G+ so it takes some time ;-)

### Make config
```bash
# Create a blank .config file to work against.
make revpi-v8_defconfig
# Menuconfig
make menuconfig
# In menu config, configure the modules how you want. Checkout RevPi4MenuConfig for my version of this config.
```

