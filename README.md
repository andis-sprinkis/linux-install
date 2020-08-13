## Installation steps (Arch Linux)

1. Disable "Secure Boot" in the BIOS. 
1. Boot into Arch Linux live installation media. Make sure that you know which disk is used for your installation. We'll assume it's `/dev/sda`. You can use `blkid` to list block devices.
1. Verify EFI boot mode by listing efivars directory:

   ```bash
   ls /sys/firmware/efi/efivars
   ```
1. *Optional* - You can launch an SSH server and continue your installation remotely from another
computer. In order to do that:

   ```bash
   passwd # set root password
   systemctl enable --now sshd
   ip a # determine host ip
   ```
   Then SSH to your installation disk from another computer and continue the installation as usual.
1. Run `cfdisk`, choose the `gpt` partitioning (if there are not options, run `cfdisk -z`).

   Create the partition table (make sure to `Write` the changes):
   | /dev/ mapping | Purpose        | Size              | Format       |
   |---------------|----------------|-------------------|--------------|
   | sda1          | Boot partition | `512M`            | `EFI System` |
   | sda2          | System root    | Rest of the drive |              |
1. Run installation scripts:
   ```bash
   reflector --country Latvia,Lithuania,Estonia,Finland,Sweden,Russia --protocol https --latest 5 --save /etc/pacman.d/mirrorlist
   rm -rf /var/lib/pacman/sync
   pacman --noconfirm -Sy git neovim
   git clone https://github.com/andis-sprinkis/linux-install /linux-install
   /linux-install/os-install/install
   systemctl-reboot
   ```
   ```bash
   /linux-install/post-os-install/install
   exit
   ```
1. Start GUI locally:
   ```bash
   startx
   ```
## Connecting to Wifi
- Installation media environent:

   ```
   iwctl station list
   iwctl station <station> scan
   iwctl station <station> get-networks
   iwctl station <station> connect <network_name>
   ```
- Finished installation environment:

  ```
  nmcli device wifi connect <SSID> password <password>
  ```
