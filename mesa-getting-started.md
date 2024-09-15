# Mesa: Getting Started with Development

_By Lina Versace (lina@kiwitree.net)_

This is my advice on how to get started on Mesa development.
It does not cover all Mesa development topics,
though I expect to add more topics over time.
This doc targets _my_ workflow.
As a consequence, it discusses only those drivers and topics that I'm interested in working on.

The guide focuses on Vulkan. I may add OpenGL to it in the future.

Topics not discussed (yet):

  - Cross-compilation
  - Non-x86 architectures
  - OpenGL/EGL/GLX development
  - dEQP (aka VK-GL-CTS)
  - Android

_Warning: There may be bugs. I typed most of this from memory._

## Conventions

### Platforms

This guides focuses on x86\_64
and a traditional Linux desktop (Fedora, Debian, Arch, etc).
Mesa does support other platforms, architectures, and OS's,
but this guide does not discuss them.

### Shell

Different people use different shells, which may not even be POSIX-compatible.
Therefore the shell commands in this guide are kept simple
and may even use non-existent utility functions,
which are documented near the end of the doc.

## Preparation

### Home Environment

Some env vars impact the _runtime_ environment,
which is used when _running_ executables and their dependent libraries.
Other env vars impact the _buildtime_ environment,
which is used when _building_ executables and libraries.

Below is a list of env vars that I configure for all shells.

- Runtime Env Vars
  - `PATH` _[man:environ(7)]()_
  - `LD_LIBRARY_PATH` _[man:environ(7)]()_, _[man:ld.so(8)]()_
- Buildtime Env Vars
  - `PKG_CONFIG_PATH` _[man:pkgconf(1)]()_, _[man:pkg-config(1)]()_
  - `CPATH` _[man:gcc(1)]()_
  - `LIBRARY_PATH` _[man:gcc(1)]()_

Env vars that contain a token-separated list, such as `LD_LIBRARY_PATH`,
_should not_ contain empty items.
That is, no initial `:`, no terminal `:`, and no two adjacent `:`.
Empty items can cause surprising, unwanted behavior.

Take care with the lists' order.
The first item in the last has highest search priority.

#### Runtime Environment

This guide may recommend installing
a few tools and libraries into your home directory,
particularly into the XDG home prefix `~/.local`.
Therefore the guide assumes that your runtime environment includes `~/.local`.

My `~/.bashrc` contains:
```
list-prepend -x PATH "$HOME/.local/bin"
list-prepend -x LD_LIBRARY_PATH "$HOME/.local/lib"
```

For multi-arch installations, my `~/.bashrc` also contains:
```
list-prepend -x LD_LIBRARY_PATH "$HOME/.local/lib/i386-linux-gnu"
list-prepend -x LD_LIBRARY_PATH "$HOME/.local/lib/x86_64-linux-gnu"
```

#### Buildtime Environment

If you decide to install development packages into your XDG home prefix,
then your buildtime environment should contain `~/.local` too.
I recommend _not_ adding `~/.local` to your shell's buildtime environment
unless you need to; even then, there exist alternative solutions.

Personally, I install development packages into home
when (a) the system's package manager lacks the package,
when (b) I want or need a newer version of the package
than provided by the system package,
(c) or when I commit to dogfooding my own package builds.

My `~/.bashrc` contains:
```
list-prepend -x PKG_CONFIG_PATH "$HOME/.local/share/pkgconfig"
list-prepend -x PKG_CONFIG_PATH "$HOME/.local/lib/pkgconfig"
list-prepend -x CPATH "$HOME/.local/include"
list-prepend -x LIBRARY_PATH "$HOME/.local/lib"
```

Again, for multi-arch installations, I additionally add x86\_64 dirs.
Do not add i368 dirs unless you really know what you're doing.
If you use CMake, then you may need to also configure CMake env vars for multi-arch.
```
list-prepend -x PKG_CONFIG_PATH "$HOME/.local/lib/x86_64-linux-gnu/pkgconfig"
list-prepend -x CPATH "$HOME/.local/include/x86_64-linux-gnu"
list-prepend -x LIBRARY_PATH "$HOME/.local/lib/x86_64-linux-gnu"
```

### Install development packages and tools

#### Fedora

```
sudo dnf install \
    glx-utils \
    mesa-demos \
    mesa-dri-drivers \
    mesa-libEGL-devel \
    mesa-libgbm-devel \
    mesa-libGL-devel \
    mesa-vulkan-drivers \
    spirv-headers-devel \
    spirv-tools \
    spirv-tools-devel \
    valgrind-devel \
    vulkan-headers \
    vulkan-loader \
    vulkan-loader-devel \
    vulkan-tools \
    vulkan-validation-layers

sudo dnf builddep \
    libdrm \
    mesa

ln -s -t ~/.local/bin \
    /usr/lib64/mesa/vkgears \
    /usr/lib64/mesa/eglinfo
```

#### Arch Linux

```
sudo pacman -S \
    mesa \
    mesa-utils \
    valgrind \
    vulkan-devel \
    vulkan-mesa-layers \
    vulkan-swrast

# Install the Vulkan drivers below that match your gpus.
sudo pacman -S \
    vulkan-intel \
    vulkan-nouveau \
    vulkan-radeon
```

## How To Fetch, Build, and Install Mesa (to a custom prefix)

This is my usual workflow.
I usually build/install Mesa into an isolated testing prefix,
one prefix per build config.

This section does not discuss how to install Mesa user-wide or system-wide.
I also do this in my day-to-day work, but haven't documented it yet.

Cross-compiling is not discussed here.
I assume the build sysroot and the host sysroot are the same.

### Filesystem Layout

A description of my usual layout for Mesa work:

  - `src_dir="$HOME/proj/mesa/$topic"`
  - `topic`: Any long-lived devel topic, like a Mesa feature or difficult bug.
  - `build_label`: Label for an invidvual build for the `src_dir`
      - Example: _debug_, _release_, _debug-valgrind_, _release-vk-only_.
  - `work_dir="$src_dir/out/$build_label"`
      - I keep build-specific scripts, notes, etc, in `work_dir`.
  - `build_dir="$work_dir/build"`
  - `prefix="$work_dir/prefix"`

A diagram of the same.
Each Mesa source dir is a separate git checkout
of the same underlying git repo, using the `git worktree` command.

```
~/proj/mesa/
    main/
        .git
        [source files]
        out/
            intel-debug/
                etc/
                    [misc temporary stuff]
                build/
                prefix/
            intel-release/
                etc/
                    [misc temporary stuff]
                build/
                prefix/
    feature-foo/
        .git
        [source files]
        out/
            intel-debug-full/
                etc/
                build/
                prefix/
            intel-release/
                etc/
                build/
                prefix/
    nvk-compiler/
        .git
        [source files]
        out/
            nvk-debug-full/
                etc/
                build/
                prefix/
```

### Fetch Mesa

`git clone https://gitlab.freedesktop.org/mesa/mesa.git`

### Configure Mesa

There are may codenames in the configuration steps below:

  - _gallium-drivers_: Mesa's OpenGL drivers
  - _zink_: OpenGL driver that is implemented atop the Vulkan API
  - _llvmpipe_ : OpenGL driver that is software-only
  - _iris_: Intel's OpenGL driver (for GPUs newer than Broadwell)
  - _radeonsi_: AMD's OpenGL driver
  - _nouveau_: NVidia's drivers
  - _gles1_ : OpenGL ES1, which is rarely used nowadays
  - _gles2_ : OpenGL ES2 and ES3
  - _glsl_ : OpenGL Shading Language
  - _nir_ : Mesa's compiler IR

#### Example 1

A _release_ build that includes all Vulkan, OpenGL, and OpenGL ES drivers
for common GPUs that may be appear in an x86\_64 machine,
as well as software drivers too.
The configuration excludes drivers for other APIs, such as OpenCL.

Even though Mesa's OpenGL software-only driver is named _llvmpipe_,
the Vulkan software-only driver is named _swrast_ in the Meson option.

```
meson setup \
    -Dbuildtype=release \
    -Dprefix="$prefix" \
    \
    -Dopengl=true \
    -Dgles1=disabled \
    -Dgles2=true \
    -Dgallium-drivers=iris,radeonsi,nouveau,zink,llvmpipe \
    -Dvulkan-drivers=intel,amd,nouveau,swrast \
    \
    -Dbuild-tests=true \
    -Dtools=nir,glsl,intel,nouveau \
    -Dllvm=enabled \
    \
    -Dgallium-vdpau=disabled \
    -Dgallium-va=disabled \
    -Dgallium-xa=disabled \
    -Dgallium-opencl=disabled \
    -Dgallium-rusticl=false \
    -Dgallium-d3d12-video=disabled \
    -Dgallium-d3d12-graphics=disabled \
    -Dglvnd=disabled \
    -Dmicrosoft-clc=disabled \
    \
    "$src_dir" \
    "$build_dir"
```

#### Example 2

A _debug_ build that includes AMD's Vulkan driver and no OpenGL drivers.

```
meson setup \
    -Dbuildtype=debug \
    -Dprefix="$prefix" \
    \
    -Dopengl=false \
    -Dgles1=disabled \
    -Dgles2=disabled \
    -Dgallium-drivers= \
    -Dvulkan-drivers=amd \
    \
    -Dbuild-tests=true \
    -Dtools=nir \
    -Dllvm=enabled \
    \
    -Dgallium-vdpau=disabled \
    -Dgallium-va=disabled \
    -Dgallium-xa=disabled \
    -Dgallium-opencl=disabled \
    -Dgallium-rusticl=false \
    -Dgallium-d3d12-video=disabled \
    -Dgallium-d3d12-graphics=disabled \
    -Dglvnd=disabled \
    -Dmicrosoft-clc=disabled \
    \
    "$src_dir" \
    "$build_dir"
```

#### Example 3

A _debug_ build with fully unoptimized code.
Useful for precise step-debugging.

```
meson setup \
    -Dbuildtype=debug \
    -Dc_args='-Og -ggdb3' \
    -Dcpp_args='-Og -ggdb3' \
    -Dprefix="$prefix" \
    ... \
    "$src_dir" \
    "$build_dir"
```

#### Misc

If your system has multiple GPUs,
then add the Mesa option `-Dvulkan-layers=device-select`.
This will provide an env var that forces selection of your preferred GPU
in the Vulkan runtime.

### Build and Install

You can skip the intermediate _build_ step if you wish.
The _install_ step will do the _build_ step automatically.

```
ninja -C "$build_dir"
ninja -C "$build_dir" install
```

### Runtime Environment

To run an executable with your custom-built Mesa drivers,
you must add their installtion prefix to the runtime environment.

First, a quick discussion of the Vulkan loader.
Application developers for Vulkan apps link their apps to `libvulkan.so`.
The library is the _Vulkan loader_ and is responsible at runtime
for discovering installed Vulkan drivers that compatible with the system's GPUs;
for Vulkan function dispatch between app and driver;
for discovering and enabling _Vulkan layers_;
which are libraries that can inject new Vulkan APIs
and modify behavior of existing APIs;
and more.

The Vulkan loader's application-facing environment variables are documented in its
[LoaderApplicationInterface.md](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderApplicationInterface.md).

The configured env vars are:

  - `PATH`
  - `LD_LIBRARY_PATH`
  - `VK_DRIVER_FILES`
      - The Vulkan loader, when deciding which Vulkan driver to load,
        prioritizes the driver metadata files listed in this env var
        above the files installed in the loader's default search locations.
        For driver enumeration, the Vulkan loader does not yet (2024-09-12) support
        a more conventional _path_ variable,
        which adds directories (rather than files) to the search path.
  - `VK_ADD_LAYER_PATH`
      - When searching for _explicit layers_,
        the Vulkan loader prioritizes the directories listed in this env var
        above the loader's default search paths.
      - The loader enables explicit layers whose names
        are listed in env var `VK_INSTANCE_LAYERS`
        or listed in `VkInstanceCreateInfo::ppEnabledLayerNames` in the Vulkan API.
  - `VK_ADD_IMPLICIT_LAYER_PATH`
      - When searching for _implicit layers_,
        the Vulkan loader prioritizes the directories listed in this env var
        above the loader's default search paths.
      - The loader enables all implicit layers
        that are found in the loader's search paths for implicit layers.
      - _Warning_: As of 2024-09-14, the loader does not support this variable,
        but will in its next release.  See pull request
        [Vulkan-Loader!1550](https://github.com/KhronosGroup/Vulkan-Loader/pull/1550).
  - `VK_INSTANCE_LAYERS`
      - A list of Vulkan layer names to explicitly enable.

If your system has [glvnd]() installed, then you may need to configure additional
env vars for [ICD enumeration](glvnd-egl).

[glvnd]: https://gitlab.freedesktop.org/glvnd/libglvnd
[glvnd-egl]: https://gitlab.freedesktop.org/glvnd/libglvnd/-/blob/master/src/EGL/icd_enumeration.md

_TODO: Name the Vulkan loader versions that do and do not support VK_ADD_IMPLICIT_LAYER_PATH._

If the Vulkan loader supports `VK_ADD_IMPLICIT_LAYER_PATH`,
then I configure the runtime environment like this:
```
list-prepend -x PATH "$prefix/bin"
list-prepend -x LD_LIBRARY_PATH "$prefix/lib"
list-prepend -x VK_DRIVER_FILES "$prefix/share/vulkan/icd.d/the_driver_you_want.json"
list-prepend -x VK_ADD_LAYER_PATH "$prefix/share/vulkan/explicit_layer.d"
list-prepend -x VK_ADD_IMPLICIT_LAYER_PATH "$prefix/share/vulkan/implicit_layer.d"
```

If the Vulkan loader does not support `VK_ADD_IMPLICIT_LAYER_PATH`,
and if I configured Mesa with `-Dvulkan-layers=device-select`,
then I configure the runtime environment like this:
```
list-prepend -x PATH "$prefix/bin"
list-prepend -x LD_LIBRARY_PATH "$prefix/lib"
list-prepend -x VK_DRIVER_FILES "$prefix/share/vulkan/icd.d/the_driver_you_want.json"
list-prepend -x VK_ADD_LAYER_PATH "$prefix/share/vulkan/explicit_layer.d"
list-prepend -x VK_ADD_LAYER_PATH "$prefix/share/vulkan/implicit_layer.d"
list-prepend -x VK_INSTANCE_LAYERS VK_LAYER_MESA_device_select
```

### Test the Vulkan drivers

Inspect, with `vulkaninfo --summary`,
which Vulkan drivers are discovered by the Vulkan loader.
If the loader finds the driver you just built,
then its `driverInfo` field will contain the Mesa git commit from which it was built.
This should match the git commit of your Mesa repo.

In the example below, GPU0 is the Intel Vulkan driver I built and installed.
Notice the line `driverInfo = Mesa 24.3.0-devel (git-6f3c003433)`.
The git commit correctly matches my Mesa repo.
But GPU1 is a software driver, _llvmpipe_,
that was _not_ built from my Mesa repo.
Its `driverInfo` lacks a git commit.
```
$ vulkaninfo --summary
...
Devices:
========
GPU0:
        apiVersion         = 1.3.292
        driverVersion      = 24.2.99
        vendorID           = 0x8086
        deviceID           = 0xa7a0
        deviceType         = PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU
        deviceName         = Intel(R) Graphics (RPL-P)
        driverID           = DRIVER_ID_INTEL_OPEN_SOURCE_MESA
        driverName         = Intel open-source Mesa driver
        driverInfo         = Mesa 24.3.0-devel (git-6f3c003433)
        conformanceVersion = 1.3.6.0
        deviceUUID         = 8680a0a7-0400-0000-0002-000000000000
        driverUUID         = ae4006fc-bdd0-5515-3822-c59af8f02109
GPU1:
        GPU1:
        apiVersion         = 1.3.278
        driverVersion      = 0.0.1
        vendorID           = 0x10005
        deviceID           = 0x0000
        deviceType         = PHYSICAL_DEVICE_TYPE_CPU
        deviceName         = llvmpipe (LLVM 18.1.6, 256 bits)
        driverID           = DRIVER_ID_MESA_LLVMPIPE
        driverName         = llvmpipe
        driverInfo         = Mesa 24.1.4 (LLVM 18.1.6)
        conformanceVersion = 1.3.1.1
        deviceUUID         = 6d657361-3234-2e31-2e34-000000000000
        driverUUID         = 6c6c766d-7069-7065-5555-494400000000
```

If your system has multiple GPUs, as in the example above,
I recommend explicitly forcing the Vulkan loader
to choose the GPU you prefer.
Below is an example from my machine.
I list all Vulkan drivers discovered by the Vulkan loader,
then select the index of the driver I want.
```
$ MESA_VK_DEVICE_SELECT=list vulkaninfo # any vk app can be used here
selectable devices:
  GPU 0: 8086:a7a0 "Intel(R) Graphics (RPL-P)" integrated GPU 0000:00:02.0
  GPU 1: 10005:0 "llvmpipe (LLVM 18.1.6, 256 bits)" CPU 0000:00:00.0
$ export MESA_VK_DEVICE_SELECT=0
```

Finally, let's draw some gears and a cube.
```
$ vkgears
$ vkcube
```

## Vulkan Testsuites

_TODO_

## Documentation for Custom Commands

```
NAME
    list-prepend

SYNOPSIS
   list-prepend [-x] <name> [<item>...]

DESCRIPTION
    Prepend each <item> to the list <name>. Item separator is ':'. If given multiple items, they are
    prepended in order so that the first <item> arg will become the first item in the list.  Flag
    '-x' exports <name> as an environment variable.

    The command takes care to not insert any empty items into the list, unless <item> is an empty
    string. This is significant for some env vars, such as LD_LIBRARY_PATH.

EXAMPLES
    Prepend to PATH.
        $ printenv PATH
        /usr/local/bin:/usr/bin
        $ list-prepend PATH ~/.local/bin ~/.cargo/bin
        $ printenv PATH
        /home/lina/.local/bin:/home/lina/.cargo/bin:/usr/local/bin:/usr/bin

    Prepend to non-existent list.
        $ printenv LD_LIBRARY_PATH
        # no output. it's empty.
        $ list-prepend -x LD_LIBRARY_PATH ~/.local/lib
        $ printenv LD_LIBRARY_PATH
        /home/lina/.local/lib
```

## License

Copyright 2024 Lina Versace (lina@kiwitree.net)

SPDX-License-Identifier: CC-BY-4.0
