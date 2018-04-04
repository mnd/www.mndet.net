---
layout: default
title: Yocto under GUIX
---
Ok, I got allowance to publish my yocto patches and finally you can use yocto to build embedded linux distros under GuixSD! All required code placed at [https://github.com/mnd/guix-mnd-pkgs](https://github.com/mnd/guix-mnd-pkgs).

What is yocto?
--------------

Yocto is set of projects to build embedded linux distros. This project based on "openembedded" project and widely used in telecommunacations and car-manafactuing.
Usage instruction

Fetch my guix packages
----------------------

    git clone https://github.com/mnd/guix-mnd-pkgs.git

Create environment with patched poky distro (main part of yocto project):

    guix environment -L /path/to/guix-mnd-pkgs --ad-hoc poky --container --network --root=${PWD}/poky.env --expose=${PWD}/poky.env=/usr  --expose=/bin/sh=/bin/bash --expose=${PWD}/poky.env/bin/pwd=/bin/pwd --expose=${PWD}/poky.env/bin/ln=/bin/ln
    export LC_ALL=en_US.UTF-8  # Use locale suitable for python3
    source /usr/share/yocto/oe-init-build-env

And then you can build your yocto-based distro. By default you have poky, but you can change `conf/bblayers.conf` file to add new meta-layers (e.g. layer with support for raspberry-pi builds).

What was made
-------------

GUIX package for poky distro with all dependencies, package for coreutils with xattr support, package for chrpath utilities, set of pactches to openembedded-core project.

Yocto patches: Support to build with `CPPFLAGS = "-isystem/usr/include"`, support to build with `HOST_CC` that contains several arguments for gcc, remove hardcoded "gcc" from openembedded-native recipes, fix for rpath'es in native perl builds, several changes to support special GuixSD environemnt (e.g. add export `GUIX_LOCPATH` to yocto build environments).
