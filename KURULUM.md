# Beitong BTP-KP40 – Linux'ta XInput (Xbox) modu düzeltmesi

## Sorun
Dongle takılınca kumanda önce `20bc:5127 BTP-KP40A XINPUT` olarak bağlanıyor,
~1,5 sn sonra kopup `057e:2009 BTP-KP40 NS` (Switch modu) olarak geri geliyor.

**Neden:** Kumanda, Windows'un gönderdiği iki "Microsoft OS descriptor" isteğini
(string `0xEE` + XUSB10 compat ID) ~1,9 sn içinde görmezse Switch moduna geçiyor.
Linux `xpad` sürücüsü bu istekleri göndermiyor (kernel 7.2.7 itibarıyla).
Düzelten kernel yaması henüz ana çekirdekte yok:
<https://lkml.iu.edu/hypermail/linux/kernel/2607.3/03645.html>

**Çözüm:** Kumanda bağlanır bağlanmaz bu iki isteği gönderen bir udev kuralı + küçük betik.

## Kurulum

Bu klasörde iki dosya olmalı: `beitong-xinput-lock` ve `99-beitong-xinput.rules`.
Klasör yoksa aşağıdaki "Dosyaların içeriği" bölümünden yeniden oluştur.

```sh
cd ~/beitong-fix
sudo install -m755 beitong-xinput-lock /usr/local/bin/
sudo install -m644 99-beitong-xinput.rules /etc/udev/rules.d/
sudo udevadm control --reload
```

Gereksinim: sadece `python3` (ek paket gerekmiyor). `xpad` modülü çekirdekte hazır geliyor.

## Test
Dongle'ı çıkar, tekrar tak:

```sh
journalctl -b | grep beitong-xinput-lock   # "ok, compat=b'XUSB10..'" görünmeli
lsusb | grep -iE 'beitong|20bc|057e'       # 20bc:5127 kalmalı, 057e:2009'a dönmemeli
```

Sorun sürerse: `journalctl -k -b | tail -40` çıktısına bak.

## Kaldırma
Yama çekirdeğe girdiğinde (ya da artık gerekmezse):

```sh
sudo rm /usr/local/bin/beitong-xinput-lock /etc/udev/rules.d/99-beitong-xinput.rules
sudo udevadm control --reload
```

Yamanın çekirdekte olup olmadığını kontrol etmek için:
```sh
zstdcat /lib/modules/$(uname -r)/kernel/drivers/input/joystick/xpad.ko.zst | strings | grep -i beitong
```
(Çıktı varsa yama girmiştir.)

## Dosyaların içeriği (yedek)

### `/usr/local/bin/beitong-xinput-lock`
```python
#!/usr/bin/env python3
# Beitong KP20/KP40 XInput modunu kilitler.
# Kumanda, host'tan Microsoft OS descriptor isteklerini ~1.9 sn içinde görmezse
# Switch (057e:2009) moduna geçiyor. Windows bunları otomatik gönderir, Linux xpad
# (henüz) göndermez. Bu betik aynı iki isteği usbfs üzerinden gönderir.
# Kullanım: beitong-xinput-lock /dev/bus/usb/BBB/DDD
import ctypes, fcntl, os, sys, syslog

class CtrlTransfer(ctypes.Structure):
    _fields_ = [("bRequestType", ctypes.c_uint8), ("bRequest", ctypes.c_uint8),
                ("wValue", ctypes.c_uint16), ("wIndex", ctypes.c_uint16),
                ("wLength", ctypes.c_uint16), ("timeout", ctypes.c_uint32),
                ("data", ctypes.c_void_p)]

USBDEVFS_CONTROL = (3 << 30) | (ctypes.sizeof(CtrlTransfer) << 16) | (ord('U') << 8) | 0

def ctrl(fd, rtype, req, value, index, length=255):
    buf = ctypes.create_string_buffer(length)
    ct = CtrlTransfer(rtype, req, value, index, length, 500, ctypes.cast(buf, ctypes.c_void_p))
    n = fcntl.ioctl(fd, USBDEVFS_CONTROL, ct)
    return buf.raw[:n]

def main():
    dev = sys.argv[1] if len(sys.argv) > 1 else os.environ.get("DEVNAME")
    fd = os.open(dev, os.O_RDWR)
    try:
        # 1) GET_DESCRIPTOR string 0xEE (MS OS 1.0 string descriptor)
        ctrl(fd, 0x80, 0x06, (0x03 << 8) | 0xEE, 0x0000)
        # 2) Vendor request 0xEE, wIndex=4 (Extended Compat ID -> "XUSB10")
        d = ctrl(fd, 0xC0, 0xEE, 0x0000, 0x0004)
        syslog.syslog(f"beitong-xinput-lock: {dev} ok, compat={d[18:26]!r}")
    except OSError as e:
        syslog.syslog(f"beitong-xinput-lock: {dev} failed: {e}")
    finally:
        os.close(fd)

main()
```

### `/etc/udev/rules.d/99-beitong-xinput.rules`
```
# Beitong KP20A/KP40A (XInput modu) takıldığında Switch moduna düşmesini engelle
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="20bc", ATTR{idProduct}=="5127", RUN+="/usr/local/bin/beitong-xinput-lock $env{DEVNAME}"
```

## Alternatif
Yamalı xpad sürücüsü (DKMS değil, elle derleme): <https://github.com/VegetablCat/Betop-driver-for-linux>
