# Complete CPU and GPU passthrough and desktop setup on Ubuntu 20.04
This guide is focused on my own hardware. If you want a general guide / tutorial, go elsewhere.

## Prerequisites
- Know what GPU passthrough is and know what you are doing. I'm not explaining a lot of things about GPU passthrough because I assume that you already read many docs. Just search Google;
- Be OK with `vim`;
- Adapt the following steps to your own hardware;
- Use a Keepass to store the passwords safely;
- Upgrade your BIOS to the latest version;
- Enable virtualisation features in the BIOS (IOMMU and Amd-v / VT-x);
- Be OK with the i3 shortcuts if you want to use it (optional);
- Know the Linux / networking basics (CLI, configure an IP address, mount a USB device, extract files from an archive, etc).

## My hardware (quiet with a high cooling capacity)
- MSI MPG B550 GAMING EDGE WIFI (see the `Issues`)
- AMD Ryzen 5 3600XT CPU
- 32 Go RAM
- 2 GPUs
  - GeForce GTX 1660 SUPER (first PCIE slot), which will be passed through to the Windows VM
  - GeForce GT 730 (second PCIE slot), used by the host because my CPU hasn't integrated graphics feature
- Be Quiet Silent Base 802 Window Black case
- Dark Rock PRO 4 ventirad
- 9 Be Quiet fans
- 2 x 1 To SSDs (Samsung and SanDisk) in a ZFS mirrored pool (to store files and VMs disks)
- 1 x 1 To Samsung NVMe SSD 970 EVO Plus (Linux host OS)

## Objectives
- [x] Use i3 on the host machine (right screen). I encountered some stability issues with KDE;
- [x] Use Windows in a VM with native performances (left and middle screens);
- [x] Use the same keyboard and mouse to manage both Windows VM and Linux host (mouse can go from the middle screen to the right screen as if it was the same machine);
- [x] Be able to use a ZFS mirrored pool (that's why i'm using a Linux host) to store data safely, perform snapshots, create secure encrypted datasets, etc;
- [x] Use a clean Ubuntu 20.04 Server distrib;
- [x] Automatically start the Windows VM when the host boots;
- [x] Be able to play games in the VM as if it was a real physical machine without latency;
- [x] Redirect the Windows VM sound to a headset without any latency.

## Global explanations and warnings
- We will use 12 Go RAM hugepages. This memory will be reserved for the Windows VM in order to increase performances. You can change the value if you want. If, like me, you use ZFS and like 5 or 6 VMs autostarted on boot (ELK, wiki, proxy, dev machines, etc), you will need a lot of RAM. I think 32 Go is the minimum in a setup like this.
- Here, the GPU will be passed through to play games in a Windows VM, but you can also pass it through a Linux VM for crypto mining.

## Create a bootable USB key
Create a bootable USB key using a Ubuntu 20.04 Server ISO https://releases.ubuntu.com/20.04/. You can also start from a Lubuntu, Xubuntu, etc ISO.

## Ubuntu installation
Boot on the USB key and install Ubuntu.
- Choose the language;
- Choose the keyboard layout;
- Don't configure the network (`Continue without network`). The NICs may not been detected; then you will have to finish the installation without internet connection;
- Don't use a proxy (`Next`);
- Use the default repo (`Next`);
- During the storage layout configuration, I recommend to encrypt the disk with luks. If you do so, set up a simple password (avoid azerty / qwerty typo issues), we will change it later. Select the correct device on which you want to install the OS (a NVMe SSD in my case);
- Confirm the storage layout;
- Configure the user name, the machine name and set a simple password (you can change it after);
- Install OpenSSH server, click `Next`;
- Once the installation is finished (it should be fast), click `Reboot now`;
- Remove the installation medium and press `Enter`.

## First boot
Boot the OS. Enter the luks password if you set up one. Login. Print the network interfaces.
```
ip a
```
On a MSI MPG B550 GAMING EDGE WIFI, the 2.5G ethernet NIC may not appear because you need to install the driver. Also note that you need some packages like `make` to compile and install the driver. On another Linux computer, you can :
- Plug and mount a USB key;
- Download the needed packages to build the driver in a folder on the USB key `apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests --no-conflicts --no-breaks --no-replaces --no-enhances --no-pre-depends build-essential | grep "^\w")`
- Go on the Realtek website https://www.realtek.com/en/component/zoo/category/network-interface-controllers-10-100-1000m-gigabit-ethernet-pci-express-software, download the `2.5G Ethernet LINUX driver r8125 for kernel up to 5.6 ` driver and put it also on the USB key.
Then on the machine on which we just installed Ubuntu 20.04:
- Plug the USB key and mount it;
- Enter the directory where we downloaded the `build-essential` packages, install them with `sudo dpkg -i *.deb` (ignore the errors you may encounter);
- Enter the directory where we downloaded the NIC driver, extract it (`tar -xvf xxx`) and install it with `sudo ./autorun.sh` (ignore the errors you may encounter).
- Run `ip a`. The ethernet NIC should be visible. Configure an IP address with the following commands (replace the IP address / netmask considering what your network address is, of course). Replace also the interface name by your own (`eno1` in my case).
```
sudo ip addr add 192.168.1.100/24 dev eno1
sudo ip link set dev eno1 up
sudo ip route add default via 192.168.1.1
```
The internet access should be OK and you should be able to `ping 8.8.8.8`. Edit `/etc/resolv.conf` and add `nameserver 8.8.8.8` at the top of the others uncommented lines (we shouldn't do that but we don't care because the `/etc/resolv.conf` file will be erased automatically when we will apply updates. We will configure the network properly later). Now you should be able to ping `google.com`. At this point, you can SSH to this machine from another one if you want. If the content of `/etc/resolv.conf` get erased, just configure it again as we did before. Update the packages.
```
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```
Configure the DNS in `/etc/resolv.conf` as we did before. Install useful packages (or not. You choose).
```
sudo apt install numlockx terminator xorg nvidia-driver-390 -y
```

## i3 installation
Configure the DNS in `/etc/resolv.conf` as we did before. I use i3, but you can use the desktop environment (or windows manager) you want. The following is a copy / paste of https://github.com/addy-dclxvi/i3-starterpack. Install i3 and the packages we will use. You can also use your own i3 config.
```
sudo apt install i3 compton hsetroot rxvt-unicode xsel rofi fonts-noto fonts-mplus xsettingsd lxappearance scrot viewnior -y
```
Configure X to use i3.
```
echo "exec i3" >> ~/.xinitrc
```
Start the X server.
```
startx
```
Press `Enter` 2 times to generate the config and use the `Windows` key as super key. On the right bottom, click on the small icon and connect to the Wifi. You can open a terminal by pressing `Windows` + `Enter`. Now you should be able to `ping google.com`.
Get the i3 configuration.
```
mkdir /tmp/i3
cd /tmp/i3
git clone https://github.com/addy-dclxvi/i3-starterpack.git
cd i3-starterpack
cp -r .Xresources .config/ .fonts/ .urxvt/ .wallpaper.png .xsettingsd ~/
```
Refresh fonts.
```
fc-cache -fv
```
Adapts the configuration to your needs. Change the NIC names. `Super` key is the `Windows` key. Use `Super + Shift + R` to refresh the running config using the config files.

## Time configuration
Run the command below to print the datetime status.
```
timedatectl status
```
If the timezone is correct, you can go to the next paragraph. If not, follow the steps below. Liste the available timezeones.
```
timedatectl list-timezones
```
Set the timezone you want. The example below is for Paris time.
```
sudo timedatectl set-timezone Europe/Paris
```

## Identify the devices to be passed through
I'm not talking about IOMMU groups. Just search Google about that. Print the PCI devices.
```
lspci -nn
```
You should get a result like that. Here are the devices I want to pass through.
```
$ lspci -nn
[...]
2b:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER] [10de:21c4] (rev a1)
2b:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
2b:00.2 USB controller [0c03]: NVIDIA Corporation TU116 USB 3.1 Host Controller [10de:1aec] (rev a1)
2b:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 [GeForce GTX 1650 SUPER] [10de:1aed] (rev a1)
[...]
```
I also want to pass my headset which is connected with USB. Print the USB devices.
```
lsusb
```
Here is the USB device I want to pass through. Note that if you are installing a new VM, you will need to pass a keyboard and a mouse (it means that you will need to have 2 keyboards and 2 mouses connected to your desktop).
```
$ lsusb
[...]
Bus 003 Device 005: ID 1038:12ad SteelSeries ApS SteelSeries Arctis 7
Bus 001 Device 008: ID 046d:c328 Logitech, Inc. Corded Keyboard K280e
Bus 001 Device 009: ID 145f:01ea Trust Trust Presenter Mouse
[...]
```
So we need to pass the devices `10de:21c4`, `10de:1aeb`, `10de:1aec`, `10de:1aed`, `1038:12ad`, `046d:c328` and `145f:01ea`.

## Prepare the host machine for the GPU passthrough
Edit `/etc/default/grub`.
```
sudo vim /etc/default/grub
```
Change the line `GRUB_CMDLINE_LINUX_DEFAULT` to the following (change the values according to your hardware). It's not needed to specify the USB devices. We need to specify all the IOMMU group.
```
GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt kvm.ignore_msrs=1 vfio-pci.ids=10de:21c4,10de:1aeb,10de:1aec,10de:1aed"
```
Update Grub.
```
sudo update-grub
```
Generate a new initramfs image.
```
sudo update-initramfs -u -k all
```
Reboot.
```
sudo reboot
```
After the reboot, print the PCI devices.
```
lspci -nnv
```
The kernel driver in use should always be `vfio-pci` for the devices we passed through (your GPU).
```
2b:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU116 [GeForce GTX 1660 SUPER] [10de:21c4] (rev a1) (prog-if 00 [VGA controller])
[...]
	Kernel driver in use: vfio-pci
[...]
2b:00.1 Audio device [0403]: NVIDIA Corporation TU116 High Definition Audio Controller [10de:1aeb] (rev a1)
[...]
	Kernel driver in use: vfio-pci
[...]
2b:00.2 USB controller [0c03]: NVIDIA Corporation TU116 USB 3.1 Host Controller [10de:1aec] (rev a1) (prog-if 30 [XHCI])
[...]
	Kernel driver in use: vfio-pci
[...]
2b:00.3 Serial bus controller [0c80]: NVIDIA Corporation TU116 [GeForce GTX 1650 SUPER] [10de:1aed] (rev a1)
[...]
	Kernel driver in use: vfio-pci
[...]
```
Now we can use the GPU in a VM.

## Hypervisor preparation
Install the needed packages for KVM.
```
sudo apt install libvirt-daemon-system virt-manager qemu-kvm ovmf bridge-utils -y
```
Reboot.
```
sudo reboot
```
Now we want to replace the `default` bridge by another one. List the KVM bridges.
```
virsh net-list --all
```
Delete the default bridge.
```
virsh net-undefine default
virsh net-destroy default
```
Create a new config file.
```
sudo vim virbr0.conf
```
Paste the following (adapt the content).
```
<network>
  <name>virbr0</name>
  <uuid>168fc909-7c5c-4333-a3cb-f74584d26321</uuid>
  <forward mode='nat'/>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:48:3f:00'/>
  <ip address='10.85.66.1' netmask='255.255.255.0'/>
</network>
```
Create the bridge using the config file.
```
virsh net-define --file virbr0.conf
```
Configure the bridge to be started automatically and start it.
```
virsh net-autostart --network virbr0
virsh net-start virbr0
```

## VM installation
Use the following command to install the Windows VM.
- Change the RAM and vCPUs values if you want;
- Always use virtio and raw disk format for better performances;
- `2b:00.0` and `2b:00.1` is the GPU. It's not mandatory to specify the USB controller and the Serial bus controller;
- `1038:12ad` is my headset;
- `046d:c328` is the keyboard we will pass to the VM (that's why you need 2 keyboards);
- `145f:01ea` is the mouse we will pass to the VM (that's why you need 2 mouses);
- Check that the file `/usr/share/OVMF/OVMF_CODE.fd` exists.
- Check the ISOs paths;
- `018-win10` is my VM name, change it if you want;
- When you will run the command below, a message will be printed (`Press any to to boot...`), then you have to be quick if you don't want to drop in the UEFI shell. If you do any mistake, just delete the VM and start again (`virsh destroy 018-win10 && sleep 5 && virsh undefine 018-win10 --nvram && sudo rm /tank/vms/storage/018-win10/018-win10-disk1.raw`).
```
virt-install \
--virt-type kvm \
--name 018-win10 \
--os-variant=win10 \
--vcpus 8 \
--memory 12288 \
--features kvm_hidden=on \
--disk path=/path/to/018-win10-disk1.raw,format=raw,bus=virtio,size=50 \
--cdrom /path/to/Win10_20H2_v2_French_x64.iso \
--disk /path/to/virtio-win-0.1.185.iso,device=cdrom \
--network bridge=virbr0,model=virtio \
--graphics none \
--hostdev 2b:00.0,address.type=pci,address.multifunction=on \
--hostdev 2b:00.1,address.type=pci \
--hostdev 1038:12ad \
--hostdev 046d:c328 \
--hostdev 145f:01ea \
--boot loader=/usr/share/OVMF/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash
```
During the disk selection in the installer :
- Click on `Load a driver`, browse the virtio CD, select `amd64`, `w10`, click `OK`, click `Next` to load the driver. The virtio disk should be visible. Now do the same for the NIC.
- Click again on `Load a driver`, browse the virtio CD, select `NetKVM`, `w10`, `amd64`, click `OK`, click `Next` to load the driver.
Once the installation is finished, you should see the Windows desktop with a bad resolution. Now, configure a static IP address (then you should be able to `ping google.com`). At this point, you still have a bad screen resolution. Shutdown the VM.

## VM import
If you already own the VM disk (backup), you can use the following command. If you had barrier configured on this VM, the SSL keys fingerprints may have changed and barrier wont work. Then, connect on the VM using RDP and accept the new key fingerprint.
```
virt-install \
--virt-type kvm \
--name 018-win10 \
--os-variant=win10 \
--vcpus 8 \
--memory 12288 \
--features kvm_hidden=on \
--import \
--disk path=/path/to/018-win10-disk1.raw,format=raw,bus=virtio \
--network bridge=virbr0,model=virtio \
--graphics none \
--hostdev 2b:00.0,address.type=pci,address.multifunction=on \
--hostdev 2b:00.1,address.type=pci \
--hostdev 1038:12ad \
--boot loader=/usr/share/OVMF/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash
```
Then `virsh shutdown` and perform the optimization steps below.

## Optimize VM performances - Set up hugepages on the host
Edit `/etc/sysctl.conf`.
```
sudo vim /etc/sysctl.conf
```
Paste the following at the end (we cant to allocate 12Go).
```
# Hugepages
vm.nr_hugepages = 6144
```
Apply the changes.
```
sudo sysctl -p
```
The following should be printed.
```
$ sudo sysctl -p
vm.nr_hugepages = 6144
```
If you run a `top`, `htop` or `free -h`, you should see that the amount of RAM you set up as hugepages is now used.

## Optimize VM performances - Configure the VM to use hugepages
Edit the VM config.
```
virsh edit 018-win10
```
Under `</currentMemory>`, add the following.
```
<memoryBacking>
<hugepages/>
</memoryBacking>
```

## Configure a random vendor ID
Edit the VM config.
```
virsh edit 018-win10
```
Between `<hyperv>` and `</hyperv>`, add the following.
```
<vendor_id state='on' value='1234567890ab'/>
```

## Optimize VM performances - Configure CPU pining
Edit the VM config.
```
virsh edit 018-win10
```
Under `<vcpu placement='static'>`, add the following.
```
<cputune>
<vcpupin vcpu='0' cpuset='2'/>
<vcpupin vcpu='1' cpuset='8'/>
<vcpupin vcpu='2' cpuset='3'/>
<vcpupin vcpu='3' cpuset='9'/>
<vcpupin vcpu='4' cpuset='4'/>
<vcpupin vcpu='5' cpuset='10'/>
<vcpupin vcpu='6' cpuset='5'/>
<vcpupin vcpu='7' cpuset='11'/>
<emulatorpin cpuset='0-1,6-7'/>
</cputune>
```
Replace `<cpu mode='host-model' check='partial'/>` by the following line.
```
<cpu mode='host-passthrough' check='none'>
<topology sockets='1' cores='4' threads='2'/>
</cpu>
```

## Screen resolution correction
Boot the Windows VM. Install all the Windows updates. The screen resolution should now be normal. You can also install the Nvidia drivers if you want. In the tasks manager, you should see your GPU.

## Use the same keyboard and mouse for both systems
Now we want to use the same keyboard and mouse (the host's ones) to manage the Windows VM and the Linux host. To do so, install `barrier` on the Linux host.
```
sudo apt install barrier
```
Install it also on the Windows VM. Configure it on both machines. Then you should be able to use the same keyboard and mouse on both systems.

## Start the Windows VM automatically on boot
Just run the following command (replace the VM name).
```
virsh autostart 018-win10
```

## Change the luks password
If you used a weak password during the installation, change it using the commands below. The device name (`/dev/nvme0n1p3`) may change for you.
```
sudo cryptsetup luksDump /dev/nvme0n1p3
```
Change the luks key.
```
sudo cryptsetup luksChangeKey /dev/nvme0n1p3 -S 0
```

## Correct the Alt Gr key not working correctly with barrier
Note that you can also use `CRTL` + `ALT`. Edit `/etc/barrier.conf`.
```
sudo vim /etc/barrier.conf
```
Add the following (adapt the content). The screen name must be the machine hostname. The Windows VM, even if it has 2 monitors (in my case), counts as one screen in the config.
```
section: screens
    my-desktop:
        halfDuplexCapsLock = false
        halfDuplexNumLock = false
        halfDuplexScrollLock = false
        xtestIsXineramaUnaware = false
        preserveFocus = false
        switchCorners = none
        switchCornerSize = 0
        meta = altgr
        altgr = shift
    018-win10:
        halfDuplexCapsLock = false
        halfDuplexNumLock = false
        halfDuplexScrollLock = false
        xtestIsXineramaUnaware = false
        preserveFocus = false
        switchCorners = none
        switchCornerSize = 0
        meta = altgr
        altgr = shift
end

section: links
        018-win10:
                right = my-desktop
        my-desktop:
                left  = 018-win10
end
```
Now, load and test the config using the barrier GUI.

## Start barrier automatically on boot
With i3 it's easy. Edit the i3 config file.
```
vim ~/.config/i3/config
```
Add the following (for example in the `#autostart` section).
```
exec --no-startup-id /usr/bin/barriers -f --no-tray --debug INFO --name my-desktop --enable-crypto -c /etc/barrier.conf --address [10.85.66.1]:24800
```
Try to reboot your computer and see if it works.
```
sudo reboot
```

## Import ZFS pools
If you have ZFS pools, you can import then using the commands below.
```
sudo apt install zfsutils-linux -y
sudo zpool import <pool_name> -f
```

## Issues
### Motherboard (MSI MPG B550 GAMING EDGE WIFI)
Note that you can't use both NVMe slots on a MSI MPG B550 GAMING EDGE WIFI when you are using both PCIE slots.

### Windows VM isn't starting on boot
Check that the VM has `autostart` enabled. Check that all the (USB) devices you redirected to this VM are connected to the host. Try to start the Windows VM manually with `virsh` and see which warning message is printed.

## Sources
- Configure hugepages on host https://help.ubuntu.com/community/KVM%20-%20Using%20Hugepages
- GPU passthrough guide for Ubuntu https://mathiashueber.com/windows-virtual-machine-gpu-passthrough-ubuntu/
- Download a package with dependencies https://stackoverflow.com/questions/13756800/how-to-download-all-dependencies-and-packages-to-directory
- i3 configuration https://github.com/addy-dclxvi/i3-starterpack
- Timezone configuration https://linuxize.com/post/how-to-set-or-change-timezone-on-ubuntu-18-04/
- Correct barrier alt gr issue https://github.com/debauchee/barrier/issues/100
