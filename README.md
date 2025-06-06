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
  packages including all the flavors of FMCs.
- `state.ext4`: Initially it doesn't exist and it is the created (empty) by the
init script.

Having all the packages in `packages.ext4` is better to ensure that all the
parts are consistent with each other.

## Initramfs init script
The init script mounts the images:
```bash
mkdir /panda /state
mount -t ext4 -o ro /boot/packages.ext4 /packages
mount -t ext4 -o rw /boot/state.ext4 /state
```

Then mount an overlayfs using packages as lower layers(immutable) and state as
upper layer:
```bash
mkdir /merge /work
mount -t overlayfs -o lowerdir=/packages,upperdir=/state,workdir=/work overlayfs /merge
```

Finally, use that as root and exec init system(possibly systemd?)
```bash
# the overlayfs mounted directory becomes the root
mount -n --move /merge /
exec /sbin/init
```

The result is a system that will merge the content of packages and state but any
write will be changing only `state.ext4` (e.g. when installing packages), any
messed up system can be recovered by deleting `state.ext4`.

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

## Notes
- If while the init script tries to create `state.ext4` it runs out of space, it
  could be interesting repartitioning to fully use the SD card if it helps.
