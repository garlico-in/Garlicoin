Gitian building
================

*Setup instructions for a Gitian build of Garlicoin Core using a Debian VM or physical system.*

Gitian is the deterministic build process that is used to build the Garlicoin
Core executables. It provides a way to be reasonably sure that the
executables are really built from the source on GitHub. It also makes sure that
the same, tested dependencies are used and statically built into the executable.

Multiple developers build the source code by following a specific descriptor
("recipe"), cryptographically sign the result, and upload the resulting signature.
These results are compared and only if they match, the build is accepted and uploaded
to garlicoin.io.

More independent Gitian builders are needed, which is why this guide exists.
It is preferred you follow these steps yourself instead of using someone else's
VM image to avoid 'contaminating' the build.

Table of Contents
------------------

- [Installing Gitian](#installing-gitian)
- [Setting up the Gitian image](#setting-up-the-gitian-image)
- [Getting and building the inputs](#getting-and-building-the-inputs)
- [Building Garlicoin Core](#building-garlicoin-core)
- [Building an alternative repository](#building-an-alternative-repository)
- [Signing externally](#signing-externally)
- [Uploading signatures](#uploading-signatures)

Preparing the Gitian builder host
---------------------------------

The first step is to prepare the host environment that will be used to perform the Gitian builds.
This guide explains how to set up the environment, and how to start the builds.

Debian Linux was chosen as the host distribution because it has a lightweight install (in contrast to Ubuntu) and is readily available.
Any kind of virtualization can be used, for example:
- [Docker](https://www.docker.com/) (covered by this guide)

```bash
curl -sL https://get.docker.com | sh
```

Installing Gitian
------------------

Clone the git repositories for garlicoin and Gitian.

```bash
git clone https://github.com/devrandom/gitian-builder.git
git clone https://github.com/GarlicoinOrg/Garlicoin garlicoin
git clone https://github.com/GarlicoinOrg/gitian.sigs.grlc.git
```

Setting up the Gitian image
-------------------------

Gitian needs a virtual image of the operating system to build in.
Currently this is Ubuntu Trusty x86_64.
This image will be copied and used every time that a build is started to
make sure that the build is deterministic.
Creating the image will take a while, but only has to be done once.

Execute the following as user `debian`:

```bash
export USE_DOCKER=1
cd gitian-builder
bin/make-base-vm --arch amd64 --suite bionic
bin/make-base-vm --arch amd64 --suite trusty
```

There will be a lot of warnings printed during the build of the image. These can be ignored.

**Note**: When sudo asks for a password, enter the password for the user *debian* not for *root*.

Getting and building the inputs
--------------------------------

At this point you have two options, you can either use the automated script (found in [contrib/gitian-build.sh](/contrib/gitian-build.sh)) or you could manually do everything by following this guide. If you're using the automated script, then run it with the "--setup" command. Afterwards, run it with the "--build" command (example: "contrib/gitian-build.sh -b signer 0.13.0"). Otherwise ignore this.

Follow the instructions in [doc/release-process.md](release-process.md#fetch-and-create-inputs-first-time-or-when-dependency-versions-change)
in the garlicoin repository under 'Fetch and create inputs' to install sources which require
manual intervention. Also optionally follow the next step: 'Seed the Gitian sources cache
and offline git repositories' which will fetch the remaining files required for building
offline.

Building Garlicoin Core
----------------

To build Garlicoin Core (for Linux, OS X and Windows) just follow the steps under 'perform
Gitian builds' in [doc/release-process.md](release-process.md#perform-gitian-builds) in the garlicoin repository.

This may take some time as it will build all the dependencies needed for each descriptor.
These dependencies will be cached after a successful build to avoid rebuilding them when possible.

At any time you can check the package installation and build progress with

```bash
tail -f var/install.log
tail -f var/build.log
```

Output from `gbuild` will look something like

    Initialized empty Git repository in /home/debian/gitian-builder/inputs/garlicoin/.git/
    remote: Counting objects: 57959, done.
    remote: Total 57959 (delta 0), reused 0 (delta 0), pack-reused 57958
    Receiving objects: 100% (57959/57959), 53.76 MiB | 484.00 KiB/s, done.
    Resolving deltas: 100% (41590/41590), done.
    From https://github.com/GarlicoinOrg/Garlicoin
    ... (new tags, new branch etc)
    --- Building for trusty amd64 ---
    Stopping target if it is up
    Making a new image copy
    stdin: is not a tty
    Starting target
    Checking if target is up
    Preparing build environment
    Updating apt-get repository (log in var/install.log)
    Installing additional packages (log in var/install.log)
    Grabbing package manifest
    stdin: is not a tty
    Creating build script (var/build-script)
    lxc-start: Connection refused - inotify event with no name (mask 32768)
    Running build script (log in var/build.log)

Building an alternative repository
-----------------------------------

If you want to do a test build of a pull on GitHub it can be useful to point
the Gitian builder at an alternative repository, using the same descriptors
and inputs.

For example:
```bash
URL=https://github.com/GarlicoinOrg/Garlicoin.git
COMMIT=master
./bin/gbuild -j 30 -m 80000 --commit garlicoin=${COMMIT} --url garlicoin=${URL} ../garlicoin/contrib/gitian-descriptors/gitian-linux.yml
./bin/gbuild -j 30 -m 80000 --commit garlicoin=${COMMIT} --url garlicoin=${URL} ../garlicoin/contrib/gitian-descriptors/gitian-win.yml
./bin/gbuild -j 30 -m 80000 --commit garlicoin=${COMMIT} --url garlicoin=${URL} ../garlicoin/contrib/gitian-descriptors/gitian-osx.yml
```

Building fully offline
-----------------------

For building fully offline including attaching signatures to unsigned builds, the detached-sigs repository
and the garlicoin git repository with the desired tag must both be available locally, and then gbuild must be
told where to find them. It also requires an apt-cacher-ng which is fully-populated but set to offline mode, or
manually disabling gitian-builder's use of apt-get to update the VM build environment.

To configure apt-cacher-ng as an offline cacher, you will need to first populate its cache with the relevant
files. You must additionally patch target-bin/bootstrap-fixup to set its apt sources to something other than
plain archive.ubuntu.com: us.archive.ubuntu.com works.

So, if you use LXC:

```bash
export PATH="$PATH":/path/to/gitian-builder/libexec
export USE_DOCKER=1
cd /path/to/gitian-builder
./libexec/make-clean-vm --suite bionic --arch amd64
./libexec/make-clean-vm --suite trusty --arch amd64

on-target -u root apt-get update
on-target -u root \
  -e DEBIAN_FRONTEND=noninteractive apt-get --no-install-recommends -y install \
  $( sed -ne '/^packages:/,/[^-] .*/ {/^- .*/{s/"//g;s/- //;p}}' ../garlicoin/contrib/gitian-descriptors/*|sort|uniq )
on-target -u root apt-get -q -y purge grub
on-target -u root -e DEBIAN_FRONTEND=noninteractive apt-get -y dist-upgrade
```

And then set offline mode for apt-cacher-ng:

```
/etc/apt-cacher-ng/acng.conf
[...]
Offlinemode: 1
[...]

service apt-cacher-ng restart
```

Then when building, override the remote URLs that gbuild would otherwise pull from the Gitian descriptors::
```bash

cd /some/root/path/
git clone https://github.com/GarlicoinOrg/Garlicoin-detached-sigs.git

BTCPATH=/some/root/path/garlicoin
SIGPATH=/some/root/path/garlicoin-detached-sigs

./bin/gbuild --url garlicoin=${BTCPATH},signature=${SIGPATH} ../garlicoin/contrib/gitian-descriptors/gitian-win-signer.yml
```

Signing externally
-------------------

If you want to do the PGP signing on another device, that's also possible; just define `SIGNER` as mentioned
and follow the steps in the build process as normal.

    gpg: skipped "laanwj": secret key not available

When you execute `gsign` you will get an error from GPG, which can be ignored. Copy the resulting `.assert` files
in `gitian.sigs` to your signing machine and do

```bash
    gpg --detach-sign ${VERSION}-linux/${SIGNER}/garlicoin-linux-build.assert
    gpg --detach-sign ${VERSION}-win/${SIGNER}/garlicoin-win-build.assert
    gpg --detach-sign ${VERSION}-osx-unsigned/${SIGNER}/garlicoin-osx-build.assert
```

This will create the `.sig` files that can be committed together with the `.assert` files to assert your
Gitian build.

Uploading signatures
---------------------

After building and signing you can push your signatures (both the `.assert` and `.assert.sig` files) to the
[garlicoin-project/gitian.sigs.ltc](https://github.com/garlicoin-project/gitian.sigs.ltc/) repository, or if that's not possible create a pull
request. You can also mail the files to thrasher (thrasher@addictionsofware.com) and he will commit them.
