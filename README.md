# How to safely virtualise an already installed (dual boot) Windows

**Warning: This is in draft and early testing.** It's working for me so far, but I haven't tested many senarios yet, please use at your own risk.

## Overview

I have Windows and Linux installed on the same SSD. I use Linux as my daily driver, and sometimes need to boot into Windows. This guide will enable you to:

1. Boot your already installed Windows from within Linux
2. Do it safely
3. With full CPU/video passthrough and 3D acceleration 
4. With hardware/OEM Windows Product Key passthrough, so you don't need to reactivate Windows

## Why would I do this?

A few reasons:

1. A raw install of Windows is easier to fix than a QCOW2 image if it becomes corrupted
2. I want Windows on an SSD, but I don't want to use an external SSD and my laptop doesn't have 2 Nvme ports
3. It gives me more options of how and when I can use Windows, on my own terms
4. It's kinda cool

## What are the risks

So far, I think the risks are:

1. Messing up your Linux drives, because you have to pass through the whole SSD to libvirt for this to work.
2. That your local Windows won't boot normally any more

(These are the risks I'm trying to mitigate.)

## Todo:

1. Use a loop device to access the Windows partition rather than accessing directly
2. Fix booting between virtualised and raw Windows

# Procedure

## Pre-setup

1. Install KVM (virt-manager)

`sudo dnf install virt-manager libvirt-client qemu-kvm-core`

2. Extract the Windows Activation data from the BIOS

```bash
midkr -p ~/.local/share/libvirt
cd ~/.local/share/libvirt
sudo cat /sys/firmware/acpi/tables/MSDM > msdm.bin
```

3. Extract hardware UUID and serial etc. from the BIOS

```bash
cd ~/.local/share/libvirt
sudo dmidecode -t 0 > smbios0.txt
sudo dmidecode -t 1 > smbios1.txt
```

_Note: I've copied these to /home because I'm exercising engineering laziness and don't need to remember to backup files in system folders 😜_

## Virtualise your dual boot Windows

In Virt-Manager:

1. Select "Manual"
2. Select "Windows 10"
3. Manually type the disk partition e.g. /dev/nvme0n1p2 for your Windows partition, leave it as SATA
4. Select e.g. 8192GB memory (half to 3/4 your RAM)
5. Select e.g. 8 logical host CPUs (half to 3/4 your threads)
6. Click "Customize"
7. Click "XML"
8. Change the first line to `<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">`
9. Add just above the last line `</domain>`, using the strings from your `smbios0.txt` and `smbios1.txt`

(Replace `username`, `uuid`, and `serial` with your own)

```xml
  <qemu:commandline>
    <qemu:arg value="-acpitable"/>
    <qemu:arg value="file=/home/insertyours/.local/share/libvirt/acpitable.bin"/>
    <qemu:arg value="-acpitable"/>
    <qemu:arg value="file=/home/insertyours/.local/share/libvirt/msdm.bin"/>
    <qemu:arg value="-smbios"/>
    <qemu:arg value="type=0,vendor=American Megatrends International LLC.,version=GA402RJ.315,date=09/21/2022,release=5.24"/>
    <qemu:arg value="-smbios"/>
    <qemu:arg value="type=1,manufacturer=ASUSTeK COMPUTER INC.,product=ROG Zephyrus G14 GA402RJ_GA402RJ,serial=insertyours,uuid=insertyours,family=ROG Zephyrus G14"/>
  </qemu:commandline>
</domain>
```

10. Click CPUs > Details
11. Click "Manually set topology"
12. Set e.g. Sockets = 1, Cores = 4, Threads = 2 to match the topology for the number of logical CPUs you've set
13. Remove "Tablet"

## Prepare to boot

1. Update the XML for the NVRAM and make it writable:

```bash
sudo cp /usr/share/OVMF/OVMF_VARS.fd $HOME/.config/libvirt/win10.nvram
sudo chmod u+w $HOME/.config/libvirt/win10.nvram
```

2. Change change the `<nvram>` entry in your XML to:

`<nvram>$HOME/.config/libvirt/win10.nvram</nvram>`

3. Click Boot Order > Check "Show boot menu"
4. Start the VM
5. Immediately hit ESC to enter the UEFI menu. We're going to program the NVRAM to boot the correct partition
6. Go into "Boot Management" > "Delete Boot Devices"
7. Delete everything except the UEFI > Confirm and Exit
8. Click "Add Boot Device"
9. You should be presented with your EFI partition. Select it
10. Select "EFI" > "Microsoft" > "bootmgfw.efi" (hidden at the bottom)
11. Enter a nice name, for example "Windows 10" > Confirm and exit
12. Select "Change Boot Order"
13. Move "Windows 10" to the top > Confirm and exit
14. Save and reboot

If all has gone well, your Windows should now boot 😊

## Clean up (necessary)

1. Right click the Start Menu > Device Manager
2. View menu > Show hidden devices
3. Expand Display Adapters
4. Right click any AMD adapters > Uninstall > and select "Delete drivers"
5. Do this for all AMD adapters, because we're about to pass them through, and will need a specific driver
6. (Reboot if requested)
7. Select virtio-win-latest.iso > Right-click > Mount
8. Open the mounted drive and install using the exe
9. Shutdown
10. Click Boot Order > Disable "Show boot menu"
11. Select the NIC > Change the model to virtio

## For the next steps and how to do VFIO passthrough, see

https://asus-linux.org/wiki/vfio-guide/
