
# Setting up OctoPrint for Raspberry Pi (Raspbian) or Odroid C1 (Ubuntu 18.04)

this document is based on: https://community.octoprint.org/t/setting-up-octoprint-on-a-raspberry-pi-running-raspbian/2337/4

---
**Heads-up**
If you want to get OctoPrint up and running as fast as possible, it is highly recommended to take a look at [OctoPi](https://github.com/guysoft/OctoPi), which is an SD card image based on Raspbian already prepared with OctoPrint, Webcamsupport, HAProxy and SSL. Just download it, flash it to an SD card and you are good to go -- you can follow [this excellent video guide](https://www.youtube.com/watch?v=MwsxO3ksxm4) by [Thomas Sanladerer](https://www.youtube.com/channel/UCb8Rde3uRL1ohROUVg46h1A) who explains all needed steps in detail.

If on the other hand you want to run the latest versions of Raspbian, OctoPrint and all the other packages, and get a sense of how it all fits together, do follow the instructions below (warning: not for the faint of heart).

---

**Important:** This guide expects you to have a more than basic grasp of the Linux command line. In order to follow it you'll need to know:

- how to issue commands on the shell,
- how to edit a text file from the command line,
- what the difference is between your user account (e.g. pi) and the superuser account root,
- how to SSH into your Pi (so you don't need to also attach keyboard and monitor),
- how to use Git and
- how to use the Internet to help you if you run into problems.

This is not a "Linux for Beginners guide", those can be found for example [here](http://elinux.org/RPi_Beginners) and [here](http://linuxcommand.org/learning_the_shell.php). For some Git basics please take a look [here](http://rogerdudler.github.io/git-guide/).

## Create user octo

Octo print doesn't run as root but as user octo. This has to be created, let's do so:

```
adduser octo
usermod -aG sudo octo
su - octo
sudo ls -la /root # test if sudo works
```

## Basic setup

For the basic package you'll need Python 2.7 (should be installed by default) and pip. OctoPrint's dependencies will be installed by pip:

```
cd ~
sudo apt update
sudo apt install python-pip python-dev python-setuptools python-virtualenv git libyaml-dev build-essential
mkdir /opt/OctoPrint && cd /opt/OctoPrint
virtualenv venv
source venv/bin/activate
pip install pip --upgrade
pip install octoprint
```

If this installs an old version of OctoPrint, pip probably still has something cached. In that case add --no-cache-dir to the install command, e.g.

```
pip install --no-cache-dir octoprint
```
To make this permanent, clean pip's cache:

```
rm -r ~/.cache/pip
```

You may need to add the pi user to the dialout group and tty so that the user can access the serial ports:

```
sudo usermod -a -G tty octo
sudo usermod -a -G dialout octo
```

You should then be able to start the OctoPrint server:

```
octo@octo ~ $ ~/OctoPrint/venv/bin/octoprint serve
 * Running on http://0.0.0.0:5000/
````

## Updating & changing release channels & rolling back

OctoPrint should offer to update itself automatically and also allow you to switch to other Release Channels out of the box.

If for whatever reason you want or need to perform any of this manually however, perform the following commands to install <version> of OctoPrint:

```
source ~/OctoPrint/venv/bin/activate
pip install octoprint==<version>
```

e.g.

```
source ~/OctoPrint/venv/bin/activate
pip install octoprint==1.3.12rc1
```

## Support restart/shutdown through OctoPrint's system menu

In Settings > Commands, configure the following commands:

- Restart OctoPrint: `sudo service octoprint restart`
- Restart system: `sudo shutdown -r now`
- Shutdown system: `sudo shutdown -h now`

Note
If you disabled Raspbian's default behaviour of allowing the pi user passwordless sudo for every command, you'll need to explicitly allow the pi user passwordless sudo access to the /sbin/shutdown program for the above to work. You'll have to add a sudo rule by creating a file /etc/sudoers.d/octoprint-shutdown (as root) with the following contents:

```
pi ALL=NOPASSWD: /sbin/shutdown
```

## Automatic start up

Download the init script files from OctoPrint's repository, move them to their respective folders and make the init script executable:

```
wget https://github.com/foosel/OctoPrint/raw/master/scripts/octoprint.init && sudo mv octoprint.init /etc/init.d/octoprint
wget https://github.com/foosel/OctoPrint/raw/master/scripts/octoprint.default && sudo mv octoprint.default /etc/default/octoprint
sudo chmod +x /etc/init.d/octoprint
```

Adjust the paths to your octoprint binary in /etc/default/octoprint. If you set it up in a virtualenv as described above make sure your /etc/default/octoprint is modified like this:

```
DAEMON=/home/octo/OctoPrint/venv/bin/octoprint
```

Note also the removed # at the start of the line, uncommenting it and making it effective!

Then add the script to autostart using:

```
sudo update-rc.d octoprint defaults.
```

This will also allow you to start/stop/restart the OctoPrint daemon via

```
sudo service octoprint {start|stop|restart}
```

## Make everything accessible on port 80

If you want to have nicer URLs or simply need OctoPrint to run on port 80 (http's default port) due to some network restrictions, I recommend using [HAProxy](http://haproxy.1wt.eu/) as a reverse proxy instead of configuring OctoPrint to run on port 80. This has the following advantages:

- OctoPrint does not need to run with root privileges, which it would need to to be able to bind to port 80 thanks to Linux privileged port restrictions
- You can make mjpg-streamer accessible on port 80 as well
- You can add authentication to OctoPrint
- Depending on the HAProxy version you can also use SSL to access OctoPrint

Setup on Raspbian is as follows:

```
octo@octo ~ $ sudo apt install haproxy
```

I'm using the following configuration in /etc/haproxy/haproxy.cfg, for further examples take a look [here](https://discourse.octoprint.org/t/reverse-proxy-configuration-examples/1107):

```
global
        maxconn 4096
        user haproxy
        group haproxy
        daemon
        log 127.0.0.1 local0 debug

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        retries 3
        option redispatch
        option http-server-close
        option forwardfor
        maxconn 2000
        timeout connect 5s
        timeout client  15min
        timeout server  15min

frontend public
        bind :::80 v4v6
        use_backend webcam if { path_beg /webcam/ }
        default_backend octoprint

backend octoprint
        reqrep ^([^\ :]*)\ /(.*)     \1\ /\2
        option forwardfor
        server octoprint1 127.0.0.1:5000

backend webcam
        reqrep ^([^\ :]*)\ /webcam/(.*)     \1\ /\2
        server webcam1  127.0.0.1:8080
```

This will make OctoPrint accessible under http://<your Raspi's IP>/ and make mjpg-streamer accessible under http://<your Raspi's IP>/webcam/. You'll also need to modify /etc/default/haproxy and enable HAProxy by setting ENABLED to 1. After that you can start HAProxy by issuing the following command

```
sudo service haproxy start
```

Pointing your browser to http://<your Raspi's IP> should greet you with OctoPrint's UI. Now open the settings and switch to the webcam tab or alternatively open ~/.octoprint/config.yaml. Set the webcam's stream URL from http://<your Raspi's IP>:8080/?action=stream to /webcam/?action=stream (leave the snapshotUrl at http://127.0.0.1:8080/?action=snapshot!) and reload the page.

If everything works you can add the following lines to ~/.octoprint/config.yaml (just create it if it doesn't exist yet) to make the server bind only to the loopback interface:

```
server:
    host: 127.0.0.1
```

Restart the server. OctoPrint should still be available on port 80, including the webcam feed (if enabled).

## Optional: Webcam

If you also want webcam and timelapse support, you'll need to download and compile MJPG-Streamer:

You must have cmake installed. You will also probably want to have a development version of libjpeg installed. I used libjpeg8-dev. e.g.

```
cd ~
sudo apt-get install cmake libjpeg8-dev
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer-experimental
make
sudo make install
```

You should then be able to start the webcam server using:

```
LD_LIBRARY_PATH=. sudo mjpg_streamer -i "input_uvc.so -d /dev/video0 -r 1280x1024 -f 9" \
  -o "output_http.so -p 8080 -w /home/octo/mjpg-streamer/mjpg-streamer-experimental/www"
```

This should give the following output:

```
MJPG Streamer Version: git rev: 5a6e0a2db163e6ae9461552b59079870d0959340
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 1280 x 1024
 i: Frames Per Second.: 9
 i: Format............: JPEG
 i: TV-Norm...........: DEFAULT
 i: The specified resolution is unavailable, using: width 1280 height 960 instead
 i: FPS coerced ......: from 9 to 10
UVCIOC_CTRL_ADD - Error at Pan (relative): Inappropriate ioctl for device (25)
[...]
 o: HTTP TCP port........: 8080
 o: HTTP Listen Address..: (null)
 o: username:password....: disabled
 o: commands.............: enabled
```

If you now point your browser to `http://<your Raspi's IP>:8080/?action=stream`, you should see a moving picture at 5fps. (If you get an error message about missing files or directories calling the output plugin with -o "./output_http.so -w ./www" should help.)

Open OctoPrint's settings dialog and under Webcam & Timelapse configured the following:

```
Stream URL: http://127.0.0.1:8080/?action=stream
Snapshot URL: http://192.168.111.164:8080/?action=snapshot
Path to FFMPEG: /usr/bin/ffmpeg
```

---

**Heads-up**

Restart the OctoPrint server, clear the cache on your browser and reload the OctoPrint page. You should now see the stream from the webcam in the "Control" tab, and a "Timelapse" tab with options.

---

If you want to be able to start and stop mjpeg-streamer from within OctoPrint, put the following in /home/octo/scripts/webcam:

```
#!/bin/bash
# Start / stop streamer daemon

case "$1" in
    start)
        /home/octo/scripts/webcamDaemon >/dev/null 2>&1 &
        echo "$0: started"
        ;;
    stop)
        pkill -x webcamDaemon
        pkill -x mjpg_streamer
        echo "$0: stopped"
        ;;
    *)
        echo "Usage: $0 {start|stop}" >&2
        ;;
esac
```

Put this in /home/octo/scripts/webcamDaemon:

```
#!/bin/bash

MJPGSTREAMER_HOME=/home/octo/mjpg-streamer/mjpg-streamer-experimental
MJPGSTREAMER_INPUT_USB="input_uvc.so"
MJPGSTREAMER_INPUT_RASPICAM="input_raspicam.so"

# init configuration
camera="auto"
camera_usb_options="-r 1280x1024 -f 9"
camera_raspi_options="-fps 10"

if [ -e "/boot/octopi.txt" ]; then
    source "/boot/octopi.txt"
fi

# runs MJPG Streamer, using the provided input plugin + configuration
function runMjpgStreamer {
    input=$1
    pushd $MJPGSTREAMER_HOME
    echo Running ./mjpg_streamer -o "output_http.so -p 8080 -w ./www" -i "$input"
    LD_LIBRARY_PATH=. ./mjpg_streamer -o "output_http.so -p 8080  -w ./www" -i "$input"
    popd
}

# starts up the RasPiCam
function startRaspi {
    logger "Starting Raspberry Pi camera"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_RASPICAM $camera_raspi_options"
}

# starts up the USB webcam
function startUsb {
    logger "Starting USB webcam"
    runMjpgStreamer "$MJPGSTREAMER_INPUT_USB $camera_usb_options"
}

# we need this to prevent the later calls to vcgencmd from blocking
# I have no idea why, but that's how it is...
vcgencmd version

# echo configuration
echo camera: $camera
echo usb options: $camera_usb_options
echo raspi options: $camera_raspi_options

# keep mjpg streamer running if some camera is attached
while true; do
    if [ -e "/dev/video0" ] && { [ "$camera" = "auto" ] || [ "$camera" = "usb" ] ; }; then
        startUsb
    elif [ "`vcgencmd get_camera`" = "supported=1 detected=1" ] && { [ "$camera" = "auto" ] || [ "$camera" = "raspi" ] ; }; then
        startRaspi
    fi

    sleep 120
done
```

Make sure both files are executable:

```
chmod +x /home/octo/scripts/webcam
chmod +x /home/octo/scripts/webcamDaemon
```

If you want different camera options put them in /boot/octopi.txt or modify the script accordingly.

If you want autostart of the webcam you need to add the following line to /etc/rc.local (Just make sure to put it above the line that reads exit 0).

```
/home/octo/scripts/webcam start
```

If you want to be able to start and stop the webcam server through OctoPrint's system menu, add the following to config.yaml:

```
system:
  actions:
   - action: streamon
     command: /home/octo/scripts/webcam start
     confirm: false
     name: Start video stream
   - action: streamoff
     command: sudo /home/octo/scripts/webcam stop
     confirm: false
     name: Stop video stream
```

---

**Note**

If you want to view the stream directly on your Pi, please be aware that Midori will not allow you to see the webcam picture. Chromium works although it is a bit slow, but it still might be useful for testing or aiming the camera:

```
sudo apt install chromium-browser
```

In any case this is only recommended for debugging purposes during setup, running a graphical user interface on the Pi will put a lot of unnecessary load on the CPU which might negatively influence print results.

---

**Note**

mjpegstreamer does not allow to bind to a specific interface to limit the accessibility to localhost only. If you want your octoprint instance to be reachable from the internet you need to block access to port 8080 from all sources except localhost if you don't want the whole world to see your webcam image.

To do this simply add iptables rules like this:

```
sudo /sbin/iptables -A INPUT -p tcp -i wlan0 ! -s 127.0.0.1 --dport 8080 -j DROP    # for ipv4
sudo /sbin/ip6tables -A INPUT -p tcp -i wlan0 ! -s ::1 --dport 8080 -j DROP         # for ipv6
```

Replace the interface with eth0, if you happen to use ethernet.

To make them persistent, they need to be saved. In order to be restored at boot time, the easiest way is to install iptables-persist:

```
sudo apt install iptables-persistent
```

The only thing left to do now, is save the rules you have added:

```
sudo /sbin/ip6tables-save > /etc/iptables/rules.v6
sudo /sbin/iptables-save > /etc/iptables/rules.v4
```

## Optional: Touch UI

Touch UI is a plugin that provides an interface for touch screens, e.G. mobile phones or the small 3,5 inch LCDs you can connect to the pi's GPIO pins.

Install the plugin using the plugin manager in the OctoPrint settings. If you want to use is for a local LCD, you need to setup epiphany to start automatically. To do so, first install xautomation to send the keypress for fullscreen later and the epiphany browser if it is not already installed:

```
sudo apt install epiphany-browser xautomation
```

Next, create a file `startTouchUI.sh` in `~/` and add:

```
#!/bin/bash

function check_octoprint {
    pgrep -n octoprint > /dev/null
    return $?
}

until check_octoprint
do
    sleep 5
done

sleep 5s
epiphany-browser http://127.0.0.1:5000 --display=:0 &
sleep 10s;
xte "key F11" -x:0
```

Make it executable:

```
chmod +x startTouchUI.sh
```

and add the following to `~/.config/lxsession/LXDE-pi/autostart`

```
@/home/octo/startTouchUI.sh
```

This will launch the mobile webinterface on startup and put it into fullscreen mode.

## Optional: Additional user authentication

In order to protect OctoPrint from unauthorized access, you have two options. For OctoPrint's built-in access control, please see this guide 362.

For additional security through authentication directly on haproxy before OctoPrint, take a look here 192.

## Optional: Reach your printer by typing its name in address bar of your browser - Avahi/zeroconf/bonjour-based

If you want to reach your printer by typing its name (the hostname of the RasPi running OctoPrint) instead of its IP into your browser's address bar, then you can use the Raspberry Pi Avahi setup (See only section "The flexible way: set up avahi / zeroconf") 217. Note: "Avahi" is called "Zeroconf", "Rendezvous" or "Bonjour", too.

Installation is simple, on your RasPi just type:

```
sudo apt update && sudo apt install avahi-daemon
```

For a network of Linux computers you are done here with the avahi setup. Jump to the paragraph relating the change of the hostname. If you want to enable Avahi support on Windows computers too you'll have to install Bonjour 85, allow traffic on UDP port 5353 within your firewall and grant internet access to the mDNSresponder.exe on these machines. Have a look here (search for "Get Bonjour for Windows") 217 for a detailed description of the Windows setup.

The next step is to change the hostname of your RasPi into something more printer specific (e.g. <yourprinter>) via editing the files etc/hostname and the etc/hosts on your RasPi. Change the default name into <yourprinter> in the hostname-file via

```
sudo nano /etc/hostname
```

and do the same (here change the name behind the 127.0.1.1 into <yourprinter>) in the hosts-file via

```
sudo nano /etc/hosts
```

Now restart your RasPi via

```
sudo reboot.
```

You can now reach your RasPi running OctoPrint within your network by pointing your browser to

```
<yourprinter>.local .
```

Note you can use this too, when you want to ssh into your RasPi:

```
ssh <username>@<yourprinter>.local.
```

## Additional Resources:

[Raspberry Pi Avahi Setup instructions on elinux.org](http://elinux.org/RPi_Advanced_Setup#Setting_up_for_remote_access_.2F_headless_operation)
[Bonjour Support for Windows from Apple](http://support.apple.com/kb/DL999)(Download)
