# Arch Linux Installation

## Create a bootable Arch Linux USB

Skip to the next section, because the creation of an UEFI bootable Arch Linux USB drive had been already automated: see https://github.com/kyberdrb/arch_linux_bootable_uefi_usb_creator

1. Download Arch Linux ISO

    - The latest version I tested to boot successfuly was the `archlinux-2021.02.01-x86_64.iso`. The newer ones failed to boot due - kernel panic due to `dell-wmi-sysman` kernel module error

1. Clone it on the USB drive

    - Linux
        Preferred method: `dd`
	
	1. Find the name of the USB stick
    
            lsblk
	    
	1. [Clone the Arch ISO on the USB stick](https://wiki.archlinux.org/index.php/USB_flash_installation_medium#Using_basic_command_line_utilities)
	
            sudo dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
	    sync
    
    - Windows
    
        - Preferred method: [Rufus](https://wiki.archlinux.org/index.php/USB_flash_installation_media#Using_Rufus)

## Boot from the USB

1. Plug the USB drive into a PC and boot from it. Keep pressing F8 / F9 / F12 to get boot device selection menu or go straight to BIOS and change boot settings there. I recommend booting the USB and installing Arch Linux in UEFI mode. The booting mode can be set in the BIOS.
    1. If your computer (like laptop HP 4530s) doesn't offer you a UEFI boot option for your USB, only the _Legacy_ one, boot from `EFI/boot/loader.efi` these steps to boot the USB in UEFI mode (make sure UEFI booting is enabled in the BIOS):
        1. Rapidly press F9
        1. Choose `Boot From EFI File`
        1. Choose option `ARCHISO_EFI_...`
        1. Select `EFI -> boot -> loader.efi`
    
    Now the USB will boot in UEFI mode a shortly you will see the ARCH EFI bootloader.

1. After you get the Arch Linux boot screen, choose `Boot Arch Linux x86_64`

## Connect to the Internet

1. To install Arch Linux you need to have Internet connection. It's recommended to use the Ethernet, but Wi-Fi connection is also possible, if your wireless adapter is supported. Below I will explain how to connect to the Internet via Ethernet and Wi-Fi.

    1. List all your wired and wireless network adapters
	
	        ip link
    
    1. Connect to the internet. Below are the steps for connecting to the wireless and wired network
        - Connect to a Wi-Fi network
    
            1. Using `wifi-menu`

                1. Execute command

                  wifi-menu

                1. Choose the network, confirm profile name and enter password. The Wi-Fi adapter is turned on automatically by the command and the IP address is obtained automatically as well via DHCP.

            - Connect to a wired network
                The Ethernet should be named similar to `enpXsY` where X and Y are numbers. Wireless adapters have a name in format similar to `wloZ` where Z is a number. Then you need to request an IP address via DHCP

                  dhcpcd enpXsY

                an IP address should be assigned to the Ethernet interface.
                For manual IP configuration see [Arch Linux Wiki - Network configuration](https://wiki.archlinux.org/index.php/Network_configuration#Static_IP_address)

	    1. Using `iwctl`

            1. First, find the device name

                    $ iwctl device list

                Let's say that the previous command outputs only one Wi-Fi network adapter with the name `wlan0`.
	    
	        - Non-interactively

		            Then the command for connecting to a standard wireless home network would be:

			        $ iwctl --passphrase PASSPHRASE station DEVICE connect SSID-NAME_OF_WIFI_NETWORK
		
		    - Interactively
	    
	    	        $ iwctl

                And in the `iwd` shell

	    	        station wlan0 scan
	    	        station wlan0 get-networks
	    	        station wlan0 connect SSID-NAME_OF_WIFI_NETWORK
		    
	        - Sources
	            - https://duckduckgo.com/?q=arch+linux+installation+wifi&ia=web
	            - https://bbs.archlinux.org/viewtopic.php?id=109672
	            - https://wiki.archlinux.org/title/Installation_guide#Connect_to_the_internet
	            - https://wiki.archlinux.org/title/Iwd#iwctl

    1. Test connectivity:
	
            ping google.com -c 4

        If the ping was successful, we can proceed with Arch Linux installation.

## Check UEFI mode

Now we neet do decide, if we want legacy or EFI install of Arch Linux. If you chose "UEFI" in the boot device selection earlier, read on, otherwise skip this part.

If you choosed to boot from USB in UEFI mode, check the list of EFI variables to see, if Arch Linux can be installed in UEFI mode:
	
	efivar -l

If it lists variables, you can install Arch Linux in UEFI mode :D
If it doesn't list any variables or prints an error "efivar: error listing variables: Function not implemented", it means, that either we booted into legacy mode, or our UEFI is not supported.

Now we can continue with the disk partitioning

## Partition the disks

First of all, we have to choose the disk, which Arch Linux will be installed on. Print the list of connected disks with command:

	lsblk

Let's say, we want to install Arch on disk "sda".
Firstly, we need to wipe the disk:

	sgdisk --zap-all /dev/sda
	
The target disk is now clean. We can proceed to the disk partitioning.

**TODO use `parted` in noninteractive mode instead of belowmentioned utilities to partition disks**  
**See [[1]](https://linuxconfig.org/how-to-manage-partitions-with-gnu-parted-on-linux), [[2]](https://askubuntu.com/questions/706608/exfat-external-drive-not-recognized-on-windows/812761#812761)**

### UEFI partitioning

For UEFI install it will be partitioned like this:

	boot (EFI), Swap, Root

#### via `sgdisk` (preferred)

    sgdisk --clear /dev/sda
    sgdisk --new=1:0:+600MiB --typecode=1:ef00 /dev/sda    
    sgdisk --new=2:0:+210GiB --typecode=2:8300 /dev/sda

#### via `cgdisk`

Run the partition tool (I will use cgdisk, but there are many others)
	
	cgdisk /dev/sda
	
	# Press any key to continue ...

UEFI partitioning configuration:

	# Create EFI partition
	Select "New"
	First sector: 4096
	Size in sectors: 512MiB
	Hex code or GUID: EF00
	Partition name: boot

	# Create Swap partition
	Select "New"
	First sector: <press Enter - use the first availible sector>
	Size in sectors: 4GiB
	Hex code or GUID: 8200
	Partition name: swap
		
	# Create Root partition
	Select "New"
	First sector: <press Enter - use the first availible sector>
	Size in sectors: <press Enter
          for HDD: use the rest of the space
	Hex code or GUID: <press Enter - 8300>
	Partition name: system
	
Write changes to disk:

	Select "Write"
	yes

	# Quit from cgdisk
	Select "Quit"

### Legacy partitioning

For legacy install it will be partitioned like this:

	Swap, Root

Remember to make some overprovisioned space if you are using a SSD (at least 20% of the free space of the system partition)!

	1: type command "cfdisk"
	2: choose dos on next screen
	3: choose "new"
	4: type half the size of ur ram
	5: press enter & choose primary
	6: choose type
	7: choose linux swap / solaris
	8: press "arrow down" and choose "new"
	9: press enter & choose primary
	10: choose bootable
(thx Gregory Nwosibe: https://www.youtube.com/watch?v=Wqh9AQt3nho)

Write changes to disk:

	Select "Write"
	yes

Quit from cfdisk

	Select "Quit"

## Verify created partitions

Verify if the changes have been successfuly applied:

	lsblk

You should see the new partitions under the name of your disk.

## Format partitions

1. Format EFI partition (if you have any)

        mkfs.fat -F32 /dev/sda1
	
Format system partition

        mkfs.ext4 -t ext4 -F /dev/sda2
        y

1. [Optional step] Turn on swap:

        mkswap /dev/sdaX
        swapon /dev/sdaX

1. Verify, if the changes have been applied:

        lsblk

1. Check partition types. Now we should see partition types (filesystems) next to our new partitions.

        fdisk -l /dev/sda

## Mount partitions

Mount system partition

    mount /dev/sda2 /mnt

### UEFI ONLY

Mount EFI partition

    mkdir /mnt/boot
    mount /dev/sda1 /mnt/boot

## Select the fastest repository servers

Update package database

        pacman - Syy
	
Install `reflector`
	
	pacman -S reflector

Backup original pacman's mirror list

	cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak

Create new mirror list. The servers which we will provide the fastest speed for downloading packages will be on the top of the list.

        reflector --sort rate > /etc/pacman.d/mirrorlist
	
The command can take a while to finish depending on the speed of your internet connection.

https://wiki.archlinux.org/index.php/mirrors#Server-side_ranking

## Install Arch Linux

Install pacman and the base system:

	pacstrap /mnt base

## Generate filesystem table

fstab using UUIDs:

	genfstab -t UUID -p /mnt >> /mnt/etc/fstab
	
	# or maybe
	
	genfstab -U -p /mnt >> /mnt/etc/fstab
	
	# Source: https://man.archlinux.org/man/genfstab.8

Edit "/etc/fstab" file:

	nano /mnt/etc/fstab
or
        vim /mnt/etc/fstab
	
	# Replace every "relatime" with "noatime" -> better HDD/SSD IO speed
	# For the boot partition (probably sda1?) change the "rw" to "ro" - more secure booting process and system runtime

Example `/etc/fstab` file

```
# /dev/sda2
UUID=cb217b7c-f7c0-4dae-b9a6-412e68b52408	/         	ext4      	rw,noatime,commit=60	0 1

# /dev/sda1
UUID=220C-B8F7      	/boot     	vfat      	ro,noatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro	0 2
```

Explanation:

* `noatime`
* `commit=60`

****************************************

Chroot into /mnt using systemd containers

    systemd-nspawn -b -D /mnt

Login as `root`

****************************************

Set up hostname

	echo whatever_hostname_you_want > /etc/hostname
	cat /etc/hostname

The hostname will be set after next reboot.

****************************************

Finish disk configuration and optimization

quick setup

```
cat /sys/block/nvme0n1/queue/{scheduler,nr_requests,read_ahead_kb}

echo 2048 | sudo tee /sys/block/nvme0n1/queue/read_ahead_kb
echo 1024 | sudo tee /sys/block/nvme0n1/queue/nr_requests
echo 'none' | sudo tee /sys/block/nvme0n1/queue/scheduler

cat /sys/block/nvme0n1/queue/{scheduler,nr_requests,read_ahead_kb}
```

check `journalctl -b -p 4` for errors occured at parameter setting when the set values don't match

persistent setup

```
$ sudo vim /etc/udev/rules.d/99-nvme-optimization.rules

ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]*", SUBSYSTEM=="block", ATTR{queue/scheduler}="none" # 'none' for minimizing latency for 4K QD1 random read/write, letting the NVMe controller schedule the operations
```

```
cat /sys/block/nvme0n1/queue/{scheduler,nr_requests,read_ahead_kb}
sudo udevadm control --reload-rules && sudo udevadm trigger --verbose --subsystem-match=block --action=add
cat /sys/block/nvme0n1/queue/{scheduler,nr_requests,read_ahead_kb}
```

****************************************

Configure pacman and its repositories by cloning my repository for Arch Linux updating

    sudo pacman -S git
    cd /tmp
    git clone https://github.com/kyberdrb/update_arch.git
    cd update_arch
    
Launch the script to update the system
    
    ./update_arch.sh

Wait until the updating completes.

## Set up root password:

	passwd
	<type your password>
	<type your password again>

## Add a new user account:

	useradd -m -g users -G wheel,storage,power -s /bin/bash laptop

## Set up password for the new user:

	passwd laptop
	<type your password>
	<type your password again>

## Allow the new user to use the `sudo` command:

1. Install `sudo`

        pacman -S sudo

1. Allow the user to use `sudo` command

        visudo

1. Find this line:

        ## Uncomment to allow members of group wheel to execute any command
        # %wheel ALL=(ALL) ALL

1. Change it by uncommenting the `wheel` line and require the root password:
	
		## Uncomment to allow members of group wheel to execute any command
		%wheel ALL=(ALL) ALL
		Defaults rootpw
                Defaults timestamp_timeout=180

This will allow for the new user to use the `sudo` command but not with the user password! Instead, the root password will be required. The root password will be valid for `180` minutes.

Save and exit

    press (Esc)
    :wq

### Install additional packages

I need these packages beacuse they simplify the work at first login.

- `dialog` - contains `wifi-menu` utility which interactively connects to the internet
- `wpa_supplicant` - dependency of  `diaalog`
- `bash-completion` - complete  commands with `Tab` key
- `vim` - text editor

        pacman -S dialog wpa_supplicant bash-completion vim

BOOTLOADER INSTALLATION

****************************************
INTEL CPU ONLY

Install Intel microcode (to improve system stability).

    pacman -S intel-ucode

****************************************

Exit from systemd container

    shutdown now
    
Enter Arch chroot environment beacuse in systemd container the directory `/sys/firmware/efi/efivars` is empty. In `arch-chroot` environment is the directory populated.

    arch-chroot /mnt

****************************************
UEFI ONLY

Mount EFI variables (efivars)
	
    mount -t efivarfs efivarfs /sys/firmware/efi/efivars

Install bootloader (systemd-boot):

    bootctl install

**TODO create a script that will automatize the creation of 'arch.conf' file that will take as arguments the path to the root partition `/` and an optional argument of the name of the kernel when multiple kernels are installed (but check if it exists in the `/etc/mkinitcpio.d/` dir), e.g. `./generate_bootctl_config.sh /dev/sda2 linux-lqx`**

Generate a system partition UUID, i.e. the partition, where you installed the operating system Arch Linux on, and then use it in the bootloader instead of the partition name (more secure):

    echo "title Arch Linux" >> /boot/loader/entries/arch.conf
    basename /boot/vmlinuz-linux >> /boot/loader/entries/arch.conf
    basename /boot/intel-ucode.img >> /boot/loader/entries/arch.conf
    basename /boot/initramfs-linux.img >> /boot/loader/entries/arch.conf
    blkid --match-tag UUID --output value /dev/sda2 >> /boot/loader/entries/arch.conf

Open boot configuration file

    # Create new file named "arch.conf"
    nano /boot/loader/entries/arch.conf
	
Edit bootloader configuration like it is shown below. Ommit the line with "intel-ucode", if you don't have an Intel CPU. Example result:

    title Arch Linux
    linux /vmlinuz-linux
    initrd /initramfs-linux.img
    options root=UUID=d65b559b-fe10-43e8-8853-09f55b3fa25d rw
    
Create seed for the random number generator - possible prevention against systemd booting message 'Failed to acquire RNG protocol: not found' [1](https://github.com/systemd/systemd/issues/13503#issuecomment-529525157) and agains journalctl message about 'crng not initialized' - which is not a problem: the systemd will initialize anyway

    bootctl random-seed

or later, as a user

    sudo bootctl random-seed

****************************************
 
LEGACY ONLY

For legacy system we need to install GRUB bootloader

	pacman -S grub-bios
	y
	grub-install /dev/sda

Generate init file for GRUB
	
	mkinitcpio --allpresets
	grub-mkconfig -o /boot/grub/grub.cfg
****************************************

If you got warning messages about missing firmwares, take note which firmwares are you missing. We will install that, after the first login.

Reboot
	exit
	reboot
	# Remove the installation USB now!

If the system doesn't boot AND you have a NVidia graphics card, boot from the USB again and install and configure graphics drivers:

	mount /dev/sda2 /mnt
	mount /dev/sda1 /mnt/boot
	arch-chroot /mnt
	pacman -S nvidia nvidia-libgl lib32-nvidia-libgl nvidia-utils lib32-nvidia-utils
	exit
	reboot

## First console login

Login as regular user i.e. under `laptop` user account.

Connect to the internet:

    sudo wifi-menu

or

    wpa_supplicant -B -i wlo1 -c <(wpa_passphrase SSID_of_the_network "network password")
    
Set time:

    sudo ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
    sudo hwclock --systohc
    
### Configure `vim`

See the `vim` entry in the [installed packages file](https://github.com/kyberdrb/Linux_tutorials/blob/master/ARCH_installed_packages_user.md)
    
## Change directory colors in terminal for better readablilty on black background

1. Export current colors of the terminal	

        dircolors --bourne-shell >> ~/.bashrc
    
1. Open the exported file

        vim ~/.bashrc

1. Find variable `LS_COLORS`
1. Modify entry `di` - i.e. _directory_ - from, e.g. `di=01;34:` (34 - blue - i don't see the directory names clearly on black background in terminal) to `di=01;33:` (33 - orange - easier to read on black background)
1. Save the file and exit
1. Test new settings by opening a new terminal window. Existing terminal windows need to be reopened in order for them to reload changes.

Sources:
- https://askubuntu.com/questions/466198/how-do-i-change-the-color-for-directories-with-ls-in-the-console/466203#466203
- https://linuxhint.com/ls_colors_bash/

### Select language

Edit locale setting (system and application language):
	
	sudo nano /etc/locale.gen

**TODO do it automatically with 'sed'**

Uncomment entries "en_US.UTF-8 UTF-8" and "en_US.ISO-8859-1"

Save file (Ctrl + O) and exit (Ctrl + X).
Generate new locale settings
	
	sudo locale-gen
	sudo localectl set-locale LANG=en_US.UTF-8
	
If the `localectl` command wouldn't be issued, we would see in the desktop environment meaningless characters

Set up automated system updating

    git clone https://github.com/kyberdrb/update_arch.git
    cd update_arch
    
Launch the script to update the system
    
    ./update_arch.sh
    
Wait until the updating completes. The script will automatically configure and update the system.

### Install X server
    
    sudo pacman -S xorg
      
### Install graphics drivers

Common graphics drivers and Vulkan support

    pacman -S mesa lib32-mesa vulkan-icd-loader lib32-vulkan-icd-loader vulkan-headers vulkan-tools mesa-utils lib32-mesa-utils
    pikaur -Syy vkmark-git assimp
    
- `vulkantools` provides the `vulkaninfo` utility to verify whether Vulkan API is enabled.
- `mesa-utils` and `lib32-mesa-utils` provide the `glxgears` utility for benchmarking [1](https://bbs.archlinux.org/viewtopic.php?pid=838572#p838572)
    - run `glxgears` utility as
    
          vblank_mode=0 glxgears -fullscreen
		
      for maximum FPS
- `vkmark-git` contains the utility `vkmark` which benchmarks GPU performance in Vulkan environment.
    - `assimp` package is required, because without it I got an error `vkmark: error while loading shared libraries: libassimp.so.5: cannot open shared object file: No such file or directory`
    
https://wiki.archlinux.org/index.php/Benchmarking#Graphics

---

Continue with the installation of the driver which is specific to your GPU vendor.

**Note**

`xorg.conf` attributes in

https://wiki.archlinux.org/index.php/Xorg#Configuration

https://wiki.archlinux.org/index.php/Intel_graphics#Xorg_configuration

https://wiki.archlinux.org/index.php/AMDGPU#Xorg_configuration

**work only with `xf86-video-*` driver of the particular GPU vendor, e. g. `xf86-video-intel`, `xf86-video-amdgpu`, `xf86-video-ati` etc.**

**TO modify a modesetting driver, use module options.** Module options are set in a text file with the same name as the module, e. g. `<module_name>.conf` and are located in the directory `/etc/modprobe.d/`. Modify the setting of the module by inserting the options to the configuration file for the modesetted driver. To query which options current module supports execute command

    modinfo -p <module_name> | less
    
All configuration in the configuration files in `/etc/modprobe.d/`, i. e. all parameters for modesetting drivers, can be defined as a kernel parameter in `/boot/loader/entries/arch.conf` (see sections about `bootctl` bootloader) [example of kernel parameters](https://wiki.archlinux.org/index.php/AMDGPU#Set_module_parameters_in_kernel_command_line) and [example of equivalent modeprobe parameters](https://wiki.archlinux.org/index.php/AMDGPU#Set_module_parameters_in_modprobe.d)
    
Keep in mind that With modesetting drivers only module options will work. The Xorg server cannot use the `intel.conf`. `amdgpu.conf` or `radeon.conf` configuration file in `modeprobe.d` directory, beacuse the modesetting driver is called `modesetting`, not `amdgpu` or `radeon` [AMDGPU - Set module parameters in modprobe.d](https://wiki.archlinux.org/index.php/AMDGPU#Set_module_parameters_in_modprobe.d), [Intel Modesetting DDX](https://wiki.gentoo.org/wiki/Intel#Modesetting_DDX). Creating these files resulted in my experiments in getting stuck in the boot messages output and inability to enter the login screen. Instead it either generates configuration by itself (recommended), or uses `modesetting.conf` and `20-modesetting.conf` when specified manually (only when automatic settings deduction is failing in some way - not tested or recommended).

Therefore "The modesetting driver doesn't support TearFree." ... "Because of that the modesetting driver is basically useless unless you use a compositor. You'll have to use xf86-video-intel driver and try to work around the bugs." [Gentoo Forum](https://forums.gentoo.org/viewtopic-t-1086534-start-0.html)  
You don't have to **always** install use a dedicated compositor, like `picom`. On my Intel HD 520 GPU I'm using vanilla XFCE4 with composing through `xfwm4` and it works without additional configuration for smooth windows movement under X-server, i. e. Xorg server. But on my Kabini build with integrated R3 8400 GPU with LXDE and `openbox-lxde` window manager, the `picom` composing manager/compositor helped to make the graphical experience under X smooth [LXDE - Using a composite manager](https://wiki.archlinux.org/index.php/LXDE#Using_a_composite_manager), [Picom (see Installation and Configuration chapters)](https://wiki.archlinux.org/index.php/Picom). So, maybe it depends on, whether the window manager is also able to compose the screen smoothly, e. g. through hardware acceleration via OpenGL.

Sources:

https://www.mankier.com/4/modesetting

https://wiki.archlinux.org/index.php/Kernel_module#Obtaining_information

#### General packages for Intel GPUs

GPU on my laptop - Intel HD Graphics 520

    sudo pacman -Syy vulkan-intel lib32-vulkan-intel intel-gpu-tools
	
The package `intel-gpu-tools` provides the utility `intel_gpu_top` which monitors the utilization of the Intel GPU. Run as `sudo intel_gpu_top`
	
---

**For Intel GPUs - Xorg `modesetting` driver - Late KMS (Kernel Mode Setting):**

[Remove all `xf86-video-*` packages.](https://bbs.archlinux.org/viewtopic.php?id=229242) In my case it were `xf86-video-intel` and `xf86-video-vesa`. [Reason for removal (look for 'Xorg searches for installed drivers automatically:' for the order in which Xorg searches for graphics drivers)](https://wiki.archlinux.org/index.php/Xorg#Driver_installation)

	sudo pacman -Runs xf86-video-intel xf86-video-vesa
	
Close all programms and reboot

	reboot
	
If everything goes well, you will see the destop environment as if nothing changed.

---

**For Intel GPUs - Xorg `modesetting` driver - Early KMS (Kernel Mode Setting):**

If the system rebooted and after you've logged in you see the desktop environment, continue with the setup to [enable early modesetting driver](https://wiki.archlinux.org/index.php/Intel_graphics#Enable_early_KMS) which may increase performance and give access to more power-saving features.
	
Start Intel graphics module during initramfs stage - [Early KMS start](https://wiki.archlinux.org/index.php/Kernel_mode_setting#Early_KMS_start) by editing the file... [Source](https://gist.github.com/lbrame/1678c00213c2bd069c0a59f8733e0ee6#using-the-modesetting-driver)

	sudo vim /etc/mkinitcpio.conf

... with content

	MODULES=(i915)
    
Save and exit by pressing `ESC + :wq`

Regenerate `initramfs` image

    sudo mkinitcpio --allpresets
	
Close all programms and reboot

	reboot
    
If everything goes well, you will see the destop environment as if nothing changed.

Now we can proceed to the modesetting driver configuration.

**Enable GuC and HuC - dangerous: taints the kernel**

More about tainting kernel here: [[1]](https://duckduckgo.com/?q=Setting+dangerous+option+enable_guc+-+tainting+kernel&ia=web), [[2]](https://unix.stackexchange.com/questions/118116/what-is-a-tainted-kernel-in-linux#118117), [[3]](https://bugs.freedesktop.org/show_bug.cgi?id=111918)

Edit this file...
	
	sudo vim /etc/modprobe.d/i915.conf
	
... with this content:
	
	options i915 enable_guc=2
        options i915 enable_gvt=0

GVT-d and GuC are mutually exclusive. When both are enabled, a kernel panic occurs and the system is unable to boot. Therefore I disable GVT-d explicitly to prevent unpleasant surprises.
	
Then regenerate the initramfs:

    ./remount_boot_part_as_writable.sh
    sudo mkinitcpio --allpresets
    
Close all open programs and reboot

Verify if GuC and HuC are enabled

	sudo cat /sys/kernel/debug/dri/0/gt/uc/guc_info
	sudo cat /sys/kernel/debug/dri/0/gt/uc/huc_info
	dmesg | grep i915
	modprobe -c | grep i915 | less


You can experiment other module settings described [on Arch Wiki](https://wiki.archlinux.org/index.php/Intel_graphics#Module-based_options) or by executing the command `modinfo -p i915 | less` and adding it into the `/etc/modprobe.d/i915.conf` file.

Intel `fastboot` option is enabled by default [1](https://wiki.archlinux.org/index.php/Intel_graphics#Module-based_options), [2](https://wiki.gentoo.org/wiki/Intel#Feature_support), [drm/i915: Enable fastboot by default on Skylake and newer](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3d6535cbed4a4b029602ff83efb2adec7cb8d28b)

---

If something breaks (blank screen, system not booting, X or desktop environment [DE] not starting etc.) try out [this method](https://www.youtube.com/watch?v=zEhAJMQYSws).

1. Boot from the Arch Linux USB

1. `chroot` into the system

        mount /dev/sda2 /mnt
        mount /dev/sda1 /mnt/boot
        arch-chroot /mnt
        
1. Revert the changes, i .e. delete the files or changes in configurations from previous guide, e. g. modifications to `/etc/mkinitcpio.conf` and then regenerating the `initramfs` image for the kernel again.
        
1. Tell Xorg to [load the modesetting driver](https://wiki.gentoo.org/wiki/Intel#Modesetting_DDX) by editing a configuration file...

        sudo vim /etc/X11/xorg.conf.d/20-modesetting.conf
        
    ... with this content:
        
        Section "Device"
            Identifier  "Intel Graphics"
            Driver      "modesetting"
            Option      "AccelMethod"    "glamor"
            Option      "DRI"            "3"
        EndSection
        
Save and exit by pressing `Esc :wq`

1. Finalize [Xorg configuration for Intel modesetting driver [see the bullet point 'Alternate Driver for Gen 4+ -- Modesetting']](https://wiki.gentoo.org/wiki/Intel#xorg.conf_3) by editing file...

        sudo vim /etc/X11/xorg.conf.d/modesetting.conf
        
    ... with this content:
    
        Section "Device"
           Identifier  "modesetting"
           Driver      "modesetting"
        EndSection

1. Exit from the `chroot` environment and reboot the system.

        exit
        reboot
        
The desktop environment will appear.

---

You can tweak the GPU properties to obtain higher GPU performance, better smoothness or enhanced power-saving features.

https://wiki.archlinux.org/index.php/Kernel_mode_setting

https://wiki.archlinux.org/index.php/Kernel_module#Setting_module_options

https://gist.github.com/lbrame/1678c00213c2bd069c0a59f8733e0ee6#using-the-modesetting-driver

https://wiki.gentoo.org/wiki/Intel

https://wiki.archlinux.org/index.php/Intel_graphics#Enable_GuC_/_HuC_firmware_loading

[What does GuC and HuC mean?](https://01.org/linuxgraphics/downloads/firmware)

Enable framebuffer compression. Benchmark before and after enabling this property to see to what extent affects this property the GPU performance.
https://wiki.archlinux.org/index.php/Intel_graphics#Framebuffer_compression_(enable_fbc)

https://kernelnewbies.org/Linux_4.11

https://archlinux.org/news/xorg-server-116-is-now-available/

https://jlk.fjfi.cvut.cz/arch/manpages/man/modesetting.4

https://gist.github.com/Brainiarc7/aa43570f512906e882ad6cdd835efe57

---

**For Intel GPUs - Xorg nomodeset driver:**

If everything else failed, use the nomodeset Intel driver

    pacman -S xf86-video-intel

    ...
    (1/2) installing libxvmc                                                  [#########################################] 100%
    (2/2) installing xf86-video-intel                                         [#########################################] 100%
    >>> This driver now uses DRI3 as the default Direct Rendering
        Infrastructure. You can try falling back to DRI2 if you run
        into trouble. To do so, save a file with the following 
        content as /etc/X11/xorg.conf.d/20-intel.conf :
          Section "Device"
            Identifier  "Intel Graphics"
            Driver      "intel"
            Option      "DRI" "2"             # DRI3 is now default 
            #Option      "AccelMethod"  "sna" # default
            #Option      "AccelMethod"  "uxa" # fallback
          EndSection
    ...

- The `/etc/X11/xorg.conf.d/20-intel.conf` is **only optional** if you're experiencing issues like tearing, stuttering, black screen, etc. Here's [another one](https://gist.github.com/radupotop/8597093) [just for curiosity].

#### For AMD/ATI graphics and AMD APUs

Currently I have a Kabini build.

For an advice which graphics driver to choose, see [this table](https://wiki.archlinux.org/index.php/Xorg#Driver_installation).

**General packages**

    pacman -S vulkan-radeon lib32-vulkan-radeon
    
**Late modesetting**

Remove all `xf86-video-*` packages.

    sudo pacman -Runs xf86-video-amdgpu
    
Close all programs ans reboot

Verify

    dmesg | grep -E "(amd|radeon)" | less
    less ~/.local/share/xorg/Xorg.0.log
    modprobe -c | grep -E "(amd|radeon)" | less
    less /var/log/Xorg.0.log

**Early modesetting**

Edit your `mkinitcpio` configuration file

    sudo vim /etc/mkinitcpio.conf
    
...with contents:
    
    MODULES=(amdgpu radeon)
    
Modules need to be in that order: first `amdgpu`, then `radeon`.

Set module parameters in `modprobe.d`.

Edit file...

    sudo vim /etc/modprobe.d/amdgpu.conf
    
...with contents:

    options amdgpu si_support=1
    options amdgpu cik_support=1
    
Edit file...

    sudo vim /etc/modprobe.d/radeon.conf

...with contents:

    options radeon si_support=0
    options radeon cik_support=0
    
Regenerate the initramfs:

**TODO this series of commands repeats here multiple times - abstract this into one file which will only be referenced from here multiple times as a link**

    ./remount_boot_part_as_writable.sh

Generate for all kernels

    sudo mkinitcpio --allpresets

or only for current kernel

    KERNEL_NAME=$(cat /boot/loader/entries/arch.conf | grep vmlinuz | cut -d'/' -f2 | cut -d'-' -f1 --complement)
    sudo mkinitcpio --preset $KERNEL_NAME
    
Close all programs ans reboot

Verify

    dmesg | grep -E "(amd|radeon)" | less
    less ~/.local/share/xorg/Xorg.0.log
    modinfo -p amdgpu | less
    systool -m amdgpu -av | less
    modinfo -p radeon | less
    systool -m radeon -av | less
    modprobe -c | grep -E "(amd|radeon)" | less
    less /var/log/Xorg.0.log
    
Test powersaving settings

In my case, for the Kabini APU, the [powersaving parameter calls `dpm`](https://wiki.archlinux.org/index.php/ATI#Powersaving) and is listed as a supported module parameter in the output of the command

    modinfo -p amdgpu | grep dpm
    
Check the [powersaving mode](https://wiki.archlinux.org/index.php/ATI#Dynamic_power_management)

    sudo cat /sys/class/drm/card0/device/power_dpm_state
    
Check the current performance level

    sudo cat /sys/class/drm/card0/device/power_dpm_force_performance_level
    
Later you can try to simplify the configuration and 
- remove `radeon` from `MODULES` in `/etc/mkinitcpio.conf` and 
- move `radeon.conf` from `/etc/modeprobe.d` into e. g. home directory.

This simplification came from the message from `vkmark` when is said in the output

    WARNING: radv is not a conformant vulkan implementation, testing use only.
    
After some searching I decided to make a simpler modules configuration according to [this post](https://forums.lutris.net/t/solved-lutris-says-radeon-hd-8670-not-vulkan-compatible/10307/3)

And when somethong goes wrong, the Arch Linux boot USB will help you to restore the previous state manually.

**`nomodeset` driver - when modesetting fails**

Install the [Xorg nomodeset driver] for AMD. To decide which one to install, decide according to the GCN version or by hardware model [here (Radeon)](https://wiki.gentoo.org/wiki/Radeon) and [here (AMD)](https://wiki.gentoo.org/wiki/AMDGPU) or by trial-and-error. Then it might be useful to have a bootable Arch Linux USB installation media nearby to boot from it and fix things when they go wrong.

If you decided to follow the trial-and-error method, first try the driver for AMD:

    sudo pacman -Syy xf86-video-amdgpu

Close all programs and reboot.

If you see the graphical desktop environment, then everything was set up correctly.

Otherwise uninstall the AMD GPU and install the ATI GPU driver

    sudo pacman -Runs xf86-video-amdgpu
    
    # or
    
    sudo pacman -Syy xf86-video-ati
    
Close all programs and reboot.

Verify

    less ~/.local/share/xorg/Xorg.0.log
    less /var/log/Xorg.0.log
    
[Monitor GPU features and frequencies](https://wiki.archlinux.org/index.php/AMDGPU#Monitoring)

    watch -n 1 "sudo cat /sys/kernel/debug/dri/0/amdgpu_pm_info | tail -n 5"
    
Check if you have enough VRAM allocated. If the values are close to the VRAM capacity, increase it in the BIOS settings

    sudo cat /sys/class/drm/card0/device/mem_info_vram_used

- https://wiki.archlinux.org/index.php/Hardware_video_acceleration#ATI/AMD
- https://wiki.archlinux.org/index.php/AMDGPU
- https://wiki.archlinux.org/index.php/Vulkan

---

If even that doesn't work, use the [`xf86-video-amdgpu`](https://wiki.gentoo.org/wiki/AMDGPU#Hardware_detection) or [`xf86-video-ati`](https://wiki.gentoo.org/wiki/Radeon#Hardware_detection) according to your GPU support. 

With these nomodeset Xorg drivers you can use the [Xorg options in `modprobe.d`](https://wiki.archlinux.org/index.php/AMDGPU#Xorg_configuration) [1](https://jlk.fjfi.cvut.cz/arch/manpages/man/amdgpu.4), [2](https://wiki.archlinux.org/index.php/Xorg#Driver_installation), [3](https://wiki.archlinux.org/index.php/Xorg#AMD).

---

To check allocated GPU memory for integrated graphics, execute command

    lspci -vvv -s $(lspci | grep VGA | cut -d' ' -f1)

https://www.cyberciti.biz/faq/howto-find-linux-vga-video-card-ram/

### Install desktop environment

XFCE4

    sudo pacman -S xfce4 xfce4-goodies xorg-apps

### Login

Now we have to decide, if we want to log in to our computer from
GUI (desktop/login manager - little unstable, but pretty) or from terminal (fast and secure)

If you use a graphical login manager and you're switching from some other graphical environment, don't forget to switch desktop enviroment in your login manager, e.g. sddm. See [how to swap desktop environments](https://github.com/kyberdrb/Linux_tutorials/blob/master/ARCH_How_to_change_desktop_enviroments.txt)

### TERMINAL LOGIN

#### Edit file `~/.xinitrc`

        sudo pacman -S xorg-xinit
        cp /etc/X11/xinit/xinitrc ~
        mv xinitrc .xinitrc #add dot before filename: mv xi<Tab> xi<Tab><Alt+b>.<Enter>
        nano ~/.xinitrc

1. Delete everything after the `fi` i.e. after the end of the condition after the line `# start some nice programs`.

1. At the very end of the file add command to start a destop environment

        exec startlxqt

    or

        exec startxfce4
    
    depending on which desktop environment is installed.

1. Save and exit.

Sample `~/.xinitrc`

    $ cat ~/.xinitrc

    #!/bin/sh

    userresources=$HOME/.Xresources
    usermodmap=$HOME/.Xmodmap
    sysresources=/etc/X11/xinit/.Xresources
    sysmodmap=/etc/X11/xinit/.Xmodmap

    # merge in defaults and keymaps

    if [ -f $sysresources ]; then







        xrdb -merge $sysresources

    fi

    if [ -f $sysmodmap ]; then
        xmodmap $sysmodmap
    fi

    if [ -f "$userresources" ]; then







        xrdb -merge "$userresources"

    fi

    if [ -f "$usermodmap" ]; then
        xmodmap "$usermodmap"
    fi

    # start some nice programs

    if [ -d /etc/X11/xinit/xinitrc.d ] ; then
     for f in /etc/X11/xinit/xinitrc.d/?*.sh ; do
      [ -x "$f" ] && . "$f"
     done
     unset f
    fi

    exec startxfce4

Edit file `~/.bash_profile`

`TODO nano...`

Add this to the end of the file

    exec startx

This will start X server right after login.
Save, exit.

Sample `~/.bash_profile`

    $ cat ~/.bash_profile

    #
    # ~/.bash_profile
    #
    
    [[ -f ~/.bashrc ]] && . ~/.bashrc
    
    exec start

1. Reboot

        reboot

    You will be greeted with a command line login prompt.

    Enter your username and password. The desktop environment will start immediately.
    
## Verifying graphics driver installation

Verify what graphics driver got activated

    LIBGL_DEBUG=1 glxinfo -B
    xdriinfo
    less ~/.local/share/xorg/Xorg.0.log
    less /var/log/Xorg.0.log
    
[Verify modesetting driver](https://wiki.archlinux.org/index.php/Intel_graphics#Module-based_options)

    modinfo -p i915 | less
    systool -m i915 -av | less
    
Run benchmark utilities for testing GPU performance and HW acceleration

Temportarily disable warning `MESA-INTEL: warning: Performance support disabled, consider sysctl dev.i915.perf_stream_paranoid=0`  
[[for permanent deactivation for development purposes see step with `dev.i915.perf_stream_paranoid` - at the time of writing it's step 7.]](https://software.intel.com/content/www/us/en/develop/articles/enabling-vulkan-vk-intel-performance-query-extension-in-ubuntu.html)

    sudo sysctl -w dev.i915.perf_stream_paranoid=0
    
Benchmark GPU performance

    vblank_mode=0 glxgears -fullscreen
    vkmark

### AUR helper utility

I'll install `pikaur` beacuse it's the most convenient and reliable package helper I've ever used.

#### Installation

Prepare packages for `pikaur`

    sudo pacman -S openssh git base-devel

- Installation: 
        
        cd /tmp
        git clone https://github.com/actionless/pikaur.git
        cd pikaur
        makepkg -fsri
	
    Reinstall `pikaur` by itself
    
        pikaur -S pikaur
        
    Source: https://github.com/actionless/pikaur#installation
      
#### Configuration

Create `pikaur` config file...

    $ vim ~/.config/pikaur.conf
    
The rest will be handled with the Arch Linux updating script.

### Enable hardware acceleration for GPU

Smoother video playback, less strain on CPU, more strain on GPU. Instead of the CPU doing the rendering, the rendering is offloaded to the GPU.

	sudo pacman -S libva lib32-libva

#### For Intel GPUs

Intel uses VAAPI to offload video decoding to the graphics processor.

Intel GPUs don't yet support VDPAU API, and there is no need to install translation layer packages to map VDPAU calls to VAAPI because it always be less efficient and slower than VAAPI optimized application. [1](https://bbs.archlinux.org/viewtopic.php?pid=1578000#p1578000), [2](https://bbs.archlinux.org/viewtopic.php?pid=1578078#p1578078)

There are multiple ways how to enable VAAPI hardware acceleration on the Linux platform.

For Intel HD 520 there are at least three VAAPI drivers I know of that enable hardware accelerated video playback:
1. `intel-media-sdk` `intel-media-driver` `intel-compute-runtime` `ocl-icd` `lib32-ocl-icd` `opencl-headers`
1. **`libva-intel-driver` `lib32-libva-intel-driver`**
1. `libva-intel-driver-hybrid` `intel-hybrid-codec-driver`

Package `intel-media-sdk` add support for Intel Quick Sync hardware acceleration for videos. That might sound good, but it doesn't accelerate the VP9 codec, so the videos and steams encoded in this codec are decoded via CPU, not GPU, which results to higher power consumption and stuttering. So, I'll rather stick to the `hybrid` drivers because of the VP9 hardware acceleration. Package `intel-media-driver` has, at the time of writing, reported some [issues with Firefox under Xorg](https://www.linuxquestions.org/questions/debian-26/intel-10th-gen-comet-lake-graphics-driver-issues-4175683860/#post6178511). There is workaround - to disable sandboxing for web media content - which might me a serious security issue. [1](https://wiki.archlinux.org/index.php/Firefox#Hardware_video_acceleration), [2](https://mastransky.wordpress.com/2020/06/03/firefox-on-fedora-finally-gets-va-api-on-wayland/). OpenCL headers are not necessary, but what if I'll develop something with it? Who knows...

[`libva-intel-driver`](https://archlinux.org/packages/extra/x86_64/libva-intel-driver/)/[`lib32-libva-intel-driver`](https://archlinux.org/packages/multilib/x86_64/lib32-libva-intel-driver/) are Intel VAAPI drivers **without** VP8 and VP9 acceleration and relying only on MP4 format encoded with AVC/H264 codecs (usually distributed as MP4 container) for hardware acceleration. With the `libva-intel-driver` VAAPI driver, `vaapi` output and [chromium://gpu](chromium://gpu) on my laptop's Intel HD 520 had showed me that the system and browser is able to decode VP8 with hardwre, but not VP9.

Because I really wanted to have the new VP9 format decoded mainly by my GPU (which is in my daily workload underutilized), not CPU (which is in my workload overutilized) I decided to install the hybrid driver to extend hardware acceleration support to VP8 and VP9 codecs install  
[`libva-intel-driver-hybrid`](https://aur.archlinux.org/packages/libva-intel-driver-hybrid/) which installs as a dependency the  
[`intel-hybrid-codec-driver`](https://aur.archlinux.org/packages/intel-hybrid-codec-driver/) which is [referenced in the Arch Wiki](https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Intel),  
but in the comment section of `intel-hybrid-codec-driver` [_randomgeek78_ recommends to use the `libva-intel-driver-hybrid`](https://aur.archlinux.org/packages/intel-hybrid-codec-driver/#comment-765336) via the environment variable `LIBVA_DRIVER_NAME=i965` which points to the `libva-intel-driver`,  
instead of directly using the `intel-hybrid-codec-driver` via e. g. `LIBVA_DRIVER_NAME=hybrid vainfo`,  
because the codec driver enhances the functionality of the `libva-intel-driver` under `ls /usr/lib/dri/i965_drv_video.so`  
and also `libva-intel-driver-hybrid` contains `intel-hybrid-codec-driver` as an optional dependency.  
`libva-intel-driver-hybrid` conflicts with the vanilla `libva-intel-driver`  
Both of these packages doesn't have their 32-bit alternatives, i. e. `lib32-libva-intel-driver-hybrid` / `lib32-intel-hybrid-codec-driver`

[`intel-hybrid-codec-driver`](https://aur.archlinux.org/packages/intel-hybrid-codec-driver/) failed at compilation and at the time of writing it was last updated at 21.12.2019, and the  `intel-hybrid-codec-driver-gcc10` was removed, therefore I'm using the `libva-intel-driver` `lib32-libva-intel-driver` combination.

As the final decision I chose the third option - the hybrid drivers:

    pikaur -Syy libva-intel-driver lib32-libva-intel-driver

Verify if the VAAPI driver is present and the formats that are accelerated through VAAPI.

    ls /usr/lib/dri/hybrid_drv_video.so
    LIBVA_DRIVER_NAME=i965 vainfo
    
    vainfo: VA-API version: 1.10 (libva 2.10.0)
    vainfo: Driver version: Intel i965 driver for Intel(R) Skylake - 2.4.1
    vainfo: Supported profile and entrypoints
          VAProfileMPEG2Simple            :	VAEntrypointVLD
          VAProfileMPEG2Simple            :	VAEntrypointEncSlice
    ...

Although I used the `intel-hybrid-codec-driver-gcc10` instead of the `intel-hybrid-codec-driver`, the `vaapi` output shows a 

sudo vim /etc/environment

Explicitly define the VAAPI driver name and try to enable hardware acceleration for MPEG4 format explicitely by defining a new [environment variable](https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Configuring_VA-API):

    ...
    LIBVA_DRIVER_NAME=i965
    VAAPI_MPEG4_ENABLED=true
    ...

Reboot

Verify VAAPI status

    vainfo

After installing the hybrid drivers, Chromium can play 1440p 60fps videos without additional configuration even smoother than mpv. mpv needs extra configuration to make it smooth [(see `mpv` package in the package list)](https://github.com/kyberdrb/Linux_tutorials/edit/master/ARCH_installed_packages_user.md) - but the smoothness is quite similar.

`libva-intel-driver-hybrid` together with `intel-hybrid-codec-driver` didn't produce the errors about missing OpenGL file in mpv player with VAAPI hw acceleration enabled '--hwdec=auto' compared to the `intel-media-driver`. I'm satisfied with the performance of the `hybrid` drivers. I'll keep them for now.

---
    
**For AMD/ATI GPUs or AMD APUs:**

    sudo pacman -Syy libva-mesa-driver lib32-libva-mesa-driver mesa-vdpau lib32-mesa-vdpau

This add support for VAAPI and VDPAU [hardware acceleration for AMD/API GPU](- https://wiki.archlinux.org/index.php/Hardware_video_acceleration#ATI/AMD).

According to [VA-API drivers supported formats table](https://wiki.archlinux.org/index.php/Hardware_video_acceleration#VA-API_drivers) we can see that the MPEG4 format is not supported for hardware acceleration. But we can still try to force it via the variable `VAAPI_MPEG4_ENABLED`.  
AMDGPU supports hardware acceleration through VAAPI and VDPAU drivers. Therefore we need to define another [enviroment variable `VDPAU_DRIVER`](https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Configuring_VDPAU)  and set it to the value given by command `grep -iE 'vdpau | dri driver' /var/log/Xorg.0.log`  Int he case of my AMD Kabini APU build, the output signalized a `radeonsi` name of the GPU type. Therefore we set the variable `VDPAU_DRIVER` to `radeonsi`. The resulting `/etc/environment` looks like this for AND Kabini APU build:

    sudo vim /etc/environment

    LIBVA_DRIVER_NAME=radeonsi
    VDPAU_DRIVER=radeonsi
    VAAPI_MPEG4_ENABLED=true
    
Although ATI and AMD GPUs support VDPAU acceleration, the VAAPI backend is in my experience always faster than VDPAU. [1](https://bbs.archlinux.org/viewtopic.php?pid=1578000#p1578000), [2](https://bbs.archlinux.org/viewtopic.php?pid=1578078#p1578078)
    
Continue with the verification of the VAAPI and VDPAU drivers
    
#### Verify hardware acceleration for graphics

Check `VAAPI` [Intel, AMD] configuration...
    
    $ sudo pacman -S libva-utils
    $ vainfo
    
    vainfo: VA-API version: 1.10 (libva 2.10.0)
    vainfo: Driver version: Intel i965 driver for Intel(R) Skylake - 2.4.1
    vainfo: Supported profile and entrypoints
          VAProfileMPEG2Simple            :	VAEntrypointVLD
          VAProfileMPEG2Simple            :	VAEntrypointEncSlice
          VAProfileMPEG2Main              :	VAEntrypointVLD
          VAProfileMPEG2Main              :	VAEntrypointEncSlice
          VAProfileH264ConstrainedBaseline:	VAEntrypointVLD
          VAProfileH264ConstrainedBaseline:	VAEntrypointEncSlice
          VAProfileH264ConstrainedBaseline:	VAEntrypointEncSliceLP
          VAProfileH264ConstrainedBaseline:	VAEntrypointFEI
          VAProfileH264ConstrainedBaseline:	VAEntrypointStats
          VAProfileH264Main               :	VAEntrypointVLD
          VAProfileH264Main               :	VAEntrypointEncSlice
          VAProfileH264Main               :	VAEntrypointEncSliceLP
          VAProfileH264Main               :	VAEntrypointFEI
          VAProfileH264Main               :	VAEntrypointStats
          VAProfileH264High               :	VAEntrypointVLD
          VAProfileH264High               :	VAEntrypointEncSlice
          VAProfileH264High               :	VAEntrypointEncSliceLP
          VAProfileH264High               :	VAEntrypointFEI
          VAProfileH264High               :	VAEntrypointStats
          VAProfileH264MultiviewHigh      :	VAEntrypointVLD
          VAProfileH264MultiviewHigh      :	VAEntrypointEncSlice
          VAProfileH264StereoHigh         :	VAEntrypointVLD
          VAProfileH264StereoHigh         :	VAEntrypointEncSlice
          VAProfileVC1Simple              :	VAEntrypointVLD
          VAProfileVC1Main                :	VAEntrypointVLD
          VAProfileVC1Advanced            :	VAEntrypointVLD
          VAProfileNone                   :	VAEntrypointVideoProc
          VAProfileJPEGBaseline           :	VAEntrypointVLD
          VAProfileJPEGBaseline           :	VAEntrypointEncPicture
          VAProfileVP8Version0_3          :	VAEntrypointVLD
          VAProfileVP8Version0_3          :	VAEntrypointEncSlice
          VAProfileHEVCMain               :	VAEntrypointVLD
          VAProfileHEVCMain               :	VAEntrypointEncSlice
          VAProfileVP9Profile0            :	VAEntrypointVLD             <<< yay :D
        
...and `VDPAU` [AMD, NVIDIA] configuration

    $ sudo pacman -Syy vdpauinfo
    $ vdpauinfo
    [outputs some video codec acceleration info]
    
If you get any kind of error in the outpu, or a `Segmentation fault` at the end of the `vdpauinfo` output, the value of the `VDPAU_DRIVER` variable is set to a value which doesn't match with the GPU type in the output. Check the output of the command `` again. Make sure the driver is available under ``. If not, reinstall VDPAU drivers with `sudo pacman -Syy mesa-vdpau lib32-mesa-vdpau`, reboot the computer and run the `vdpauinfo` utility again.

Reboot to activate hardware acceleration, if you haven't already.

Sources:

https://wiki.archlinux.org/index.php/Hardware_video_acceleration#VA-API_drivers - Note #4 - force MPEG4 VAAPI support

[[SOLVED] So, which is better; VDPAU or VAAPI?](https://bbs.archlinux.org/viewtopic.php?pid=1343287#p1343287)

https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Verifying_VA-API

https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Verifying_VDPAU
    
## Sound

Install sound server and graphical front-end

    pikaur -S pulseaudio pavucontrol alsa-utils
    alsactl restore
    
Installing `alsa-utils` package and issuing command `alsactl restore` enlived the audio output from the headphone jack on `Dell Latitude E5470`.
    
Reboot

    reboot
    
Sources:
- https://bbs.archlinux.org/viewtopic.php?pid=1749454#p1749454
- https://www.reddit.com/r/archlinux/comments/7ahv5n/linux_41310_no_audio/?st=ja1l4lq2&sh=65b079bd
- https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture#ALSA_Utilities
    
## Enable tap-to-click and natural scrolling for touchpad

### Automatic

[Tap-to-Click](https://github.com/kyberdrb/Linux_utils_and_gists/blob/master/touchpad_tap-to-click_enable.sh)

[Natural Scrolling](https://github.com/kyberdrb/Linux_utils_and_gists/blob/master/touchpad_natural_scrolling-enable.sh)	

### TODO-Scriptify touchpad features autostart

Start these scripts at start by creating following files in XFCE4 autostart directory. Below are examples of such files. 

	vim ~/.config/autostart/Touchpad_Tap-to-Click.desktop

	[Desktop Entry]
	Encoding=UTF-8
	Type=Application
	Name=Touchpad_Tap-to-Click
	Comment=
	Exec=/home/laptop/git/kyberdrb/Linux_utils_and_gists/touchpad_tap-to-click_enable.sh
	OnlyShowIn=XFCE;
	RunHook=0
	StartupNotify=false
	Terminal=false
	Hidden=false

	vim ~/.config/autostart/Touchpad_Natural_Scrolling.desktop

	[Desktop Entry]
	Encoding=UTF-8
	Type=Application
	Name=Touchpad_Natural_Scrolling
	Comment=
	Exec=/home/laptop/git/kyberdrb/Linux_utils_and_gists/touchpad_natural_scrolling-enable.sh
	OnlyShowIn=XFCE;
	RunHook=0
	StartupNotify=false
	Terminal=false
	Hidden=false

Source: https://edoceo.com/sys/xfce-autostart-apps

The entries should be then visible in the `Applications - Settings - Session and Startup - Application Autostart` tab.

### Manual

1. List IDs of all input devices

        $ xinput list
        
        ...
        ⎜   ↳ Virtual core XTEST pointer              	id=4	[slave  pointer  (2)]
        ⎜   ↳ AlpsPS/2 ALPS DualPoint Stick           	id=13	[slave  pointer  (2)]
        ⎜   ↳ AlpsPS/2 ALPS DualPoint TouchPad        	id=14	[slave  pointer  (2)]
        ...

1. Identify the touchpad device

    This line is telling me the ID of the touchpad device which is `14`:

    **AlpsPS/2 ALPS DualPoint TouchPad        	id=14	[slave  pointer  (2)]**

1. List the capabilities of the touchpad

        $ xinput list-props 14

        Device 'AlpsPS/2 ALPS DualPoint TouchPad':
            Device Enabled (165):	1
            Coordinate Transformation Matrix (167):	1.000000, 0.000000, 0.000000, 0.000000, 1.000000, 0.000000, 0.000000, 0.000000, 1.000000
            libinput Tapping Enabled (318):	0
            ...
            libinput Natural Scrolling Enabled (300):	0
            ...
	    
1. Enable `Tapping` and `Natural Scrolling`

        xinput set-prop 14 318 1
        xinput set-prop 14 300 1

1. Verify touchpad settings

        $ xinput list-props 14

        Device 'AlpsPS/2 ALPS DualPoint TouchPad':
            Device Enabled (165):	1
            Coordinate Transformation Matrix (167):	1.000000, 0.000000, 0.000000, 0.000000, 1.000000, 0.000000, 0.000000, 0.000000, 1.000000
            libinput Tapping Enabled (318):	1
            ...
            libinput Natural Scrolling Enabled (300):	1
            ...

1. Add it to autostart
    - XFCE4
        1. Go to `Applications -> Settings -> Session and Startup`
	1. Tab `Application Autostart`
	1. Click on `Add` button
	1. Create a _tap-to-click_ startup task
	    - **Name:** `Touchpad: tap-to-click`
	    - **Command:** `xinput set-prop 14 318 1`
	    - ***Trigger:* `on login`
	1. Create a _natural scrolling_ startup task
	    - **Name:** `Touchpad: tap-to-click`
	    - **Command:** `xinput set-prop 14 300 1`
	    - ***Trigger:* `on login`
	    
	1. Log out and back in to verify the startup tasks are effective.

Sources:
- https://askubuntu.com/questions/1087328/lubuntu-18-10-how-to-activate-tap-to-click/1089387#1089387
- https://wiki.archlinux.org/index.php/Libinput#Via_xinput

****************************************
POST-INSTALL

Now we all see our desktop environment. 
Let's make some changes.



sudo pacman -S breeze-icons

---

Configure screen locking

- When I pressed `Fn + Insert` to put the system to sleep, the lock screen appeared instead of going immediately to sleep. When I logged back in after a while the Power Manager displayed a message _None of the screen lock tools ran successfully, the screen will not be locked_. The Power Manager had the _Lock screen when system is going to sleep_ checkbox checked. when I unchecked this checkbox, the system went to sleep immediately. The system goes to sleep immediately after I executed the xfce4 suspend command. Then I found out to bind the xfce suspend command to the same keyboard shortcut `Fn + Insert`. But I got the same lockscreen with the error message after logging back in. Then I tried to bind it to a different keyboard shortcut. Then the system went to sleep immediately just like with the xfce suspend command.
    - **Solution:**
    
    	1. Go to `Applications -- Settings -- Power Manager -- System [tab]`.
        1. Uncheck checkbox _Lock screen when system is going to sleep_.
        1. Go to `Applications -- Settings -- Screensaver -- Lock Screen [tab]`.
        1. Check checkbox _Lock Screen with System Sleep_.
	    The checkbox _Lock screen when system is going to sleep_ under `Applications -- Settings -- Power Manager -- System [tab]` will also be checked [these two settings are probably linked].
        1. Go to `Applications -- Settings -- Keyboard -- Application Shortcuts [tab]`.
        1. Click _Add_ button to add a new keyboard shortcut to put the system to sleep.
        1. As for command, enter `xfce4-session-logout --suspend`
        1. Press _OK_ to confirm the command.
        1. A _Command Shortcut_ popup window appears. 
	    Press `Ctrl + Alt + Insert` to assign keyboard shortcut to the command.
        1. Test the new keyboard shortcut.
	    The system will go to sleep immediately without any error or warning messages.
	    After waking up the system will display a lock screen to enter the password, although the checkbox _Lock screen when system is going to sleep_ in Power Manager is unchecked. 
            The checked option _Lock Screen with System Sleep_ in Screensaver is taking care of it.

    https://wiki.archlinux.org/index.php/Xfce#Suspend
    
---

Sometimes when I press `Fn + Insert`, i. e. `Fn + SleepButton` (Insert is my functional Fn-Sleep button) and **close the lid**, then when I open the lid after some time (don't know how long, because immediately after sleep and closing the lid the computer wakes up alright) instead of waking up and displaying the xscreenserver password prompt, **it reboots**.

**_Note: As of April 19th 2022 I use only the default kernel parameter, and I removed the added kernel parameters in `arch.conf`. I will observe, how the sleep and wakeup from sleep will work on my laptop, and whether the sleep issue comes up again._**

Fix:

1. Go to Application Menu - Settings - Keyboard - (tab) Application Shortcuts
1. Click `Add` to add a new keyboard shortcut. Enter these parameters
    - Command: `xfce4-session-logout --suspend`
    - Map this command to keyboard shortcut `Ctrl + Alt + Insert` (or more generally `Ctrl + Alt + SleepButton` whatever key your sleep function button is located at)

Close the Keyboard settings window and test the new sleep-keyboard shortcut. Close the lid. Open the lid. The computer will resume to the xscreensaver password prompt after any amount of time. I don't know what causes it.

It seems that the problem is much deeper than I thought. It looks like it's a firmware or CPU issue. Nevermind. Following procedure fixed the awakening from sleep for my laptop:

    - installing packages `cpupower cpupower-gui turbostat`
    - Changing CPU frequency and disabling C-States, i.e. locking the CPU at the highest turbo frequency, using kernel parameters to resolve the issue of rebooting at awakening laptop from sleep.
      - added/appended kernel parameters into `/boot/loader/entries/arch.conf` next to `options`:

              options <other kernel parameters> quiet splash acpi_sleep=nonvs intel_idle.max_cstate=0 processor.max_cstate=0 idle=poll

    - Using installed utilities to monitor CPU frequency
    - Using Sensor Monitor to check CPU temperature
    - Sources
        - https://wiki.archlinux.org/index.php/CPU_frequency_scaling#cpupower
        - https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.txt
        - https://askubuntu.com/questions/716957/what-do-the-nomodeset-quiet-and-splash-kernel-parameters-mean/716966#716966
        - https://unix.stackexchange.com/questions/291546/laptop-reboots-instead-of-resuming-from-systemd-suspend-when-on-battery-power-s/435937#435937
        - https://bugzilla.kernel.org/show_bug.cgi?id=108801#c46
        - https://xuantuyen311.wordpress.com/2015/08/01/disablce-c-states-in-ubuntu/ - maybe the most helpful article
        - https://kernel.org/doc/html/v5.18-rc1/admin-guide/kernel-parameters.html
        - https://duckduckgo.com/?q=psi+Enable+or+disable+pressure+stall+information+tracking&ia=web

---

Question:  
My system doesn't boot after upgrading to 5.11 kernel. Booting ends up with a kernel panic.

Answer:  
Add kernel flag `module_blacklist=dell_wmi_sysman`, but that results in a slower system. See `options` line in `/boot/loader/entries/arch.conf`

    options <other kernel parameters> module_blacklist=dell_wmi_sysman

Then any of the 5.11.X kernels booted up.  
Alternatively you can specify for kernel parameters `acpi=off` or `iommu=off` and see, how your system behaves: Does it boot? Is the responsiveness different? Slower?

A better solution is to either switch to the latest 5.10.X _stable_ kernel version, or to Linux _LTS (Long-Term Support)_ kernel.

Sources:
- https://duckduckgo.com/?q=dell+5.11+kernel+panic&ia=web
- https://bbs.archlinux.org/viewtopic.php?id=264340
- https://bugzilla.kernel.org/show_bug.cgi?id=212069
- https://wiki.gentoo.org/wiki/IOMMU_SWIOTLB
- https://bbs.archlinux.org/viewtopic.php?id=264136
- https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.txt
- https://wiki.archlinux.org/index.php/Kernel_module#Using_kernel_command_line_2
- https://wiki.archlinux.org/index.php/Intel_graphics#Enable_GuC_/_HuC_firmware_loading
- https://wiki.archlinux.org/index.php/Intel_GVT-g#Prerequisite
- https://askubuntu.com/questions/922140/how-to-know-i-have-to-blacklist-acer-wmi/922141#922141
- [Bug 211895 - dell_wmi_sysman causes unbootable system](https://bugzilla.kernel.org/show_bug.cgi?id=211895)
- [FS#69702 - [linux][linux-zen] 5.11, dell_wmi_sysman causes unbootable system](https://bugs.archlinux.org/task/69702)

****************************************
PROXY

Edit /etc/environment file:

  sudo nano /etc/environment

And edit its content to something like this:

    <snip>
    #http_proxy=http://192.168.0.3:3128/
    #https_proxy=http://192.168.0.3:3128/
    #ftp_proxy=http://192.168.0.3:3128/
    #no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
    #HTTP_PROXY=http://192.168.0.3:3128/
    #HTTPS_PROXY=http://192.168.0.3:3128/
    #FTP_PROXY=http://192.168.0.3:3128/
    #NO_PROXY="localhost,127.0.0.1,localaddress,.localdomain.com"
    <snip>

At the top we setting system variables VDPAU and LIBVA to enable graphics acceleration for Intel graphics. Then we set up proxy server address for various protocols.

****************************************

### SSD OPTIMIZATION - TRIM

Check whether the SSD supports TRIM

	lsblk --discard
	
If columns `DISC-GRAN` and `DISC-MAX` are non-zero, the drive supports `TRIM`.

Enable Periodic TRIM

	sudo systemctl enable --now fstrim.timer
	
This will execute TRIM command on all TRIM-capable drives once a week (Periodic TRIM). You can verify contents of the timer service with command

	systemctl cat fstrim.timer
	
Check the status of the service:

	systemctl status fstrim.timer
	
    ● fstrim.timer - Discard unused blocks once a week
         Loaded: loaded (/usr/lib/systemd/system/fstrim.timer; enabled; preset: disabled)
         Active: active (waiting) since Thu 2022-09-29 11:32:29 CEST; 1 day 5h ago
          Until: Thu 2022-09-29 11:32:29 CEST; 1 day 5h ago
        Trigger: Mon 2022-10-03 00:18:54 CEST; 2 days left
       Triggers: ● fstrim.service
           Docs: man:fstrim

... and the service, to see that it's triggered by the timer service

	systemctl status fstrim.service
	
    ○ fstrim.service - Discard unused blocks on filesystems from /etc/fstab
        Loaded: loaded (/usr/lib/systemd/system/fstrim.service; static)
        Active: inactive (dead)
    TriggeredBy: ● fstrim.timer
        Docs: man:fstrim(8)

Periodic TRIM is safer, and more supported and less prone to errors than Continuous TRIM.

You can run TRIM manually with commands:

    /usr/bin/fstrim --listed-in /etc/fstab:/proc/self/mountinfo --verbose --quiet-unsupported

for running TRIM on all mounted disks, or with

    /usr/bin/fstrim --fstab
    
for running TRIM only on entries in `/etc/fstab`, where usually internal disks are present

After the time period indicated in `Trigger` line from the status of the `timer` service it's recommended to check whether the TRIM timer really triggers the TRIM service. Check the logs

    journalctl | grep --ignore-case discard

- Sources:
	- https://wiki.archlinux.org/title/Solid_state_drive#Periodic_TRIM
	- https://wiki.archlinux.org/title/Solid_state_drive#Periodic_TRIM
	- https://duckduckgo.com/?q=check+trim+support+linux&ia=web
	- https://wiki.archlinux.org/title/Installation_guide

****************************************
## NETWORK MANAGEMENT

### Terminal

I manage all networks in terminal.

For wireless networks I use `wifi-menu`.

For wired networks I use `dhcpcd`.

### Graphical

If we need to connect to the `eduroam` network, or to connect to the Ethernet automatically on plugging in the cable we can install a `Network Manager` utility together with its applet for the notification area

    pacman -S networkmanager network-manager-applet
    systemctl enable NetworkManager.service
    reboot
    
#### Connect to `eduroam` network

I failed to connect to the `eduroam` network with `wpa_supplicant` with a configuration generated by the `Configuration Assistant Tool` from [eduroam webpage](https://cat.eduroam.org).

This is a case where the `Network Manager` helped me to connect to the network.

I used these settings:
- Security: `WPA & WPA2 Enterprise`
- Authentication: `Protected EAP (PEAP)`
- Anonymous identity: `<I've left this field blank>`
- Domain: `<I've left this field blank>`
- Check `No CA certificate is required`
- PEAP version: `Automatic`
- Inner authentication: `MSCHAPv2`
- Username: `<student_id>@uniza.sk`
- Password: `<LDAP_password>`

## XFCE4 CONFIGURATION

### Quick connecting to a network

1. Open application finder

        Alt + F2

    - Wireless network

            xfce4-terminal -e 'bash -c "sudo wifi-menu"'
	
    - Ethernet

            xfce4-terminal -e 'bash -c "sudo dhcpcd enp0s31f6"'

First time you need to execute the entire command.

Next time it will be enough to type `wifi`. The command pops-up below the command line.

Source: https://askubuntu.com/questions/980720/open-xfce-terminal-window-and-run-command-in-same-window/983865#983865

---

- Terminal
    1. Open Terminal (Ctrl + Alt + T)
    1. In the menu bar click on `Terminal` item
    1. Disable `Scroll on output`
    1. `Edit -> Preferences`
    1. Tab `General`
        1. Uncheck `Show unsafe paste dialog`
    1. Tab `Appearance`
        1. Font
            1. Click on font
            1. Find `Monospace Bold`
            1. Set the font size to `14`
            1. Close the dialog window
        1. Tabs
            1. Check option `Use custom styling to make tabs slim`
        1. Close the Preferences window
        1. Check the settings by reopening a terminal window

- Desktop
    1. Go to the Desktop (Ctrl + Alt + D)
    1. Right click on empty space on the Desktop
    1. Choose `Desktop Settings`
    1. Background
        1. `Style:` None
        1. `Color:` Solid color
        1. Click on the first color box next to `Color` menu
        1. Choose _black_ color
        1. Select
    1. Icons
        1. Appearance
            1. `Icon type:` None
	    
    1. Go back to the Desktop
    1. Right click on the bottom panel
    1. Navigate to `Panel -> Panel Preferences`
    1. On the top of the dialog window click on a button with a minus sign on it. This will remove the panel.
    
1. Go to `Applications menu -> Settings`

- Session and Startup
    1. Tab `Application Autostart`
        1. `Add`
            - `Name:` Redshift
            - `Command:` `redshift`
            - `Trigger:` on login

- Keyboard
    - `Layout` tab
        1. Uncheck `User system defaults`
        1. Change layout option: Alt+Shift
        1. Click on "Add" button
        1. Add German layout
        1. Add a Slovak layout the same way
        1. Remove English layout

- Panel
    - `Items` tab
        - Workspace Switcher
        - PulseAudio Plugin - Volume icon
        - Power Manager Plugin - Battery indicator
        - Clock
        - Keyboar layout switcher

- Power Manager
    - `General` tab
        - Buttons
            - When power button is pressed: Shutdown (battery and plugged-in)
            - When sleep button is pressed: Suspend (battery and plugged-in)
            - Set the rest of the actions to `None`.
	    - Enable `Handle display brightness keys`. This prevents from occasional brigntness-key-blocking.
        - Laptop Lid
            - When laptop lid is closed: Suspend (battery and plugged-in)
    - `System` tab
        - Critical power
            - On critical battery power: `Ask`
    - `Display` tab
        - Display power management
	    - set everything to `Never`

- Removable Drives and Media [Optional]
    - Storage tab
        - Enable `Mount removable drives when hot-plugged`
        - Enable `Mount removable drives when inserted`
	
- Screensaver
    - `Lockscreen` tab
    	- disable `Lock Screen with Screensaver`. I will lock the screen manually with the keyboard shortcut `Ctrl + Alt + L`
    Source: https://askubuntu.com/questions/259717/power-manager-gui-settings-not-to-shut-down-the-display-are-not-followed/259723#259723
    
- Window Manager
    - Advanced tab
        - Windows snapping: Enable `To other windows`

- Window Manager Tweaks
    - Cycling tab
        - Enable `Cycle through windows in a list`
    - Compositor tab
        - Disable all shadow effects
        - Set all opacity to maximum, i. e. `Opaque`

- Workspaces
    - General tab
        - Number of workspaces: `2`
    - Margins tab
        - Left margin: `1`
        - Right margin: `1`
	
- [Change panel popup and popdown delay when autohide is enabled](https://forum.xfce.org/viewtopic.php?pid=60337#p60337)
    1. vim ~/.config/gtk-3.0/gtk.css
    1. Insert this contents [by pressing `i Ctrl+Shit+V`]:
    
            * {
            -XfcePanelWindow-popup-delay: 300;
            -XfcePanelWindow-popdown-delay: 1;
            -XfcePanelWindow-autohide-size: 0;
            } 

        When `autohide-size` property is set to `0` pixels, the panel will be hidden into a dot in the corner instead of a bar at 1px+.
    
    1. Then adjust panel properties by your preferences to your satisfaction.
    1. Close all programms and logout/reboot. Settings will update after XFCE restart.
	
---

Additional settings can be found in:
- https://github.com/kyberdrb/Linux_tutorials/blob/master/XFCE_ARCH_Installation_and_configuration.txt
- https://github.com/kyberdrb/Linux_tutorials/blob/master/XFCE_ARCH_Tearing_fix.txt
- https://github.com/kyberdrb/Linux_tutorials/blob/master/XFCE_repair_icons_in_application_menu.txt
- and other `XFCE` files...

## Additional programs

See "ARCH_installed_packages_user.txt".

For package installation automation, see
  https://bbs.archlinux.org/viewtopic.php?pid=470581#p470581
  
### Automount devices and enable Trash

Install required packages

    pikaur -S gvfs
  
For automounting mobile devices:

    pikaur -S gvfs-mtp gvfs-gphoto2

1. Go to `Applications menu -> Settings -> Removable Drives and Media`
1. check `Mount removable drives when hot-plugged` and `Mount removable media when inserted`
1. Disconnect the removable disk, if it's connected.
1. Plug in your removable device.
1. Open file manager.

The removeable device will appear in the left column.


If not, you can still mount the drive manually via terminal:

Check, which label corresponds your USB stick

  lsblk
or
  sudo fdisk -l

Then mount it

  gvfs-mount -d /dev/sdX#

where X is the drive name and # is the partition number, e.g.

  gvfs-mount -d /dev/sdb1


Sources:
  https://forum.voidlinux.eu/t/solved-xfce-missing-trash-and-other-icons-on-the-desktop/2010/2
  https://bbs.archlinux.org/viewtopic.php?pid=1608701#p1608701
  https://docs.xfce.org/xfce/thunar/using-removable-media
  https://www.youtube.com/watch?v=fYlBVkB1gn4
  
### Generate SSH certificate for GitHub

1. https://help.github.com/en/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent
1. https://help.github.com/en/articles/adding-a-new-ssh-key-to-your-github-account
1. https://help.github.com/en/articles/testing-your-ssh-connection

Set global `git` configuration attributes

    git config --global user.email "you@example.com"
    git config --global user.name "Your Name"
    
Now is the `git` SSH communication active and ready to communicate with GitHub servers.

## Updating

    cat update_arch.sh

    #!/bin/bash

    PIKAUR_INSTALLED=$(pacman -Q | grep pikaur)
    if [[ -z $PIKAUR_INSTALLED ]]; then 
        rm -rf /tmp/pikaur-git
        mkdir /tmp/pikaur-git
        curl https://aur.archlinux.org/cgit/aur.git/snapshot/pikaur-git.tar.gz --output /tmp/pikaur-git.tar.gz
        tar -xvzf /tmp/pikaur-git.tar.gz --directory /tmp/pikaur-git
        cd /tmp/pikaur-git/pikaur-git
        makepkg --ignorearch --clean --syncdeps --noconfirm
        PIKAUR_PACKAGE_NAME=$(ls *.tar*)
        sudo pacman -U $PIKAUR_PACKAGE_NAME --noconfirm
        rm -rf /tmp/pikaur-git
    fi

    rm -rf ~/.libvirt
    pikaur -Syyuu
    pikaur -Scc --noconfirm

    COMMAND=$1
    SHUTDOWN_TIME=$2
    if [[ $COMMAND == "--shutdown" ]]; then
        shutdown $SHUTDOWN_TIME
    fi


## Troubleshooting

Question:  
Can't update/upgrade system via pacman. Pacman displays these error messages:

    error: Partition /boot is mounted read only
    error: not enough free disk space
    error: failed to commit transaction (not enough free disk space)
    Errors occurred, no packages were upgraded.

Answer:  
Remount boot partition as `rw` - see https://github.com/kyberdrb/update_arch/blob/master/utils/remount_boot_part_as_writable.sh

Here is a script to run the entire update procedure without user intervention (unattended)- https://github.com/kyberdrb/update_arch

---

Question:  
The laptop reboots after waking up from sleep / The laptop doesn't go to sleep but locks the screen instead.

Answer:  
Reset the sleep mode

    $ cat /sys/power/mem_sleep 
    s2idle [deep]
    $ echo s2idle | sudo tee /sys/power/mem_sleep
    [sudo] password for root: 
    s2idle
    $ cat /sys/power/mem_sleep 
    [s2idle] deep
    $ echo deep | sudo tee /sys/power/mem_sleep
    deep
    $ cat /sys/power/mem_sleep 
    s2idle [deep]

Sources:
- Dell XPS 15 9570 Linux Sleep Suspend Fix: https://www.youtube.com/watch?v=f-u7Zk_itUU

Alternative:
- XFCE Session Restore: not fully functioning (some applications don't open, e.g. Atom) - XFCE save session: https://docs.xfce.org/xfce/xfce4-session/preferences

---

Question:  
In the output of `journalctl --boot` is listed an error message **`iwlwifi 0000:01:00.0: No beacon heard and the time event is over already...`**

Answer:  
Fixed by upgrading UEFI. **But from Windows**, not from Linux by `fwupd`. I rather do firmware upgrades from Windows because of the better support from the manufacturer.

Sources:  
- https://bugs.archlinux.org/task/58457

---

Question:  
In the output of `journalctl --boot` and in the terminal login screen is listed an error message **`Bluetooth: hci0: Reading supported features failed (-16)`**

Answer:  
Installing `linux-firmware-iwlwifi-git` fixes the issue with the firmware and removes the error message from the `journalctl` logs. Possible reasons for that is that at the time of writing (02/2020) the `linux-firmware` provided `iwlwifi` firmware from 2014 for my bluetooth device, whereas `linux-firmware-iwlwifi-git` reported firmware version from 2018.

    Bluetooth: hci0: Firmware revision 0.0 build 10 week 41 2018
    
Another solution is to disable bluetooth in BIOS/UEFI alltogether, and uninstalling everything bluetooth related in the system.

Sources:
- https://bbs.archlinux.org/viewtopic.php?id=126603
- https://wiki.archlinux.org/index.php/Bluetooth
- https://wiki.archlinux.org/index.php/Blueman
- https://wiki.archlinux.org/index.php/Laptop/Dell#Latitude
- https://aur.archlinux.org/packages/iwlwifi/
- https://aur.archlinux.org/packages/iwlwifi-next/
- https://duckduckgo.com/?q=arch+iwlwifi&ia=web
- https://aur.archlinux.org/packages/linux-firmware-iwlwifi-git/

---

Question:  
In the output of `journalctl --boot` is listed an error message **`psi: task underflow!`**

Answer:  
Disable `psi` in kernel parameters. In my case, I'm using `systemd-boot`, so I appended `psi=0` to the `/boot/loader/entries/arch.conf`

    title ARCH LINUX
    linux /vmlinuz-linux-tkg-muqss-skylake
    initrd /intel-ucode.img
    initrd /initramfs-linux-tkg-muqss-skylake.img
    options root=UUID=f4e9c51b-06a6-46e0-9af3-0d8f227a8917 rw psi=0

Sources:
- https://ck-hack.blogspot.com/2018/12/linux-420-ck1-muqss-version-0185-for.html?showComment=1546575567073#c4526898740671031723
- https://github.com/torvalds/linux/tree/master/Documentation/admin-guide
- https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.txt
- https://wiki.archlinux.org/index.php/Kernel_parameters#systemd-boot
- https://wiki.archlinux.org/index.php/Systemd-boot#Kernel_parameters_editor_with_password_protection

---

Question:  
In the output of `journalctl --boot` is listed an error message

    DMAR: [Firmware Bug]: No firmware reserved region can cover this RMRR [0x000000a9800000 end: 0x000000abffffff], contact BIOS vendor for fixes
    DMAR: [Firmware Bug]: Your BIOS is broken; bad RMRR [0x000000a9800000 end: 0x000000abffffff]

Answer:  
Fixed by upgrading UEFI. **But from Windows**, not from Linux by `fwupd`. I rather do firmware upgrades from Windows because of the better support from the manufacturer.

---

Question:  
Why didn't I updated firmware on my ADATA SU800 SSD disk?

Answer:  
I evaluated it as a too risky thing. Official upgrade procedure will likely soft-brick the device and make it temporary unusable. I say temporary, because there are ways to fix this messed up upgrade, but I want to save myself the hassle. The SSD works still very well, and I'm very satisfied with it. For that money, excelent value.

Sources:
- https://www.adata.com/en/download/410
- https://www.adata.com/en/ss/software-6/
- https://www.reddit.com/user/michel_olvera/comments/fzmf2p/adata_su800_firmware_recovery_upgrade/
- https://forums.anandtech.com/threads/which-firmware-for-adata-su800-ssd.2553663/
- https://blog.elcomsoft.com/2019/01/identifying-ssd-controller-and-nand-configuration/

---

Fix "Failed to acquire RNG protocol: not found" at boot

    systemctl status systemd-boot-system-token.service
    systemctl enable systemd-boot-system-token.service -- fails to enable; journalctl reports "systemd[1]: Condition check resulted in Store a System Token in an EFI Variable being skipped."
    sudo bootctl random-seed -- doesn't help, message still present
    
Leaving it as it is. Maybe they will fix this error and informational messager in the version 248. 

Therefore I'll not add the flag `random-seed-mode off` in `/boot/loader/loader.conf` which might possibly fix this issue.

I'll wait until systemd 248 is out.

- https://github.com/systemd/systemd/issues/13503#issuecomment-529545326
- https://github.com/systemd/systemd/issues/13603
- [The fix is close...](https://github.com/systemd/systemd/commit/391719682bf68134b01cf422eb92e3ec4686fa7b)
- [bootctl random-seed + systemd-boot-system-token.service failing to start]https://bbs.archlinux.org/viewtopic.php?pid=1864646#p1864646
- https://wiki.archlinux.org/index.php/Systemd-boot#Loader_configuration
- [Failed to start Store a System Token in an EFI Variable.](https://bbs.archlinux.org/viewtopic.php?id=250379)

---

added `intremap=no_x2apic_optout` to the `/boot/loader/entries/arch.conf` to the `options` line.

Reason: fixing error messages in `journalctl`

    ...
    kernel: DMAR: RMRR base: 0x000000ab53b000 end: 0x000000ab55afff
    kernel: DMAR: RMRR base: 0x000000ad800000 end: 0x000000afffffff
    kernel: DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
    kernel: DMAR-IR: HPET id 0 under DRHD base 0xfed91000
    kernel: DMAR-IR: x2apic is disabled because BIOS sets x2apic opt out bit.
    kernel: DMAR-IR: Use 'intremap=no_x2apic_optout' to override the BIOS setting.
    kernel: DMAR-IR: Enabled IRQ remapping in xapic mode
    kernel: x2apic: IRQ remapping doesn't support X2APIC mode
    ...

- https://stackoverflow.com/questions/51261999/check-if-vt-d-iommu-has-been-enabled-in-the-bios-uefi/51266134#51266134

After the flag

    ...
    kernel: DMAR-IR: IOAPIC id 2 under DRHD base  0xfed91000 IOMMU 1
    kernel: DMAR-IR: HPET id 0 under DRHD base 0xfed91000
    kernel: DMAR-IR: Queued invalidation will be enabled to support x2apic and Intr-remapping.
    kernel: DMAR-IR: Enabled IRQ remapping in x2apic mode
    kernel: x2apic enabled
    kernel: Switched APIC routing to cluster x2apic.
    ...

---

journalctl reports

    ACPI: [Firmware Bug]: BIOS _OSI(Linux) query ignored

- https://bbs.archlinux.org/viewtopic.php?pid=1930423#p1930423
- https://shantanugoel.com/2008/08/03/hi-bios-my-name-is-linux-or-is-it/

Solution: add flag `acpi_osi=Linux` to the `options` line in the `/boot/loader/entries/arch.conf`

- https://bugzilla.kernel.org/show_bug.cgi?id=69921
- https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1275985

After the flag

    ...
    ACPI: [Firmware Bug]: BIOS _OSI(Linux) query honored via cmdline
    ...

...and the rest was still the same.

But after that the login screen of xscreensaver/xfce4-screensaver froze before displaying leaving the system unusable, so don't do it and leave it as it is. Don't add this flag otherwise you risk an unstable system.

---

Automatic Bluetooth enabling in order to prevent error messages in `journalctl`

    sudo vim /etc/bluetooth/main.conf

    ...
    AutoEnable=true
    ...

Enable Bluetooth services

    systemctl enable bluetooth.service
    systemctl enable bluetooth-mesh.service
    systemctl enable blueman-mechanism.service

Edit modules at boot - early modesetting for Bluetooth module

    sudo vim /etc/mkinitcpio.conf

Append to the `MODULES` list the `bluetooth` module found by `lsmod | less` but missing in the output

`lsmod | less` output

    bnep                   28672  2
    ...
    btusb                  69632  0
    btrtl                  28672  1 btusb
    ...
    btbcm                  20480  1 btusb
    ...
    bluetooth             745472  16 btrtl,btintel,btbcm,bnep,btusb

Appended Intel bluetooth module in `/etc/mkinitcpio.conf`

    MODULES=(i915 btintel)

Reboot

- https://askubuntu.com/questions/938228/how-to-enable-bluetooth-at-startup-16-04-lts/938235#938235
- https://wiki.archlinux.org/index.php/Bluetooth#Auto_power-on_after_boot
- https://bbs.archlinux.org/viewtopic.php?pid=1861643#p1861643
- https://www.cyberciti.biz/faq/add-remove-list-linux-kernel-modules/
- https://askubuntu.com/questions/1278580/bluetooth-blueman-manager-not-working-on-ubuntu-20-04-1-lts
- https://superuser.com/questions/1358968/bluetooh-hci0-command-0x1009-tx-timeout/1497228#1497228
- https://bbs.archlinux.org/viewtopic.php?id=238924

Anyway, none of these solutions really worked and my Bluetooth device became unusable - detected but can't enable Bluetooth: `blueman-manager` reports no default Bluetooth adapters. Therefore I disabled the Bluetooth device in BIOS/UEFI. No more problems.

---

`journalctl` reports an error

    kernel: DMAR: [INTR-REMAP] Request device [f0:1f.0] fault index 0 [fault reason 37] Blocked a compatibility format interrupt request
    kernel: DMAR: DRHD: handling fault status reg 2

Solution: Disable VT-d/Virtualization for Directed I/O in BIOS/UEFI

Maybe adding a kernel flag `intel_iommu=igfx_off` but I didn't test it.

https://bbs.archlinux.org/viewtopic.php?pid=1375998#p1375998

---

Question:  
At update procedure and/or at upgrade of some package this message gets shown: `Please add your user to the brlapi group.`

Answer:  
Add user to the group with

    sudo gpasswd -a $USER brlapi


## Linux hardening

Commands

    lscpu | grep --ignore-case vulnerability

If each of the vulnerability has a "Mitigations", then you're protected. Enabled SMT/HyperThreading might be still a source of some security vulnerabilities, though.

    journalctl --boot | grep --ignore-case "CPU bug"
    
Shows currently active CPU bugs. In my case they're all related to enabled SMT/HyperThreading.

    grep -r . /sys/devices/system/cpu/vulnerabilities/
    grep -r . /sys/devices/system/cpu/vulnerabilities/ | wc -l
    grep -r "Mitigation" /sys/devices/system/cpu/vulnerabilities/ | wc -l
    
    find /sys/devices/system/cpu/vulnerabilities/* | nl
    find /sys/devices/system/cpu/vulnerabilities/* | xargs cat | nl
    
Double check if all security vulnerabilities are mitigated. When you see a "Mitigation" in each of the file, the system is protected by the kernel patches that mitigate these vulnerabilities.

As soon as I disabled HyperThreading, these `CPU bug` error messages didn't display as well, so disabling SMT/HyperThreading makes the system more secure at the one hand, at the other hand occasional stutterring and longer waiting times for CPU-bount tasks might appear.

- https://unix.stackexchange.com/questions/456425/what-does-the-bugs-section-of-proc-cpuinfo-actually-show/456446#456446
- https://wiki.archlinux.org/title/Security#Hardware_vulnerabilities

### Checking for other system issues

Log messages:

    journalctl --reverse --since "$(date '+%Y-%m-%d') 00:00:00" --until "$(date '+%Y-%m-%d') 23:59:59" | grep "\<bug\>" | cut -d' ' -f1,2,3,4,5 --complement | sort | uniq | less
    
or

    journalctl --reverse --boot | grep "\<bug\>" | cut -d' ' -f1,2,3,4,5 --complement | sort | uniq | less

    i915 0000:00:02.0: [drm] Panel advertises DPCD backlight support, but VBT disagrees. If your backlight controls don't work try booting with i915.enable_dpcd_backlight=1. If your machine needs this, please file a _new_ bug report on drm/i915, see https://gitlab.freedesktop.org/drm/intel/-/wikis/How-to-file-i915-bugs for details.
    L1TF CPU bug present and SMT on, data leak possible. See CVE-2018-3646 and https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/l1tf.html for details.
    MDS CPU bug present and SMT on, data leak possible. See https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/mds.html for more details.
    PCI: Using host bridge windows from ACPI; if necessary, use "pci=nocrs" and report a bug
    TAA CPU bug present and SMT on, data leak possible. See https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/tsx_async_abort.html for more details.

For more error messages execute command

    journalctl --reverse --boot | grep -e "\<bug\>" -e "\<failed\>" -e "\<off\>" -e "\<rejected\>" -e "\<error\>" | less

For full log output execute command

    journalctl --reverse --boot"

Mitigation

    ./remount_boot_part_as_writable.sh
    sudo vim /boot/loader/entries/arch.conf

Add in the file flags that will help to mitigate detected vulnerabilities and errors as needed.


Sources:

- https://wiki.archlinux.org/index.php/Security
- https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/l1tf.html
- https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/mds.html
- https://www.kernel.org/doc/html/latest/admin-guide/hw-vuln/tsx_async_abort.html
- https://askubuntu.com/questions/1250040/how-do-i-fix-mds-cpu-bug-present-and-smt-on-data-leak-possible-errors-from-lo/1250060#1250060
- https://www.cyberciti.biz/faq/check-linux-server-for-spectre-meltdown-vulnerability/
- https://forum.linuxfoundation.org/discussion/856998/lfs201-chapter-11-sys-devices-system-cpu-vulnerabilities
- https://lwn.net/Articles/785934/
- https://unix.stackexchange.com/questions/554908/disable-spectre-and-meltdown-mitigations/554922#554922
- https://askubuntu.com/questions/1248273/lscpu-vulnerabilities#1248288
- https://software.intel.com/content/www/us/en/develop/articles/intel-trusted-execution-technology-intel-txt-enabling-guide.html
- https://www.techulator.com/resources/6372-How-enable-NX-or-XD-option-BIOS-install-Windows-8-RP.aspx
- https://access.redhat.com/solutions/2936741

---

Question:  
When running `mkinitcpio` or upgrading the kernel, during the process appears a message `WARNING: Possibly missing firmware for module: xhci_pci`.

Answer:
Test the USB ports. If all of the USB ports are working, you can safely ignore this message.

If some of the USB doesn't detect a USB device, install the firmware for Renesas USB3.0 Controllers [`upd72020x-fw`](https://aur.archlinux.org/packages/upd72020x-fw/)

- https://github.com/linux-surface/linux-surface/issues/306#issuecomment-709203268
- https://www.reddit.com/r/archlinux/comments/k71mlr/warning_possibly_missing_firmware_for_module_xhci/geoag7c/?utm_source=reddit&utm_medium=web2x&context=3
- https://bbs.archlinux.org/viewtopic.php?pid=1922334#p1922334

---

Question:  
When installing the skylake optimized tkg kernel with muqss be running `pikaur -Sy linux-tkg-muqss-skylake linux-tkg-muqss-skylake-headers` I got a lot of errors in the sense of:
    
    linux-tkg-muqss-skylake: /usr/lib/modules/5.11.3-131-tkg-MuQSS/... exists in filesystem (owned by linux-tkg-muqss)

and
    
    linux-tkg-muqss-skylake-headers: /usr/lib/modules/5.11.3-131-tkg-MuQSS/... exists in filesystem (owned by linux-tkg-muqss-headers)

Answer:  
The skylake kernel specialization of the same kernel didn't install because I had the generic packages `linux-tkg-muqss` and `linux-tkg-muqss-headers` installed. Uninstalling them and installing the skylake optimized kernel again solved the problem.

    pikaur -Runs linux-tkg-muqss linux-tkg-muqss-headers
    pikaur -Sy linux-tkg-muqss-skylake linux-tkg-muqss-skylake-headers
    
- https://joshtronic.com/2020/09/13/reflector-service-exists-in-filesystem-owned-by-reflector-timer/

## Sources

[GloriousEggroll - 2017 Arch Linux EFI Install Guide](https://www.youtube.com/watch?v=iF7Y8IH5A3M)
- GloriousEggroll - 2016 Arch Linux EFI Install Guide Part 1 - Preparation and Disk Partitioning: https://www.youtube.com/watch?v=MMkST5IjSjY
- GloriousEggroll - 2016 Arch Linux EFI Install Guide Part 2 - Installing Arch and Making it Boot: https://www.youtube.com/watch?v=0WBB8v-tiz8
- GloriousEggroll - 2016 Arch Linux EFI Install Guide Part 3 - Making it user friendly and adding a desktop environment: https://www.youtube.com/watch?v=n5UK66GF99A
- GloriousEggroll - 2016 Arch Linux NetworkManager / Wifi Setup guide: https://www.youtube.com/watch?v=MAi9DurTRQc
- goguda55 Tech Tutorials - How to Install Arch Linux: https://www.youtube.com/watch?v=Wqh9AQt3nho
- https://wiki.archlinux.org/index.php/Solid_State_Drives#Periodic_TRIM
- https://www.digitalocean.com/community/tutorials/how-to-configure-periodic-trim-for-ssd-storage-on-linux-servers
- https://wiki.archlinux.org/index.php/pacman#Cleaning_the_package_cache
- https://wiki.archlinux.org/index.php/Pacman/Tips_and_tricks#Removing_unused_packages_.28orphans.29
- https://apple.stackexchange.com/questions/10139/how-do-i-increase-sudo-password-remember-timeout/51763#51763
- https://wiki.archlinux.org/index.php/Environment_variables#Defining_variables
- https://wiki.archlinux.org/index.php/Xinit#xinitrc

