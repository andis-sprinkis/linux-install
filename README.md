# Arch Linux setup

With LVM on LUKS, systemd-boot bootloader, hibernation, applying user personal configuration files and preferences.

## Setup process

1. [Download](https://archlinux.org/download/) the Arch Linux installer image.
1. Write the installation image to the installation media.

    - To write the image from a \*nix system:
        ```sh
        < path/to/archlinux-version-x86_64.iso > /dev/sdx
        ```
    - To write the image from Microsoft Windows, use [Rufus](https://rufus.ie/en/).
    - Alternatively, copy the downloaded image to [a Ventoy prepared](https://www.ventoy.net/en/doc_start.html) device.

1. Disable "Secure Boot" in the BIOS of the installation target computer.
1. Boot installation target computer into Arch Linux installation media environment.
1. Verify EFI boot mode by listing efivars directory.
    ```sh
    ls /sys/firmware/efi/efivars
    ```
1. To continue the installation remotely from another computer:
    1. Set the installation media root user password.
        ```sh
        passwd
        ```
    1. Enable and start SSH server.
        ```sh
        systemctl enable --now sshd
        ```
    1. Determine installation target computer IP address.
        ```sh
        ip a
        ```
    1. From another device SSH into installation target computer to continue the setup.
        ```sh
        ssh root@192.168.1.99
        ```
1. Wipe the installation target disk. This document assumes installation target disk is `/dev/nvme0n1` (use `lsblk` to list block devices).
    ```sh
    cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped
    dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress 2> /dev/null
    cryptsetup close to_be_wiped
    ```
1. Create the top level physical partitions. Choose the option `GPT partitioning`.
    ```sh
    cfdisk /dev/nvme0n1
    ```
    | /dev/ mapping  | Size              | Type             |
    | -------------- | ----------------- | ---------------- |
    | /dev/nvme0n1p1 | `512M`            | EFI System       |
    | /dev/nvme0n1p2 | rest of the drive | Linux filesystem |
1. Format the LUKS container partition. Must provide the password.
    ```sh
    cryptsetup luksFormat /dev/nvme0n1p2
    ```
1. Open the LUKS container.
    ```sh
    cryptsetup luksOpen /dev/nvme0n1p2 nvme0n1_luks0
    ```
1. Create physical volume in LUKS container.
    ```sh
    pvcreate /dev/mapper/nvme0n1_luks0
    ```
1. Create a logical volume group and add the physical volume of the LUKS container to it.
    ```sh
    vgcreate nvme0n1_luks0_volgrp0 /dev/mapper/nvme0n1_luks0
    ```
1. Create the logical partitions in the volume group.
    ```sh
    lvcreate -L 128G nvme0n1_luks0_volgrp0 -n root
    lvcreate -L 20G nvme0n1_luks0_volgrp0 -n swap
    lvcreate -l 100%FREE nvme0n1_luks0_volgrp0 -n home
    ```
    To determine the swap partition size:
    - RAM <=1 GB – at least the size of RAM, at most double the size of RAM.
    - RAM >1 GB – at least equal to the square root of the RAM size and at most double the size of RAM.
    - With hibernation – equal to size of RAM + the square root of the RAM size.
1. Reduce /home partition by 256MiB for e2scrub use.
    ```sh
    lvreduce -L -256M nvme0n1_luks0_volgrp0/home
    ```
1. Format the partitions of each logical volume.
    ```sh
    mkfs.ext4 /dev/nvme0n1_luks0_volgrp0/root
    mkfs.ext4 /dev/nvme0n1_luks0_volgrp0/home
    mkswap /dev/nvme0n1_luks0_volgrp0/swap
    ```
1. Format the /boot partition.
    ```sh
    mkfs.vfat -F32 /dev/nvme0n1p1
    ```
1. Create mount points and mount the system partitions.
    ```sh
    mkdir -p /mnt
    mount /dev/mapper/nvme0n1_luks0_volgrp0-root /mnt
    mkdir /mnt/{boot,home}
    mount /dev/mapper/nvme0n1_luks0_volgrp0-home /mnt/home
    mount /dev/nvme0n1p1 /mnt/boot -o fmask=0137,dmask=0027
    ```
1. Initialize /swap partition.
    ```sh
    swapon /dev/mapper/nvme0n1_luks0_volgrp0-swap
    ```
1. Update Arch official package repository mirrors.
    ```sh
    reflector --country Latvia,Lithuania,Estonia,Finland,Sweden,Poland --protocol https --latest 10 --save /etc/pacman.d/mirrorlist
    ```
1. Install base packages.
    ```sh
    pacstrap /mnt base linux linux-firmware networkmanager openssh sudo neovim git terminus-font lvm2
    ```
1. Generate fstab.
    ```sh
    genfstab -U /mnt >> /mnt/etc/fstab
    ```
1. Change root path of the system.
    ```sh
    arch-chroot /mnt
    ```
1. Install root user Neovim configuration.
    ```sh
    mkdir -p /root/.config && cd /root/.config
    git clone https://github.com/andis-sprinkis/nvim-user-config nvim
    cd nvim && git checkout minimal-config
    ```
1. Add boot-loader directories.
    ```sh
    mkdir -p /boot/loader/entries
    ```
1. Get the LUKS container partition UUID.
    ```sh
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

    ***

    Set TTY default screen rotation by specifying the `fbcon=rotate:X` boot options value.

    For a counter-clockwise rotation set:

    ```
    options ... fbcon=rotate:1
    ```

1. Configure boot-loader.
   Add file `/boot/loader/loader.conf`:
    ```
    #timeout 0
    #console-mode keep
    ```
1. Update `/etc/mkinitcpio.conf` variable `HOOKS`, adding `encrypt lvm2 resume`:
    ```sh
    HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems fsck encrypt lvm2 resume)
    ```
1. Regenerate initfram file.
    ```sh
    mkinitcpio -P
    ```
1. Install systemd-boot bootloader.
    ```sh
    bootctl --path=/boot install
    ```
1. Add pacman update hook for systemd-boot bootloader:

    1. ```sh
       mkdir /etc/pacman.d/hooks
       ```
    1. Add file `/etc/pacman.d/hooks/100-systemd-boot.hook`:

        ```systemd
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
    ```sh
    systemctl enable NetworkManager
    ```
1. Set hardware clock.
    ```sh
    hwclock --systohc
    ```
1. Set system locale.
    1. Add to file `/etc/locale.gen`:
        ```
        en_US.UTF-8 UTF-8
        lv_LV.UTF-8 UTF-8
        ```
    1. ```sh
       locale-gen
       ```
    1. Add file `/etc/locale.conf`
        ```sh
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
1. Set the console font and keymap.
   Add to file `/etc/vconsole.conf`:
    ```sh
    FONT=ter-v24b
    KEYMAP=lv
    ```
1. Set console typematic delay and rate (keyboard input speed).

    1. Add file `/etc/systemd/system/console-kbdrate.service`:

        ```systemd
        [Unit]
        Description=Console typematic delay and rate (kbdrate).

        [Service]
        Type=oneshot
        RemainAfterExit=yes
        StandardInput=tty
        StandardOutput=tty
        ExecStart=/usr/bin/kbdrate --silent --delay 165 --rate 55

        [Install]
        WantedBy=multi-user.target
        ```

    1. ```sh
       systemctl enable console-kbdrate.service
       ```

1. Set hostname.
    ```sh
    echo "archpc0" > /etc/hostname
    ```
1. Set root user password.
    ```sh
    passwd root
    ```
1. Create a regular user.
    ```sh
    useradd -m usr0
    usermod -aG wheel usr0
    passwd usr0
    ```
1. Set sudo-ers.
    1. ```sh
       EDITOR=nvim
       visudo
       ```
    1. Add or uncomment:
        ```
        %wheel ALL=(ALL:ALL) ALL
        ```
1. Create user mount directories.
    ```sh
    dirs=$(eval "echo /mnt/nvme{1..5} /mnt/sata{1..5} /mnt/usb{1..5} /mnt/pc{1..5} /mnt/nas{1..5} /mnt/vm{1..5} /mnt/mobile{1..5}")
    mkdir -p $dirs
    chown usr0:usr0 $dirs
    ```
1. Exit from /mnt root shell and reboot, then log in as the regular user.
    ```sh
    exit
    reboot
    ```
1. Enable Network Time Protocol.
    ```sh
    sudo timedatectl set-ntp on
    ```
1. Set the time zone.
    ```sh
    sudo timedatectl set-timezone Europe/Riga
    ```
1. Clone the repository containing the user package lists.
    ```sh
    cd $HOME
    git clone https://github.com/andis-sprinkis/linux-install
    cd linux-install
    ```
1. Install the Arch official package repository packages.
    ```sh
    sudo pacman -S --needed $(echo $(cat ./pkg_pacman))
    ```
1. If installation target computer is a VirtualBox guest, install and enable the VirtualBox guest utilities.
    ```sh
    sudo pacman -S virtualbox-guest-utils
    sudo systemctl enable --now vboxservice.service
    ```
1. Add the regular user to `vboxusers` group for the guest OS USB port access in VirtualBox VMs running on this host.
    ```sh
    sudo usermod -aG vboxusers usr0
    ```
1. Install AUR helper.
    ```sh
    temp_path=$(mktemp -d)
    git clone https://aur.archlinux.org/yay.git $temp_path
    cd $temp_path
    makepkg -si
    cd $HOME/linux-install
    ```
1. Install AUR packages.
    ```sh
    yay -S --needed $(echo $(cat ./pkg_aur))
    ```
1. Install user general configuration.
    ```sh
    git_url_cfg=https://github.com/andis-sprinkis/nix-user-config
    dir_cfg_git=$HOME/.local/state/dotfiles_git
    temp_path=$(mktemp -d)
    git clone --separate-git-dir=$dir_cfg_git $git_url_cfg $temp_path
    rsync --recursive --verbose --exclude '.git' $temp_path/ $HOME
    git --git-dir=$dir_cfg_git --work-tree=$HOME config --local status.showUntrackedFiles no
    git --git-dir=$dir_cfg_git --work-tree=$HOME submodule update --init
    ```
1. Install user Neovim configuration.
    ```sh
    cd $HOME/.config
    git clone https://github.com/andis-sprinkis/nvim-user-config nvim
    ```
1. Create user download directries.
    ```sh
    mkdir -p $HOME/dl/{_chrm,_eph,_ff,_jd2,_mnt,_qbt,_scdl,_ytdlp}
    ```
1. Switch shell to ZSH for both root and the regular user and execute ZSH.
    ```sh
    sudo chsh -s /usr/bin/zsh root
    sudo chsh -s /usr/bin/zsh usr0
    exec zsh
    ```
1. Install AppImage packages.
    ```sh
    for p in $(echo $(cat ./pkg_appimage)); do curl --location --output-dir "$HOME/.local/opt/appimage" --remote-name "$p"; done
    chmod +x $HOME/.local/opt/appimage/*
    ```
1. Install npm packages.
    ```sh
    volta install $(echo $(cat ./pkg_npm))
    ```
1. Install PyPi packages.
    ```sh
    for p in $(echo $(cat ./pkg_pypi)); do pipx install $p; done
    ```
1. Enable the audio system.
    ```sh
    systemctl --user enable --now pipewire.socket pipewire-pulse.socket wireplumber.service
    ```
1. Enable the PC/SD Smart Card Daemon service.
    ```sh
    sudo systemctl enable pcscd.service
    ```
1. Enable non-root users to be able to use `allow_other` mount option with FUSE.
   In file `/etc/fuse.conf` add or uncomment line
    ```
    user_allow_other
    ```
1. Detect the hardware sensors.
    ```sh
    sudo sensors-detect
    ```
1. To customize functions of the device power buttons:
    1. Update file `/etc/systemd/logind.conf`.
        ```sh
        sudo nvim /etc/systemd/logind.conf
        ```
    1. Restart the systemd-logind.service.
        ```sh
        sudo systemctl restart systemd-logind.service
        ```
1. Log out and log in again.
    ```sh
    exit
    ```

## Encryption, automatic unlocking and mounting of an another drive on the system

LVM on LUKS.

1. Wipe the target disk. This document assumes the target disk is `/dev/nvme1n1` (use `lsblk` to list block devices).
    ```sh
    sudo su
    cryptsetup open --type plain -d /dev/urandom /dev/nvme1n1 to_be_wiped
    dd if=/dev/zero of=/dev/mapper/to_be_wiped bs=1M status=progress 2> /dev/null
    cryptsetup close to_be_wiped
    ```
1. Create the top level physical partition. Choose the option `GPT partitioning` and set the entire drive as `Linux filesystem`.
    ```sh
    cfdisk /dev/nvme1n1
    ```
1. Generate the keyfile.
    ```sh
    dd bs=512 count=4 if=/dev/random of=/nvme1.key iflag=fullblock
    ```
1. Set keyfile access permissions.
    ```sh
    chmod a=,u=rw /nvme1.key
    ```
1. Format the LUKS container partition. Must provide the password.
    ```sh
    cryptsetup luksFormat /dev/nvme1n1p1
    ```
1. Associate the keyfile with the LUKS container partition.
    ```sh
    cryptsetup luksAddKey /dev/nvme1n1p1 /nvme1.key
    ```
1. Open the LUKS container.
    ```sh
    cryptsetup luksOpen /dev/nvme1n1p1 nvme1n1_luks0 --key-file /nvme1.key
    ```
1. Create the physical volume in LUKS container.
    ```sh
    pvcreate /dev/mapper/nvme1n1_luks0
    ```
1. Create a logical volume group and add the physical volume of the LUKS container to it.
    ```sh
    vgcreate nvme1n1_luks0_volgrp0 /dev/mapper/nvme1n1_luks0
    ```
1. Create the logical partition in the volume group.
    ```sh
    lvcreate -l 100%FREE nvme1n1_luks0_volgrp0 -n data
    ```
1. Reduce the /data logical partition by 256MiB for e2scrub use.
    ```sh
    lvreduce -L -256M nvme1n1_luks0_volgrp0/data
    ```
1. Format the partition of the logical volume.
    ```sh
    mkfs.ext4 /dev/nvme1n1_luks0_volgrp0/data
    ```
1. Get the LUKS container partition UUID.
    ```sh
    blkid --match-tag UUID -o value /dev/nvme1n1p1
    ```
1. Add the LUKS container partition entry to file `/etc/crypttab`:
    ```
    nvme1          UUID=<LUKS container partition UUID>    /nvme1.key
    ```
1. Get the logical volume partition UUID.
    ```sh
    blkid --match-tag UUID -o value /dev/mapper/nvme1n1_luks0_volgrp0-data
    ```
1. Add the logical volume partition entry to file `/etc/fstab`:
    ```
    # /dev/mapper/nvme1n1_luks0_volgrp0-data
    UUID=<Logical volume partition UUID>  /mnt/nvme1 ext4 rw,relatime 0 0
    ```
1. Re-mount `/etc/fstab` file specified devices.
    ```sh
    mount -a
    systemctl daemon-reload
    ```
1. Change the mounted file system ownership to the regular user.
    ```sh
    chown -R usr0:usr0 /mnt/nvme1
    ```
1. Reboot.
    ```sh
    systemctl reboot
    ```

## Connecting to Wi-Fi

-   Installation media environment:
    ```sh
    iwctl station list
    iwctl station $station scan
    iwctl station $station get-networks
    iwctl station $station connect $network_name
    ```
-   The installed OS environment:
    -   Interactively:
        ```sh
        nmtui
        ```
    -   Non-interactively:
        ```sh
        nmcli device wifi connect $ssid password $password
        ```

## Related resources

-   `xorg.conf`, `xrandr` manpages
-   [AMDGPU - ArchWiki](https://wiki.archlinux.org/title/AMDGPU)
-   [Backlight - ArchWiki](https://wiki.archlinux.org/title/Backlight)
-   [Intel graphics - ArchWiki](https://wiki.archlinux.org/title/Intel_graphics)
-   [Intel graphics - LinuxReviews](https://linuxreviews.org/Intel_graphics)
-   [NVIDIA - ArchWiki](https://wiki.archlinux.org/title/NVIDIA)
-   [Network UPS Tools - ArchWiki](https://wiki.archlinux.org/title/Network_UPS_Tools)
-   [Network UPS Tools - Hardware compatibility list](https://networkupstools.org/stable-hcl.html)
-   [Persistent block device naming - ArchWiki](https://wiki.archlinux.org/title/Persistent_block_device_naming)
-   [Smartcards - ArchWiki](https://wiki.archlinux.org/title/Smartcards)
-   [The Framebuffer Console — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/fb/fbcon.html)
-   [User/Group Name Syntax](https://systemd.io/USER_NAMES/)
-   [Xorg - ArchWiki](https://wiki.archlinux.org/title/Xorg)
