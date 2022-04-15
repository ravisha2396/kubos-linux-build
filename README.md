# Kubos Linux

## Overview

Check out the official [Kubos Linux](https://docs.kubos.com/latest/ecosystem/index.html#kubos-linux) documentation for the most current instructions on how to setup and interact with Kubos Linux on specific boards.
Kubos uses BuildRoot as its main Linux build tool.  This repo contains the configuration and patch files required by BuildRoot to build Linux for each of the OBCs that Kubos supports.

## Contributing

Want to get your code to space? Become a contributor! Check out our doc on [contributing to KubOS](https://docs.kubos.com/latest/contributing/contribution-process.html)
and come talk to us on [Slack](https://slack.kubos.co/) to join our community.
Or, if you're just looking to give some feedback,
submit an [issue](https://github.com/kubos/kubos-linux-build/issues) with your feature requests or bug reports!

## Files and folders

### external.desc

This file contains the name and description for the kubos-linux-build folder.  BuildRoot will use it to create a link between the
main BuildRoot folder and the kubos-linux-build folder.

### external.mk

Currently (intentionally) empty.  This file is required by BuildRoot when creating the external link to the kubos-linux-build folder,
but no content is actually required.

### Config.in

Currently empty.  Same as the external.mk file, BuildRoot requires that this file exist, but not that it have any content.

### board/kubos/{board}

Inside of each of these folders are the patches and configuration files for each of the components required to build Linux for
a particular board.

### configs

This folder contains the BuildRoot configuration files needed to build each OBC supported by Kubos.

## Configuring the Beaglebone Black from a fresh buildroot

> This document is using Buildroot 2022-02, the latest LTS build at the time of writing.

Make a new folder for the project which will contain all the related files, and download
the buildroot file to it.

After downloading the buildroot file, decompress and unpack it; this will create the
`buildroot-2022.02` folder containing the buildroot system.

### Configuring buildroot with BR2_EXTERNAL
If you don't want don't want to manually configure buildroot yourself or aren't trying out new
changes, you can use a preconfigured buildroot. After creating a new folder and unpacking
buildroot to it, clone the [external config](https://github.com/ASU-cubesat/kubos-linux-build)
and place it next it next to the buildroot folder. After it's finished cloning, you must check out
the branch with the changes for buildroot-2022:
```bash
cd kubos-linux-build
git checkout buildroot-2022-02
```
then move back  into the buildroot folder:

```bash
cd ../buildroot-2022.02
```

Next we tell buildroot to use our external tree as configuration:

```bash
make BR2_EXTERNAL=../kubos-linux-build/ beaglebone-black_defconfig
```

and finally build the system:

```bash
make
```

Once it's finished, it will create an `output/images` folder which will contain a file
called `sdcard.img`. This is the file that you burn to the SD card containing your new
buildroot system.

This system should already be configured to allow serial access over the power cable
through `/dev/ttyACM0`, create an ethernet adapter on the host similar to `enp0s20f0u1`,
and register the device as a `Linux Multifunction Composite Gadget` which can be seen
in the output of `lsusb`. Note that the device must be fully booted before these will
appear on the host.

### Creating your own BR2_EXTERNAL tree
If you want to create your own external buildroot tree or modify the one from above, the
process is very straightforward. To base your changes off the premade config from above,
follow the steps up to and including the `make BR2_EXTERNAL=` command, but don't build
the full system. If you want to start fresh, just unpack buildroot as normal.

After unpacking buildroot and optionally starting with an external configuration as a
base, proceed to make any changes to the system via `make menuconfig` or
`make linux-menuconfig` and friends as you see fit. Once your changes are made, you can save
your changes to buildroot itself by running

```bash
make savedefconfig BR2_DEFCONFIG=/path/to/safe/defconfig
```

obviously adjusting `BR2_DEFCONFIG` as you see fit. For example, to save the new
buildroot defconfig in a file called `new_defconfig` above the root of the buildroot
directory, run the following from the root:

```bash
make savedefconfig BR2_DEFCONFIG=../new_defconfig
```

Similarly, to save your changes to the linux configuration after `make linux-menuconfig`,
run

```bash
make linux-savedefconfig
```

There might be a variable to control where this file gets saved, but I haven't been
able to find it. By default the generated config is stored at

```
output/build/linux-custom/defconfig
```

To store both of these in an external buildroot tree, they must be placed in a specific
directory structure. The entire tree is available in the Buildroot Manual under
`Project-specific customization > Keeping customizations outside of Buildroot > Example layout`,
but I've copied the folders and files relevant to us here:

```
/path/to/br2-ext-tree/
  |- configs/beaglebone-black_defconfig
  |
  |- board/kubos/linux.config
  |- board/kubos/overlay/
```

The file in `configs/` is what you pass to `make BR2_EXTERNAL=../kubos-linux-build`, so
if you called it `my_cool_defconfig`, for example,, you would instead run `make BR2_EXTERNAL=../kubos-linux-build my_cool_defconfig`.
`board/kubos/linux.config` is the defconfig to use for the kernel.

Any files/folders in the `board/kubos/overlay` folder will be copied over the root filesystem after creating the image, and will be
baked into the `sdcard.img` file that is generated. For example, to provide a configured `/etc/inittab` file for the image, create the
file `board/kubos/overlay/etc/inittab` with the desired contents and it will be put into the root filesystem of the generated image.

With this in mind, and the two defconfigs saved from the previous step, place them in their respective locations in the external tree.
The file created by `make savedefconfig` goes in `configs/beaglebone-black_defconfig`, and the file generated by `make linux-savedefconfig`
goes in `board/kubos/linux.config`. Any files you wish to be present in the root filesystem then go in their respective location in
`board/kubos/overlay`.

Once all the files are in place, you can then use this structure to recreate additional buildroot
builds with the same configuration via `make BR2_EXTERNAL=/path/to/my/tree beaglebone-black_defconfig`.


### Manually configuring buildroot

To manually configure the system after unpacking buildroot, `cd` in to `buildroot-2022.02` to begin
configuration.

The first step is to create a base `.config` file, which must be done before we can configure the kernel.
Luckily, Buildroot comes with a default configuration for the Beaglebone black. To set up the default
Beaglebone config, run

```bash
make beaglebone_defconfig
```

from the root of the `buildroot-2022.02` directory, which will generate a `.config` file with
defaults for the beaglebone.

Next, run `make menuconfig` to customize the `buildroot` settings.
You can navigate `menuconfig` with the up and down arrows, options can be
enabled/disabled by pressing the space bar with them selected, and submenus
can be viewed by pressing enter.

Change the C Library to `glibc` from `uClibc-ng`:
```
Toolchain
  -> C Library
    -> glibc
```

You can also optionally change the system hostname and set a root password:
```
System configuration
  (buildroot) System hostname
    ...enter hostname...
  () Root password
    ...enter password...
```

this step is optional; without doing this the system hostname will be `buildroot`
and the root account will have no password.

You can also optionally add extra tools and packages to the system via the `Target Packages` menu.
Some packages which may be especially helpful are `openssh` and `autossh`, which can be found under the following:
```
Target packages
-> Networking applications
  -> [*] autossh
...scroll way down...
  -> [*] openssh
    -> [ ] client
    -> [*] server
    -> [*] key utilities
```
Client utilies are only needed if you intend on connecting to another device over SSH
from the beaglebone, so it can be disabled if not needed.

`autossh` is a package to automatically start and monitor an `ssh` server on the device, making sure it
starts at boot and stays running.
> NOTE: `autossh` will not show up until _after_ `openssh` is selected


The `monit` process monitor is available under
```
System tools
  -> [*] monit
```
I haven't tested the system with `monit` installed yet, though.

Some tools from `util-linux` may also be helpful while developing and troubleshooting:
```
System tools
  -> [*] util-linux --->
    ...pick tools that might be helpful to you...
```


After Buildroot has been configured, exit `menuconfig` by using the left/right arrow keys to
and select the option `< Exit >`, repeating until prompted to save your changes, and select
`< Yes >`.

Next we configure the linux kernel to enable the Multifunction Composite Gadget functionality
and enable all the drivers we need. To configure the kernel, run:
```
make linux-menuconfig # to configure the kernel
```
and ensure all of these options are set as follows. If not, press the `m` key to
enable them as modules, and space to disable if needed
```
Device Drivers
  [*] USB Support --->
      <M> Support for Host-side USB
      <M> EHCI HCD (USB 2.0) support
      <M> EHCI support for OMAP3 and later chips
      <M> OHCI HCD (USB 1.1) support
      <M>   OHCI support for OMAP3 and later chips
      <M> Inventra Highspeed Dual Role Controller
            MUSB Mode Selection (Dual Role mode)  --->
      <M>   TI DSPS platforms
      <M> USB Gadget Support --->
          [*]   Debugging messages (DEVELOPMENT)
          [_]     Verbose debugging Messages (DEVELOPMENT)
          [*]   Debugging information files (DEVELOPMENT)
          [*]   Debugging information files in debugfs (DEVELOPMENT)
          (2)   Maximum VBUS Power usage (2-500 mA)
          (2)   Number of storage pipeline buffers
          [_]   Serial gadget console support
                USB Peripheral Controller  --->
          <M>   USB Gadget functions configurable through configfs
          [*]     Generic serial bulk in/out
          [_]     Abstract Control Model (CDC ACM)
          [_]     Object Exchange Model (CDC OBEX)
          [_]     Network Control Model (CDC NCM)
          [*]     Ethernet Control Model (CDC ECM)
          [*]     Ethernet Control Model (CDC ECM) subset
          [*]     RNDIS
          [*]     Ethernet Emulation Model (EEM)
          [_]     Phonet protocol
          [*]     Mass storage
          [_]     Loopback and sourcesink function (for testing)
          [*]     Function filesystem (FunctionFS)
          [_]     Audio Class 1.0
          [_]     Audio Class 1.0 (legacy implementation)
          [_]     Audio Class 2.0
          [_]     MIDI function
          [_]     HID function
          [_]     USB Webcam function
          [_]     Printer function
```
I'm not sure _all_ of these are needed, but I know it works with the above.

After configuring the kernel exit the menu and save your changes, then run `make` to build the system. Once it finishes compiling,
it will create the `output/images` folder, which will contain a file called `sdcard.img`;
this is the image to be burned on to the SD card for the beaglebone.

After burning the image, there should be two partitions on the card. On Linux, these
should be `/dev/mmcblk0p1` and `/dev/mmcblk0p2`, with `mmcblk0p2` holding the root
filesystem. Before we can boot using the card, we need to add a few files to the
root filesystem to correctly configure it as a gadget device and allow serial over
the power cable.

To enable the Multifunction Composite Gadget functionality, we need to add a startup
script to configure it. Startup scripts are placed in `/etc/init.d` and are run in
descending order, sorted alphabetically; e.g. `a_script` will run before `b_script`.

I recommend naming the script something like `S99` to ensure it runs after everything
else is set up. The following script is known to work for Multifunction Composite Gadget,
TTY over the power cable, and making the beaglebone show up as an ethernet device:

```bash
# Magic numbers pulled from the script used by the official Debian image
usb_idVendor="0x1d6b"
usb_idProduct="0x0104"
usb_bcdDevice="0x0404"
usb_bcdUSB="0x0200"
usb_serialnr="000000"
usb_product="Multi Gadget"
usb_iserialnumber="1234BBBK5678"
usb_imanufacturer="BeagleBoard.org"
usb_iproduct="BeagleBoneBlack"

# Load the stuff needed for configfs
modprobe libcomposite

# Set up our configfs filesystem
mount none /sys/kernel/config -t configfs

# Make a folder for our gadget
mkdir /sys/kernel/config/usb_gadget/g1
cd /sys/kernel/config/usb_gadget/g1

# Set up some metadata
echo $usb_idProduct > idProduct
echo $usb_idVendor > idVendor

mkdir strings/0x409

echo $usb_iserialnumber > strings/0x409/serialnumber
echo $usb_imanufacturer > strings/0x409/manufacturer
echo $usb_product > strings/0x409/product

mkdir configs/c.1

echo 500 > configs/c.1/MaxPower

mkdir -p configs/c.1/strings/0x409
echo "BeagleBone Composite" > configs/c.1/strings/0x409/configuration

mkdir functions/acm.0
mkdir functions/ecm.0
mkdir functions/rndis.0

ln -s functions/rndis.0 configs/c.1
ln -s functions/acm.0 configs/c.1

# Turn it on
echo "musb-hdrc.0" > /sys/kernel/config/usb_gadget/g1/UDC
```
> This is a slimmed down version of my modified script, see my actual file
> for more information and additional helpful links

This script will then run on boot and "advertise" the beaglebone to the device it's
plugged in to as a Multifunction Composite Gadget, creating both an ethernet adapter
(called something similar to `enp0s20f0u1` on my device) and a serial device
(called `/dev/ttyACM0` on my device).

After placing these files on the SD card, you're almost done. It will create the serial
device, but you won't be able to log in to the board through it because there won't be
a login prompt being displayed on that device.

To start the login prompt on the serial device, we need to start a `getty` service at
boot which is done by modifying the `/etc/inittab` file. Add the lines
```
# Put a getty on the USB power cable
ttyGS0::respawn:/sbin/getty -L ttyGS0 115200 xterm
```
to the file. The location doesn't matter, but I placed mine below
```
# Put a getty on the serial port
ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100 # GENERIC_SERIAL
```

to keep the login services together.

