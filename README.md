# beitong-linux-fix

Beitong BTP-KP40 (ve KP20) kumandasının Linux'ta dongle ile bağlanınca XInput (Xbox)
modundan ~1,5 sn sonra Switch (NS) moduna düşmesini engelleyen udev kuralı + küçük betik.

Kurulum, test ve kaldırma adımları: [KURULUM.md](KURULUM.md)

Hızlı kurulum:

```sh
sudo install -m755 beitong-xinput-lock /usr/local/bin/
sudo install -m644 99-beitong-xinput.rules /etc/udev/rules.d/
sudo udevadm control --reload
```
