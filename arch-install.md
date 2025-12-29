# Minimal Arch + i3-gaps WM

*Take into account that this guide is primarily written for my personal setup and tailored to my needs; however, it can still be considered universal, as the customizations are minimal and mainly involve the choice of programs—such as the terminal, text editor, and basic utilities—which can of course be changed ad hoc.*

## Pre-installation

### EFI-enabled BIOS

This guide assumes you have EFI-enabled BIOS. You can check it with:

       # ls /sys/firmware/efi/efivars

### Setup Keyboard
For a list of all acceptable keymaps:

       # localectl list-keymaps


Set up your keyboard layout.In my case:

       # loadkeys us



### Internet connection

To make sure you have an internet connection:

       # ping 8.8.8.8

Typical output:

> ```text
> PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
> 64 bytes from 8.8.8.8: icmp_seq=1 ttl=115 time=18.3 ms
> 64 bytes from 8.8.8.8: icmp_seq=2 ttl=115 time=17.9 ms
> 64 bytes from 8.8.8.8: icmp_seq=3 ttl=115 time=18.1 ms
> ```
###  Wi-Fi setup (iwctl)

Use this section to configure Wi-Fi when installing on a laptop or on systems without an Ethernet connection.

Start interactive shell

       # iwctl


List devices 

       # device list

Typical output:

> ```text
> Name          Address          Powered   Adapter   Mode
> wlan0         xx:xx:xx:xx:xx   on        phy0      station
> ```

Power on the device (if needed)

       # device "name" set-property Powered on


Power on the adapter (if needed)

       # adapter "adapter-name" set-property Powered on


To initiate a scan for networks (note that this command will not output anything): 

       # station "name" scan

You can then list all available networks: 

       # station "name" get-networks

Finally, to connect to a network: 

       # station "name" connect "SSID"

If a passphrase is required, you will be prompted to enter it. Alternatively, it can be provided directly as a command-line argument:

       # iwctl --passphrase "passphrase" station "name" connect "SSID"

To exit from iwctl:

       # exit

To make sure that internet is working:

       # ping 8.8.8.8

Typical output:

> ```text
> PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
> 64 bytes from 8.8.8.8: icmp_seq=1 ttl=115 time=18.3 ms
> 64 bytes from 8.8.8.8: icmp_seq=2 ttl=115 time=17.9 ms
> 64 bytes from 8.8.8.8: icmp_seq=3 ttl=115 time=18.1 ms
> ```


### Add best Arch mirrors


Refresh the pacman package database and install the reflector utility.
This step ensures access to the latest mirror information in the Arch Linux live installation environment.

       # pacman -Sy reflector


Generate a new pacman mirror list using the five most recently synchronized mirrors, sorted by download speed.
The generated list replaces the existing mirror configuration to improve download speed and reliability.
       
       # reflector --latest 5 --sort rate --save /etc/pacman.d/mirrorlist

### Partition disk

Your primary disk will be referred to as `sda` for the remainder of this installation guide.
Before proceeding, verify that `sda` corresponds to the correct physical disk to avoid accidental data loss.

       # lsblk

If you plan to install Arch Linux on a secondary drive, replace all references to `sda` in this guide with the appropriate device name (for example, `sdb`, `nvme0n1`, etc.).

This guide assumes a single-boot Arch Linux installation on a 250 GB disk.
The disk will be partitioned into five partitions. You may adapt the partition layout, sizes, or number of partitions according to your own requirements.

* `/dev/sda1` boot partition (1G).
* `/dev/sda2` swap partition (4G).
* `/dev/sda3` root partition (50G).
* `/dev/sda4` home partition (100G).
* `/dev/sda5` data partition (remaining disk space).

Before proceeding, remove all existing partitions on the target disk and create the new partition layout described in this guide.

       # gdisk /dev/sda

This interactive command-line utility allows you to manage disk partitions by entering specific commands.
Only the commands required for this installation will be shown in the following steps.

#### Clear partitions table

       Command: O
       Y

#### EFI partition (boot)

       Command: N
       ENTER
       ENTER
       +1G
       EF00

#### SWAP partition

       Command: N
       ENTER
       ENTER
       +4G
       8200

#### Root partition (/)

       Command: N
       ENTER
       ENTER
       +50G
       8304

#### Home partition

       Command: N
       ENTER
       ENTER
       +100G
       8302

#### Data partition

       Command: N
       ENTER
       ENTER
       ENTER
       ENTER

#### Save changes and exit

       Command: W
       Y

### Format partitions

       # mkfs.fat -F32 /dev/sda1
       # mkswap /dev/sda2
       # mkfs.ext4 /dev/sda3
       # mkfs.ext4 /dev/sda4
       # mkfs.ext4 /dev/sda5

### Mount partitions

       # swapon /dev/sda2
       # mount /dev/sda3 /mnt
       # mkdir /mnt/{boot,home}
       # mount /dev/sda1 /mnt/boot
       # mount /dev/sda4 /mnt/home

Running the *lsblk* command will display a list of detected block devices and their partitions.
The output should resemble the following example:

       NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
       sda      8:0    0 232.9G  0 disk
       ├─sda1   8:1    0     1G  0 part /mnt/boot
       ├─sda2   8:2    0     4G  0 part [SWAP]
       ├─sda3   8:3    0    50G  0 part /mnt
       ├─sda4   8:4    0   100G  0 part /mnt/home
       └─sda5   8:5    0  77.9G  0 part

## Installation

### Update the system clock

       # timedatectl set-ntp true

### Install Arch packages

       # pacstrap /mnt base base-devel openssh linux linux-firmware neovim

### Generate fstab file

       # genfstab -U /mnt >> /mnt/etc/fstab

## Add basic configuration

### Enter the new system

       # arch-chroot /mnt

### Language-related settings

       # nvim /etc/locale.gen

Uncomment the locale corresponding to your preferred language (for example, `en_US.UTF-8 UTF-8`).

       # locale-gen
       # nvim /etc/locale.conf

Add this content to the file:

       LANG=en_US.UTF-8
       LANGUAGE=en_US
       LC_ALL=C
Then:

       # nvim /etc/vconsole.conf

Add this content to the file:

       KEYMAP=us

### Configure timezone

This example uses the Europe/Rome time zone.
Adjust the value to match your local time zone as needed.

       # ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
       # hwclock —-systohc

### Enable SSH, NetworkManager and DHCP

These services will be enabled to start automatically at system startup.

       # pacman -S dhcpcd networkmanager network-manager-applet
       # systemctl enable sshd
       # systemctl enable dhcpcd
       # systemctl enable NetworkManager

### Install bootloader

       # pacman -S grub-efi-x86_64 efibootmgr
       # grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch
       # grub-mkconfig -o /boot/grub/grub.cfg

You may replace `arch` with any identifier you prefer, as long as it remains consistent throughout the installation.
### Choose a name for your computer

Assuming the system hostname is set to `arch-rabbit`:

       # echo arhc-rabbit > /etc/hostname

### Adding content to the hosts file

       # nvim /etc/hosts

Add the following content to the file:

       127.0.0.1    localhost.localdomain     localhost
       ::1          localhost.localdomain     localhost
       127.0.0.1    arch-rabbit.localdomain   arch-rabbit

Replace `arch-rabbit` with your system hostname.

### Install other useful packages

       # pacman -S iw wpa_supplicant dialog intel-ucode git reflector lshw unzip btop
       # pacman -S wget pulseaudio alsa-utils alsa-plugins pavucontrol xdg-user-dirs

### Update root password

       # passwd

### Final steps

       # exit
       # umount -R /mnt
       # swapoff /dev/sda2
       # reboot

## Post-install configuration

After the system restarts, the login prompt will appear on the `tty1` console.
Log in as the `root` user using the password set in the previous step:

>```text 
>login:root
>password: <the password you set earlier>
>```

### Add your user

Assuming your chosen user is `junior`:

       # useradd -m -g users -G wheel,storage,power,audio junior
       # passwd junior

### Grant root access to our user

       # EDITOR=nvim visudo

Configure `sudo` access for users in the `wheel` group by uncommenting one of the following lines in the sudoers file.

To allow passwordless execution of commands with `sudo`(not recomended):

       %wheel ALL=(ALL) NOPASSWD: ALL


To require password authentication for all commands executed with `sudo` (standard behavior on most Linux distributions):
     
       %wheel ALL=(ALL) ALL
    

Only one of these lines should be enabled/uncommented at a time.

### Login into newly created user

       # su - junior
       $ xdg-user-dirs-update

### Install AUR package manager

       $ mkdir Sources
       $ cd Sources
       $ git clone https://aur.archlinux.org/yay.git
       $ cd yay
       $ makepkg -si

### Manage Bluetooth

Install Bluetooth support and enable the service:

       $ sudo pacman -S --needed bluez bluez-utils blueman
       $ sudo systemctl enable bluetooth



### Improve laptop battery life

Install power management tools:

       $ sudo pacman -S --needed tlp tlp-rdw powertop acpi
       $ sudo systemctl enable tlp
       $ sudo systemctl enable tlp-sleep
       $ sudo systemctl mask systemd-rfkill.service
       $ sudo systemctl mask systemd-rfkill.socket



### Enable SSD TRIM

TRIM should be enabled on systems using solid-state drives (SSD) to maintain performance and extend the lifespan of the storage device.

       $ sudo systemctl enable fstrim.timer

---
## i3-gaps WM related steps

### Install graphical environment and i3

       $ sudo pacman -S xorg-server xorg-apps xorg-xinit
       $ sudo pacman -S i3-gaps i3blocks i3lock numlockx

### Install display manager

       $ sudo pacman -S lightdm lightdm-gtk-greeter --needed
       $ sudo systemctl enable lightdm

### Install some basic fonts

       $ sudo pacman -S noto-fonts ttf-ubuntu-font-family ttf-dejavu ttf-freefont
       $ sudo pacman -S ttf-liberation ttf-droid ttf-roboto terminus-font

### Install some useful utilities on i3

       $ sudo pacman -S rxvt-unicode yazi rofi alacritty dmenu neovim tmux firefox zathura zathura-pdf-mupdf pcmanfm 
       stow feh code --needed

#
### Apply all previous settings

       $ sudo reboot





