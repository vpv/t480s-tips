Installation notes for my "new" T480s
=====================================

# Background
I bought a used Thinkpad T480s. I found it for a reasonable price at [PCPiste](https://pcpiste.fi/kauppa/). I'm not affiliated with the store in any way, but things went so well that I might as well mention the store here.

I wanted some model of the T480 series, because it's the first series with a four-core CPU. The GPU in the 8th Gen Intel processors is also the first one, which is able to decode VP9 (used in Youtube) in hardware.

The actual T480 models are still quite expensive here in Finland. For some reason the T480s models cost a bit less, that's why I ended up getting one.

Here are my installation notes. I'm using Fedora and have been for more than 15 years.

# First tasks

- Install a stock Fedora desktop.
- Disable blinking cursor in the Gnome terminal. The profiles have the option to do it, in dconf it seems to be `cursor-blink-mode='off'`.
- Install apostrophe for editing md files (`dnf install apostrophe`).
- Install keepassxc for passwords (`dnf install keepassxc`).
- I also installed a Nextcloud client, but did not make good enough notes about it. Anyway, you could `dnf install nextcloud-client-nautilus` and configure it. It seems to work better than Gnome's own Nextcloud integration.
- Install vim, Fedora offers to do that for you if you try to start vim on the command line.
- Then do `dnf install vim-default-editor --allowerasing` to have vim as the default editor.
- Install the Gnome tweak tool, (`dnf install gnome-tweaks`), which allows you to set Caps Lock as Control.
- Install gnome-open because it's very useful: `dnf install libgnome`
	- Apparently it's deprecated, but I've not found anything else that works as well. Contact me, if you know of good alternatives.
- Install vinagre for remote support: `dnf install vinagre`
- Install wireguard-tools for obvious reasons (I'm running wireguard at home): `dnf install wireguard-tools`


# Extra packages and repositories
- Enable the `google-chrome` repository with Gnome's Software application and then install Chrome: `dnf install google-chrome-stable`
- `dnf install gnome-extensions-app`, might be useful at some point
- `dnf install htop powertop`
- Enable RPMFusion:
```
dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
dnf groupupdate core
dnf groupupdate multimedia --setop="install_weak_deps=False" --exclude=PackageKit-gstreamer-plugin
dnf install rpmfusion-free-release-tainted rpmfusion-nonfree-release-tainted
```

- `dnf install vlc`

## Packages specific to Fedora packaging
- `dnf install koji rpmdevtools fedora-packager`
	- A new find, there's a command called `rpmls` in the `rpmdevtools` package.


## Install yadm for dotfiles management
- [There's a Fedora repository in the OpenSuse Build Service.](https://software.opensuse.org//download.html?project=home%3ATheLocehiliosan%3Ayadm&package=yadm)
- [Instructions on using yadm with Gnome's dconf in this blog entry.](https://peterbabic.dev/blog/keep-gnome-shell-settings-dotfiles-yadm/)
- The file `~/.config/yadm/pre_commit` should look like this:
```
#!/bin/bash

dconfFile="$HOME/.config/dconf/settings.ini"

dconf dump / > "$dconfFile"
yadm add "$dconfFile"
```

- Make the script executable: `chmod +x config/yadm/pre_commit`
- Then handle the bootstrapping by adding the following to the file `~/.config/yadm/bootstrap`:
```
#!/bin/bash

dconf load / < "$HOME/.config/dconf/settings.ini"
```

- Again, make the script executable: `chmod +x ~/.config/yadm/bootstrap`

# Hardware tuning
## General
- `dnf install s-tui thermald-monitor acpitool nvme-cli gnome-power-statistics smartmontools`
- For Thermald do `usermod -aG power <your_username_here>`

## Intel GPU stuff

I decided to enable Framebuffer Compression, GuC and HuC although to be honest this is somewhat cargo cultish. Note that enable_fbc and enable_guc both will taint your kernel! The system has been stable in my use.

If someone can explain whether GuC and HuC are actually useful or not, please contact me!

- Add the following to the file `/etc/modprobe.d/i915.conf`:
```
options i915 enable_fbc=1 enable_guc=2
```

- `enable_guc=2` seems to also mean it enables HuC.
- Run `dracut --force` to regenerate the initrd, which actually enables these options at boot.

- These will be useful with the Intel GPU: `dnf install ffmpeg libva libva-utils libva-intel-driver intel-gpu-tools`

Enabling VP9 hardware decoding (in Youtube, for example), needs these settings in Firefox's `about:config`:

- `media.ffmpeg.vaapi.enabled` to `true`
- `media.ffvpx.enabled` to `false`

## SD card reader

This is a straight copy-paste from the [Arch wiki](https://wiki.archlinux.org/title/Lenovo_ThinkPad_T480s#SD_card_reader):

>According to various reports the SD card reader drains several
>watts of power. If you do not want to disable it in bios
>because you use it semi-regularly, you can turn it off by
>unbinding its driver using this command:
>
>`# tee /sys/bus/usb/drivers/usb/unbind <<< 2-3`
>
>You can then turn the reader back on by running:
>
>`# tee /sys/bus/usb/drivers/usb/bind <<< 2-3`

## TLP
I read somewhere that you don't actually need to limit the maximum charge of the battery any more, because the Thinkpad firmware will do it for you. I decided to set a limit just in case.

- `dnf install tlp`
- Add something like this to `/etc/tlp.d/01-battery.conf`:
```
# Battery charge thresholds (ThinkPad only).
# May require external kernel module(s), refer to the output of tlp-stat -b.
# Charging starts when the remaining capacity falls below the
# START_CHARGE_THRESH value and stops when exceeding the STOP_CHARGE_THRESH
# value.

# Main / Internal battery (values in %)
# Default: <none>

START_CHARGE_THRESH_BAT0=25
STOP_CHARGE_THRESH_BAT0=75

# Ultrabay / Slice / Replaceable battery (values in %)
# Default: <none>

#START_CHARGE_THRESH_BAT1=75
#STOP_CHARGE_THRESH_BAT1=80

# Restore charge thresholds when AC is unplugged: 0=disable, 1=enable.
# Default: 0

#RESTORE_THRESHOLDS_ON_BAT=1
```

- Enable tlp by running `systemctl enable tlp`
- When you need to charge the battery full, run `tlp fullcharge`

- Running `tlp-stat` as root with tlp 1.5 gives these warnings:
```
Error: conflicting power-profiles-daemon.service is enabled, power saving will not apply on boot.
>>> Invoke 'systemctl mask power-profiles-daemon.service' to correct this!
Warning: systemd-rfkill.service is not masked, radio device switching may not work as configured.
>>> Invoke 'systemctl mask systemd-rfkill.service' to correct this.
Warning: systemd-rfkill.socket is not masked, radio device switching may not work as configured.
>>> Invoke 'systemctl mask systemd-rfkill.socket' to correct this.
```

This is what I have done. If you ever wish to turn back to using power-profiles-daemon and systemd-rfkill, you will have to unmask these services. At that point it would probably be good to just remove the tlp package.

# Spotify installation
- `flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo`
- `flatpak install flathub com.spotify.Client`

[See the Fedora documentation.](https://docs.fedoraproject.org/en-US/quick-docs/installing-spotify/)

# Firmware updates
The T480s is supported by [LVFS](https://fwupd.org/) and [fwupd](https://github.com/fwupd/fwupd). In a default Fedora Gnome installation [Gnome Software](https://gitlab.gnome.org/GNOME/gnome-software) will notify you of new firmware updates. In September 2021 I got the first firmware update through fwupd and Gnome Software. After a reboot, the update never started. [This is a known bug.](https://github.com/fwupd/fwupd/wiki/LVFS-Triaged-Issue:-Failed-to-run-update-on-reboot). The bug has probably already been fixed in upstream [shim](https://github.com/rhboot/shim/), but for some reason the fixes are not in Fedora 34 yet.

The solution is to disable Secure Boot from the UEFI/BIOS configuration screen. With Secure Boot disabled, I was able to install the firmware update with Gnome Software. After installing the update, you should be able to re-enable Secure Boot.

# Personal reminder
- To ease up things when working in different networks and connecting back to home, you have disabled IPv6 in Gnome's Network settings. At some point it might be more harmful than useful.

# License for original text written by Ville-Pekka Vainio
Shield: [![CC BY 4.0][cc-by-shield]][cc-by]

This work is licensed under a
[Creative Commons Attribution 4.0 International License][cc-by].

[![CC BY 4.0][cc-by-image]][cc-by]

[cc-by]: http://creativecommons.org/licenses/by/4.0/
[cc-by-image]: https://i.creativecommons.org/l/by/4.0/88x31.png
[cc-by-shield]: https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg
