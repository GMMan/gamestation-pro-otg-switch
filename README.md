USB OTG Switch for Atari Gamestation Pro
========================================

This repo contains a RAMDISK for enabling USB peripheral mode on Atari
Gamestation Pro. Entering peripheral mode allows you to access the device
through ADB. Due to the SoC used, USB is switched between the on-board hub
(which connects the on-board wireless controller dongle and front USB ports
upstream) and the OTG port on the back; you cannot use both the hub and the
OTG port at the same time. This RAMDISK injects updated scripts so you can
select whether to operate in peripheral mode or host mode at boot time.

Installation
------------
1. Download the latest `boot.img` from GitHub's releases page.
2. Download the firmware update package linked on
   [MyArcade's product page](https://www.myarcadegaming.com/products/atari-gamestation-pro).
3. Follow the firmware update instructions up until the step where you plug in
   the Gamestation Pro.
4. If you have already updated to the latest firmware, uncheck the box beside
   the `firmware` partition. If you haven't updated, leave the box checked.
5. Add a new partition entry (right click the box and click Add Item), with
   address `0x00000800`, name `boot`, and browse for the `boot.img` that you
   downloaded (use the blank button at the end of the row).
6. Continue following the rest of the update instructions.

Usage
-----
To enter peripheral mode, connect the Gamestation Pro to your computer, and
hold the Home button while you turn the console on. Continue holding the button
until you see the boot animation. After the boot animation, you should see the
device connected on your computer. Use ADB to shell in and transfer files.
The screen may remain black for a while after the boot animation finishes.
This is normal because the menu is looking for input devices and not finding
any.

After you are done with ADB, shell in and run the `otg_switch` command. This
will switch USB back into host mode, and connect the controllers. If the screen
is black, it should change to the menu shortly.

To enter host mode at boot, don't hold the Home button when you turn the
console on.

UART port is also enabled with this mod.

If you want to access QA/ENG tests that were originally accessible by holding
Home while booting, you can put a text file at `game/GBX107_pcba.txt` on a
microSD card and insert it into the console before turning it on.

Building and technical details
------------------------------
See [tech.md](tech.md).
