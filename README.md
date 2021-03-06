# USB Turntable to Sonos or Pi Musicbox streaming guide
## for darkice, libaacplus, et al on raspberry pi with raspbian jessie netinstall
to facilitate streaming audio from one of these turntables with USB audio codec output
- Audio Technica AT-LP120-USB (http://amzn.to/1TZvF2y)
- Ion TTUSB or Max (http://amzn.to/1Uv6KDY)

via a Raspberry Pi B or B+

- Raspberry Pi B+ (http://amzn.to/1LzwOeh)
- Raspberry PI B (http://amzn.to/21ChHCJ)

using Raspbian Jessie (https://www.raspberrypi.org/downloads/raspbian/)

## Install raspbian
For a < 4GB install, use the [raspbian-ua-netinst](https://github.com/debian-pi/raspbian-ua-netinst) minimal net installer. Grab [Pi Filler](http://ivanx.com/raspberrypi/) to write the image to your 2GB or less SD card. Then create the file `installer-config.txt` in the root of the card next to `config.txt` and add the following to make sure sshd is running (via [here](https://github.com/debian-pi/raspbian-ua-netinst#installer-customization)):
```
preset=server
packages= # comma separated list of extra packages
mirror=http://mirrordirector.raspbian.org/raspbian/
release=jessie
hostname=vinyl
domainname=
rootpw=raspbian
cdebootstrap_cmdline=
bootsize=+128M # /boot partition size in megabytes, provide it in the form '+<number>M' (without quotes)
rootsize=     # / partition size in megabytes, provide it in the form '+<number>M' (without quotes), leave empty to use all free space
timeserver=time.nist.gov
ip_addr=dhcp
ip_netmask=0.0.0.0
ip_broadcast=0.0.0.0
ip_gateway=0.0.0.0
ip_nameservers=
online_config= # URL to extra config that will be executed after installer-config.txt
usbroot= # set to 1 to install to first USB disk
cmdline="dwc_otg.lpm_enable=0 console=ttyAMA0,115200 kgdboc=ttyAMA0,115200 console=tty1 elevator=deadline"
rootfstype=ext4
rootfs_mkfs_options=
rootfs_install_mount_options='noatime,data=writeback,nobarrier,noinit_itable'
rootfs_mount_options='errors=remount-ro,noatime'
```
Power up your pi, wait about 15 minutes for the netinstall to complete, and ssh in as *root* with the password *raspbian*.

## Dependencies
```
apt-get -y install aptitude apt-utils sudo unzip autoconf libtool libtool-bin checkinstall libssl-dev libasound2-dev libmp3lame-dev libpulse-dev alsa-utils avahi-daemon darkice
```
We will install the darkice package, but compile it later to add AAC+ support

## Compiling libaacplus
```
wget http://tipok.org.ua/downloads/media/aacplus/libaacplus/libaacplus-2.0.2.tar.gz
tar -xzf libaacplus-2.0.2.tar.gz
cd libaacplus-2.0.2
./autogen.sh --with-parameter-expansion-string-replace-capable-shell=/bin/bash --host=arm-unknown-linux-gnueabi --enable-static
make
sudo make install
```

## Compliling libx264 (not required, SKIP)
```
cd /home/pi/src
git clone git://git.videolan.org/x264
cd x264
./configure --host=arm-unknown-linux-gnueabi --enable-static --disable-opencl
make
sudo make install
```

## Compiling libfaac (not required, SKIP)
```
cd /home/pi/src
curl -#LO http://downloads.sourceforge.net/project/faac/faac-src/faac-1.28/faac-1.28.tar.gz
tar xzvf faac-1.28.tar.gz
cd faac-1.28
vi common/mp4v2/mpeg4ip.h
```
then change near line **126**:
```
#ifdef __cplusplus
extern "C" {
#endif
#ifndef _STRING_H
char *strcasestr(const char *haystack, const char *needle);
#endif
#ifdef __cplusplus
}
#endif
```
## Get darkice source
enable the raspbian source repo
```
vi /etc/apt/sources.list
```
uncomment or add the deb-src line
```
deb-src http://archive.raspbian.org/raspbian jessie main contrib non-free rpi
apt-get update
```
get the source
```
cd ~
mkdir src
cd src
apt-get source darkice
cd darkice-1.2
```

## Compiling darkice

```
./configure --with-aacplus --with-aacplus-prefix=/usr/local --with-pulseaudio --with-pulseaudio-prefix=/usr/lib/arm-linux-gnueabihf --with-lame --with-lame-prefix=/usr/lib/arm-linux-gnueabihf --with-alsa --with-alsa-prefix=/usr/lib/arm-linux-gnueabihf --with-jack --with-jack-prefix=/usr/lib/arm-linux-gnueabihf
make
make install
```

Had `--with-faac --with-faac-prefix=/usr/local` in there, but faac doesn't work with icecast2/darkice

## Configure /etc/darkice.cfg
```
# this section describes general aspects of the live streaming session
[general]
duration        = 0         # duration of encoding, in seconds. 0 means forever
bufferSecs      = 1         # size of internal slip buffer, in seconds
reconnect       = yes       # reconnect to the server(s) if disconnected
realtime        = yes       # run the encoder with POSIX realtime priority
rtprio          = 3         # scheduling priority for the realtime threads

# this section describes the audio input that will be streamed
[input]
device          = hw:1,0    # OSS DSP soundcard device for the audio input
sampleRate      = 48000     # other settings have crackling audo, esp. 44100
bitsPerSample   = 16        # bits per sample. try 16
channel         = 2         # channels. 1 = mono, 2 = stereo

# this section describes a streaming connection to an IceCast2 server
# there may be up to 8 of these sections, named [icecast2-0] ... [icecast2-7]
# these can be mixed with [icecast-x] and [shoutcast-x] sections
[icecast2-0]
bitrateMode     = cbr
# format          = mp3
format          = aacp
# bitrate         = 320
bitrate         = 64
server          = vinyl
port            = 8000
password        = vinyl   # or whatever you set your icecast2 password to
mountPoint      = listen
name            = Vinyl
description     = DarkIce on Raspberry Pi
url             = http://vinyl
genre           = vinyl
public          = no
localDumpFile   = recording.m4a
```

## icecast2
```
aptitude install icecast2
```
For the hostname, use `vinyl`, and for both *hackme* passwords, use `vinyl`

Then, for the admin password, leave it blank unless you trust no one

Note: heaven forbid you mess up the icecast2 text GUI config... you'll need to run
```
apt-get autoremove icecast2
apt-get purge icecast2
```
and then reinstall it
```
aptitude install icecast2
```
to get that crappy GUI back... unless there's an easier, undocumented way? and even then, where are the icecast.xml config files? not in /etc/icecast2/ ...

## Autostart darkice
These are from an Ubuntu install and don't exactly match the startup script, but they are close enough and do solve the startup problem

### 1
In /etc/init.d/darkice find:
```
DAEMON=/usr/bin/$NAME
```
and change it to the AAC+ complied version:
```
DAEMON=/usr/local/bin/$NAME
```

### 2
In /etc/init.d/darkice find:
```
start-stop-daemon --start --quiet --pidfile $PIDFILE \
```
and replace it with:
```
start-stop-daemon --start --quiet -m --pidfile $PIDFILE \
```

### 3
In /etc/init.d/darkice find:
```
stop_server() {
# Stop the process using the wrapper
        start-stop-daemon --stop --quiet --pidfile $PIDFILE \
            --exec $DAEMON
        errcode=$?
```
add after (with the new line):
```
    rm $PIDFILE
```

### 4
In /etc/init.d/darkice find:
```
running() {
# Check if the process is running looking at /proc
# (works for all users)
```
add after (with the new line):
```
    sleep 1
```

### 5
In /etc/default/darkice check that you have
```
RUN=yes
```

### 6
```
systemctl daemon-reload
```

### 7
Add default user nobody to the audio group (in my case, to work with ALSA):
```
adduser nobody audio
```

### 8
Fix upstart problem (it seems Darkice is trying to start on boot too early):
```
update-rc.d -f darkice remove
update-rc.d darkice defaults 99
```

See [this forum](http://ubuntuforums.org/showthread.php?t=2183222)

## Final note
While AAC+ is neat, on a local network you might as well stream 320Kbps MP3 for better sound quality, or if you're so inclined, uncompressed WAV 
