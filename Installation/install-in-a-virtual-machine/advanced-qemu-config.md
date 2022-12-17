---
layout: post
title: 'Advanced tweaks for qemu'
date: '2021-10-27'
---
# Advanced configuration for qemu

Qemu is a very powerfull tool that may be confusing for some people. however when using it we can get some very detailed setups, and advanced setups, this page will document some more advanced features that folk may wish to utilize with qemu. the majority of this page will assume running on linux. many of these will not be compatible with windows. 

## Evdev

one of the more useful and easy to implement features of qemu is evdev passthrough. doing this can enable low latency and a better by using hotkeys to globally toggle device passthrough. 

there are two methods of doing this, we can use `virtio-input-host` and we can attach evdev to `virtio-mouse` and `virtio-keyboard`. using `virtio-input-host` will give the guest exclusive accsess to the device, and will implicitly setup the proper device. this may cause bugs, and is likely not what all users would like, however this might be helpful for unique devices that don't fall under traditional inputs, and won't work with usb passthrough 

`virtio-input-host` is fairly easy to setup simply add the below to add a multi-touch device through to the VM.

```
-device virtio-input-host,id=touch0,evdev=/dev/input/eventX
```

a minimal example of this would be 

```bash
 #!/bin/bash
 qemu-system-x86_64 \
 -enable-kvm \
 -M q35 -m 4096 -smp 4 -cpu host \
 -bios /usr/share/ovmf/x64/OVMF.fd \
 -drive file=disks/bliss.qcow2,if=virtio \
 -device virtio-vga-gl -display sdl,gl=on \
 -device virtio-input-host,id=touch0,evdev=/dev/input/eventx
```

attaching to `virtio-keyboard` and `virtio-mouse` are probably the route most will like to go. this will allow you to attach evdev to an existing device. we need to attach the below output to qemu corresponding device ids 

```
 -object input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME
 -object input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on
```

a functional example of this would be 

```bash
 #!/bin/bash
 qemu-system-x86_64 \
 -enable-kvm \
 -M q35 -m 4096 -smp 4 -cpu host \
 -bios /usr/share/ovmf/x64/OVMF.fd \
 -drive file=disks/bliss.qcow2,if=virtio \
 -device virtio-tablet,id-mouse1 \
 -device virtio-keyboard,id=kbd1 \
 -device virtio-vga-gl -display sdl,gl=on \
 -object input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME \
 -object input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on
```


## Flexible bash scripting

some folks who add and remove options from qemu may find that the current script setups are more of a hassle to use. however we can set up scripts using arrays instead this can allow users to comment out arguments from the script without breaking script, which is useful for prototyping. this does however pose some issues, that will be explained under the `booting a physical install` section


```bash
#!/bin/bash

## cd image "bliss.iso"

array=(
-enable-kvm
-M q35
-m 6000
-smp 3
-cpu host
-bios /usr/share/ovmf/x64/OVMF.fd
-drive file=android.qcow2,if=virtio
#-cdrom ~/Downloads/bliss.iso
-device virtio-tablet
-device virtio-keyboard
-device qemu-xhci,id=xhci
-machine vmport=off
-device virtio-vga-gl
-display sdl,gl=on
-net nic,model=virtio-net-pci -net user,hostfwd=tcp::4444-:5555
)

qemu-system-x86_64 ${array[@]}
```

## VFIO

Many folk may wish to pass a physical PCIe through to the VM, this can be helpful for people looking to make a gaming setup using bliss, or perhaps more unique setups. Qemu and Crosvm both allow doing this. with libvirt this is unbinding, and binding handled mostly automatically. you still need to load the vfio module yourself (see below).

first we need to prepare the device we are going to passthrough, the command to achieve this is below, however it requires root being done by root, not simply with root permissions, so the easiest way to do this is to use `su -c`. but first we need the pcie function addr and the pcie driver, to get this we can use `lspci | grep -i <vendor>` to find the function address. we can then use `lspci -vv FINISH THIS` to get the pci device driver too.



```
su -c 'echo <function_address> > /sys/bus/pci/drivers/<pci_device_driver>/unbind' 
```

after we have unmounted the drive we need to bind to vfio this is done in a similar manor with the command below however we need to load the vfio module and use the vid:pid of the device, 

```
sudo modprobe vfio
su -c 'echo vid:pid > /sys/bus/pci/drivers/vfio/bind' 
```

you can instead add this to module autoload using your init system, and example of a systemd setup would be 

```
FINISH THIS
```

after the card is loaded into VFIO we can then load it into qemu

```
FINISH THIS
```

To Be Finished

## Booting physical media

below is an example of booting a physical install where a second drive is mounted to `nvme` and bliss is installed in the folder called `android`. However there is a small issue, due to how arays work under bash, this can complicate how variables and strings work, meaning that sometimes it is better to simply avoid using them in the array, thankfully that does not pose too much issues in the majority of cases.


```bash
#!/bin/bash

install_location=/nvme/android

#common kernel arguments;
#root=/dev/ram0 console=ttyS0 HWC=drm_minigbm GRALLOC=minigbm_arcvm video=800x480 DATA=/dev/vdb quiet DEBUG=2
#'-drive index=0,if=virtio,id=system,file=${install_location}system.efs,format=raw,readonly=on'


args=(
    #CPU
    '-smp 4'
    '-M q35'
    '-m 4096'
    '-cpu host'
    '-accel kvm'
    #GPU
    '-device virtio-vga-gl,edid=on'
    '-display sdl,gl=on,show-cursor=true'
    #devices
    '-usb'
    #'-audiodev pa,id=snd0'
    #'-device AC97,audiodev=snd0'
    '-device virtio-tablet'
    '-device virtio-keyboard'

    #net
    '-net nic,model=virtio-net-pci -net user,hostfwd=tcp::4444-:5555'

    #drives
    '-drive index=0,if=virtio,id=system,file=/nvme/android/system.efs,format=raw,readonly=on'
    '-drive index=0,if=virtio,id=system,file=/nvme/android/data.img,format=raw,readonly=on'
    '-initrd /nvme/android/initrd.img'
    '-kernel /nvme/android/kernel'

    #misc
    '-monitor stdio'
    #'-serial mon:stdio'
)



qemu-system-x86_64 ${args[@]} \
      -append "root=/dev/ram0 console=ttyS0 HWC=drm_minigbm GRALLOC=minigbm_arcvm DATA=/dev/vdb" 
```

## USB passthrough
FINISH ME

## EGL-headless for remote VMs
FINISH ME

## Low latency Jack/Pipewire audio
FINISH ME