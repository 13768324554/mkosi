# mkosi - Create legacy-free OS images

A fancy wrapper around `dnf --installroot`, `debootstrap`,
`pacstrap` and `zypper` that may generate disk images with a number of
bells and whistles.

# Supported output formats

The following output formats are supported:

* Raw *GPT* disk image, with ext4 as root (*raw_ext4*)

* Raw *GPT* disk image, with btrfs as root (*raw_btrfs*)

* Raw *GPT* disk image, with xfs as root (*raw_xfs*)

* Raw *GPT* disk image, with squashfs as read-only root (*raw_squashfs*)

* Plain directory, containing the *OS* tree (*directory*)

* btrfs subvolume, with separate subvolumes for `/var`, `/home`,
  `/srv`, `/var/tmp` (*subvolume*)

* Tarball (*tar*)

When a *GPT* disk image is created, the following additional
options are available:

* A swap partition may be added in

* The image may be made bootable on *EFI* systems

* Separate partitions for `/srv` and `/home` may be added in

* The root, /srv and /home partitions may optionally be encrypted with
  LUKS.

* A dm-verity partition may be added in that adds runtime integrity
  data for the root partition

# Compatibility

Generated images are *legacy-free*. This means only *GPT* disk
labels (and no *MBR* disk labels) are supported, and only
systemd based images may be generated. Moreover, for bootable
images only *EFI* systems are supported (not plain *MBR/BIOS*).

All generated *GPT* disk images may be booted in a local
container directly with:

```bash
systemd-nspawn -bi image.raw
```

Additionally, bootable *GPT* disk images (as created with the
`--bootable` flag) work when booted directly by *EFI* systems, for
example in *KVM* via:

```bash
qemu-kvm -m 512 -smp 2 -bios /usr/share/edk2/ovmf/OVMF_CODE.fd -drive format=raw,file=image.raw
```

*EFI* bootable *GPT* images are larger than plain *GPT* images, as
they additionally carry an *EFI* system partition containing a
boot loader, as well as a kernel, kernel modules, udev and
more.

All directory or btrfs subvolume images may be booted directly
with:

```bash
systemd-nspawn -bD image
```

# Other features

* Optionally, create an *SHA256SUMS* checksum file for the result,
  possibly even signed via gpg.

* Optionally, place a specific `.nspawn` settings file along
  with the result.

* Optionally, build a local project's *source* tree in the image
  and add the result to the generated image (see below).

* Optionally, share *RPM*/*DEB* package cache between multiple runs,
  in order to optimize build speeds.

* Optionally, the resulting image may be compressed with *XZ*.

* Optionally, btrfs' read-only flag for the root subvolume may be
  set.

* Optionally, btrfs' compression may be enabled for all
  created subvolumes.

* By default images are created without all files marked as
  documentation in the packages, on distributions where the
  package manager supports this. Use the `--with-docs` flag to
  build an image with docs added.

# Supported distributions

Images may be created containing installations of the
following *OS*es.

* *Fedora*

* *Debian*

* *Ubuntu*

* *Arch Linux*

* *openSUSE*

* *Mageia*

* *CentOS*

* *Clear Linux*

In theory, any distribution may be used on the host for building
images containing any other distribution, as long as the necessary
tools are available. Specifically, any distro that packages
`debootstrap` may be used to build *Debian* or *Ubuntu* images. Any
distro that packages `dnf` may be used to build *Fedora* or *Mageia*
images. Any distro that packages `pacstrap` may be used to build *Arch
Linux* images. Any distro that packages `zypper` may be used to build
*openSUSE* images. Any distro that packages `yum` (or the newer
replacement `dnf`) may be used to build *CentOS* images.

Currently, *Fedora* packages all relevant tools as of Fedora 26.

# Files

To make it easy to build images for development versions of
your projects, mkosi can read configuration data from the
local directory, under the assumption that it is invoked from
a *source* tree. Specifically, the following files are used if
they exist in the local directory:

* `mkosi.default` may be used to configure mkosi's image
  building process. For example, you may configure the
  distribution to use (`fedora`, `ubuntu`, `debian`, `archlinux`,
  `opensuse`, `mageia`) for the image, or additional
  distribution packages to install. Note that all options encoded
  in this configuration file may also be set on the command line,
  and this file is hence little more than a way to make sure simply
  typing `mkosi` without further parameters in your *source* tree is
  enough to get the right image of your choice set up.
  Additionally if a `mkosi.default.d` directory exists, each file in it
  is loaded in the same manner adding/overriding the values specified in
  `mkosi.default`. Command-line arguments, as shown in the help
  description, have to be included in a configuration block (e.g.
  "[Packages]") corresponding to the argument group (e.g. "Packages"),
  and the argument gets converted as follows: "--with-network" becomes
  "WithNetwork=yes".

* `mkosi.extra/` or `mkosi.extra.tar` may be respectively a directory
  or archive. If any exist all files contained in it are copied over the
  directory tree of the image after the *OS* was installed. This may be used to
  add in additional files to an image, on top of what the distribution includes
  in its packages. When using a directory file ownership is not preserved:
  all files copied will be owned by root. To preserve ownership use a tar
  archive.

* `mkosi.skeleton/` or `mkosi.skeleton.tar` may be respectively a directory
  or archive, and they work in the same way as
  `mkosi.extra`/`mkosi.skeleton.tar`. However the files are copied before
  anything else so to have a skeleton tree for the OS. This allows to change
  the package manager and create files that need to be there before anything is
  installed. When using a directory file ownership is not preserved:
  all files copied will be owned by root. To preserve ownership use a tar
  archive.

* `mkosi.build` may be an executable script. If it exists the image
  will be built twice: the first iteration will be the *development*
  image, the second iteration will be the *final* image. The
  *development* image is used to build the project in the current
  working directory (the *source* tree). For that the whole directory
  is copied into the image, along with the mkosi.build build
  script. The script is then invoked inside the image (via
  `systemd-nspawn`), with `$SRCDIR` pointing to the *source*
  tree. `$DESTDIR` points to a directory where the script should place
  any files generated it would like to end up in the *final*
  image. Note that `make`/`automake` based build systems generally
  honour `$DESTDIR`, thus making it very natural to build *source*
  trees from the build script. After the *development* image was built
  and the build script ran inside of it, it is removed again. After
  that the *final* image is built, without any *source* tree or build
  script copied in. However, this time the contents of `$DESTDIR` are
  added into the image.

  When the source tree is copied into the *build* image, all files are
  copied, except for `mkosi.builddir/`, `mkosi.cache/` and
  `mkosi.output/`. That said, `.gitignore` is respected if the source
  tree is a `git` checkout. If multiple different images shall be
  built from the same source tree it's essential to exclude their
  output files from this copy operation, as otherwise a version of an
  image built earlier might be included in a later build, which is
  usually not intended. An alternative to excluding these built images
  via `.gitignore` entries is making use of the `mkosi.output/`
  directory (see below), which is an easy way to exclude all build
  artifacts.

* `mkosi.postinst` may be an executable script. If it exists it is
  invoked as last step of preparing an image, from within the image
  context. It is once called for the *development* image (if this is
  enabled, see above) with the "build" command line parameter, right
  before invoking the build script. It is called a second time for the
  *final* image with the "final" command line parameter, right before
  the image is considered complete. This script may be used to alter
  the images without any restrictions, after all software packages and
  built sources have been installed. Note that this script is executed
  directly in the image context with the final root directory in
  place, without any `$SRCDIR`/`$DESTDIR` setup.

* `mkosi.nspawn` may be an nspawn settings file. If this exists
  it will be copied into the same place as the output image
  file. This is useful since nspawn looks for settings files
  next to image files it boots, for additional container
  runtime settings.

* `mkosi.cache/` may be a directory. If so, it is automatically used as
  package download cache, in order to speed repeated runs of the tool.

* `mkosi.builddir/` may be a directory. If so, it is automatically
  used as out-of-tree build directory, if the build commands in the
  `mkosi.build` script support it. Specifically, this directory will
  be mounted into the build countainer, and the `$BUILDDIR`
  environment variable will be set to it when the build script is
  invoked. The build script may then use this directory as build
  directory, for automake-style or ninja-style out-of-tree
  builds. This speeds up builds considerably, in particular when
  `mkosi` is used in incremental mode (`-i`): not only the disk images
  but also the build tree is reused between subsequent
  invocations. Note that if this directory does not exist the
  `$BUILDDIR` environment variable is not set, and it is up to build
  script to decide whether to do in in-tree or an out-of-tree build,
  and which build directory to use.

* `mkosi.rootpw` may be a file containing the password for the root
  user of the image to set. The password may optionally be followed by
  a newline character which is implicitly removed. The file must have
  an access mode of 0600 or less. If this file does not exist the
  distribution's default root password is set (which usually means
  access to the root user is blocked).

* `mkosi.passphrase` may be a passphrase file to use when LUKS
  encryption is selected. It should contain the passphrase literally,
  and not end in a newline character (i.e. in the same format as
  cryptsetup and /etc/crypttab expect the passphrase files). The file
  must have an access mode of 0600 or less. If this file does not
  exist and encryption is requested the user is queried instead.

* `mkosi.secure-boot.crt` and `mkosi.secure-boot.key` may contain an
  X509 certificate and PEM private key to use when UEFI SecureBoot
  support is enabled. All EFI binaries included in the image's ESP are
  signed with this key, as a late step in the build process.

* `mkosi.output/` may be a directory. If it exists, and the image
  output path is not configured (i.e. no `--output=` setting
  specified), or configured to a filename (i.e. a path containing no
  `/` character) all build artifacts (that is: the image itself, the
  root hash file in case Verity is used, the checksum and its
  signature if that's enabled, and the nspawn settings file if there
  is any) are placed in this directory. Note that this directory is
  not used if the image output path contains at least one slash, and
  has no effect in that case. This setting is particularly useful if
  multiple different images shall be built from the same working
  directory, as otherwise the build result of a preceeding run might
  be copied into a build image as part of the source tree (see above).

All these files are optional.

Note that the location of all these files may also be
configured during invocation via command line switches, and as
settings in `mkosi.default`, in case the default settings are
not acceptable for a project.

# Examples

Create and run a raw *GPT* image with *ext4*, as `image.raw`:

```bash
# mkosi
# systemd-nspawn -b -i image.raw
```

Create and run a bootable btrfs *GPT* image, as `foobar.raw`:

```bash
# mkosi -t raw_btrfs --bootable -o foobar.raw
# systemd-nspawn -b -i foobar.raw
# qemu-kvm -m 512 -smp 2 -bios /usr/share/edk2/ovmf/OVMF_CODE.fd -drive format=raw,file=foobar.raw
```

Create and run a *Fedora* image into a plain directory:

```bash
# mkosi -d fedora -t directory -o quux
# systemd-nspawn -b -D quux
```

Create a compressed image `image.raw.xz` and add a checksum file, and
install *SSH* into it:

```bash
# mkosi -d fedora -t raw_squashfs --checksum --xz --package=openssh-clients
```

Inside the source directory of an `automake`-based project,
configure *mkosi* so that simply invoking `mkosi` without any
parameters builds an *OS* image containing a built version of
the project in its current state:

```bash
# cat > mkosi.default <<EOF
[Distribution]
Distribution=fedora
Release=24

[Output]
Format=raw_btrfs
Bootable=yes

[Packages]
Packages=openssh-clients httpd
BuildPackages=make gcc libcurl-devel
EOF
# cat > mkosi.build <<EOF
#!/bin/sh
cd $SRCDIR
./autogen.sh
./configure --prefix=/usr
make -j `nproc`
make install
EOF
# chmod +x mkosi.build
# mkosi
# systemd-nspawn -bi image.raw
```

To create a *Fedora* image with hostname:
```bash
# mkosi -d fedora --hostname image
```

Also you could set hostname in configuration file:
```bash
# cat mkosi.default
...
[Output]
Hostname=image
...
```

# Requirements

mkosi is packaged for various distributions: Debian, Ubuntu, Arch (in AUR), Fedora.
It is usually easiest to use the distribution package.

The current version requires systemd 233 (or actually, systemd-nspawn of it).

When not using distribution packages make sure to install the
necessary dependencies. For example, on *Fedora* you need:

```bash
dnf install arch-install-scripts btrfs-progs debootstrap dosfstools edk2-ovmf e2fsprogs squashfs-tools gnupg python3 tar veritysetup xfsprogs xz zypper
```

On Debian/Ubuntu it might be necessary to install the `ubuntu-keyring`,
`ubuntu-archive-keyring` and/or `debian-archive-keyring` packages explicitly,
in addition to `debootstrap`, depending on what kind of distribution images
you want to build. `debootstrap` on Debian only pulls in the Debian keyring
on its own, and the version on Ubuntu only the one from Ubuntu.

Note that the minimum required Python version is 3.5.

If SecureBoot signing is to be used, then the "sbsign" tool needs to
be installed as well, which is currently not available on Fedora, but
in a COPR repository:

```bash

dnf copr enable msekleta/sbsigntool
dnf install sbsigntool
```
# References

* [Primary mkosi git repository on GitHub](https://github.com/systemd/mkosi/)
* [mkosi — A Tool for Generating OS Images](http://0pointer.net/blog/mkosi-a-tool-for-generating-os-images.html) indroductory blog post by Lennart Poettering
* [The mkosi OS generation tool](https://lwn.net/Articles/726655/) story on LWN
