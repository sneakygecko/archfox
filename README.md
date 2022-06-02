# ArchFox Description
This will allow you to run Firefox from inside a lxc container.

My Host:
- Arch Linux
- KDE

The Container:
- Using LXC
- Arch Linux
- Unpriviledged
- Non root user account (fox)
- Shared folder host:/home/USER/Downloads/archfox <--> container:/home/fox/Downloads
- Using wayland
- Sound works
- Copy & Paste work between host and browser
- GPU acceleration works
- Auto Updates via systemd

Not Working:
- The Firefox window does not have minimse button. It still minimses by clicking task bar.

TODO:
- Change lxc profile to allow running as a vm
  - `sudo pacman -Syu qemu-full`      # on host
  - `lxc launch images:archlinux archfox --vm -c "security.secureboot=false" -p default`
  - Error: Invalid expanded devices: Device validation failed for "PulseSocket": Only NAT mode is supported for proxies on VM instances

# LXC Setup
Refer to these:

https://wiki.archlinux.org/title/Linux_Containers

https://wiki.archlinux.org/title/LXD

On your Arch Linux Host:
```bash
sudo pacaman -Syu lxc arch-install-scripts
```

Set up some files for lxc to run unprivileged containers

/etc/lxc/default.conf
```
lxc.idmap = u 0 100000 65536
lxc.idmap = g 0 100000 65536
```

/etc/subuid
```
root:100000:65536
```

/etc/subgid
```
root:100000:65536
```

Add your user to the lxd group so you can run lxc without sudo
```bash
sudo usermod -aG lxd $USER
```

Log out and back in for the user groups to update.
Check that your user account id is 1000 otherwise edit as needed.
```bash
id
```

# Installing the Container

On your host:

```bash
lxc profile create archfox
lxc profile edit archfox
```

- Change the DownloadsFolder -> source path to your host username
- Change all the 1000 if your id is different

```yaml
config:
  environment.XDG_RUNTIME_DIR: "/run/user/1000"
  environment.WAYLAND_DISPLAY: "wayland-0"
  environment.QT_QPA_PLATFORM: "wayland"
  environment.MOZ_ENABLE_WAYLAND: "1"
  environment.PULSE_SERVER: "/run/user/1000/pulse/native"
description: Archlinux with Firefox working with wayland and sound
devices:
  DownloadsFolder:
    path: /home/fox/Downloads
    shift: "true"
    source: /home/USER/Downloads/archfox
    type: disk
  PulseSocket:
    bind: container
    connect: unix:/run/user/1000/pulse/native
    gid: "1000"
    listen: unix:/mnt/pulse1/native
    mode: "0777"
    security.gid: "1000"
    security.uid: "1000"
    type: proxy
    uid: "1000"
  WaylandSocket:
    bind: container
    connect: unix:/run/user/1000/wayland-0
    gid: "1000"
    listen: unix:/mnt/wayland1/wayland-0
    mode: "0777"
    security.gid: "1000"
    security.uid: "1000"
    type: proxy
    uid: "1000"
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  mygpu:
    type: gpu
  root:
    path: /
    pool: lxdpool
    type: disk
name: archfox
used_by: []
```

On your Host:
```bash
lxc launch images:archlinux archfox -p default
```

Then get inside the archfox

```bash
lxc exec archfox bash
```

And run the following.

```bash
# Make a non root user
useradd -m fox

# Mounts get cleared on reboot recreate on startup
echo '#!/bin/sh' > /root/startup_stuff.sh
echo 'mkdir -p /run/user/1000/pulse /mnt/wayland1 /mnt/pulse1' >> /root/startup_stuff.sh
echo 'ln -s /mnt/wayland1/wayland-0 /run/user/1000/wayland-0' >> /root/startup_stuff.sh
echo 'ln -s /mnt/pulse1/native /run/user/1000/pulse/native' >> /root/startup_stuff.sh
echo 'chown -R fox:fox /run/user/1000/' >> /root/startup_stuff.sh
chmod +x /root/startup_stuff.sh
echo '[Unit]' > /etc/systemd/system/startup_stuff.service
echo 'Description=Get stuff ready' >> /etc/systemd/system/startup_stuff.service
echo '' >> /etc/systemd/system/startup_stuff.service
echo '[Service]' >> /etc/systemd/system/startup_stuff.service
echo 'ExecStart=/root/startup_stuff.sh' >> /etc/systemd/system/startup_stuff.service
echo '' >> /etc/systemd/system/startup_stuff.service
echo '[Install]' >> /etc/systemd/system/startup_stuff.service
echo 'WantedBy=multi-user.target' >> /etc/systemd/system/startup_stuff.service
chmod +x /etc/systemd/system/startup_stuff.service
systemctl enable startup_stuff

# Daily autoupdates at 16:00
echo '[Unit]' > /etc/systemd/system/autoupdate.service
echo 'Description=Weekly auto update' >> /etc/systemd/system/autoupdate.service
echo 'After=network-online.target' >> /etc/systemd/system/autoupdate.service
echo '' >> /etc/systemd/system/autoupdate.service
echo '[Timer]' >> /etc/systemd/system/autoupdate.service
echo 'OnCalendar= *-*-* 16:00:00' >> /etc/systemd/system/autoupdate.service
echo 'Persistent=true' >> /etc/systemd/system/autoupdate.service
echo '' >> /etc/systemd/system/autoupdate.service
echo '[Service]' >> /etc/systemd/system/autoupdate.service
echo 'ExecStart=/usr/sbin/pacman -Syu --noconfirm' >> /etc/systemd/system/autoupdate.service
echo '' >> /etc/systemd/system/autoupdate.service
echo '[Install]' >> /etc/systemd/system/autoupdate.service
echo 'WantedBy=timers.target' >> /etc/systemd/system/autoupdate.service
chmod +x /etc/systemd/system/autoupdate.service
systemctl enable autoupdate

# Install Firefox
pacman -Syu --noconfirm firefox
echo 'Done'
exit
```

Now we have to connect wayland and pulse using the profile above. If we did this initially it would hang on boot.

On your host:

```bash
lxc stop archfox
lxc profile assign archfox archfox
lxc start archfox
lxc exec archfox -- runuser -u fox -- firefox
```

That last command can then be added as an application menu entry.

I also themed my host and container firefox differently so I can tell the difference.
