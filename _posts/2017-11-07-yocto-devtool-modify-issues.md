---
layout: default
title: Possible issues with yocto 'devtool modify' utility
---
This post was originally [published](https://www.linkedin.com/pulse/possible-issues-yocto-devtool-modify-utility-nikolay-merinov/) on linkedin.

Yocto delivered with great "devtool" tool that allow to work on specific package locally. This tool allow you create patches to any recipes like this:

    devtool modify package
    cd workspace/sources/package
    # Edit source as you want
    devtool build package
    # Finish editing
    git commit -am 'This new commit in sources will be exported as patch to yocto recipe'
    devtool finish package /path/to/meta-layer-with-package

You can find full description for "devtool modify" utility in yocto [documentation](http://www.yoctoproject.org/docs/latest/sdk-manual/sdk-manual.html#sdk-devtool-use-devtool-modify-to-modify-the-source-of-an-existing-component).

For recipe with `S="${WORKDIR}/internal/subpath/for/sources"` devtool will export directory `"${WORKDIR}/internal"` as source directory and reuse "externalsrc" mechanism to allow to build from exported sources. To track changes and generate patches devtool create "git" repository at a root of exported sources.

All this mechanisms can fail because of yocto flexibility. Let's describe a possible issues with examples based on current (yocto 2.4) [poky](https://www.yoctoproject.org/tools-resources/projects/poky) distro.

Possible failures:
------------------

### Missed sources

externalsrc.bbclass change "fetch" task to fetch only "file://" URI's, while 'devtool' save only one subdirectory of "${WORKDIR}" as sources. As result if you fetch more that one remote URI for recipe "devtool modify" will lose part of files required for build.

#### Example:

    devtool modify libxml2 && devtool build libxml2

This command will fail to build because libxml2 fetch two tarballs, but devtool save content only from one of tarballs as a sources.

### Lost changes

If you have git repository in subdirectory of "${S}" than "devtool" will lose all your changes in this subdirectory, because "devtool finish" looks only at toplevel git.

#### Example:

    devtool modify cross-localedef-native
    cd workspace/sources/cross-localedef-native/localedef/
    # Change files
    git commit -am 'This changes will be lost'
    devtool finish cross-localedef-native /path/to/meta-layer

### Missed patches

devtool also can fail if you recipe patch some files fetched with "file://" fetcher. There is no such recipes in poky, but it possible to imagine next example: 

    recipe.bb:
      SRC_URI = "https://recipe.org/recipe.tar.gz file://additional_file"
    recipe.bbappend
      SRC_URI += "file://patch_for_additional_file.patch;patchdir=../"

In this case after "devtool modify" you will work with original "additional_file" without patches.
