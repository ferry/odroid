# Odroid C1 met wifi TP-Link TL-WN725N V2 dongle (RTL8188EU)

## Otional: driver from source

clone https://github.com/lwfinger/rtl8188eu to a usb memory dongle

place dongle in droid, mount and compile driver:

```
mount /dev/sda1 /mnt
cd /mnt/rtl8188eu
make all
sudo make install
```

## Disable NetworkManager and use systemd instead

source: https://forum.manjaro.org/t/how-to-use-systemd-networkd-to-manage-your-wifi/1557

In most manjaro editions you connect to your wifi with networkmanager. This tutorial shows you the basics of how to use systemd-networkd and wpa_supplicant instead.

Why would you want to do this?
Networkmanager usually does great job in managing your network. However, it uses a lot of resources to do this (albeit this is neglible on most modern hardware). Systemd-networkd can do the same job for a lot less - On my system networkmanager consumes about 20mb ram, plus about 15mb if I use a tray icon for it. Networkd and resolvd use together less than 2mb ram. And it is already installed on your system if you use systemd, so there is nothing extra to install.

Why not use netctl instead?
Because you don’t have to. For me, netctl has always been very buggy and unreliable. Networkd just works.

Are there any downsides?
Yes!

You need sudo to access to connect to a new wifi. However, when you connect once, you automatically connect to that wifi-without any action from your part.
There is no nice interactive interface to this. Seriously, even the awful netctl has more options for this. And networkmanager has excellent user friendly interface for both command line and gui.
To connect to a new network, you need command line. I plan on writing some kind of interactive interface for this when I have the time. And also to automate setting this up. There is wpa_gui (using qt) and wpa_cli that you can use, but they really arent optimal for basic wifi management.
You need to use systemd. For some people, this is a dealbreaker. But most of us use systemd anyway.
if you are not carefull, your wifi password might get stored in plain text in your shell command history. To get around this, you can manually delete those lines from your history, use dash shell (instead of bash, fish or zsh) that does not log your command history, or use a wrapper script or disable command history temporarily.
Okay, I still want to try. How to do this?
You need to create a few configuration files first. You need sudo for this.
For this, you need to know the name of your wireless interface (I think you do). You can get it by running this command:

```
networkctl | awk '/wlan/ {print $2}'
```

This should give you output something like “wlp1s0” but that 1 can be a different number. If that does not work, run just

```
networkctl
```

to get more detailed list.

Then create following files with text editor of your choice, replacing wlp1s0 with name of your interface:

/etc/systemd/network/wireless.network

```
# /etc/systemd/network/wireless.network
[Match]
Name=wl*

[Network]
DHCP=yes
RouteMetric=20
IPv6PrivacyExtensions=true
## to use static IP uncomment these instead of DHCP
#DNS=192.168.1.254
#Address=192.168.1.87/24
#Gateway=192.168.1.254
/etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf
```

/etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf

```
# /etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=wheel
update_config=1
eapol_version=1
ap_scan=1
fast_reauth=1
```

Optional: to make networkd also manage your wired connection, also create

/etc/systemd/network/wired.network

```
# /etc/systemd/network/wired.network
[Match]
Name=en*

[Network]
DHCP=yes
RouteMetric=10
IPv6PrivacyExtensions=true
## to use static IP uncomment these instead of DHCP
#DNS=192.168.1.254
#Address=192.168.1.87/24
#Gateway=192.168.1.254
```

Disable networkmanager, connman, wicd or any other service you might have managing your internet. For network manager, the command is

```
sudo systemctl disable networkmanager
sudo systemctl stop networkmanager
```

I also had to uninstall networkmanager to prevent it from autostarting, but that might not be the case for you. Many packages, such as gnome, depend on networkmanager.

Run following commands to enable right services and replace your resolv.conf with a symlink to systemd folder. Again, replace wlp1s0 with the name of your interface.

```
sudo -i
rm /etc/resolv.conf
systemctl enable systemd-networkd
systemctl enable wpa_supplicant@wlp1s0
systemctl enable systemd-resolved
systemctl start systemd-networkd
systemctl start wpa_supplicant@wlp1s0
systemctl start systemd-resolved
ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Add your wifi to your configuration with

```
set +o history
sudo wpa_passphrase your-ESSID your-wifi-passphrase >> /etc/wpa_supplicant/wpa_supplicant-wlp1s0.conf
set -o history
```

is the name of your wifi and is your password.

Reboot. It should work.

Sources (contain also instructions how to do this for wired connection and how to use static ip instead of dhcpd):
https://wiki.archlinux.org/index.php/systemd-networkd 319
https://beaveris.me/systemd-networkd-with-roaming/ 192
http://dabase.com/blog/Good_riddance_netctl/ 125
http://blog.volcanis.me/2014/06/01/systemd-networkd/ 158
