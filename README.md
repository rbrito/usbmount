# USBmount

The USBmount package automatically mounts USB mass storage devices (e.g.,
USB pen drives or HDs in USB enclosures) when they are plugged in.  The
mountpoints (`/media/usb[0-7]` by default), filesystem types to consider,
and mount options are configurable. When multiple devices are plugged
in, the first available mountpoint is automatically selected.

If the device plugged provides a model name, a symbolic link at
`/var/run/usbmount/MODELNAME` pointing to the mountpoint is automatically
created.

If the device plugged has a label (e2label), a symbolic link at
`/var/run/usbmount/LABEL` pointing to the mountpoint is automatically
created.

When the device is not present anymore in the system (e.g.,
after it has been unplugged), usbmount deletes the symbolic links that
were created.

The script that does the (un)mounting is called by the `udev` daemon.
Therefore, USBmount requires a 2.6 (or newer) kernel.

USBmount is intended as a lightweight solution which is independent of a
desktop environment. Users which would like an icon to appear when an
USB device is plugged in should use other alternatives.

The comments in the configuration file `/etc/usbmount/usbmount.conf`
describe how to configure the package.


## Generic Comments about Flash Drives

Users should be aware that, independently of the filesystem used by the
mass storage device, *ANY* filesystem that resides in flash memory will
become unreadable after some time. This unfortunate situation is
intrinsic to the storage medium and better quality flash drives perform
a "wear levelling" operation, distributing the load of operations across
the whole device. [*]

Filesystems using flash memory and mounted with the `sync` option can
degrade earlier due to the fact that the sync mount option forces the
operating system to write data more frequently to the device than if it
were mounted without the `sync` option.

So, why mount filesystems with the `sync` option then? The reason is to
keep the written data on the drive reflecting what the user thinks is on
the flash drive, and, more importantly, to avoid the problem of the user
unplugging the device before it is finished receiving data that the
kernel has on the memory of the computer and that is meant to be written
to the device.

If you don't like the `sync` option with your filesystems, then you can
remove it from the configuration file of usbmount and use your devices
with better performance and longer life time. **BUT** you should always
make sure that you use the `sync` command (on a shell) to ensure that
there is no writes pending for the device in question, so that you don't
loose any data when you unplug the device from the computer.

[*] You can see if your flash drives support wear levelling by seeing
    the technical specifications of your specific drives in the
    manufacturer's site (e.g., the manufacturer Kingston provides such
    information regarding its drives and others quite probably do that
    too).

Of course, usbmount doesn't only work with flash drives. Common hard
drives put into enclosures are perfectly used with usbmount and
usbmount, despite its name, can mount drives connected via Firewire
ports, provided that the kernel has support for it (most distribution
kernels, including the ones shipped with Debian and Ubuntu do).

## Generic comments about package dependencies

This package depends on a few other packages for installation. These are
properly declared in the built package and apt-get will install all
required packages if the package is installed from a remote repository
but dpkg doesn't install dependencies when the package is installed
from a local file.

There are a few ways to deal with dependencies when installing files
directly, ex:

Option 1: let apt-get fix missing dependencies.

    # Try install, will not necessarily complete if you're missing a dependency
    dpkg -i <package>.deb

    # Will install missing dependencies and finish the install process
    apt-get install -f


Option 2: use a package installer that fetches dependencies even for local packages (ex. gdebi).

    # Only required the first time you do this with any package
    apt-get install gdebi

    # Actually install the package and it's dependencies
    gdebi <package>.deb

## Technical Considerations

### Control of Filesystems Mounted by USBmount

You can choose which filesystems you want usbmount to automatically
handle by listing the filesystem types provided by the operating system
in the configuration variable `FILESYSTEMS`.

### Recommendations for vfat Filesystems

The vfat filesystem is one of the most commonly used filesystems in pen
drives. Unfortunately, due to its age, it is very poor regarding
features and, in particular, it doesn't feature the most basic access
control present in Unix systems, namely: permissions on files.

Linux works around this by creating "virtual" permissions and
restrictions based on who mounted the filesystem. As usbmount is used,
the user assigned to the vfat filesystem is, by default, root.

For a more flexible configuration, some useful options for vfat
filesystems are to specify explicitly who the user and permissions are.
Please, read the manpage of the mount command to get details.

An example is to specify `-fstype=vfat,gid=floppy,dmask=0007,fmask=0117`
in the `FS_MOUNTOPTIONS` variable of the configuration file. The
particular options specified in the example mean that members of the
floppy group can read from and write to the medium, but nobody else can
access it.

## Troubleshooting USBmount

No software is free of problems and the situation isn't different with
USBmount. To ease the troubleshooting of problems, you may try to check
the following:

* Do you have HAL running? Any GNOME or KDE daemon automounting devices?

* Let's suppose that the partition containing the filesystem that you
  want USBmount to automatically handle is `/dev/sda1` (your case may,
  quite possibly, vary). Then, check the result of the following
  command:

        udevadm test --action=add /sys/class/block/sda1

  The command above just gives diagnostics of what USBmount would do
  with the device, but it doesn't actually mount or interfere with the
  device. It is intended for debugging purposes. Be careful that it
  generates *a lot* of output. Many screens, depending on the device.

* Under the same assumptions as the above, another good diagnostic tool
  is the following:

        udevadm info -a -p $(udevadm info -q path -n /dev/sdb1)

# Hook Scripts

After a device or partition has been mounted, the command `run-parts
/etc/usbmount/mount.d` is executed. This runs all scripts in
`/etc/usbmount/mount.d` which adhere to a certain naming convention; see
the `run-parts` manual page for details.

The following environment variables are available to the scripts (in
addition to those set in `/etc/usbmount/usbmount.conf` and by the hotplug
and udev systems):

|    Variable     | Description                                              |
|-----------------|----------------------------------------------------------|
|`UM_DEVICE`      | file name of the device node                             |
|`UM_MOUNTPOINT`  | mountpoint                                               |
|`UM_FILESYSTEM`  | filesystem type                                          |
|`UM_MOUNTOPTIONS`| mount options that have been passed to the mount command |
|`UM_VENDOR`      | vendor of the device (empty if unknown)                  |
|`UM_MODEL`       | model name of the device (empty if unknown)              |

Likewise, the command `run-parts /etc/usbmount/umount.d` is executed
after a device or partition has been unmounted. The scripts can make use
of the environment variables `UM_DEVICE`, `UM_MOUNTPOINT` and `UM_FILESYSTEM`.
Note that vendor and model name are no longer easily available when the
device has been removed. If you need this information in an unmount hook
script, write it to a file in a mount hook script and read it back in
the unmount hook script.


## Safely unmounting filesystems

As it is not possible for the system to detect when the device should be
unmounted (such information is only present when the device has already
been unplugged, which is too late for some clean ups, like flushing
unwritten buffers to disk and marking the filesystem as clean), the user
should manually unmount the device that were automatically mounted.

This situation is similar to those in graphical desktop environments
where the user has to click on an icon and inform the system that it
wants to remove the device from the computer.

A recommended solution for this problem is to use the `pumount` command
(provided by the `pmount` package), which acts as a wrapper around the
regular mount command and lets regular users (i.e., not root) to unmount
the filesystems, conveniently.

**Warning:** carelessly removing the device/filesystem without unmounting it
first can (and does) lead to massive filesystem corruption and should
only be performed if you know exactly what you are doing.


## The Special Case of FUSE Filesystems

Many users use removable drives with NTFS filesystems and the
user-space filesystem NTFS-3g, since it provides more flexibility than
the native module present in the Linux kernel.

Such users have difficulty when unmounting the filesystems, since they
are present in the system `/etc/mtab` with a filesystem type of `fuseblk`,
not with `ntfs` (or `ntfs-3g`) as one might expect.

For such filesystems, it may be convenient to:

* add `ntfs-3g` to `/etc/usbmount/usbmount.conf`'s variable `FILESYSTEMS`
  (for mounting purposes)
* add `fuseblk` to `/etc/usb/usbmount.conf`'s variable `FILESYSTEMS`
  (for unmounting purposes).

Similar comments may apply to other FUSE-managed filesystems.  In general,
if you need a FUSE filesystem, it may be a good idea to add the name of
that filesystem to the `FILESYSTEMS` variable as well as making sure that the
special `fuseblk` filesystem is contained in that list.


This subsection is an adaptation of descriptions made by Thomas Jancar
and Jan Schulz.


## Remounting filesystems without physical removal

usbmount operates (read: "mounts or unmounts filesystems") based on events
issued by the Linux kernel/udev. As a consequence, if you happen to unmount a
filesystem and want to mount it again, you have basically two choices:

1. unplug and plug the device, which may not be desired, for a number of
    reasons.
2. make the kernel/udev generate another event so that usbmount knows that it
    has some work to do.

The latter can be accomplished by the use of the command

	udevadm trigger --action=add /dev/sdd2

where `/dev/sdd2` should be substituted with the proper partition. This
command is likely needed to be run with superuser privileges.
Triggering kernel events is also a way to get a particular filesystem
mounted after a cold boot.

## Building the package

In order to build this package, debhelper and build-essential are
required. The package can be built with the simple commands:

```
# Install dependencies
sudo apt-get update && sudo apt-get install -y debhelper build-essential

# Build
sudo dpkg-buildpackage -us -uc -b
```
