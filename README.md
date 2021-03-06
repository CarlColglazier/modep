![modep-intro](https://raw.githubusercontent.com/wiki/BlokasLabs/pisound-docs/images/modep-intro.PNG)

MODEP is an open-source, community-based [MOD DUO](https://www.moddevices.com/) emulator that lets you play around with hundreds of LV2 audio plugins ranging from a simple reverb to a complex FM synth using your Raspberry Pi and [Pisound](https://blokas.io/pisound) or any other [Raspberry Pi](https://www.raspberrypi.org/) supported sound card!

MOD DUO team has done a remarkable job for the whole Linux audio open-source community, so if you like this emulator you should go get the real thing!

You can find more information about MODEP on [our website](https://blokas.io/modep) and [documentation page](https://blokas.io/modep/docs).

# Found a Bug?

If you're here because you want to report an issue in MODEP, please visit [our community forum](https://community.blokas.io/c/modep) or open an issue in this repository.

# Contributing

Pull-requests are more than welcome! You can find a complete MODEP image build instructions bellow. If you want to add an additional plugin, follow [these instructions](https://blokas.io/modep/docs/Contribution-Guide/) and here is the list of all related repositories:

- UI software for MODEP - https://github.com/BlokasLabs/mod-ui
- Jack LV2 host for MODEP - https://github.com/BlokasLabs/mod-host
- Pisound Button scripts for MODEP - https://github.com/BlokasLabs/modep-btn-scripts
- Pisound App scripts for MODEP - https://github.com/BlokasLabs/modep-ctl-scripts
# Building MODEP Image

This repository contains a tool used to create the MODEP images, based on [pi-gen](https://github.com/RPi-Distro/pi-gen) which is used to create the Raspbian images. We recommending building the image using the Docker-based workflow. You can find more information about the build process bellow.


## Dependencies

pi-gen runs on Debian based operating systems. Currently it is only supported on either Debian Stretch or Ubuntu Xenial and is known to have issues building on earlier releases of these systems.
To install the required dependencies for pi-gen you should run:

    apt-get install docker.io quilt parted qemu-user-static debootstrap zerofree pxz zip \
    dosfstools bsdtar libcap2-bin grep rsync xz-utils file git

The file `depends` contains a list of tools needed. The format of this package is `<tool>[:<debian-package>]`.

## Starting the Build

After all the dependencies are installed, start the build by running:

```
./build-docker.sh
```

It may take a few hours before it completes.

## Build Config

Upon execution, `build.sh` will source the file `config` in the current working directory. This bash shell fragment is intended to set needed environment variables.
The following environment variables are supported:

    IMG_NAME required (Default: unset)
- The name of the image to build with the current stage directories. Setting `IMG_NAME=Raspbian` is logical for an unmodified RPi-Distro/pi-gen build, but you should use something else for a customized version. Export files in stages may add suffixes to `IMG_NAME`.
    APT_PROXY (Default: unset)
- If you require the use of an apt proxy, set it here. This proxy setting will not be included in the image, making it safe to use an `apt-cacher` or similar package for development.
- If you have Docker installed, you can set up a local apt caching proxy to like speed up subsequent builds like this:
    docker-compose up -d
    echo 'APT_PROXY=http://172.17.0.1:3142' >> config
    BASE_DIR (Default: location of build.sh)
- **CAUTION**: Currently, changing this value will probably break build.sh
- Top-level directory for `pi-gen`. Contains stage directories, build scripts, and by default both work and deployment directories.
    WORK_DIR (Default: "$BASE_DIR/work")
- Directory in which `pi-gen` builds the target system. This value can be changed if you have a suitably large, fast storage location for stages to be built and cached. Note, `WORK_DIR` stores a complete copy of the target system for each build stage, amounting to tens of gigabytes in the case of Raspbian.
- **CAUTION**: If your working directory is on an NTFS partition you probably won't be able to build. Make sure this is a proper Linux filesystem.
    DEPLOY_DIR (Default: "$BASE_DIR/deploy")
- Output directory for target system images and NOOBS bundles.
    USE_QEMU (Default: "0")
- Setting to '1' enables the QEMU mode - creating an image that can be mounted via QEMU for an emulated environment. These images include "-qemu" in the image file name.

A simple example for building Raspbian:

    IMG_NAME='MODEP'
## How the build process works

The following process is followed to build images:

- Loop through all of the stage directories in alphanumeric order
- Move on to the next directory if this stage directory contains a file called "SKIP"
- Run the script `prerun.sh` which is generally just used to copy the build directory between stages.
- In each stage directory loop through each subdirectory and then run each of the install scripts it contains, again in alphanumeric order. These need to be named with a two digit padded number at the beginning. There are a number of different files and directories which can be used to control different parts of the build process:
  - **00-run.sh** - A unix shell script. Needs to be made executable for it to run.
  - **00-run-chroot.sh** - A unix shell script which will be run in the chroot of the image build directory. Needs to be made executable for it to run.
  - **00-debconf** - Contents of this file are passed to debconf-set-selections to configure things like locale, etc.
  - **00-packages** - A list of packages to install. Can have more than one, space separated, per line.
  - **00-packages-nr** - As 00-packages, except these will be installed using the `--no-install-recommends -y`parameters to apt-get.
  - **00-patches** - A directory containing patch files to be applied, using quilt. If a file named 'EDIT' is present in the directory, the build process will be interrupted with a bash session, allowing an opportunity to create/revise the patches.
- If the stage directory contains files called "EXPORT_NOOBS" or "EXPORT_IMAGE" then add this stage to a list of images to generate
- Generate the images for any stages that have specified them

It is recommended to examine build.sh for finer details.

## Docker Build
    vi config         # Edit your config file. See above.
    ./build-docker.sh

If everything goes well, your finished image will be in the `deploy/` folder. You can then remove the build container with `docker rm -v pigen_work`
If something breaks along the line, you can edit the corresponding scripts, and continue:

    CONTINUE=1 ./build-docker.sh

After successful build, the build container is by default removed. This may be undesired when making incremental changes to a customized build. To prevent the build script from remove the container add

    PRESERVE_CONTAINER=1 ./build-docker.sh

There is a possibility that even when running from a docker container, the installation of `qemu-user-static` will silently fail when building the image because `binfmt-support` *must be enabled on the underlying kernel*. An easy fix is to ensure `binfmt-support` is installed on the host machine before starting the `./build-docker.sh` script (or using your own docker build solution).

## Stage Anatomy

**MODEP Stage Overview**
The build of MODEP image is divided up into several stages for logical clarity and modularity. This causes some initial complexity, but it simplifies maintenance and allows for more easy customization.


- **Stage 0** - bootstrap. The primary purpose of this stage is to create a usable filesystem. This is accomplished largely through the use of `debootstrap`, which creates a minimal filesystem suitable for use as a base.tgz on Debian systems. This stage also configures apt settings and installs `raspberrypi-bootloader` which is missed by debootstrap. The minimal core is installed but not configured, and the system will not quite boot yet.
- **Stage 1** - truly minimal system. This stage makes the system bootable by installing system files like `/etc/fstab`, configures the bootloader, makes the network operable, and installs packages like raspi-config. At this stage the system should boot to a local console from which you have the means to perform basic tasks needed to configure and install the system. This is as minimal as a system can possibly get, and its arguably not really usable yet in a traditional sense yet. Still, if you want minimal, this is minimal and the rest you could reasonably do yourself as sysadmin.
- **Stage 3** - mod-ui, mod-host and other mod-related software gets installed, as well as optional software for [Pisound](https://blokas.io/pisound) and other things like realtime kernel
- **Stage 4** - LV2 Plugins get built as it was done in the original MODEP release
- **Stage 5** - More LV2 Plugins get built, structured in a new way


**Stage specification**
If you wish to build up to a specified stage (such as building up to stage 2 for a lite system), place an empty file named `SKIP` in each of the `./stage` directories you wish not to include.
Then add an empty file named `SKIP_IMAGES` to `./stage4` (if building up to stage 2) or to `./stage2` (if building a minimal system).

    # Example for building a lite system
    echo "IMG_NAME='MODEP'" > config
    touch ./stage3/SKIP ./stage4/SKIP ./stage5/SKIP
    touch ./stage4/SKIP_IMAGES ./stage5/SKIP_IMAGES
    sudo ./build.sh  # or ./build-docker.sh
## Skipping stages to speed up development

If you're working on a specific stage the recommended development process is as follows:

- Add a file called SKIP_IMAGES into the directories containing EXPORT_* files (currently stage2, stage4 and stage5)
- Add SKIP files to the stages you don't want to build. For example, if you're basing your image on the lite image you would add these to stages 3, 4 and 5.
- Run build.sh to build all stages
- Add SKIP files to the earlier successfully built stages
- Modify the last stage
- Rebuild just the last stage using `sudo CLEAN=1 ./build.sh`
- Once you're happy with the image you can remove the SKIP_IMAGES files and export your image to test
## Troubleshooting
    binfmt_misc

Linux is able execute binaries from other architectures, meaning that it should be possible to make use of `pi-gen` on an x86_64 system, even though it will be running ARM binaries. This requires support from the `[binfmt_misc](https://en.wikipedia.org/wiki/Binfmt_misc)` kernel module.
You may see the following error:

    update-binfmts: warning: Couldn't load the binfmt_misc module.

To resolve this, ensure that the following files are available (install them if necessary):

    /lib/modules/$(uname -r)/kernel/fs/binfmt_misc.ko
    /usr/bin/qemu-arm-static

You may also need to load the module by hand - run `modprobe binfmt_misc`.


