# Maginon WL-755 Arbitrary Code Execution

This Writeup covers a Arbitrary Code Execution exploit on a commonly sold Maginon WL-755. It will also go over restoring a bootloop caused by a malformed u-boot-env using a custom OpenWRT image.

## About

The Maginon WL-755 is a cheap Router/AP/WiFi-Extender sold a few years ago by Aldi, a german supermarket chain. It is based on the Wavlink WL-WN577A2 and uses a MediaTek MT7628AN MIPS processor with integrated 2,4GHz and LAN. It also comes with a MediaTek MT7610E for the 5GHz network. The Firmware underneath is a simple linux-based system with a BusyBox shell. It uses the "Das U-Boot" bootloader for bootup but unfortunately does not come with any exposed UART pins. 

## The Exploit

Logging in for the first time, the default user and password is admin/admin. After logging in it prompts you to change your password - but it can easily be dismissed by clicking "cancel". Navigating to the password settings, it allows you to set a new password and change the login username. 

<img width="928" height="379" alt="login_page" src="https://github.com/user-attachments/assets/89756c8c-c305-47cb-ad89-b3264ef377b0" />



Setting `$(reboot)` as the password will cause the connection to be dropped and the router will restart. When it boots up again, shortly after establishing a connection over LAN, it will reboot again. As the Reset button is monitored by a script that is run after the system properly booted up it has no effect and the router will bootloop forever. Instead of reboot other commands can be put in, for example, here I made the router establish a telnet connection on port 4444 using `$(telnetd${IFS}-p${IFS}4444)`:

<img width="565" height="189" alt="nmap_2" src="https://github.com/user-attachments/assets/0c148640-c4f5-40f3-9021-0c6d4c94cb99" />

It has to be noted that the password field does not allow whitespaces. As such, ${IFS} has to be used for whitespaces.
Using this an attacker could inject malicious code into the router. These products are often sold as "plug and play", meaning users are likely to not change the default password once it starts working. This poses a high risk to anyone using these devices. 

## Recovery

During the discovery of this expoit I had to recover the router from the bootloop. After trying to press the reset button for different lengths to no avail, I figured out that holding the WPS button while booting up the Router will cause it to stop bootlooping and request a file called "firmware.bin" over TFTP on the IP address 192.168.10.100. Flashing the original firmware on it has no effect however, as the nvram is not affected by flashing such an image. Surprisingly enough, OpenWRT offers a build for this model of router and flashing that onto the router exposes an SSH shell to the OpenWRT ASH. 

Using cat /proc/mtd I figured out that there are 4 MTD blocks on the SoC:

| dev  | size     | erasesize | name        |
|------|----------|-----------|-------------|
| mtd0 | 00030000 | 00010000  |  u-boot    |
| mtd1 | 00010000 | 00010000  | u-boot-env |
| mtd2 | 00010000 | 00010000  |  factory   |
| mtd3 | 007b0000 | 00010000  |  firmware  |

Transferring them onto my PC and inspecting them with hexedit it became clear that u-boot-env held all the configuration data for the firmware. Erasing this should clear the configuration and allow the system to boot. However, mtd erase /dev/mtd1 will throw the exception "Could not open device /dev/mtd1". OpenWRT is built with the MTD blocks being read-only and as such the solution is to compile OpenWRT without the read-only flag. Doing that allows /dev/mtd1 to be erased and after flashing the original firmware the router will boot up normally again. I included a precompiled OpenWRT image in the firmware folder of the repository, along with the original firmware and MTD dumps for preservation. 

## Miscellaneous

- The Router exposes a telnet shell on port 2323 by default. Changing the password in any way however completely destroys the login and while the telnet port is exposed, login is not possible. It is unknown if this is intentional or not, however, as the telnet shell stays open it is likely unintended and caused by the chpasswd script not working correctly - running it will manually will produce the same result.
- The Busybox version even on the latest version is outdated - it's v1.12.1 (2019-07-08 10:17:20 CST).
- The Linux Version is also outdated - `Linux version 2.6.36 (root@ubuntu) (gcc version 4.6.3 (Buildroot 2012.11.1) ) #1730 Mon Jul 8 10:23:14 CST 2019`
- The credentials are stored in plain text in u-boot-env:

<img width="441" height="194" alt="hexedit" src="https://github.com/user-attachments/assets/5293fb9c-a374-406a-9c57-a530ca7d6a52" />
