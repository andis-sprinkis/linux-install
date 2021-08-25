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
1. Run the installation script:
   ```bash
      curl -LO https://raw.githubusercontent.com/andis-sprinkis/linux-install/master/install && sh ./install
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
