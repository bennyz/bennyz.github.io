+++
title = "Providing Conainer Images to Firecracker with nbdkit"
date = 2022-06-26
+++


Important note before I begin: Unforunately, this is not really practical for actual use since currently the OCI image format specifies that layers have to be in `tar` format, and a tar archive cannot be randomly accessed. But I decided to procceed anyway since this looked like a good oppurtunity to learn something new, like using `initrd`.

I had this idea to start [Firecracker micro VMs](https://firecracker-microvm.github.io/) with OCI images, without pulling it with, say, `docker pull`, but just by providing the layer's digest. 

## Setting Up the Docker Registry

Firecracker requires a root file system and a kernel image. I can use any supported kernel image in this exercise. To provide the root file system some work is going to be required.
To start I setup a local registry to avoid having to deal with authentication and rate limits, this is as simple as:

```shell
$ docker run -d -p 5000:5000 --name registry registry:2
eba0f7d77509a9d9d966df18fa33231250e1b4f2db6673e1273642414d9e9788
```
Let's do a small sanity check:

```shell
$ curl http://localhost:5000/v2/_catalog
{"repositories":[]}
```

## Preparing a Container Image

Since containers only run a single program they lack an init process. To add it, I will modify an existing image:

```shell
$ docker run -it python bash
root@762a0427aa13:/# apt-get update
root@762a0427aa13:/# apt-get install systemd # systemd will be the init process

# Setup the tty
root@762a0427aa13:/# ln -s agetty /etc/init.d/agetty.ttyS0
root@762a0427aa13:/# echo ttyS0 > /etc/securetty
root@762a0427aa13:/# systemctl enable getty@ttyS0
# Set root password
root@762a0427aa13:/# passwd root
```

I made changes to the container, now I am going to export it, and import it back to upload a flat image to the registry:

```shell
$ docker export --output=python.tar 762a0427aa13

# And import it back as flat-python:latest
$ docker import python.tar flat-python:latest
sha256:67d16081116dc3bf5fca2f633a4104bffedc785cb53069710a1e052001ca65f1
```

Now, upload it to the registry:

```shell
$ docker tag flat-python:latest localhost:5000/flat-python
$ docker push localhost:5000/flat-python
Using default tag: latest
The push refers to repository [localhost:5000/flat-python]
ce2b37ca4342: Pushed
latest: digest: sha256:f16fbcc2845c385cef37211bf6a3587ca3ad390ad362cf67f56119592e874e83 size: 529
```

## Exposing the Root Filesystem via NBD

So now I have a suitable OCI image layer to use as the root file system. I can expose it using `nbdkit`:

```shell
# To find out the url to provide `nbdkit`'s curl plugin, I'll query the manifest and because I flattened to image there is only one layer:
$ curl -s localhost:5000/v2/flat-python/manifests/latest | jq '.fsLayers[].blobSum
"sha256:cba57c55ee0f84a7d37c79d0524ff29be2856e7844430f8e212d87bb4a87e837"

# Start nbdkit with the curl plugin, sending it through a gzip filter because in this case it's also gzipped:
$ nbdkit -r curl http://localhost:5000/v2/flat-python/blobs/sha256:cba57c55ee0f84a7d37c79d0524ff29be2856e7844430f8e212d87bb4a87e837 --filter=gzip

# Sanity check
nbdinfo nbd://localhost
protocol: newstyle-fixed without TLS
export="":
        export-size: 969963008
        content: POSIX tar archive
        uri: nbd://localhost:10809/
        contexts:
                base:allocation
        is_rotational: false
        is_read_only: true
        can_cache: true
        can_df: true
        can_fast_zero: false
        can_flush: false
        can_fua: false
        can_multi_conn: true
        can_trim: false
        can_zero: false
```

And now I have the `tar`ed root file system exposed via an NBD server, I will map it to a device because the "nbd://" URI scheme is not supported:

```shell
# Load the nbd kernel module
$ sudo modprobe nbd

# Map the server
$ sudo nbd-client localhost 10809 /dev/nbd0

# Give myself permissions:
sudo chown ${USER} /dev/nbd0
```

Now I can use `/dev/nbd0` as the path of the drive in Firecracker's configuration. However, it will not boot because the path is a `tar` archive, so I need to somehow extract the filesystem inside this `tar`.

## In Comes initrd

Not too long ago, support for [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk) was added to Firecracker. By using an `initrd`, I can do things before the kernel loads the real root filesystem that is archived in a `tar` on `/dev/nbd0`.
What I essentially do is, I use the `/dev/nbd0` device which is now attached to Firecracker, containing the `tar`ed root filesystem, and extract it. Then it can be used to load the system as it's an actual filesystem.

Let's start by creating a work directory and populate it with all the goodness needed by initrd:

```shell
mkdir /tmp/initrd
mkdir -p bin dev etc home mnt proc sys tmp usr
```

Now I have the basic layout of the initrd, and I need to create the `init` script that will be executed to load the real root filesystem:

```shell
pushd /tmp/initrd

# Put the script in an init file
cat >> init << EOF
#!/bin/busybox sh

mount -t devtmpfs  devtmpfs  /dev
mount -t proc      proc      /proc
mount -t sysfs     sysfs     /sys
mount -t tmpfs     tmpfs     /tmp

# Create loop device backed by a file
truncate -s 2G fs
# Why loop5? I don't know
losetup /dev/loop5 fs
mkfs.ext2 /dev/loop5

# Extract tar file exposed via /dev/vda to temporary directory
mkdir mnt_fs
mount -o loop /dev/loop5 mnt_fs
tar -C mnt_fs -xf /dev/vda
umount /mnt_fs

# Mount extract rootfs
mkdir new_root
mount -t ext2 /dev/loop5 /new_root

# Cleanup
umount /proc
umount /sys
umount /tmp
umount /dev

# Switch root
exec switch_root /new_root /sbin/init
EOF

chmod +x init
```

This script uses commands like `mount` and `truncate`, all of these need to be made available in the final initrd, this can be done using BusyBox, which provides multiple utilities in a single file.

```shell
$ pushd /tmp/initrd/bin

$ curl -s https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox > busybox

# Make it executable
$ chmod +x busybox

# Now I can create links for all the utilities I use in the init script above:
$ ln -s busybox mount
$ ln -s busybox truncate
$ ln -s busybox losetup
$ ln -s busybox mkdir
$ ln -s busybox mkfs.ext2
$ ln -s busybox tar
$ ln -s busybox umount
$ ln -s busybox switch_root

$ popd
```

And now I create the initrd by packing everything into a `cpio` archive

```shell
$ find . -print0 | cpio --null --create --verbose --format=newc > initrd.cpio
```

The full script can be found [here](https://gist.github.com/bennyz/b96f01b7f7c447927502b48b22051c26)

Now I have the `initrd` ready, I can finally create the micro VM, I will use a configuration file:

```shell
# Notes: hello-vmlinux.bin is the official example kernel, taken from:
# https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md
$ cat vmconfig-initrd.json
{
  "boot-source": {
    "kernel_image_path": "hello-vmlinux.bin",
    "boot_args": "console=ttyS0 reboot=k panic=1 pci=off",
    "initrd_path": "/tmp/initrd/initrd.cpio"
  },
  "drives": [
    {
      "drive_id": "rootfs",
      "path_on_host": "/dev/nbd0",
      "is_root_device": false,
      "is_read_only": false
    }
  ],
  "machine-config": {
    "vcpu_count": 2,
    "mem_size_mib": 4500
  }
}
```

Since initrd is run in-memory, there needs to be enough of it to do the `tar` extraction. Another important note, is that the rootfs drive should be marked as `is_root_device: false`, because Firecracker passes the initrd as the root filesystem device.

Note:
The `init` script creates a loop block device on which an ext2 filesystem is created. The `tar` I use contains an ext4 filesystem, but it still works. BusyBox doesn't come with newer filesystems, likely on purpose, so if I wanted to use ext4 I would have to provide `e2fsprogs` myself.

## Run the Thing

Now the micro VM can be started with:

```shell
 $ ./firecracker --no-api --config-file vmconfig-initrd.json
```

And voilà!
[![asciicast](https://asciinema.org/a/PSrhDUbTVj8pPIzeTupuKW76f.svg)](https://asciinema.org/a/PSrhDUbTVj8pPIzeTupuKW76f)

