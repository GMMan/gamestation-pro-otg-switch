Technical Details
=================

The hardware
------------
The SoC used in Gamestation Pro is Rockchip RK3032, a feature-reduced version
of RK3036. There is only one USB OTG controller instead of separate USB OTG
and USB host controllers found on RK3036. Due to this, a USB switch is
connected between an on-board USB hub and the rear USB OTG port to enable
both the use of USB controllers and USB firmware downloading. The on-board
wireless controller dongle and the front USB ports are all connected to the
USB hub, so you can either have input or have peripheral connection to a
different host, e.g. a computer, but not both at the same time.

There are two GPIO pins for controlling the USB switching and the power to the
dongle. However, due to the fact the board uses a USB-C connector for the OTG
port so there's no connection to the OTG ID pin, and the driver configuration
of the kernel means it doesn't support overriding host/OTG mode through a
driver in userspace, switching the OTG controller between modes is tricky.
Fortunately, `/dev/mem` and the `devmem` BusyBox applet is available, and
an ID pin override is available on register `GRF_UOC_CON5[10:9]`, it is
actually possible to switch modes from userspace.

Device tree
-----------
There is a device called `usb_control` that is allegedly used for controlling
various USB-related things, however in reality the original control system
provided by the driver doesn't actually work because it is part of the `dwc310`
driver, whereas the `dwc2` driver is actually in use. Instead, the factory has
used it as an anchor point for a kitchen sink of miscellaneous GPIO and state
controls. There are also three pins named `host_drv`, `usb_en`, and `dongle_en`
that are set up as fixed regulators, despite the fact that the `usb_control`
driver actually supports toggling them as GPIOs. The USB OTG controller is also
set to `host` mode instead of `otg` mode, which means without device tree
modification, it is only possible to use the controller as host in Linux (this
kernel does not have device tree overlays enabled).

Theory of operation
-------------------
First, the device tree is modified so that the OTG controller is in `otg` mode
so that it can switch roles depending on the software ID pin value. The three
control pins are then removed as fixed regulators and attached to `usb_control`
so they may be toggled.

To swap between modes, we first toggle the USB switch `usb_en` to disconnect
the current device, then poke the `GRF_UOC_CON5` register to force the ID pin
value. This will trigger the USB controller driver to detect a role switch and
set the controller into the correct mode. `dongle_en` is set on afterwards
when switching to host mode to connect the wireless controllers after the hub
becomes connected. It is switched off in OTG mode to reduce power draw.

The `host_drv` pin toggles power to the front USB ports, and also the LEDs on
the top panel. This should typically be left untouched to keep the LEDs on as
originally designed.

File injection
--------------
To avoid redistributing the whole firmware image and as much of the copyrighted
contents as possible, an initramfs is used to perform the initial rootfs
overlay. Because the kernel does not have `overlay` filesystem built-in, it is
built as an module and `insmod`ed at init. A stripped down BusyBox with the
bare minimum applets built using `musl libc` for minimal size is used.

Build instructions
==================

Unpacking the firmware update
-----------------------------
The original firmware update can be obtained from a link on
[MyArcade's product page](https://www.myarcadegaming.com/products/atari-gamestation-pro).
Unzip the file, and locate `Firmware.img` inside the `3. Firmware` rolder. This
is an GPT-partitioned disk image. Extract the `boot` partition from it. You
could mount it as a loop and copy partition 3 to a file, or on Windows, use
7-Zip to extract the partition image.

Unpacking the boot image
------------------------
The boot image is in Android boot image format. To unpack it, you can use
`unpackbootimg` from [here](https://github.com/osm0sis/mkbootimg).

This extracts `boot.img` into a `bootimg` directory:
```sh
mkdir bootimg
mkbootimg/unpackbootimg -i boot.img -o bootimg
```

Unpacking the resource file
---------------------------
The resource file is a Rockchip custom format and contains the device tree
blob. It needs to be unpacked and repacked to update the device tree.

You will need `resource_tool`, which you can get from Rockchip's
[rkbin repository](https://github.com/rockchip-linux/rkbin/blob/master/tools/resource_tool)
or Rockchip's [U-Boot repository](https://github.com/rockchip-linux/u-boot) if
you can't use the prebuilt version.

Extract the resources to folder `rkres_out`:
```sh
resource_tool --unpack --image=bootimg/boot.img-second
mv out rkres_out
```

Compiling the device tree
-------------------------
A pre-modified device tree has been supplied at
[dts/rk-kernel_usbctrl.dts](dts/rk-kernel_usbctrl.dts). You will need to have
the `device-tree-compiler` package installed to compile device trees.

Compile the device tree:
```sh
dtc -I dts -O dtb dts/rk-kernel_usbctrl.dts -o rkres_out/rk-kernel.dtb
```

Note: you can also decompile device tree blobs like this:
```sh
dtc -I dtb -O dts rkres_out/rk-kernel.dtb -o dts/rk-kernel_decomp.dts
```

Repacking the resource file
---------------------------
To repack the resource file, make sure you are inside the resource directory
(file names will be incorrect otherwise), then use `resource_tool` to build:
```sh
cd rkres_out/
resource_tool --pack --image=../bootimg/boot.img-second rk-kernel.dtb logo.bmp
cd ..
```

Building the `overlay` kernel module
------------------------------------
A prebuilt copy of the module has been included, but if you want to build your
own, do the following:

1. Clone the kernel from
   https://github.com/RK3128-CFW/rockchip-linux/tree/powkiddy-a13.
   ```sh
   git clone https://github.com/RK3128-CFW/rockchip-linux.git -b powkiddy-a13 --single-branch --depth=1
   ```
2. Save https://github.com/RK3128-CFW/batocera.linux/blob/master/board/batocera/rockchip/rk3128/linux-defconfig.config
   as `.config` under your newly cloned kernel repo
3. Set your env vars. You will need to find your own copy of a suitable ARM GCC
   toolchain.
   ```sh
   export ARCH=arm
   export CROSS_COMPILE=arm-linux-gnueabihf-
   ```
4. Update the following config options through whatever method you prefer:
   ```
   # CONFIG_DEBUG_INFO is not set
   CONFIG_OVERLAY_FS=m
   # CONFIG_CGROUP_SCHED is not set
   ```
5. Make the kernel module
   ```sh
   make fs/overlayfs/overlay.ko
   ```
6. Copy `fs/overlayfs/overlay.ko` to
   `ramdisk/lib/modules/4.4.194/kernel/fs/overlayfs/overlay.ko` in the repo.

Building BusyBox
----------------
A prebuilt copy of BusyBox has been included, but if you want to build your
own, do the following:

1. Download [BusyBox](https://busybox.net/) source archive and extract it.
2. Copy [busybox.config](busybox.config) as `.config` under BusyBox's source
   tree.
3. Obtain `musl` cross-compiler from [musl.cc](https://musl.cc/). You want
   `armv7l-linux-musleabihf-cross.tgz`. Unpack it somewhere.
4. Set your env vars. Adjust toolchain path as necessary.
   ```sh
   export CROSS_COMPILE=armv7l-linux-musleabihf-
   ```
5. Build BusyBox
   ```sh
   make
   ```
6. Copy `busybox` to `ramdisk/bin/busybox` in the repo.

Building the initramfs
----------------------
The init script was loosely based off of
https://gist.github.com/m13253/e4c3e3a56a23623d2e7e6796678b9e58.

To build the initramfs:
```sh
cd ramdisk
chmod 4755 bin/busybox
cd overlay/upperdir
chmod 755 usr usr/bin usr/bin/otg_switch etc etc/init.d etc/init.d/S49launcher etc/init.d/S21mountall.sh
cd ../..
find . | sort | cpio -o -H newc -R 0:0 | xz -9 --check=crc32 > ../initramfs.img
cd ..
```

Rebuilding the boot image
-------------------------
Use `mkbootimg`:
```sh
mkbootimg/mkbootimg --kernel bootimg/boot.img-kernel \
                    --second bootimg/boot.img-second \
                    --ramdisk initramfs.img \
                    --header_version 0 --hashtype sha1 --pagesize 2048 \
                    --base 0x10000000 \
                    --tags_offset 0x00000100 --kernel_offset 0x00008000 \
                    --second_offset 0x00f00000 --ramdisk_offset 0xf0000000 \
                    -o boot_new.img
```

`boot_new.img` is ready to be flashed to your Gamestation Pro.
