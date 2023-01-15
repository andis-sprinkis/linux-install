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
1. For each disk run `cfdisk /dev/sdX`, choose the `gpt` partitioning (if there are not options, run `cfdisk -z /dev/sdX`).

   Create the partition table (make sure to `Write` the changes):
   | /dev/ mapping | Directory        | Size              | Format             |
   |---------------|------------------|-------------------|--------------------|
   | sda1          | /boot            | `512M`            | `EFI System`       |
   | sda2          | /                | Rest of the drive | `Linux filesystem` |
   | sdb1          | /home            | Rest of the drive | `Linux filesystem` |
   | sdc1          | /var             | Rest of the drive | `Linux filesystem` |

1. Update pacman, clone this repository and start the installation:
   ```bash
   rm -rf /var/lib/pacman/sync && pacman --noconfirm -Sy git vi
   git clone https://github.com/andis-sprinkis/linux-install /linux-install --recurse-submodules

   vi /linux-install/config # optional double-checking of the installation config
   /linux-install/main
   ```
   **Or start the installation automatically (including the commands above):**
   ```
   curl -LO https://raw.githubusercontent.com/andis-sprinkis/linux-install/master/autostart && sh ./autostart
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
