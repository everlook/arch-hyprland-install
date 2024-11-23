### Arch Install with UEFI and Encrypted Root

* Set variables for install
```bash
export USER=username
export DRIVE=/dev/vda
export HOST_NAME=vm1
```

* Setup wifi for laptop install
```bash
iwctl
[iwd]# device list
[iwd]# station wlan0 scan # this will not output anything
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID
ping -c 5 archlinux.org
```
* Enable ssh and set root password so we can ssh in for install.
```bash
systemctl start sshd
passwd
```

* Update mirrors
```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
reflector --latest 20 --protocol https --sort rate --country US --save /etc/pacman.d/mirrorlist
```

* Update the system clock
```bash
timedatectl set-ntp true
```

* For EFI boot create first partion as an EFI System
  * 100M
  * For an EFI Partition use FAT32 (mkfs.vfat -F32 ${DRIVE}1)

* For an encrypted root create second partition for grub
  * 512MB
  * Format as linux(83)

* For the swap
  * For 64Gb of RAM use 72Gb if you need to hibernate

* Rest of disk ${DRIVE}3 linux(83)

* Partition the disk
```bash
lsblk
cfdisk ${DRIVE}
```

* For NVME drives add a **p** before the parition number

* Encrypt root parition
```bash
cryptsetup luksFormat --verify-passphrase --key-size 512 --hash sha512 ${DRIVE}4
```

* Open encrypted disk
```bash
cryptsetup open ${DRIVE}4 cryptroot
```

* Format partitions
```bash
mkfs.fat -F32 ${DRIVE}1
mkfs.ext4 ${DRIVE}2
mkfs.ext4 /dev/mapper/cryptroot
mkswap ${DRIVE}3
```

* Mount partitions
```bash
mount /dev/mapper/cryptroot /mnt
mkdir -p /mnt/boot
mount ${DRIVE}2 /mnt/boot
mkdir -p /mnt/efi
mount ${DRIVE}1 /mnt/efi
lsblk
swapon -a
```

```bash
NAME          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
vda           254:0    0    40G  0 disk
├─vda1        254:1    0   512M  0 part  /mnt/efi
├─vda2        254:2    0   512M  0 part  /mnt/boot
├─vda3        254:3    0     2G  0 part
└─vda4        254:4    0    37G  0 part
  └─cryptroot 253:0    0    37G  0 crypt /mnt
```

* Install base system
```bash
pacstrap -K /mnt base base-devel linux linux-firmware vim
```

* Generate fastab
```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

* Chroot
```bash
arch-chroot /mnt
```

* Set local time

```bash
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
hwclock --systohc
```

* Uncomment locale in /etc/locale.gen
```bash
vim /etc/locale.gen
```

* Generate locale
```bash
locale-gen
```

* Create /etc/locale.conf
```bash
echo LANG=en_US.UTF-8 > /etc/locale.conf
```

* Add hostname
```bash
echo ${HOST_NAME} > /etc/hostname
cat /etc/hostname
```

* Add to /etc/hosts
```bash
echo "127.0.0.1   localhost" >> /etc/hosts
echo "::1         localhost" >> /etc/hosts
echo "127.0.1.1   ${HOST_NAME}.localdomain  ${HOST_NAME}" >> /etc/hosts
cat /etc/hosts
```

* Change root password
```bash
passwd
```

* Add additional core packages
```bash
pacman -S grub efibootmgr man-db man-pages networkmanager openssh \
  network-manager-applet sudo git zsh unzip wget reflector 
```

* For Intel CPU install the ucode
```bash
pacman -S intel-ucode
```

* For AMD CPU install the ucode
```bash
pacman -S amd-ucode
```

* Add user
```bash
useradd -m -G wheel -s /usr/bin/zsh ${USER}
passwd ${USER}
```

* Run visudo, uncomment out the wheel group
```bash
visudo
%wheel ALL=(ALL) ALL
```

* Update /etc/default/grub
* Find UUID for root, this is the decrypted disk, not the Luks
  * becf3228-af91-47d0-8013-60c63cad9bcc

```bash
blkid
vim /etc/default/grub
```
* GRUB_CMDLINE_LINUX="cryptdevice=UUID=[use blkid]:cryptroot root=/dev/mapper/cryptroot"

* Edit /etc/mkinitcpio.conf. Make sure encrypt is before filesystems

```bash
vim /etc/mkinitcpio.conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt filesystems fsck)
```

* Buikd the initcpio
```bash
mkinitcpio -p linux
```

* For an Nvidia GPU, for drivers > 560 only the mkinitcpio.conf needs to be updated. DRM will be enabled by default
```bash
pacman -S nvidia
```

* Add Nvidia to mkinitcpio.conf modules. MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
  * Be sure to remove KMS from the hooks
```bash
vim /etc/mkinitcpio.conf
mkinitcpio -p linux
```

* Create the Nvidia module conf
```bash
vim /etc/modprobe.d/nvidia.conf

options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_TemporaryFilePath=/var/tmp
```

* Enable the Nvidia suspend services
```bash
sudo systemctl enable nvidia-suspend.service
sudo systemctl enable nvidia-hibernate.service
sudo systemctl enable nvidia-resume.service
```

* Install grub
```bash
grub-install --efi-directory=/efi --target=x86_64-efi --bootloader-id=ArchLinux --recheck
grub-mkconfig --output /boot/grub/grub.cfg
```

* On 4k displays we should use a bigger font with grub
```bash
# install a font
pacman -S ttf-fira-code
grub-mkfont --output=/boot/grub/fonts/FiraCode32.pf2 /usr/share/fonts/TTF/FiraCode-Regular.ttf
# update grub config to set the font 
# GRUB_FONT=/boot/grub/fonts/FiraCode32.pf2
vim /etc/default/grub
grub-mkconfig --output /boot/grub/grub.cfg
```

* Setup NTP
```bash
vim /etc/systemd/timesyncd.conf
```

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org 
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

* Enable the time sync service
```bash
systemctl enable systemd-timesyncd.service
```

* Enable networkmanager
```bash
systemctl enable NetworkManager
systemctl enable sshd
```

* Exit and reboot
```bash
exit
umount -R /mnt
cryptsetup close cryptroot
reboot
```
