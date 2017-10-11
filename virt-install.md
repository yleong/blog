Title: How to use libvirt without graphics
Date: Wed Aug 23 00:16:13 +08 2017
Category: Howto
Tags: howto, libvirt, virsh, virt-install, kvm

I am trying out using `kvm` with `libvirt`. The front end tools are
`virt-install` and `virsh`.  I want to install graphicless guests on
graphicless host.  That requires passing `--graphics none` to `virt-install`.
The guest's installer must output a text console to its first serial port
`ttyS0`. Most installers nowadays are graphical by default. `virt-install` will
need `--extra-args` to redirect console output to the guest's serial port. 

I first tried installing debian 9 relying on an external vnc viewer.

`virt-install --name deb1 --memory 512 --disk 'size=3' --cdrom /root/debian-9.1.0-amd64-netinst.iso  --os-type=linux --graphics 'vnc,listen=0.0.0.0' --os-variant=debian8` 

After installation, I wanted to use the VM via `virsh console`. That requires
Debian's kernel to redirect console to serial port. Using vnc, I edited the
VM's `/etc/default/grub`. I added `GRUB_TERMINAL='serial console'` and
`GRUB_CMDLINE_LINUX_DEFAULT='console=tty0 console=ttyS0'`. I regenerated with
`update-grub` and reboot, then `virsh console deb1` works.

I then tried taking out vnc altogether. This needs internet access to install.

`virt-install --name deb2 --memory 512 --disk 'size=3' --location http://ftp.us.debian.org/debian/dists/stable/main/installer-amd64/ --os-type=linux --graphics none --os-variant=debian8 --extra-args='console=ttyS0'` 

The `extra-args` boots the Debian installer to console. After installation,
grub will try to boot to a graphical console by default. This will cause `virsh
console deb2` to stop working. To workaround this, press `e` at the grub menu
and replace `quiet` with `console=ttyS0`. Boot to Debian, then commit this
change to `/etc/default/grub`. The steps to do so are exactly the same as
before.

Alternatively, right before ending the installation, you can drop into a shell
and `chroot /target` and edit the grub files that way, then reboot.

It has been a fun exercise to mess around with `virsh` and `virt-install`. But
`vagrant` with `libvirt` plugin may be easier to use.
