### Printer install

```bash
$ sudo pacman -Sy hplip cups system-config-printer
$ sudo systemctl enable --now cups
```

* Use cups web interface to install printer
    * http://localhost:631

### Scanner install

```bash
$  sudo pacman -Sy sane-airscan simple-scan
```
