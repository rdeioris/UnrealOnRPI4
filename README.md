# UnrealOnRPI4
Running Unreal Engine 4 Projects on RaspberryPI 4

Author: Roberto De Ioris
Updated: 20201202

This is a tutorial for configuring an RPI4b (2, 4 and 8 GB versions) for running Linux AArch64 builds of Unreal Engine projects.

NOTE: This is NOT for running the Unreal Engine 4 Editor, only packaged builds!.

In addition to RPI4 configuration, you will need to slightly modify Unreal Engine sources to support the RPI4 GPU.

This tutorial could be useful for porting Unreal Engine projects to other Linux arm64 platforms (with a GPU vulkan driver available).

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

Time to build and install:

```sh
ninja -C build
ninja -C build install
```

Your RPI vulkan ICD driver is in /home/pi/unreal_on_rpi4/

```sh
pi@raspberrypi:~/mesa $ find /home/pi/unreal_on_rpi4/
/home/pi/unreal_on_rpi4/
/home/pi/unreal_on_rpi4/lib
/home/pi/unreal_on_rpi4/lib/aarch64-linux-gnu
/home/pi/unreal_on_rpi4/lib/aarch64-linux-gnu/libvulkan_broadcom.so
/home/pi/unreal_on_rpi4/share
/home/pi/unreal_on_rpi4/share/vulkan
/home/pi/unreal_on_rpi4/share/vulkan/icd.d
/home/pi/unreal_on_rpi4/share/vulkan/icd.d/broadcom_icd.aarch64.json
/home/pi/unreal_on_rpi4/share/drirc.d
/home/pi/unreal_on_rpi4/share/drirc.d/00-mesa-defaults.conf
```

We can now run vkcube by specifying (using an environment variable) where the ICD driver is (NOTE: technically weuse the path to a json file specyfying where to find the shared object):

```sh
VK_ICD_FILENAMES=/home/pi/unreal_on_rpi4/share/vulkan/icd.d/broadcom_icd.aarch64.json vkcube
```

If you see the vulkan cube spinning, Congratulations!, we can now move to Unreal Engine.

## Step 3: Preparing Unreal Engine (The Editor) for Linux AArch64 (Cross Compilation from Win64)

Just download the toolchain from your editor version: https://docs.unrealengine.com/en-US/Platforms/Linux/GettingStarted/index.html

Once installed you will get Linux and Linux AArch64 as packaging target.

Create an empty Unreal Engine project and package it for Linux AArch64 and copy the resulting directory to your RPI4.

Time to fail: run the .sh script into the directory you just uploaded onto the RPI to see it miserabily crash :(

## Step 4: First Engine Sources modifications

Why Unreal crashed ?

The first reason is that the Engine makes a check during Vulkan setup for the type of driver discovered. The current (as Unreal 4.26) list contains the following vendors in 'Engine/Source/Runtime/RHI/Public/RHIDefinitions.h' :

```cpp
enum class EGpuVendorId
{
        Unknown         = -1,
        NotQueried      = 0,

        Amd                     = 0x1002,
        ImgTec          = 0x1010,
        Nvidia          = 0x10DE,
        Arm                     = 0x13B5,
        Qualcomm        = 0x5143,
        Intel           = 0x8086,
};
```

No Broadcom here (The RPI GPU vendor).

This is an easy fix (just add the 'Broadcom' entry for the VendorId 0x14e4:

```cpp
enum class EGpuVendorId
{
        Unknown         = -1,
        NotQueried      = 0,

        Amd                     = 0x1002,
        ImgTec          = 0x1010,
        Nvidia          = 0x10DE,
        Arm                     = 0x13B5,
        Qualcomm        = 0x5143,
        Intel           = 0x8086,
        Broadcom        = 0x14e4,
};
```

In addition to this, go below in the code til this inline function:

```cpp
inline EGpuVendorId RHIConvertToGpuVendorId(uint32 VendorId)
{
        switch ((EGpuVendorId)VendorId)
        {
        case EGpuVendorId::NotQueried:
                return EGpuVendorId::NotQueried;

        case EGpuVendorId::Amd:
        case EGpuVendorId::ImgTec:
        case EGpuVendorId::Nvidia:
        case EGpuVendorId::Arm:
        case EGpuVendorId::Qualcomm:
        case EGpuVendorId::Intel:
                return (EGpuVendorId)VendorId;

        default:
                break;
        }

        return EGpuVendorId::Unknown;
}
```

An easy fix again:

```cpp
inline EGpuVendorId RHIConvertToGpuVendorId(uint32 VendorId)
{
        switch ((EGpuVendorId)VendorId)
        {
        case EGpuVendorId::NotQueried:
                return EGpuVendorId::NotQueried;

        case EGpuVendorId::Amd:
        case EGpuVendorId::ImgTec:
        case EGpuVendorId::Nvidia:
        case EGpuVendorId::Arm:
        case EGpuVendorId::Qualcomm:
        case EGpuVendorId::Intel:
        case EGpuVendorId::Broadcom:
                return (EGpuVendorId)VendorId;

        default:
                break;
        }

        return EGpuVendorId::Unknown;
}
```

Time to rebuild our project...
