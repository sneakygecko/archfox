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
- Shared folder host:/home/arch/Downloads/archfox <--> container:/home/fox/Downloads
- Using wayland
- Sound works
- Copy & Paste work between host and browser
- GPU acceleration works
- Auto Updates via systemd

Not Working:
- The Firefox window does not have minimse button. It still minimses by clicking task bar.

# LXC Setup
Refer to these:

https://wiki.archlinux.org/title/Linux_Containers

https://wiki.archlinux.org/title/LXD

On your Arch Linux Host:
```bash
sudo pacaman -Syu lxc arch-install-scripts
```

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

Check that your user account id is 1000 otherwise edit as needed
```bash
id
```

