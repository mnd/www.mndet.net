---
layout: default
title: Build Systems from Distribution Developer Point of View
---
Distribution developers often solve several tasks:

 1. Add a package to distribution.
 2. Update a package.
 3. Change an implementation or version of the core system dependencies: compiler, kernel, libc, build systems itself.
 4. Gather overall system statistics: compile-time and run-time dependencies between packages, optimization flags used for all packages in the system.
 5. Change a compilation configuration for all packages: Change optimizations from a speed to a size, enable stack guards, enable the "-fPIE" for all applications, etc.

I have an experience with different types of distributions: Yocto-based, rpmspec-based, GNU GUIX. Part of issues (e.g. package creation, extracting compile-time dependencies from recipes) should be solved in distribution-specific manner; other issues are specific to a build system used for the package. Today I want to discuss the latter type of the issues.

# Formal Description of Interested Tasks

I will try to describe how the main build systems (plane make, autotools and cmake) are friendly to the following tasks:

 1. To use a custom application as a C or C++ compiler.

    It means providing something like "/usr/bin/ccache /opt/local/clang/bin/clang --resource-dir /opt/local/clang/lib/clang/6.0.0 --sysroot /usr/src/build/sysroot" as the compiler command. A build system should use it in correct manner passing each option as a separate argument.
 2. To append compilation flags for shared library sources.

    It is useful in cases when we want to enable some specific optimization options or to disable diagnostic errors that was added in new version of a compiler.
 3. To append compilation flags for application sources.

    This task differ from flags for shared library sources because flags like the "-fPIE" flag should be passed only for sources of an executable.
 4. To append linking flags in same cases.

    There are useful linking flags that can be enabled system-wide: the "--print-map" flag can be used to gather a build information, the "-z,noexecstack" flag can be required by security reasons, etc.
    
    Linker flags divided on executable-related flags like the "-pie" flag and shared library related flags like "-shared" flag in same manner as compilation flags.
 5. To append preprocessor flags.

    I do not know cases when we should pass different preprocessor flags for libraries and executables. Therefore I will not distinguish this cases.

I will not describe prepending/replacing all flags provided by build system. As a rule compilers prioritize the last option from a command line and prepending/replacing became much less useful then appending.

# Common Methods to Perform Described Tasks

Regardless of the build system you use to create packages for your distribution you always have several methods to solve the tasks described above:

 1. Pass environment variables to whole build. e.g. add `export LIBRARY_PATH=/opt/local/gcc/lib/x86_64/6.0.2` before configuration/compilation commands.
 2. Pass specific options to the configuration command. e.g. `./configure --disable-silent-rules` or `cmake -DCMAKE_VERBOSE_MAKEFILE:BOOL=ON`
 3. Pass specific options to the compilation command. e.g. A construction like the `make CC="${CC}"` can be used to override value pf the `CC` variable even if it was predefined in the Makefile itself.
 4. Patch sources before configuration. Most of the package build systems supports a source patching as a separate step in an unpack-patch-configure-build-install process and they have a special support to apply patches to sources before configuration.
 5. Modify sources after the a configuration process. There are cases when we want to change files created during the configuratoin process and build systems allows to make such changes.

I will assume that a method with a lower number is a preferred method. The first three methods in most of cases can be used universally for all packages, while the last two methods should be applied on a per-package base.

# Make Based Packages

For packages that builds with plane makefiles there are can be two cases: makefiles that use the [implicit rules](https://www.gnu.org/software/make/manual/html_node/Catalogue-of-Rules.html) and a makefiles that use custom rules.

## Makefiles Based on the Implicit Rules

Consider the following example:

    SOURCES = application.c a.c b.c d.c
    OBJECTS = $(patsubst %.c,%.o,$SOURCES)
    # You should have file with same name as resulting
    # application to use implicit linking rules
    application: $(OBJECTS) -ldl
    a.o : a.h

This code is enough to compile application. All compilation will be made using implicit rules. Implicit make rules allow you pass: `CC`, `CXX` as compilers; `CPPFLAGS` for preprocessor flags; `CFLAGS`, `CXXFLAGS` as compilation flags; `LDFLAGS` for linking flags except `-lname` option; `LDLIBS` for additional libraries.

If Makefile does not provide values for standard variables then you can specify values through environment variables. If Makefile provide any values for standard variables (e.g. `CFLAGS = -Wall` in text of Makefile) then you have no easy way to *append* additional values to this variables. In this case you must check code of Makefile carefully and decide if you want completely *override* variable value with `make VARIABLE="new value"` command or modify variables with patch to Makefile.

### Recommendations

In order to provide custom compiler always override C and C++ compiler variables with `make CC="c-compiler" CXX="c++-compiler"`. This code will work even for makefiles that provides own value for the compiler variables. In order to provide custom flags try to export environment variables; if environment variables did not helped then you should consider the code and decide what will be safer: to override variable completely or to patch makefiles.

## Makefiles with Own Set of Rules
    
Often developers writes own rules for compilation:

    all: application
    application : a.o b.o c.o
        $(CC) -Wl,-z,relro $(LINKER_FLAGS) $^ $(LDLIBS) -o $@
    %.o : %.c common.h
        $(CC) $(CPPFLAGS) -fstack-protector $(CFLAGS) -c -o $@ $< 

Or creates make-based build-systems as set of rules that can be configured with variables defined in a makefile. Such systems may look like following code:

    LIBRARY = name
    SOURCES = a.c b.c c.c
    PUBLIC_HEADERS = a.h
    PRIVATE_HEADERS = b.h c.h
    LIBRARY_TYPE = shared
    LIBRARY_CFLAGS = -Wall
    include build-system-rules.mk

Makefiles of this types can provide any set of rules and they can use any names for their variables: consider the `LINKER_FLAGS` name instead of the common `LDFLAGS` in the example above.

### Recommendations

Without knowledge about a particular Makefile code I can suggest same recommendation as for a Makefile that was based on the implicit rules. but if you want to create own make-based-build-system consider the following recommendations that will make much simpler work of distribution developers:

 1. Use standard variables for compilers: `CC`, `CXX`. Assume that this variables can be *overridden*. Do not try to provide a custom behavior based on value of this variables. The following example

        CC_SAVED := $(CC)
        CC := false
        rule:
            # Make sure that subdir can't use CC
            $(MAKE) -C subdir 

    will work in the unexpected manner if user will call `make CC=gcc`.
 2. For all flags use two variables: one variable with a name specific to your system and one with a standard name. The good example:

        %.o : %.c
            $(CC) $(MY_CPPFLAGS) $(CPPFLAGS) $(MY_CFLAGS) $(CFLAGS) $< -c -o $@
        $(APPNAME) : $(OBJS)
            $(CC) $(MY_LDFLAGS) $(LDFLAGS) $^ -o $@ $(MY_LDLIBS) $(LDLIBS)

    In this example the application developer should use `MY_*` flags, while flags without prefix will be used by person who will create package for the application.

# Autoconf+Automake Based Packages

The Automake always generate makefiles with the compile commands that made in the following manner:

    compilation command:
    $(CC) $(automake/autoheader specific flags)            \
        $(either AM_CPPFLAGS or prog_CPPFLAGS) $(CPPFLAGS) \
        $(either AM_CFLAGS or prog_CFLAGS)     $(CFLAGS)   \
        -c -o source.o source.c

    link command:
    CCLD = $(CC)
    $(CCLD) \
        $(either AM_CFLAGS or prog_CFLAGS) $(CFLAGS) \
        $(either AM_LDFLAGS or prog_LDFLAGS) $(LDFLAGS) \
        -o prog $(prog_OBJECTS) $(either prog_LDADD or prog_LIBADD) $(LIBS)

As you can see all the compilation commands have pairs of the flags: automake specific `AM_*FLAGS` or `prog_*FLAGS` and user provided `*FLAGS`. *NOTE*: `AM_*FLAGS` are `Makefile.am`-wide flags, `prog_*FLAGS` are flags for specific output; it's good practice to define your per-application flags in the following manner: `prog_CPPFLAGS = $(AM_CPPFLAGS) other flags`.

If developer try to override `*FLAGS` through `Makefile.am` then automake print warning:

    warning: 'CFLAGS' is a user variable, you should not override it;
    use 'AM_CFLAGS' instead

As result it's a rare situation when a package code broke a possibility to append flags through an environment variable.

The Automake do not provide a way to pass different flags for executables and shared libraries, but the Libtool is able to filter-out PIE flags from PIC objects compilation flags. As result if you compile a project with the `CFLAGS=-fPIE LDFLAGS=-pie` flags then applications, automake static libraries, and libtool static libraries will be compiled with PIE flags, while libtool shared libraries will be compiled only with PIC flags. This solution is good enough because the "-fPIE -pie" flags as far as I know is the only flags that must be different between executables and shared libraries.

## Recommendations

In most of the cases there will be enough to export environment variables before configuration:

    # A configure script will check if a compiler is able to compile a code
    # If the test will pass then a user-supplied compiler will be used
    export CC="gcc -m64"
    export CXX="ccache /usr/bin/g++"
    # Will be added after flags defined in Makefile.am's
    export CPPFLAGS="-DDEBUG=1"
    export CFLAGS="-Wno-format-literal"
    export CXXFLAGS="-fpermissive"
    
    ./configure && make && make install

There are always some cases when the configuration through environment variables do not work. In this cases you should consider a source code to understand how you should build a project. Use the following rules:

 1. If `configure.ac` uses non-standard environment variables it's better to pass it through `./configre` argument instead of environment: `./configure CUSTOM_CONFIG=value`. In this case `./configure` script will be able to memorize it and re-use if the `make` command will trigger re-configuration. 
 2. If `configure.ac` override your flags try to pass them as make override: `make CFLAGS="--flags-from-configure --user-additions"`.
 3. You can try to patch sources if previous suggestions does not work.

The main case when user-provided flags is not appended or when project add own flag after user-provided is when `configure.ac` file modify this flags by itself. e.g.

    configure.ac:
    # In next cases user will lost ability to disable -Wformat warnings through CFLAGS variable
    CFLAGS="${CFLAGS} -Wformat=2 -Werror"
    AX_APPEND_COMPILE_FLAGS([-Wformat=2 -Werror], [CFLAGS])

If you write own `configre.ac` then you should prefer an additional variable for `configure`-provided flags rather then a modification to user flags:

    configure.ac:
    AX_APPEND_COMPILE_FLAGS([-Wformat=2 -Werror], [APPLICATION_CFLAGS])
    AC_SUBST([APPLICATION_CFLAGS])
    
    Makefile.am
    AM_CFLAGS = $(APPLICATION_CFLAGS)


# CMake Based Packages

The CMake is an other widely used build system. It has some advantages over the Autoconf (the CMake works faster, the Autoconf is able to generate only Makefiles, while the CMake also able to generate "ninja" build scripts and project files for several IDE's) and I would expect most new software project to choose the CMake as a build system.

The CMake has own set of variables that configure compilation commands, but it also provides some compatibility layer that try to convert old `CC`/`CFLAGS`/`LDFALGS` variables to a default value for the CMake-specific variables. Unfortunately the CMake does not support the Automake concept of [User Variables](https://www.gnu.org/software/automake/manual/html_node/User-Variables) and all your flags are just initial values.

## Custom Compiler

In order to compile a C code the CMake uses the following variables:

 1. `CMAKE_C_COMPILER` -- a full path to the compiler.
 2. `CMAKE_C_COMPILER_ARG1` -- mandatory arguments for all calls to the compiler.
 3. `CMAKE_C_COMPILER_LAUNCHER` -- a compiler launcher. e.g. "ccache".

Additionally some arguments, that with other build systems can be used as a part of the `CC` variable, provided through special variables:

 1. `CMAKE_C_COMPILER_TARGET` -- a target architecture that used for cross-compilers, such as the CLang, that supports several targets from the same binary. In case of the CLang compiler this variable contains a value for the `--target=` key.
 2. `CMAKE_SYSROOT` -- a value for the `--sysroot=` compiler argument.
 
If the `CMAKE_C_COMPILER` variable is not defined then the CMake tried to extract compiler related variables command from the `CC` variable. First word from this variable used as value for the `CMAKE_C_COMPILER` variable, other words from the `CC` variable passed to the `CMAKE_C_COMPILER_ARG1` variable. *NOTE*: If the `CMAKE_C_COMPILER` variable already supplied somehow then all arguments from the `CC` variable will be missed. They would not be used as an initial value for the `CMAKE_C_COMPILER_ARG1` variable.

## Compilation flags

For code compilation the CMake uses three types of variables:

 1. Definitions. New definitions for a compilation can be added with the following commands:
      + The `target_compile_definitions` function can add a new "-DVAR" flag on a per-target base.
      + The `add_definitions` function can add flags globally for all projects that will be defined after this command.
      + With modification to the `COMPILE_DEFINITIONS` target and directory properties. 
 2. Include paths. New paths to header files can be added with the following commands:
      + The `target_include_directories` function can add new "-Ipath" or "-isystem path" flags on a per-target base.
      + The `include_directories` function can add new include path for all projects that will be defined after this command.
      + With modification to the `INCLDUE_DIRECTORIES` target and directory properties.
 3. All other compilation flags. New flags can be defined with the following commands:
      + Regardless of its name the `add_definitions` function can add any compilation flag to a compilation command.
      + The `add_compile_options` and `target_compile_options` function can add compilation flags globally or on a per-target base.
      + With modification to the `COMPILE_OPTIONS` target and directory properties.
      + Through the language-specific `CMAKE_C_FLAGS` variable.
      + Through the language- and build-type-specific `CMAKE_C_FLAGS_<CMAKE_BUILD_TYPE>` variables.
      
    You'll have the following order for compilation options:
     1. `${CMAKE_C_FLAGS}`
     2. `${CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE}}`
     3. options added through `add_definitions`
     4. options added through `add_compile_options`
     5. options added through `target_compile_options`
         
In order to control a default value of compilation flags you can:
 1. Define the `CFLAGS` environment variable. This will provide default value for the `CMAKE_C_FLAGS` cmake variable.
 2. Provide a value for the `CMAKE_C_FLAGS` variable with the `-D` key for the `cmake` configuration command.
 3. Provide values for the both `CMAKE_BUILD_TYPE` and `CMAKE_C_FLAGS_${CMAKE_BUILD_TYPE}` variables during configuration.
 
*NOTE*: The CMake does not use the `CPPFLAGS` environment variable at all. Custom pre-processor flags should be either added to `CMakeLists.txt` files, or passed as compilation flags.

## Linking Flags

In order to link binaries the CMake uses the following mechanisms:

 1. Global variables `CMAKE_EXE_LINKER_FLAGS`, `CMAKE_STATIC_LINKER_FLAGS`, `CMAKE_SHARED_LINKER_FLAGS`, `CMAKE_MODULE_LINKER_FLAGS` and their `CMAKE_*_LINKER_FLAGS_<CMAKE_BUILD_TYPE>` variants can be used to provide flags for specific types of outputs.
 2. The `add_link_options` and `target_link_options` function used to add link options either globally or on a per-target base.
 3. The `link_libraries` and `target_link_libraries` function used to add extra libraries during linking. *NOTE*: If you use the `link_libraries` function to link with other CMake targets then all compilation and linker flags added to this targets with the `PUBLIC` or `INTERFACE` keywords will be inherited by your new targets.

In order to control default value of linker flags you can:
 1. Define the `LDFLAGS` environment variable. This will provide default value for the `CMAKE_EXE_LINKER_FLAGS`, `CMAKE_SHARED_LINKER_FLAGS`, and `CMAKE_MODULE_LINKER_FLAGS` variables, but not for the `CMAKE_STATIC_LINKER_FLAGS` variable.
 2. Provide value for the `CMAKE_*_LINKER_FLAGS` variable with the `-D` key for the CMake configuration.
 3. Provide values for both the `CMAKE_BUILD_TYPE` and `CMAKE_*_LINKER_FLAGS_${CMAKE_BUILD_TYPE}` variables during configuration.

## Recommendations

The CMake ability for configuration of compilation and linking flags is very flexible and the CMake provide handy API for developers. Unfortunately the CMake allows to provide only initial values for all used mechanisms during a configuration process. As a result package developers have a limited possibility to control flags without modification to the CMake files itself. My personal recommendations is the following:

 1. If your packaging system does not provide specific CMake support then you should try to use the standard `CC`, `CFLAGS`, `LDFLAGS` environment variables. *NOTE*: There is no `CPPFLAGS` support and you should always move your `CPPFLAGS` to `CFLAGS` environment variable.
 2. If you have control over your packaging system then you should consider creation of own code that will translate standard environment variables to the CMake variables. In this case you can provide your default values either through separate `-D` options or through the `-DCMAKE_TOOLCHAIN_FILE=/path/to/toolchain.cmake` configuration option and the `toolchain.cmake` file with default configuration.
 3. Consider using of the `CMAKE_*_FLAGS_${CMAKE_BUILD_TYPE}` variables and try always explicitly provide the `CMAKE_BUILD_TYPE` for builds.

But still you can control only initial values of variables and new flags from the `CMakeLists.txt` files often appended to initial values there is high chance that you will fail to provide your flags in sane manner without patches to the `CMakeLists.txt` files.

# Conclusion

With any build system there will be situations that force distribution developers to write patches to the build system files itself, but according to my experience with the Autotools it is much likely that compilation/linking flags can be added just through environment variables. 

If you are going to write a new configuration/build system consider the following requirements to make life of distribution developers simpler:

 1. Provide mechanism like the Automake's [User Variables](https://www.gnu.org/software/automake/manual/html_node/User-Variables.html) for all flags. Unfortunately the CMake do not have such mechanism.
 2. Provide the way to supply different flags for different types of outputs (executables, shared libraries, plug-ins). The CMake have handy the `CMAKE_EXE_LINKER_FLAGS`, `CMAKE_STATIC_LINKER_FLAGS`, `CMAKE_SHARED_LINKER_FLAGS`, `CMAKE_MODULE_LINKER_FLAGS` variables for linker flags, but it lacks the same mechanism for compilation flags. In case of the autotools "libtool" have possibility to filter out both inappropriate compilation and linker flags, but this works only for flags that was hard-coded in the libtool code.
 3. Provide the way to override flags completely during compilation. If your build system will be based on the "make" build system then try to use everything through variables: compilers, tools, each types of flags. 

    Try to avoid partial solutions like in the CMake with Makefile back-end: The CMake allow you re-define compilation flags during compilation with the `make C_DEFINES= C_INCLUDES= CFLAGS=` parameters, but the CMake does not provide possibility to override a C compiler application or linker flags in same time.

Currently the Autotools build system supports most of the tricks that makes work on a big distribution simpler.
