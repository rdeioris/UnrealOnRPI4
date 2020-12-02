# UnrealOnRPI4
Running Unreal Engine 4 Projects on RaspberryPI 4

Author: Roberto De Ioris
Updated: 20201202

This is a tutorial for configuring an RPI4b (2, 4 and 8 GB versions) for running Linux AArch64 builds of Unreal Engine projects.

NOTE: This is NOT for running the Unreal Engine 4 Editor, only packaged builds!.

In addition to RPI4 configuration, you will need to slightly modify Unreal Engine sources to support the RPI4 GPU.

## Step 1: Preparing the RPI4 for Linux AArch64

The first step is installing a 64bit version of raspios.

You can get the latest images here:

https://downloads.raspberrypi.org/raspios_arm64/images/

Once the os is booted, you need to install the vulkan-utils package:

```sh
sudo apt install vulkan-utils
```

Our objective is to allow Vulkan based applications (like Unreal Engine on Linux) to run on the RPI4.

As of raspios 2020-08-24 the included version of mesa (the open source implementaton of OpenGL, OpenGLES, Vulkan and so on) does not include a vulkan driver for the RPI.

By running

```sh
vkcube
```

You will get an ugly error about missing ICD drivers (read: Vulkan drivers)

## Step 2: Installing the latest Mesa (for Vulkan drivers)

Let's clone the mesa official repository:

```sh
git clone https://gitlab.freedesktop.org/mesa/mesa
```

For building mesa we need a bunch of packages:

```sh
sudo apt install meson python3-mako libdrm-dev libxcb1-dev libx11-xcb-dev libxcb-dri2-0-dev libxcb-dri3-dev libxcb-present-dev libxshmfence-dev libxrandr-dev bison flex
```

We can now move to the mesa repository (read: the mesa directory you created by cloning the repository) and configure the build system:

```sh
meson --prefix=/home/pi/unreal_on_rpi4 -Dvulkan-drivers=broadcom -Ddri-drivers= -Dgallium-drivers= -Dplatforms=x11 build
```

Do not be worried about the 'red NO' lines. If all goes well you should see something like this

```txt
Message: Configuration summary:

        prefix:          /home/pi/unreal_on_rpi4
        libdir:          lib/aarch64-linux-gnu
        includedir:      include

        OpenGL:          no (ES1: no ES2: no)
        OSMesa:          no

        EGL:             no
        GBM:             no
        EGL/Vulkan/VL platforms:   x11 surfaceless

        Vulkan drivers:  broadcom
        Vulkan ICD dir:  share/vulkan/icd.d

        llvm:            no

        Gallium:         no
        HUD lmsensors:   no

        Shared-glapi:    no
```
