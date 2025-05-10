# Radxa-Orion-O6-Arch-PKGBUILD-Kernel-pkg
Radxa Orion O6 Arch Linux PKGBUILD &amp; prebuilt Kernel image &amp; headers

Simply download the Radxa Cix Orion O6 linux kernel from
```sh
https://gitlab.com/cix-linux/cix_opensource/linux
```
configure the kernel how you want it or use the included config. Download the arch linux flavor of your choice then chroot into the installation. Make sure to rename the kernel source folder to linux-cix then execute.
```sh
makepkg -sif
```
