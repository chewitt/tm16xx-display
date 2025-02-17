# tm16xx-display
Linux kernel driver for auxiliary displays based on led controllers such as tm16xx family and alike

## Supported Android TV box
See [Device Table](DEVICES.md)

## Compatible Controllers

*Other similar controllers may already be compatible using one of these identifiers.*

* Shenzhen TITAN MICRO Electronics
  * titanmec,tm1618 ([datasheet](datasheets/TM1618_V2.1_EN.pdf))
  * titanmec,tm1620 ([datasheet](datasheets/TM1620_V2.1_EN.pdf))
  * titanmec,tm1628 ([datasheet](datasheets/TM1628_V1.1_EN.pdf))
  * titanmec,tm1650 ([datasheet](datasheets/TM1650_V2.2_EN.pdf))
* FUDA HISI MICROELECTRONICS
  * fdhisi,fd620 (no datasheet)
  * fdhisi,fd628 (no datasheet)
  * fdhisi,fd650 ([datasheet](datasheets/FD650.pdf))
  * fdhisi,fd655 ([datasheet](datasheets/FD655.pdf))
  * fdhisi,fd6551 ([datasheet](datasheets/FD6551.pdf))
* Princeton Technology Corp
  * princeton,pt6964 ([datasheet](datasheets/PT6964.pdf))
* Unknown
  * hbs,hbs658 (no datasheet)

# Installation Instructions

## Prerequisites
### Building the driver
Linux kernel headers installed
```sh
armbian-config
```
Then go to Software -> Headers

### Display service
To blink the time seperator
- `timer` led triger (`ledtrig_timer` module)

To manage lan, wlan and bluetooth LEDs:
- `netdev` led triger (`ledtrig_netdev` module)

As well as built-in led triggers:
- `usbport` for the usb port
- `mmc0` for the sd card

## Download
```sh
git clone https://github.com/jefflessard/tm16xx-display.git
```

## Configure the device tree
:warning: **KEEP A BACKUP OF YOUR CUREENT DTB**

1. Find your device in the [Device Table](DEVICES.md)

2. Update your dtb

*Option 1: Use device tree overlay, if supported*
  * Build overlay
```sh
# This will create the overlay in release/{YOUR_DEVICE_NAME}.dtbo
make {YOUR_DEVICE_NAME}.dtbo 
```
   * Copy dtbo in `/boot/overlay-user/`
```sh
cp release/{YOUR_DEVICE_NAME}.dtbo /boot/overlay-user/tm16xx.dtbo
```

   * Edit `/boot/armbianEnv.txt` to load the overlay
```
user_overlays=tm16xx
```

*Option 2: Create an updated dtb*
  * Copy your current dtb file to `original.dtb`:

```sh
# run this command only once.
# we must always start from the
# original dtb when merging overlay
make extract-dtb ORIGINAL_DTB=original.dtb
```

  * Merge the display dtb overlay with your current dtb
```sh
# This will create the dtb in release/{YOUR_DEVICE_NAME}.dtb
make {YOUR_DEVICE_NAME}.dtb ORIGINAL_DTB=original.dtb

# Replace your current dtb with the new dtb, for example:
#cp release/{YOUR_DEVICE_NAME}.dtb /boot/dtb/{YOUR_DTB_PATH}.dtb
```

3. Reboot to apply changes
```sh
reboot
```

## Kernel module and display service
*Option 1: build and install in a single command*

Builds then installs module and service
```sh
make install
```

*Option 2: step by step commands*
```sh
make module
make module-install
make service-install
```

## Check your display configuration
```sh
display-utils -c
```
3 phase display check
1. Check that all leds are ON
2. Check the order of digits and segment mapping (you should see "1234")
3. Check each led name (ex: LAN icon is ON while "LAN" text is shown on the digits)

In case you want to experiment with alternative digits ordering or segment mapping, you can update them from user space without editing the dtb or rebooting. This may be useful to test your configuration changes before editing the dtb.

### Assisted segments and digits disovery
```sh
display-utils -a
```
Interactive prompt to quickly and easily identify segments mapping and digits ordering, plus will generate the related device tree configuration.

Example session:
```
Segment Mapping:
Original segments: [0 1 2 3 4 5 6]

   --A--
  |     |
  F     B
  |     |
   --G--
  |     |
  E     C
  |     |
   --D--

Enter blinking segment 0 (A-G or empty): d
Enter blinking segment 1 (A-G or empty): e
Enter blinking segment 2 (A-G or empty): f
Enter blinking segment 3 (A-G or empty): a
Enter blinking segment 4 (A-G or empty): b
Enter blinking segment 5 (A-G or empty): c
Enter blinking segment 6 (A-G or empty): g
All 7 segments have been mapped.
Segment mapping: [3 4 5 0 1 2 6]

Validating digit order
Original digits: [2 1 4 3]

  1234

Enter the position of digit 1 (1-4): 2
Enter the position of digit 2 (1-4): 1
Enter the position of digit 3 (1-4): 4
Enter the position of digit 4 (1-4): 3
Digit order: [1 2 3 4]

Update your device tree configuration to:

        tm16xx,digits = [1 2 3 4];
        tm16xx,segment-mapping = [3 4 5 0 1 2 6];

```

### Manual alternative digits ordering
```sh
# show current digit ordering
cat /sys/class/leds/display/digits

# update digit ordering
echo "1 2 3 4" > /sys/class/leds/display/digits

# check configuration
display-utils -c
```

### Manual alternative segment mapping
```sh
# show current segment mapping
cat /sys/class/leds/display/segments

# update segment mapping
echo "0 1 2 3 4 5 6" > /sys/class/leds/display/segments

# check configuration
display-utils -c
```

# Advanced Device Configuration

## Convert existing [OpenVFD](https://github.com/arthur-liberman/linux_openvfd/tree/master) vfd.conf
Existing compatible [OpenVFD](https://github.com/arthur-liberman/linux_openvfd/tree/master) [vfd-configurations](https://github.com/arthur-liberman/vfd-configurations/) are already converted. Find them in the [Device Table](DEVICES.md)

### Convert a single vfd.conf
```sh
./vfdconf-convert {path_to_your_vfd.conf_file} devices/{your_device_name}.dtso
```

### Convert multiple vfd.conf
```sh
./vfdconf-convert -r {path_to_vfd-configurations_directory} devices
```

## Create your own configuration
Create a .dtso file in `devices` directory
  * `display-client`
    * Option 1 : 3-wire serial (SPI) controller
      * `compatible = "spi-gpio"`
      * `mosi-gpios`: data gpio pin
      * `gpio-sck`: clock gpio pin
      * `cs-gpios`: chip select gpio pin
    * Option 2 : 2-wire serial (I2C) controller
      * `compatible = "i2c-gpio"`
      * `sda-gpios`: data gpio pin
      * `scl-gpios`: clock gpio pin
  * `display-controller`
    * `compatible`: your display controller chip
    * `tm16xx,transposed` : optional flag indicating that the data matrix should be transposed when writing to the controller (grids and segments are inverted), required on some devices
    * `tm16xx,digits`: variable lengh byte array determining the number of text grid digits and their index position 
    * `tm16xx,segment-mapping`: array of 7 bytes specifying which bit of a grid digit should be used for each ascii map segment
  * `led@X,Y`
    * X: grid cell index
    * Y: segment index
    * `reg`: must match `<X Y>` above
    * `function`: sets the sysfs name of the led


# Usage

## Service basics
Start:
```sh
systemctl start display
```

Stop:
```sh
systemctl stop display
```

Restart:
```sh
systemctl restart display
```

## Show scrolling text on the display
```sh
display-utils -t "{your_message}"
```

## Customize display service
Just edit the bash script at `/sbin/display-service`

## Customize display from shell
```sh
# turn on display
cat /sys/class/leds/display/max_brightness > /sys/class/leds/display/brightness

# dim brightness display
# value between 1 and max_brightness (usually 8)
echo 1 > /sys/class/leds/display/brightness

# turn off display
echo 0 > /sys/class/leds/display/brightness

# write text on the display (supports 7-segment ascii mapping)
echo "boot" > /sys/class/leds/display/value

# clear the display text
echo > /sys/class/leds/display/value

# list available leds/symbols
ls /sys/class/leds/display\:\:*

# turn on a specific led/symbol
echo 1 > /sys/class/leds/display\:\:lan/brightness

# turn off a specific led/symbol
echo 0 > /sys/class/leds/display\:\:lan/brightness

# automatically turn on/off usb led when usb device is connected on a specific port
echo usbport > /sys/class/leds/display::usb/trigger
echo 1 > /sys/class/leds/display::usb/ports/usb1-port1

# turn on led on wifi connect + blink on activity (requires ledtrig-netdev module)
echo netdev > /sys/class/leds/display::wlan/trigger
echo wlan0 > /sys/class/leds/display::wlan/device_name
echo 1 > /sys/class/leds/display::wlan/link
echo 1 > /sys/class/leds/display::wlan/rx
echo 1 > /sys/class/leds/display::wlan/tx
```
