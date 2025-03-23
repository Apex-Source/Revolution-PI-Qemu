# Networking

> [!warning]
> Danger: Im a bit of a kernel n00b. So please do not follow any instruction from this page. If you do, you might see a lot of brainfarts which doesnt make sense to any senior (C/C++/Kernel) developer and it might even destroy things. Writing things down and sharing with the public is just a way how like learn stuff.

For now i have discovered that the piControl kernel module is missing a backend device. QEMU supports character devices.

## Scenario
Started QEMU instance with instructions from the guide. It should at least have a working internet connection. All other fancy stuf is out of scope here.

## Bridge Setup
This may change, since i have never get it working.
```
sudo ip link add name br0 type bridge
sudo ip link set br0 up
## qemu should create a nic.
## link that nic to the bridge.
sudo ip link set tapX master br0
```

## The QEMU instruction
```
sudo qemu-system-aarch64 -M virt -m 2048M -cpu max -smp 4 -kernel Image.gz  -drive file=revolution-pi-filesystem.img,format=raw,if=none,id=hd0 -initrd initramfs8  -append "kgdboc=ttyAMA0,115200 kgdbwait root=/dev/vda2 rootfstype=ext4 fsck.repair=yes rootwait console=ttyAMA0" -device virtio-blk-device,drive=hd0 -accel tcg -d guest_errors -D log3.txt -nic bridge,id=custom,br=br0 -serial mon:stdio
```

## Expected result
I would have expected that there was a network device available in the QEMU instance, but there is none...
when running 'ip a' i only see a loopback device (lo).

## Logs & Errors
PiControl does recognize the config.rsc file and device /dev/piControl0 is created.
```
[   29.051223] piControl: loading out-of-tree module taints kernel.
[   29.102901] piControl: RevPi Core
[   29.102973] piControl: MAJOR-No.  : 240
[   29.104840] piControl: MAJOR-No.  : 240  MINOR-No.  : 0
[   29.136767] piControl: read file finished, f_pos=1358
[   29.141784] piControl: 1 devices found
[   29.141845] piControl: 9 entries in total
[   29.142737] piControl: cl-comp:  0 addr  6  bit ff  len   8
[   29.142804] piControl: cl-comp:  1 addr 11  bit 00  len  16
[   29.154766] piControl: cannot find thermal zone
[   29.154839] piControl: piControlInit done
[   29.623598] i2c_dev: i2c /dev entries driver
```

After systemd is started, i see the following errors
```
# Wont start because of missing network device i guess (TCP)???
[FAILED] Failed to start ssh.servic…[0m - OpenBSD Secure Shell server.

# After login:
[   80.305692] piControl: problem at driver initialization
[   80.305752] piControl: no piControl reset possible, a firmware update is running
[   80.781912] piControl: problem at driver initialization
[   80.781958] piControl: no piControl reset possible, a firmware update is running
[   83.685748] piControl: problem at driver initialization
...

```

## The challenge
I have figured out that the piControl device which is created, is missing a backend device. Which might be created when using a DTB file.
The thread piIO Control is never started.

## DTB File missing?
I do not see any output from the instance when running with DTB file. Not sure why, im guessing something with the UART device defined in there.
But the DTB file describes the network devices and stuf, and it feels like this is missing. 

## WSL Limits?
Alpine works, at least it creates a network interface...

## piTest -x
Returns:
```
pi@RevPi100358:~$ piTest -x
Cannot reset: No such device
```

### DMESG of a Real RevPI:
```
[   29.272529] piControl: loading out-of-tree module taints kernel.
[   29.274401] piControl: RevPi Connect 4
[   29.274410] piControl: MAJOR-No.  : 240
[   29.274597] piControl: MAJOR-No.  : 240  MINOR-No.  : 0
[   29.291205] uart-pl011 fe201a00.serial: timeout while draining hardware tx queue
[   29.345715] systemd[1]: Starting systemd-remount-fs.service - Remount Root and Kernel File Systems...
[   29.394458] EXT4-fs (mmcblk0p2): re-mounted ecd26a72-871b-4b2a-90ff-294e4d095cb9 r/w. Quota mode: none.
[   29.703190] piControl: read file finished, f_pos=1357
[   29.703217] piControl: 1 devices found
[   29.703220] piControl: 9 entries in total
[   29.703237] piControl: cl-comp:  0 addr  6  bit ff  len   8
[   29.703240] piControl: cl-comp:  1 addr 11  bit 00  len  16
[   29.704384] piControl: piIO thread started
[   29.704389] piControl: RevPiDevice_init()
[   29.704397] piControl: Enter Init State
[   29.704564] piControl: piControlInit done
[   29.705298] piControl: Enter PresentSignalling1 State
[   29.713953] uart-pl011 fe201a00.serial: timeout while draining hardware tx queue
[   29.734870] i2c_dev: i2c /dev entries driver
[   29.746498] piControl: Enter InitialSlaveDetectionRight State
[   29.747354] piControl: Enter InitialSlaveDetectionLeft State
[   29.751694] piControl: Enter ConfigLeftStart State
[   29.762242] piControl: Enter ConfigDialogueLeft State
[   29.773136] piControl: Error sending gate request: -5
[   29.781351] piControl: piIoComm_sendRS485Tel(GetDeviceInfo) failed -5
[   29.792735] piControl: Error sending gate request: -5
[   29.801730] systemd[1]: Starting systemd-udev-trigger.service - Coldplug All udev Devices...
[   29.805382] piControl: piIoComm_sendRS485Tel(GetDeviceInfo) failed -5
[   29.816068] piControl: Error sending gate request: -74
[   29.825348] piControl: piIoComm_sendRS485Tel(GetDeviceInfo) failed -74
[   29.845359] piControl: GetDeviceInfo: Id 118
[   29.857370] piControl: found 2. device on left side. Moduletype 118. Designated address 31
[   29.857382] piControl: input offset     13  len  34
[   29.857385] piControl: output offset    47  len  27
[   29.857908] piControl: Enter SlaveDetectionLeft State
[   29.870302] piControl: Enter EndOfConfig State

[   29.870311] piControl: Device  0: Addr  0 Type 136  Act 1  In   6 Out   7
[   29.870316] piControl:            input offset      0  len   6
[   29.870318] piControl:            output offset     6  len   7
[   29.870321] piControl:            serial number 121332  version 1.0
[   29.870324] piControl: Device  1: Addr 31 Type 118  Act 1  In  34 Out  27
[   29.870327] piControl:            input offset     13  len  34
[   29.870329] piControl:            output offset    47  len  27
[   29.870331] piControl:            serial number 62119  version 1.2
[   29.870334] piControl:
[   29.870432] piControl: Adjust: base 0 in 0 out 6 conf 0
[   29.870438] piControl: After Adjustment
[   29.870440] piControl: Device  0: Addr  0 Type 136  Act 1  In   6 Out   7
[   29.870444] piControl:            input offset      0  len   6
[   29.870446] piControl:            output offset     6  len   7
[   29.870448] piControl: Device  1: Addr 31 Type 118  Act 0  In  34 Out  27
[   29.870452] piControl:            input offset     13  len  34
[   29.870454] piControl:            output offset    47  len  27
[   29.870456] piControl:
[   29.906024] systemd[1]: Started systemd-journald.service - Journal Service.
[   29.977348] piControl: start data exchange
[   30.197602] piControl: set BridgeState to running

```

### Key diff between virtual and real.
Im pretty sure that the overlay is not determined. (DTB?) 

Output Virtual:
```
[   29.051223] piControl: loading out-of-tree module taints kernel.
[   29.102901] piControl: RevPi Core
```
Output Real:
```
[   29.272529] piControl: loading out-of-tree module taints kernel.
[   29.274401] piControl: RevPi Connect 4
```

Missing in virtual:
```
[   29.977348] piControl: start data exchange
[   30.197602] piControl: set BridgeState to running
```
I am also receiving errors about missing EEPROM stuff which are also described in dtb.


## Brainfarts
Something is missing, but nut sure exactly what and therefore not sure how to emulate.
For now i'm thinking about an custom virtio driver for the revpi, to emulate the character backend for piControl.


### GDB Debugging
1) Check memory address of the piControl module
```
cat /sys/module/piControl/sections/.text #returns something like 0xffffffeda1f27000
gdb-multiarch piControl.ko 
cd $HOME/revpi-guide/piControl
add-symbol-file piControl.ko 0xffffffeda1f27000. #This tells GDB where to find the kernel module in the QEMU instance.

```
