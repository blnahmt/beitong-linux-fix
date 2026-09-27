# beitong-linux-fix

A udev rule + small script that stops the Beitong BTP-KP40 (and KP20) controller from
dropping out of XInput (Xbox) mode into Switch (NS) mode ~1.5 s after connecting via the
wireless dongle on Linux.

Installation, testing and removal: [INSTALL.md](INSTALL.md)

Quick install:

```sh
sudo install -m755 beitong-xinput-lock /usr/local/bin/
sudo install -m644 99-beitong-xinput.rules /etc/udev/rules.d/
sudo udevadm control --reload
```
