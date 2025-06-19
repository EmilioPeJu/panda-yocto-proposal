# Panda Yocto Structure Proposal
# Requirements
- Req 1: Having a simple upgrade process (e.g. copying a few files to the SD
  card).
- Req 2: Having a persistent state for supporting tests, designs and logging.
- Req 3: Having a simple test process for development.
- Req 4: Supporting 3rd party packages.
- Req 5: The versions between all the packages should always be consistent.
- TODO: Maximum size requirement?

# Proposal
![](./panda-yocto.drawio.png)
- Fully updating a panda just requires to copy 5 files (which could come from an
  automatically created bundle), no repartitioning
  needed. Why is this good? repartitioning at boot time is a fragile task (with
  many failure modes) that I've seen failing a few times in the past. The new
  way can only fail (in an obvious way) if you don't have enough room to copy
  the files (in which case, get a bigger sdcard).
- We can still keep `config.txt` to facilitate network configuration from
  outside.
- Panda-yocto could have the option to get the FPGA part either from a release
  or from a directory pointed by the user.

## Images
The images are:
- `initramfs.cpio.tar`: it contains the needed packages to mount and move to the
  definitive rootfs.
- `packages.ext4`: definitive rootfs, it contains system packages and panda
  packages including all the flavors of FMCs. Even though it is called
  `packages.ext4` for convenience, this is actually
  `petalinux-image-minimal.ext4` with the panda packages added (and possibly
  some extra system packages).
- `state.ext4`: Initially it doesn't exist and it is the created (empty) by the
init script.

Having all the packages in `packages.ext4` is better to ensure that all the
parts are consistent with each other.

## Initramfs init script
The init script mounts the images:
```bash
mount -t tmpfs tmpfs /tmp
mkdir /tmp/packages /tmp/state
mount -t ext4 /boot/packages.ext4 /tmp/packages
mount -t ext4 /boot/state.ext4 /tmp/state
```

Then mount an overlayfs using packages as lower layers(immutable) and state as
upper layer:
```bash
mkdir -p /rootfs /tmp/state/upper /tmp/state/work
mount -t overlay -o lowerdir=/tmp/packages/,upperdir=/tmp/state/upper/,workdir=/tmp/state/work/ overlay /rootfs/
```

Finally, switch root and exec init system(possibly systemd?)
```bash
mkdir -p /rootfs/boot /rootfs/proc /rootfs/sys /rootfs/dev /rootfs/tmp
mount --move /boot /rootfs/boot
mount --move /proc /rootfs/proc
mount --move /sys /rootfs/sys
mount --move /dev /rootfs/dev
mount --move /tmp /rootfs/tmp
exec switch_root -c /dev/console /rootfs/ /sbin/init
```
The result is a system that will merge the content of packages and state but any
write will be changing only `state.ext4` (e.g. when installing packages), any
messed up system can be recovered by deleting `state.ext4`.

A proof-of-concept init script can be found [here](/init).

## FAQ
### How do we test a part of software in development?
We have 2 options: either you create a package using the panda-yocto repository
and then install it or you scp the built files individually; any of those
actions will produce the modification of only `state.ext4`.
The last approach can be improved by making the panda repositories create
image directories as a result, for example, PandABlock-server could
create the following build result:
- `/build/image/usr/bin/panda-server`
- `/build/image/lib/modules/.../panda.ko`
- `/build/image/etc/systemd/system/panda-server.service`
- ...
Then, the only thing you need to do to update the panda is:
`rsync -ra /build/image/ panda-host:/`

### While testing, the system broke, how to recover?
Delete `state.ext4` from the sdcard. Another option is manually mounting
 `state.ext4` and going over its content to fix the problem (from outside the
 panda).

### How can we include all the FMC variants in one image without conflicts?
This will require changes in some of the panda repos to rename output files
differently so that they can coexists in the same image.
Additionally, the server will need to accept a parameter to know the variant so
that it can use the proper `registers` and `config` files.
The selection of the flavor can be automated in the server start script based on
information coming from the I2C bus. An alternative way to specify the flavor is
by setting a variable in the `config.txt`.

### How do we upgrade an old panda?
- Offline upgrade: we just repartition the SD card to use all the space and copy
the 5 required files.
- Live upgrade: we replace the initramfs in /boot with the new one and copy the
  5 required files to /opt/, then reboot, the init script in initramfs
  if it doesn't have all the required files, will try to mount /opt, copy the
  files to RAM, repartition to use the full SD card in one fat32 partition,
  then copy the files back.
  For convenience to allow upgrading from the web interface, we package the 5
  required files and a service script in a zpkg (this could be our default
  release format to support this upgrade process always), the service script
  will do the steps described earlier.

### Error `operation not supported error, dev loop1, sector 1608 op 0x9:(WRITE_ZEROES) flags 0x800 phys_seg 0 prio class 2`
This error appears when the state.ext4 image is created, apparently some
initialization does the WRITE_ZEROES operation which the underlying vfat
filesystem doesn't support, the system continues running ... so, it probably did
the operation in the slow way and then this message doesn't appear anymore. More
 investigation is needed to confirm this is fine.
 
 A workaround was found, mkfs.ext4 with options `lazy_journal_init=0` and
 `lazy_itable_init=0`. This initialization is the only thing that seems to need
 the `WRITE_ZEROES` operation.
