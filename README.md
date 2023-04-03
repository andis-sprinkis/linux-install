# Arch Linux setup

With LVM on LUKS, systemd-boot bootloader, hibernation, applying user personal configuration.

## Setup process

1. [Download](https://archlinux.org/download/) the Arch Linux installer image.
1. Write the installation image to the installation media.
   ```bash
   cat path/to/archlinux-version-x86_64.iso > /dev/sdx
   ```
1. Disable "Secure Boot" in the BIOS of the installation target computer.
1. Boot installation target computer into Arch Linux installation media environment.
1. Verify EFI boot mode by listing efivars directory.
   ```bash
   ls /sys/firmware/efi/efivars
   ```
1. To continue the installation remotely from another computer:
   1. Set the installation media root user password.
      ```bash
      passwd
      ```
   1. Enable and start SSH server.
      ```bash
      systemctl enable --now sshd
      ```
   1. Determine installation target computer IP address.
      ```bash
      ip a
      ```
   1. From another device SSH into installation target computer to continue the setup.
      ```bash
      ssh root@192.168.1.99
      ```
1. Wipe the installation target disk. This document assumes installation target disk is `/dev/nvme0n1` (use `blkid` to list block devices).
   ```bash
   cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped
   dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress 2> /dev/null
   cryptsetup close to_be_wiped
   ```
1. Create the top level physical partitions. Choose the option `GPT partitioning`.
   ```
   cfdisk /dev/nvme0n1
   ```
   | /dev/ mapping  | Size              | Type             |
   | -------------- | ----------------- | ---------------- |
   | /dev/nvme0n1p1 | `512M`            | EFI System       |
   | /dev/nvme0n1p2 | rest of the drive | Linux filesystem |
1. Format the LUKS container partition. Must provide the password.
   ```bash
   cryptsetup luksFormat /dev/nvme0n1p2
   ```
1. Open the LUKS container.
   ```bash
   cryptsetup luksOpen /dev/nvme0n1p2 nvme0n1_luks0
   ```
1. Create physical volume in LUKS container.
   ```bash
   pvcreate /dev/mapper/nvme0n1_luks0
   ```
1. Create a logical volume group and add the physical volume of the LUKS container to it.
   ```bash
   vgcreate nvme0n1_luks0_volgrp0 /dev/mapper/nvme0n1_luks0
   ```
1. Create the logical partitions in the volume group.
   ```bash
   lvcreate -L 128G nvme0n1_luks0_volgrp0 -n root
   lvcreate -L 20G nvme0n1_luks0_volgrp0 -n swap
   lvcreate -l 100%FREE nvme0n1_luks0_volgrp0 -n home
   ```
   To determine the swap partition size:
   - RAM <=1 GB – at least the size of RAM, at most double the size of RAM.
   - RAM >1 GB – at least equal to the square root of the RAM size and at most double the size of RAM.
   - With hibernation – equal to size of RAM + the square root of the RAM size.
1. Reduce /home partition by 256MiB for e2scrub use.
   ```bash
   lvreduce -L -256M nvme0n1_luks0_volgrp0/home
   ```
1. Format the partitions of each logical volume.
   ```bash
   mkfs.ext4 /dev/nvme0n1_luks0_volgrp0/root
   mkfs.ext4 /dev/nvme0n1_luks0_volgrp0/home
   mkswap /dev/nvme0n1_luks0_volgrp0/swap
   ```
1. Format the /boot partition.
   ```bash
   mkfs.vfat -F32 /dev/nvme0n1p1
   ```
1. Create mount points and mount the system partitions.
   ```bash
   mkdir /mnt
   mount /dev/mapper/nvme0n1_luks0_volgrp0-root /mnt
   mkdir /mnt/{boot,home}
   mount /dev/mapper/nvme0n1_luks0_volgrp0-home /mnt/home
   mount /dev/nvme0n1p1 /mnt/boot
   ```
1. Initialize /swap partition.
   ```bash
   swapon /dev/mapper/nvme0n1_luks0_volgrp0-swap
   ```
1. Update Arch official package repository mirrors.
   ```bash
   reflector --country Latvia,Lithuania,Estonia,Finland,Sweden,Poland --protocol https --latest 10 --save /etc/pacman.d/mirrorlist
   ```
1. Install base packages.
   ```bash
   pacstrap /mnt base linux linux-firmware networkmanager openssh sudo neovim git lvm2
   ```
1. Generate fstab.
   ```bash
   genfstab -U /mnt >> /mnt/etc/fstab
   ```
1. Change root path of the system.
   ```bash
   arch-chroot /mnt
   ```
1. Install root user Neovim configuration.
   ```bash
   mkdir -p /root/.config && cd /root/.config
   git clone https://github.com/andis-sprinkis/nvim-user-config nvim
   cd nvim && git checkout minimal-config
   ```
1. Add boot-loader directories.
   ```bash
   mkdir -p /boot/loader/entries
   ```
1. Get the LUKS container partition UUID
   ```bash
   blkid --match-tag UUID -o value /dev/nvme0n1p2
   ```
1. Add boot-loader entry.
   Add file `/boot/loader/entries/arch.conf`:
   ```
   title Arch Linux
   linux /vmlinuz-linux
   initrd /initramfs-linux.img
   options cryptdevice=UUID=<LUKS container partition UUID>:nvme0n1_luks0 root=/dev/nvme0n1_luks0_volgrp0/root resume=/dev/nvme0n1_luks0_volgrp0/swap module_blacklist=pcspkr,snd_pcsp
   ```
1. Configure boot-loader.
   Add file `/boot/loader/loader.conf`:
   ```
   #timeout 3
   #console-mode keep
   ```
1. Update `/etc/mkinitcpio.conf` variable `HOOKS`, adding `encrypt lvm2 resume`
   ```bash
   HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems fsck encrypt lvm2 resume)
   ```
1. Regenerate initfram file.
   ```bash
   mkinitcpio -P
   ```
1. Install systemd-boot bootloader.
   ```bash
   bootctl --path=/boot install
   ```
1. Add pacman update hook for systemd-boot bootloader.
   Add file `/etc/pacman.d/hooks/100-systemd-boot.hook`:
   ```ini
   [Trigger]
   Type = Package
   Operation = Upgrade
   Target = systemd

   [Action]
   Description = Updating systemd-boot
   When = PostTransaction
   Exec = /usr/bin/bootctl update
   ```
1. Enable NetworkManager service.
   ```bash
   systemctl enable NetworkManager
   ```
1. Set hardware clock.
   ```bash
   hwclock --systohc
   ```
1. Set system locale.
   1. Add to file `/etc/locale.gen`:
      ```
      en_US.UTF-8 UTF-8
      lv_LV.UTF-8 UTF-8
      ```
   1. ```bash
      locale-gen
      ```
   1. Add to file `/etc/locale.conf`
      ```
      LANG=en_US.UTF-8
      LC_ADDRESS=lv_LV.UTF-8
      LC_COLLATE=lv_LV.UTF-8
      LC_CTYPE=lv_LV.UTF-8
      LC_MEASUREMENT=lv_LV.UTF-8
      LC_MONETARY=lv_LV.UTF-8
      LC_NUMERIC=lv_LV.UTF-8
      LC_PAPER=lv_LV.UTF-8
      LC_TELEPHONE=lv_LV.UTF-8
      LC_TIME=lv_LV.UTF-8
      ```
1. Set hostname.
   ```bash
   echo "arch-pc-00" > /etc/hostname
   ```
1. Set root user password.
   ```bash
   passwd root
   ```
1. Create a regular user.
   ```bash
   useradd -m user-00
   usermod -G wheel -a user-00
   passwd user-00
   ```
1. Set sudo-ers.
   1. ```bash
      visudo
      ```
   1. Add or uncomment
      ```
      %wheel ALL=(ALL) ALL
      ```
1. Create user mount directories.
   ```bash
   dirs=$(eval "echo /mnt/nvme{1..5} /mnt/sata{1..5} /mnt/usb{1..5} /mnt/pc{1..5} /mnt/nas{1..5} /mnt/vm{1..5} /mnt/mobile{1..5}")
   mkdir -p $dirs
   chown user-00:user-00 $dirs
   ```
1. Exit from /mnt root shell and reboot.
   ```bash
   exit
   reboot
   ```
1. Enable Network time protocol.
   ```bash
   sudo timedatectl set-ntp on
   ```
1. Clone the repository containing user package lists.
   ```bash
   cd $HOME
   git clone https://github.com/andis-sprinkis/linux-install
   cd linux-install
   ```
1. Install the Arch official package repository packages.
   ```bash
   sudo pacman -S --needed $(echo $(cat ./pkg_pacman))
   ```
1. If installation target computer is a VirtualBox guest, install and enable the VirtualBox guest utilities.
   ```bash
   sudo pacman -S virtualbox-guest-utils
   sudo systemctl enable --now vboxservice.service
   ```
1. Install AUR helper.
   ```bash
   temp_path=$(mktemp -d)
   git clone https://aur.archlinux.org/yay.git $temp_path
   cd $temp_path
   makepkg -si
   cd $HOME
   ```
1. Install AUR packages.
   ```bash
   yay -S --needed $(echo $(cat ./pkg_aur))
   ```
1. Install user general configuration.
   ```bash
   git_url_cfg=https://github.com/andis-sprinkis/nix-user-config
   dir_cfg_git=$HOME/.dotfiles_git
   temp_path=$(mktemp -d)
   git clone --separate-git-dir=$dir_cfg_git $git_url_cfg $temp_path
   rsync --recursive --verbose --exclude '.git' $temp_path/ $HOME
   git --git-dir=$dir_cfg_git --work-tree=$HOME config --local status.showUntrackedFiles no
   git --git-dir=$dir_cfg_git --work-tree=$HOME submodule update --init
   ```
1. Install user Neovim configuration.
   ```bash
   cd $HOME/.config
   git clone https://github.com/andis-sprinkis/nvim-user-config nvim
   ```
1. Switch shell to ZSH for both root and the regular user and execute ZSH.
   ```bash
   sudo chsh -s /usr/bin/zsh root
   sudo chsh -s /usr/bin/zsh user-00
   exec zsh
   ```
1. Install Node.js.
   ```bash
   volta install node
   ```
1. Install npm packages.
   ```bash
   volta install $(echo $(cat ./pkg_npm))
   ```
1. Install PyPi packages.
   ```bash
   pip3 install $(echo $(cat ./pkg_pypi))
   ```
1. Enable the audio system.
   ```bash
   systemctl --user enable --now pipewire.socket pipewire-pulse.socket wireplumber.service
   ```
1. Enable non-root users to be able to use `allow_other` mount option with FUSE.
   In file `/etc/fuse.conf` add or uncomment line 
   ```
   user_allow_other
   ```
1. To customize functions of the device power buttons:
   1. Update file `/etc/systemd/logind.conf`.
      ```bash
      sudo nvim /etc/systemd/logind.conf
      ```
   1. Restart the systemd-logind.service.
      ```bash
      sudo systemctl restart systemd-logind.service
      ```
1. Log out and log in again.
   ```bash
   exit
   ```

## Connecting to Wi-Fi

- Installation media environment:
  ```
  iwctl station list
  iwctl station <station> scan
  iwctl station <station> get-networks
  iwctl station <station> connect <network_name>
  ```
- Finished installation OS environment:
  - Interactively:
    ```bash
    nmtui
    ```
  - Non-interactively:
    ```bash
    nmcli device wifi connect <SSID> password <password>
    ```

## Hardware configuration and troubleshooting

- [AMDGPU - ArchWiki](https://wiki.archlinux.org/title/AMDGPU)
- [Backlight - ArchWiki](https://wiki.archlinux.org/title/Backlight)
- [Intel graphics - ArchWiki](https://wiki.archlinux.org/title/Intel_graphics)
- [Intel graphics - LinuxReviews](https://linuxreviews.org/Intel_graphics)
- [NVIDIA - ArchWiki](https://wiki.archlinux.org/title/NVIDIA)

## To do

- Describe encrypting and using crypttab for unlocking and mounting other drives.
