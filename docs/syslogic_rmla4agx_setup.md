# TIER IV ISX021 GMSL2 Camera Driver Setup (Jetson AGX Orin / L4T r36)

This guides provides a step-by-step installation guide for the TIER IV ISX021
GMSL2 camera driver on Syslogic RML-A4AGX with L4T r36.x. (Jetpack 6.x) The
instructions include building the driver from source, fixing kernel header
issues, generating and applying a device tree overlay (DTBO), and verifying
camera functionality.

The original camera driver is hosted here:
<https://github.com/tier4/tier4_automotive_hdr_camera>

A driveblocks fork is hosted here:
<https://github.com/driveblocks/tier4_automotive_hdr_camera/tree/driveblocks_rml4agx-jp6>

## Overview
This guide covers:
- building and installing the TIER IV camera driver
- fixing kernel header issues
- generating and applying a DT overlay (DTBO)
- verifying camera functionality via V4L2

---

## 1. Clone Repository

```bash
git clone https://github.com/driveblocks/tier4_automotive_hdr_camera.git
cd tier4_automotive_hdr_camera
git checkout driveblocks_rml4agx-jp6
```

---

## 2. Install Dependencies

```bash
sudo apt update
sudo apt install -y \
    make build-essential debhelper debmake devscripts dkms \
    device-tree-compiler python3 v4l-utils
```

Optional (for testing video pipelines):
```bash
sudo apt install -y gstreamer1.0-tools gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good gstreamer1.0-plugins-bad
```

---

## 3. Build Debian Package

```bash
cd pkg
dpkg-buildpackage -b -rfakeroot -us -uc
ls ../*.deb
# Expected:
# tier4-camera-gmsl_2.1.0_arm64.deb
```

---

## 4. Install Driver

```bash
cd ..
sudo apt install ./tier4-camera-gmsl_2.1.0_arm64.deb
```

---

## 5. Fixing Kernel Headers (if needed)

If the installation of the debian package fails with errors related to kernel
headers, you may need to fix the symlinks for the kernel headers before
rebuilding the DKMS module.

### 5.1 Check kernel header symlinks

```bash
uname -r

ls -l /lib/modules/$(uname -r)/build
ls -l /lib/modules/$(uname -r)/source

readlink -f /lib/modules/$(uname -r)/build
test -f /lib/modules/$(uname -r)/build/Makefile && echo "OK" || echo "BROKEN"
```

### 5.2 Fix symlinks (Jetson JP6 / L4T r36.x)

```bash
KVER=$(uname -r)
KSRC=/usr/src/linux-headers-${KVER}-ubuntu22.04_aarch64/3rdparty/canonical/linux-jammy/kernel-source

sudo ln -sfn "$KSRC" /lib/modules/${KVER}/build
sudo ln -sfn "$KSRC" /lib/modules/${KVER}/source
```

### 5.3 Rebuild DKMS module (TIER IV camera)

```bash
sudo dkms remove tier4-camera-gmsl/2.1.0 --all
sudo dkms add -m tier4-camera-gmsl -v 2.1.0
sudo dkms build -m tier4-camera-gmsl -v 2.1.0
sudo dkms install -m tier4-camera-gmsl -v 2.1.0
```

### 5.4 Verify

```bash
dkms status
```

---

## 6. Generate Device Tree Overlay (DTS)

```bash
cd /usr/src/tier4-camera-gmsl-2.1.0
sudo python3 make_overlay_dts_rml-a4agx-r36.py R36 -8 C1
```

---

## 7. Compile DTBO

```bash
DTS=tier4-isx021-gmsl-device-tree-overlay-rml-a4agx-r36.dts
DTBO=${DTS%.dts}.dtbo

sudo dtc -O dtb -@ -o "$DTBO" "$DTS"
sudo cp "$DTBO" /boot/
```

---

## 8. Configure extlinux.conf

Edit:
```bash
sudo vim /boot/extlinux/extlinux.conf
```

### Ensure safe default
```conf
TIMEOUT 30
DEFAULT primary
```

### Add overlay entry
```conf
LABEL JetsonIO
      MENU LABEL Custom Header Config: <CSI TIERIV ISX021 GMSL2 Camera Device Tree Overlay>
      LINUX /boot/Image
      INITRD /boot/initrd
      FDT /boot/dtb/kernel_tegra234-brla4-agx-orin-64gb.dtb
      APPEND ${cbootargs} root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyTCU0,115200 console=ttyAMA0,115200
      OVERLAYS /boot/tier4-isx021-gmsl-device-tree-overlay-rml-a4agx-r36.dtbo
```

⚠️  Important:
- `FDT` must point to a `.dtb`
- `OVERLAYS` must point to the `.dtbo`
- keep `primary` as fallback

---

## 9. Boot with Overlay

```bash
sudo reboot
```

At boot:
- press any key
- select **JetsonIO**

⚠️ ⚠️ ⚠️  If you are certain that your overlay is working fine, you could also
add it to the `primary` entry, but it is safer to keep a fallback option in case
of issues.

---

## 10. Verify Camera Detection

### List devices
```bash
v4l2-ctl --list-devices
```

### Inspect device
```bash
v4l2-ctl -d /dev/video0 --all
```

---

## 11. Verify Overlay Applied

```bash
dmesg | grep -i -e isx021 -e gmsl -e camera
```

---


## 12. Test Camera using gst-launch-1.0 (Gstreamer)

Open a live preview window for each camera device:

```bash
#!/bin/bash

xset q

devices=$(ls /dev/gmsl)

for device in $devices; do
  echo "Launch gstreamer for camera '$device'"
  gst-launch-1.0 v4l2src io-mode=0 device=/dev/gmsl/${device} do-timestamp=true ! 'video/x-raw, width=1920, height=1280, framerate=30/1, format=UYVY' ! videoscale ! xvimagesink sync=false &
done

wait
```