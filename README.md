# Core22 snap for snapd & Ubuntu Core

This is a base snap for snapd & Ubuntu Core that is based on Ubuntu 22.04

# Building locally

To build this snap locally you need snapcraft. The project must be built as real root.

For i386 and amd64
```
$ sudo snapcraft
```

For any other architecture we recommend remote-build as multipass has limited
support for cross-building, and lack of stable releases for some architectures. 
To use remote-build you need to have a launchpad account, and follow the instructions [here](https://snapcraft.io/docs/remote-build)
```
$ sudo snapcraft remote-build --build-on={arm64,armhf,ppc64el,s390x}
```

# Testing with spread

## Prerequisites for testing

You need to have the following software installed before you can test with spread
 - Go (https://golang.org/doc/install or ```sudo snap install go```)
 - Spread (install from source as per below)

## Installing spread

You can install spread by simply using ```snap install spread```, however this does not allow for the lxd-backend to be used.
To use the lxd backend you need to install spread from source, as the LXD profile support has not been upstreamed yet.
This document will be updated with the upstream version when this happens. To install spread from source you need to do the following.

```
git clone https://github.com/Meulengracht/spread
cd spread
cd cmd/spread
go build
go install
```

## QEmu backend

1. Install the dependencies required for the qemu emulation
```
sudo apt update && sudo apt install -y qemu-kvm autopkgtest
```
2. Create a suitable ubuntu test image (focal) in the following directory where spread locates images. Note that the location is different when using spread installed through snap.
```
mkdir -p ~/.spread/qemu # This location is different if you installed spread from snap
cd ~/.spread/qemu
autopkgtest-buildvm-ubuntu-cloud -r focal
```
3. Rename the newly built image as the name will not match what spread is expecting
```
mv autopkgtest-focal-amd64.img ubuntu-20.04-64.img
```
4. Now you are ready to run spread tests with the qemu backend
```
cd ~/core22 # or wherever you checked out this repository
spread qemu-nested
```

## LXD backend
The LXD backend is the preffered way of testing locally as it uses virtualization and thus runs a lot quicker than
the qemu backend. This is because the container can use all the resources of the host, and we can support
qemu-kvm acceleration in the container for the nested instance.

This backend requires that your host machine supports KVM.

1. Setup any prerequisites and build the LXD image needed for testing. The following commands will install lxd
and yq (needed for yaml manipulation), download the newest image and import it into LXD.
```
sudo snap install lxd
sudo snap install yq
curl -o lxd-core22-img.tar.gz https://storage.googleapis.com/snapd-spread-core/lxd/lxd-spread-core22-img.tar.gz
lxc image import lxd-core22-img.tar.gz --alias ucspread22
lxc image show ucspread22 > temp.profile
yq e '.properties.aliases = "ucspread22,amd64"' -i ./temp.profile
yq e '.properties.remote = "images"' -i ./temp.profile
cat ./temp.profile | lxc image edit ucspread22
rm ./temp.profile ./lxd-core22-img.tar.gz
```
2. Import the LXD core22 test profile. Make sure your working directory is the root of this repository.
```
lxc profile create core22
cat tests/spread/core22.lxdprofile | lxc profile edit core22
```
3. Set environment variable to enable KVM acceleration for the nested qemu instance
```
export SPREAD_ENABLE_KVM=true
```
4. Now you can run the spread tests using the LXD backend
```
spread lxd-nested
```

# Writing code

The usual way to add functionality is to write a shell script hook
with the `.chroot` extenstion under the `hooks/` directory. These hooks
are run inside the base image filesystem.

Each hook should have a matching `.test` file in the `hook-tests`
directory. Those tests files are run relative to the base image
filesystem and should validates that the coresponding `.chroot` file
worked as expected.

The `.test` scripts will be run after building with snapcraft or when
doing a manual "make test" in the source tree.

# Bootchart

It is possible to enable bootcharts by adding `core.bootchart` to the
kernel command line. The sample collector will run until the system is
seeded (it will stop when the `snapd.seeded.service` stops). The
bootchart will be saved in the `ubuntu-data` partition, under
`/var/log/debug/boot<N>/`, `<N>` being the boot number since
bootcharts were enabled. If a chart has been collected by the
initramfs, it will be also saved in that folder.

**TODO** In the future, we would want `systemd-bootchart` to be started
only from the initramfs and have just one bootchart per boot. However,
this is currently not possible as `systemd-bootchart` needs some changes
so it can survive the switch root between initramfs and data
partition. With those changes, we could also have `systemd-bootchart` as
init process so we get an even more accurate picture.


# Writable paths

In Ubuntu Core some paths are bind mounted from `/writable`.

In order to add a new path you need to
 * Add an fstab entry in static/etc/fstab
 * Add a tmpfiles.d entry in static/usr/lib/tmpfiles.d/core-writable.conf
 * Eventually install template directory or file into /usr/share/factory

## Entry in fstab

If your mount is for a path within `/etc`, then you need t make sure
that prefix of the source path uses `/run/mnt/data` rather than
`/writable`. You also need to make sure that the directory or empty
file exists in the file system.

For aother paths (other than `/etc`), just bind mount
`/writable/system-data/some/path` to `/some/path`.

## Entry in tmpfiles.d

### Without template data

If you are binding a file that starts empty, add a `f` entry. For
example:
```
f /writable/system-data/var/lib/mydata
```

If you are binding a directory that starts empty, add a `d` entry.
For example:
```
d /writable/system-data/var/lib/mydir
```

### With template data

If you are binding a file or a directory that is initialized by a
template, then add a `C` entry.
```
C /writable/system-data/var/lib/mydir
```

You also need to install the initial data in `/usr/share/factory`.  In
the previous example, data has to be installed in
`/usr/share/factory/writable/system-data/var/lib/mydir`.

If the data is already installed in the sysroot. In the previous example
it would be installed in `/var/lib/mydir`. The you can use hook
`993-factory-files.chroot` to copy the files. Just add the path
(in this case `/var/lib/mydir`) in the `dirs` variable.
