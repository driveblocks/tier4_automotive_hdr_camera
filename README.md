# TIERIV Automotive HDR Camera C1/C2 Device Driver
This repository contains the device driver for the [TIERIV Automotive HDR Camera C1/C2](https://sensor.tier4.jp/automotive-hdr-camera). 

If you want to use the official driver, please refer to the [Installation and Usage section](#installation-and-usage) and proceed with the installation.

If you want to modify the driver, please refer to the [How to create a package from source section](#create-debian-dkms-package).


## Installation and Usage

1. Download the latest release of the driver from the [releases page](https://github.com/tier4/tier4_automotive_hdr_camera/releases).
2. Follow the installation instructions provided in the quickstart guide.

### Note for ADLINK ROSCube-X RQX-58G users:
The device type (i.e., C1 or C2) assignment is fixed at every GMSL2 port.
The following table shows the default assignment:

| GMSL2 port No. | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  |
|----------------|----|----|----|----|----|----|----|----|
| Camera type    | C1 | C1 | C2 | C2 | C1 | C1 | C2 | C2 |

This assignment can be changed by the following steps:

- Generate a `dts` file to describe user desired assignment
    ```bash
    $ cd drivers/tools/
    # Specify device type assignment from port #1 to #8 one by one
    $ python3 make_overlay_dbt.py c1 c1 c1 c1 c2 c2 c2 c2
    ```
    - Port #(2n-1) and #(2n), where n = [1, 2, 3, 4], must be the same assignment. If invalid assignments are specified, the script returns errors, otherwise a dts file will be created.
    - By the above example, a file named `tier4-isx021-imx490-c1-c1-c2-c2-device-tree-overlay.dts` will be created.

- Generate `dtbo` from `dts`
    ```bash
    $ dtc -O dtb -o ${OVERLAY_DTBO_FILE} -@ ${OVERLAY_DTS_FILE}
    ```
    - `${OVERLAY_DTBO_FILE}` indicates the string that file name extension is replaced from `dts` to `dtbo`.
       - e.g., `tier4-isx021-imx490-c1-c1-c2-c2-device-tree-overlay.dtbo`
    - `${OVERLAY_DTS_FILE}` is the dts file generated at the above step.

- Copy the result to appropriate directory and run configure
    ```bash
    $ sudo cp ${OVERLAY_DTBO_FILE} /boot/
    $ sudo fdtoverlay \
        -o /boot/kernel_tegra194-rqx-58g-user-custom.dtb \
        -i /boot/dtb/kernel_tegra194-rqx-58g.dtb \
        /boot/${OVERLAY_DTBO_FILE}
    ```
- Reboot the system

## How to create a package from source 

### Preparation

```
$> sudo apt update
$> sudo apt install make build-essential debhelper debmake devscripts dkms 
```

### Create debian-dkms package

```
$> dpkg-buildpackage -b -rfakeroot -us -uc
```

You can confirm the package file with the following command.
```
$> ls ../*deb
```

## Contributing

We welcome contributions to the Automotive HDR Camera Device Driver. 
If you have a bug fix or new feature that you would like to contribute, please submit a issue or pull request.
