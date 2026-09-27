# Beitong BTP-KP40 – XInput (Xbox) mode fix for Linux

## Problem
When the dongle is plugged in, the controller first enumerates as `20bc:5127 BTP-KP40A XINPUT`,
then disconnects after ~1.5 s and comes back as `057e:2009 BTP-KP40 NS` (Switch mode).

**Cause:** If the controller does not see the two "Microsoft OS descriptor" requests that
Windows sends (string `0xEE` + XUSB10 compat ID) within ~1.9 s, it switches to Switch mode.
The Linux `xpad` driver does not send these requests (as of kernel 7.2.7).
The kernel patch that fixes this is not in mainline yet:
<https://lkml.iu.edu/hypermail/linux/kernel/2607.3/03645.html>

**Fix:** A udev rule + small script that sends those two requests as soon as the controller connects.

## Installation

This folder should contain two files: `beitong-xinput-lock` and `99-beitong-xinput.rules`.
If they are missing, recreate them from the "File contents" section below.

```sh
cd beitong-linux-fix
sudo install -m755 beitong-xinput-lock /usr/local/bin/
sudo install -m644 99-beitong-xinput.rules /etc/udev/rules.d/
sudo udevadm control --reload
```

Requirements: only `python3` (no extra packages). The `xpad` module ships with the kernel.

## Testing
Unplug the dongle and plug it back in:

```sh
journalctl -b | grep beitong-xinput-lock   # should show "ok, compat=b'XUSB10..'"
lsusb | grep -iE 'beitong|20bc|057e'       # should stay 20bc:5127, not switch to 057e:2009
```

If it still fails, check the output of `journalctl -k -b | tail -40`.

## Removal
Once the patch lands in the kernel (or if you no longer need it):

```sh
sudo rm /usr/local/bin/beitong-xinput-lock /etc/udev/rules.d/99-beitong-xinput.rules
sudo udevadm control --reload
```

To check whether your kernel already includes the patch:
```sh
zstdcat /lib/modules/$(uname -r)/kernel/drivers/input/joystick/xpad.ko.zst | strings | grep -i beitong
```
(Any output means the patch is included.)

## File contents (backup)

### `/usr/local/bin/beitong-xinput-lock`
```python
#!/usr/bin/env python3
# Locks Beitong KP20/KP40 controllers into XInput mode.
# If the controller does not receive the Microsoft OS descriptor requests from the
# host within ~1.9 s, it switches to Switch mode (057e:2009). Windows sends these
# automatically; Linux xpad does not (yet). This script sends the same two requests via usbfs.
# Usage: beitong-xinput-lock /dev/bus/usb/BBB/DDD
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
# Keep Beitong KP20A/KP40A (XInput mode) from falling back to Switch mode when plugged in
ACTION=="add", SUBSYSTEM=="usb", ENV{DEVTYPE}=="usb_device", ATTR{idVendor}=="20bc", ATTR{idProduct}=="5127", RUN+="/usr/local/bin/beitong-xinput-lock $env{DEVNAME}"
```

## Alternative
Patched xpad driver (manual build, not DKMS): <https://github.com/VegetablCat/Betop-driver-for-linux>
