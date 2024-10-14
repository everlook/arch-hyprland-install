### Post Install Setup

* If on wifi and it does not start
```bash
nmcli device wifi list
nmcli device wifi connect SSID --ask
```

* Install Oh My Zsh
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

* Install yay package manager
```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

* Install plymouth boot splash screen for encrypet root
```bash
sudo pacman -S plymouth
```

* Add plymouth to the hooks for initcpio
```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block plymouth encrypt filesystems fsck)
```

* Install plymouth themes
```bash
yay -S plymouth-theme-archlinux plymouth-theme-arch-logo plymouth-theme-catppuccin-mocha-git
# show themes
plymouth-set-default-theme -l
```

* Create a plymouth conf
```bash
sudo vim /etc/plymouth/plymouthd.conf
```

```
[Daemon]
Theme=catppuccin-mocha
DeviceScale=an-integer-scaling-factor # for HiDPI
```

* After setting the theme rebuild the initramfs
```bash
sudo mkinitcpio -p linux
```

* Add **splash** to the grub bootloader
```bash
sudo vim /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet splash cryptdevice=UUID=<UUID>:cryptroot root=/dev/mapper/cryptroot"
sudo grub-mkconfig --output /boot/grub/grub.cfg
```
