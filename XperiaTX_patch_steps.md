# Patch Guide for Xperia TX Running Lineage OS 14.1

## Compiling Kernel

* I use kernel from Lineage OS repository. The kernel version is `3.4.113` (Oct., 2017).
  * Repo : https://github.com/LineageOS/android_kernel_sony_msm8x60
  * Commit : `6fd23f41f7babaf8d3c0c57709f14fecd47a5ec2`
* Make a new directory, download `kernel-3.4.patch` to the directory.
* `cd` into the directory, launch following commands.
```
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8
git clone https://github.com/LineageOS/android_kernel_sony_msm8x60
export PATH=`pwd`/arm-eabi-4.8/bin:$PATH
export ARCH=arm
export SUBARCH=arm
export CROSS_COMPILE=arm-eabi-
cd android_kernel_sony_msm8x60
patch -p1 < ../kernel-3.4.patch
```
You should be greeted with the below output. Notice that some patches failed.
```
patching file drivers/usb/gadget/Makefile
patching file drivers/usb/gadget/android.c
Hunk #1 FAILED at 74.
Hunk #2 succeeded at 2329 with fuzz 1 (offset 241 lines).
Hunk #3 FAILED at 2153.
Hunk #4 FAILED at 2478.
3 out of 4 hunks FAILED -- saving rejects to file drivers/usb/gadget/android.c.rej
patching file drivers/usb/gadget/f_hid.c
Hunk #7 succeeded at 403 (offset -9 lines).
Hunk #8 succeeded at 422 (offset -9 lines).
patching file drivers/usb/gadget/f_hid.h
patching file drivers/usb/gadget/f_hid_android_keyboard.c
patching file drivers/usb/gadget/f_hid_android_mouse.c

```
Let's patch those manually. Open `android_kernel_sony_msm8x60/drivers/usb/gadget/android.c` in a text editor. <br>
Find the line `#include "f_accessory.c"`, add the below code after it.
```
#include "f_hid.h"
#include "f_hid_android_keyboard.c"
#include "f_hid_android_mouse.c"
```
Find the line `&uasp_function,`, add the below code after it.
```
&hid_function,
```
Find the line `/* Free uneeded configurations if exists */`, add the below code <strong>before</strong> it.
```
/* HID driver always enabled, it's the whole point of this kernel patch */
android_enable_function(dev, conf, "hid");
```
That's all. Save the file, then launch the following commands.
```
make blue_hayabusa_defconfig
make -j4
```
After it finishes, there will be a file called `zImage` in `android_kernel_sony_msm8x60/arch/arm/boot`. This is the kernel.

## Making boot.img
I use a simpler way that doesn't require compiling the whole ROM from source. Basic concept is replacing the kernel in an existing `boot.img` by using unpack/repack boot.img tool. The tricky part is finding the right tool that works with Sony's `boot.img` format. Luckily, I find [this tool](https://github.com/AdrianDC/multirom_libbootimg) working.<br><br>
#### First, clone the repo and `cd` into `multirom_libbootimg/src`, launch command `make` to compile it.<br><br>
Note: If you got this error
```
../include/libbootimg.h:14:25: fatal error: cutils/klog.h: No such file or directory`, 
```
Open `multirom_libbootimg/include/libbootimg.h`.<br>
Comment the line
```
#include <cutils/klog.h>
```
Find the line
```
#define LOG_DBG(fmt, ...) klog_write(3, "<3>%s: " fmt, "libbootimg", ##__VA_ARGS__)
```
Change it to this and compile again
```
#define LOG_DBG(fmt, ...) printf("[DBG] %s", fmt);
```

#### Then, extract `boot.img` from Lineage OS flashable zip, launch the below command (replace file paths with yours)
```
./bbootimg -u {path/to/unmodified/boot.img} -k {path/to/patched/zImage}
```

#### Use `fastboot` to flash the modified `boot.img`
* Some useful commands
```
sudo apt-get install android-tools-fastboot
sudo fastboot devices
sudo fastboot flash boot {path/to/boot.img}
```

## Stuff for debugging 
#### binary diff
```
diff -y <(xxd -s13440000 -l10000 bootimg/boot.img) <(xxd -s13440000 -l10000 bootimg/image-new.img) | colordiff
```
* -s{start} -l{length}
