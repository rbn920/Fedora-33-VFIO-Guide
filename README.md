# Fedora 33 VFIO Guide
This is how I got gpu passthrough working on Fedora 33. It is based on the guide at [level1techs](https://forum.level1techs.com/t/fedora-33-ultimiate-vfio-guie-for-2020-2021-wip/163814/31). The original post has a few typos in it. The corrections can be found lower in the post but I wanted to have a source with it all in one place.

## Pre-Tasks 
### ssh
It is a good idea to make sure you can ssh into the box in case you have issues and it boots up with no screen.
```sh
sudo dnf install openssh-server
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload
sudo systemctl start sshd
sudo systemctl enable sshd
```

### IOMMU Groups
create a file called `iommu_groups.sh` and place the following inside:
```sh
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
Make it executable with `chmod +x iommu_groups.sh` and run it `./iommu_groups.sh`.
This will let you make show you what groups your hardware is in. You can only pass whole groups through. There are some workarounds but that is outside of scope of this guide.

My guest gpu groups:
```sh
IOMMU Group 22:
	33:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 XL/XT [Radeon RX Vega 56/64] [1002:687f] (rev c1)
IOMMU Group 23:
	33:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 HDMI Audio [Radeon Vega 56/64] [1002:aaf8]
```

## Installing Packages
Install the `virtualization` package group and then add yourself the the `libvert` group so you don't have to run `virt-manager` as root/sudo.
```sh
sudo dnf install @virtualization
sudo usermod -a -G libvirt YOUR_USERNAME
```
## Configure Grub
AMD:
```sh
sudo grubby --update-kernel=ALL --args="amd_iommu=on rd.driver.pre=vfio-pc"
```
Intel:
```sh
sudo grubby --update-kernel=ALL --args="intel=on rd.driver.pre=vfio-pc"
```

## Inital Ramdisk
Create the file `/usr/sbin/vfio-pci-override.sh` and place the following inside. Modify the `DEVS=` line for your setup. Make sure to use the correct PCIe device prefix, mine is `0000`.
```sh
#!/bin/sh
PREREQS=""
DEVS="0000:33:00.0 0000:33:00.1"

for DEV in $DEVS; do
        echo "vfio-pci" > /sys/bus/pci/devices/$DEV/driver_override
done

modprobe -i vfio-pci
```

Change the permissions with:
```sh
sudo chmod 755 /usr/sbin/vfio-pci-override.sh
```

Next create the a directory and below:
```sh
sudo mkdir /usr/lib/dracut/modules.d/20vfio
```

Create the file `/usr/lib/dracut/modules.d/20vfio/module-setup.sh` and place the following inside:
```sh
#!/usr/bin/bash
check() {
  return 0
}
depends() {
  return 0
}
install() {
  declare moddir=${moddir}
  inst_hook pre-udev 00 "$moddir/vfio-pci-override.sh"
}
```

and change the permissions:
```sh
sudo chmod 755 /usr/lib/dracut/modules.d/20vfio/module-setup.sh
```

Create a symbolic linkto the `vfio-pci-override.sh` script we created earlier:
```sh
sudo ln -s /usr/sbin/vfio-pci-override.sh /usr/lib/dracut/modules.d/20vfio/vfio-pci-override.sh
```

Create the file `/etc/dracut.conf.d/vfio.conf` and add the following:
```sh
add_dracutmodules+=" vfio "
force_drivers+=" vfio vfio-pci vfio_iommu_type1 "
install_items="/usr/sbin/vfio-pci-override.sh /usr/bin/find /usr/bin/dirname"
```

Finally regenerate the initramfs with:
```sh
sudo dracut -fv
```

If you run `sudo lsinitrd | grep vfio` you should see something similar to below:
```sh
etc/modprobe.d/vfio.conf
usr/lib/dracut/hooks/pre-udev/00-vfio-pci-override.sh -> ../../../../sbin/vfio-pci-override.sh
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio/pci
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio/pci/vfio-pci.ko.xz
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio/vfio_iommu_type1.ko.xz
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio/vfio.ko.xz
usr/lib/modules/5.10.18-200.fc33.x86_64/kernel/drivers/vfio/vfio_virqfd.ko.xz
usr/sbin/vfio-pci-override.sh
```

Reboot and cross your fingers....

### Post Reboot
Assuming everything came up fine after the reboot check to see that the `vfio-pci` kernel drivers loaded:
```sh
lspci -nnv
```

```sh
.
.
.
33:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 XL/XT [Radeon RX Vega 56/64] [1002:687f] (rev c1) (prog-if 00 [VGA controller])
	.
	.
	.
	Kernel driver in use: vfio-pci
	Kernel modules: amdgpu
3:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Vega 10 HDMI Audio [Radeon Vega 56/64] [1002:aaf8]
	.
	.
	.
	Kernel driver in use: vfio-pci
	Kernel modules: snd_hda_intel
.
.
.
```

From there you should be ready to set up your VMs
