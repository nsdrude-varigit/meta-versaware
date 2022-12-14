# Purpose

This Yocto layer includes the customizations needed to build Yocto and B2Qt images
for Versaware hardware.

# Building Images

To get started, the follow the steps below, which are identical to https://variwiki.com/index.php?title=B2QT_Build_Release&release=RELEASE_DUNFELL_B2QT_V1.2_VAR-SOM-MX8M-NANO, with the addition of adding meta-variscite to conf/bblayers.conf:

**The very first build:**

```
$ mkdir -p ~/var-b2qt/sources/
$ cd ~/var-b2qt/sources/
$ git clone https://github.com/varigit/meta-variscite-boot2qt.git -b dunfell-var01
$ git clone https://github.com/nsdrude-varigit/meta-versaware.git -b dunfell
$ cd ..
$ ./sources/meta-variscite-boot2qt/b2qt-init-build-env init --device imx8mn-var-som
$ MACHINE=imx8mn-var-som source ./setup-environment.sh
```

Add meta-versaware to conf/bblayers.conf:

```
BBLAYERS ?= " \
  ${BSPDIR}/sources/poky/meta \
  ${BSPDIR}/sources/poky/meta-poky \
  ${BSPDIR}/sources/meta-freescale \
  ${BSPDIR}/sources/meta-freescale-3rdparty \
  ${BSPDIR}/sources/meta-variscite-fslc \
  ${BSPDIR}/sources/meta-versaware \
  ${BSPDIR}/sources/meta-openembedded/meta-oe \
  ${BSPDIR}/sources/meta-openembedded/meta-python \
  ${BSPDIR}/sources/meta-openembedded/meta-networking \
  ${BSPDIR}/sources/meta-openembedded/meta-initramfs \
  ${BSPDIR}/sources/meta-openembedded/meta-multimedia \
  ${BSPDIR}/sources/meta-boot2qt/meta-boot2qt \
  ${BSPDIR}/sources/meta-boot2qt/meta-boot2qt-distro \
  ${BSPDIR}/sources/meta-mingw \
  ${BSPDIR}/sources/meta-qt6 \
  "s
```

Finally, build the image:

```
$ MACHINE=imx8mn-var-som bitbake b2qt-embedded-qt6-image
```

**Future builds:**

When returning to build again in the future, you only need to follow these steps:

```
$ cd ~/var-b2qt/
$ MACHINE=imx8mn-var-som source ./setup-environment.sh
$ MACHINE=imx8mn-var-som bitbake b2qt-embedded-qt6-image
```

# Program image to a recovery sd card:

Follow the guide here to build a recovery sd card: https://variwiki.com/index.php?title=B2QT_Build_Release&release=RELEASE_DUNFELL_B2QT_V1.2_VAR-SOM-MX8M-NANO#Create_an_extended_SD_card

After booting this sd card, run '# install_yocto.sh' to install to eMMC.

# Program image to eMMC using U-Boot UUU:

eMMC can be programmed by connecting a USB cable between the versaware type c
connector and the host computer:

1. Connect USB cable to PC
2. Interrupt the boot process in U-Boot and run `ums 0 mmc 2`
3. On host computer, use dmesg to determine the new device node and run:
```
 $ zcat tmp/deploy/images/imx8mn-var-som/b2qt-embedded-qt6-image-imx8mn-var-som.wic.gz | sudo dd of=/dev/sdX bs=1M && sync
```
4. Reset the board to run the new image

# Updating the kernel

To update the Linux kernel, change the commit id, branch, and/or git repository in [recipes-kernel/linux/linux-variscite_5.4.bbappend](recipes-kernel/linux/linux-variscite_5.4.bbappend)

# Hardware API

## USB OTG

The USB OTG connector is configured for peripheral mode in both U-Boot and Linux.
To use it, plug a USB Type C cable from your host computer to the carrier board.

### USB OTG U-Boot

In U-Boot, you can access and program the eMMC using:

```
u-boot> ums 0 mmc 2
```

On your host computer, look for a new device /dev/sdX to appear.


### USB OTG Linux

In Linux, you can use an ethernet or mass storage gadgets to access the device.
For more information, see: https://variwiki.com/index.php?title=DART-MX8M_USB_OTG&release=mx8mn-b2qt-hardknott-5.10.72_2.2.1-v1.0#Mass_Storage_Device

## LEDs

LEDs are controlled using /sys/class/leds. For example:

```
# echo 1 > /sys/class/leds/led1/brightness
# echo 0 > /sys/class/leds/led1/brightness
# echo 1 > /sys/class/leds/led2/brightness
# echo 0 > /sys/class/leds/led2/brightness
```

## Buzzer

The buzzer is controlled using /sys/class/pwm:

```
# echo 0 > /sys/class/pwm/pwmchip0/export
# echo 1000000 > /sys/class/pwm/pwmchip0/pwm0/period
# echo 500000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle
# echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable
# echo 0 > /sys/class/pwm/pwmchip0/pwm0/enable
```

## User Button

Use evtest utility. For more information, see https://variwiki.com/index.php?title=IMX_UserButtons

## MCP3221 Battery Monitor

**Usage:**

```
# gpioset gpiochip3 31=1
# cat /sys/bus/i2c/drivers/mcp3021/1-004e/in0_input
# gpioset gpiochip3 31=0
```

**Troubleshooting**

Verify i2c-device 0x4e:

```
root@b2qt-imx8mn-var-som:~# i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- UU -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --
```
Normal dmesg logs:

```
root@b2qt-imx8mn-var-som:~# dmesg | grep -i mcp
[    1.038397] mcp3021 1-004e: hwmon_device_register() is deprecated. Please convert the driver to use hwmon_device_register_with_info().
```

## MAX8808 Charger

GPIO will be controlled by user application. Use libgpiod to read/write
these lines:

| NET          | J1 | PAD        | pinctrl                            | gpiochip | gpio pin | i/o   | usage                  |
|--------------|----|------------|------------------------------------|----------|----------|-------|------------------------|
| CHG_Iset     | 40 | GPIO1_IO13 | MX8MN_IOMUXC_GPIO1_IO13_GPIO1_IO13 | 0        | 13       | out   | gpioset gpiochip0 13=1 |
| CHG_ACOK     | 52 | SAI3_TXC   | MX8MN_IOMUXC_SAI3_TXC_GPIO5_IO0    | 4        | 0        | input | gpioget gpiochip4 0    |
| CHG_EN       | 51 | SAI3_RXD   | MX8MN_IOMUXC_SAI3_RXD_GPIO4_IO30   | 3        | 30       | out   | gpioset gpiochip2 30=1 |
| CHG_CHGstate | 50 | SAI3_RXC/  | MX8MN_IOMUXC_SAI3_RXC_GPIO4_IO29   | 3        | 29       | input | gpioget gpiochip3 29   |

## HX711 Load Cell

**Usage:**
```
$ cat /sys/bus/iio/devices/iio\:device0/in_voltage0_raw
```