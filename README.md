LED Controller of UGREEN's DX4600 Pro NAS
==

UGREEN's DX4600 Pro is a four-bay NAS with a built-in system based on OpenWRT called UGOS. It can install Debian or other open-source NAS systems, but the issue is that the installed system does not have drivers for the six LED lights on the front panel (indicating power, network card, and four hard drives). By default, only the power indicator light flashes, and other indicator lights cannot be controlled.

This repository describes the control logic of UGOS for these LED lights and provides a Linux command-line tool to control them. For the process of understanding this control logic, please refer to [my blog (in Chinese)]().

## Preparation

UGOS communicates with the control chip of the LED via I2C, corresponding to the device with address 0x3a on `/dev/i2c-1`. Before proceeding, you first need to load the `i2c-dev` module and install the `i2c-tools` tool.

```
$ apt install -y i2c-tools
$ modprobe -v i2c-dev
```

Now, you can check if the device located at address 0x3a is visible.

```
$ i2cdetect -l
i2c-0   i2c             Synopsys DesignWare I2C adapter         I2C adapter
i2c-1   smbus           SMBus I801 adapter at efa0              SMBus adapter
i2c-2   i2c             Synopsys DesignWare I2C adapter         I2C adapter

$ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         08 -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: 30 -- -- -- -- 35 UU UU -- -- 3a -- -- -- -- --
40: -- -- -- -- 44 -- -- -- -- -- -- -- -- -- -- --
50: UU -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

## Build

After cloning the current repository, run `make` to build this project. Once the build is complete, you can use `dx4600_leds_cli` to modify the LED states (requires root permissions).

## Usage

```
Usage: dx4600_leds  [LED-NAME...] [-on] [-off] [-blink T_ON T_OFF]
                    [-color R G B] [-brightness BRIGHTNESS] [-status]

       LED_NAME:    separated by white space, possible values are
                    { power, netdev, disk1, disk2, disk3, disk4, all }.
       -on / -off:  turn on / off corresponding LEDs.
       -blink:      set LED to the blink status. This status keeps the
                    LED on for T_ON millseconds and then keeps it off
                    for T_OFF millseconds.
                    T_ON and T_OFF should belong to [0, 65535].
       -color:      set the color of corresponding LEDs.
                    R, G and B should belong to [0, 255].
       -brightness: set the brightness of corresponding LEDs.
                    BRIGHTNESS should belong to [0, 255].
       -status:     display the status of corresponding LEDs.
```

## Communication Protocols

The IDs for the six LED lights on the front panel of the chassis are as follows: 

- power indicator light = 0;
- network device indicator light = 1;
- four hard drive indicator lights = 2 - 5.

### Query status

Reading 11 bytes from the address `0x81 + LED_ID` allows you to obtain the current status of the corresponding LED. The meaning of these 11 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| 0x00    | LED status: 0 - off, 1 - on, 2 - blink, 3 - breath |
| 0x01    | LED brightness |
| 0x02    | LED color (Red component in RGB) |
| 0x03    | LED color (Green component in RGB) |
| 0x04    | LED color (Blue component in RGB) |
| 0x05    | Milliseconds needed to complete one blink cycle (high 8 bits) |
| 0x06    | Milliseconds needed to complete one blink cycle (low 8 bits) |
| 0x07    | Milliseconds the LED is on during one blink cycle (high 8 bits) |
| 0x08    | Milliseconds the LED is on during one blink cycle (low 8 bits) |
| 0x09    | Checksum of data in the range 0x00 - 0x08 (high 8 bits) |
| 0x0a    | Checksum of data in the range 0x00 - 0x08 (low 8 bits) |

The checksum is a 16-bit value obtained by summing all the data at the corresponding positions as unsigned integers.

You can directly use i2cget on the command line to read from the relevant registers. For example, below is the status of the power indicator light (purple, blinking once per second, lit for 40% of the time, with a brightness of 180/256):

```
$ i2cget -y 0x01 0x3a 0x81 i 0x0b
0x02 0xb4 0xff 0x00 0xff 0x03 0xe8 0x01 0x90 0x04 0x30
```

### Change status

By writing 12 bytes to the address `0x00 + LED_ID`, you can modify the current status of the corresponding LED. The meaning of these 12 bytes is as follows:

| Address | Meaning of Corresponding Data |
|---------|--------------------------------|
| 0x00    | LED ID |
| 0x01    | Constant: 0xa0 |
| 0x02    | Constant: 0x01 |
| 0x03    | Constant: 0x00 |
| 0x04    | Constant: 0x00 |
| 0x05    | If the value is 1, it indicates modifying brightness.
            If the value is 2, it indicates modifying color.
            If the value is 3, it indicates setting the on/off state.
            If the value is 4, it indicates setting the blink state. |
| 0x06    | First parameter |
| 0x07    | Second parameter |
| 0x08    | Third parameter |
| 0x09    | Fourth parameter |
| 0x0a    | Checksum of data in the range 0x01 - -0x09 (high 8 bits) |
| 0x0b    | Checksum of data in the range 0x01 - -0x09 (low 8 bits) |

For the four different modification types at address 0x05:

- If you need to modify brightness, the first parameter contains brightness information.
- If you need to modify color, the first three parameters represent RGB information.
- If you need to toggle the on/off state, the first parameter is either 0 or 1, representing off or on, respectively.
- If you need to set the blink state, the first two parameters together form a 16-bit unsigned integer in big-endian order, representing the number of milliseconds needed to complete one blink cycle. The next two parameters, also in big-endian order, represent the number of milliseconds the LED is on during one blink cycle.

If using the `i2cset` provided by `i2c-tools`, the content of 0x00 should be ignored. The actual command requires writing the following 11 bytes. Below is an example command line for turning off and on the power indicator light:

```
$ i2cset -y 0x01 0x3a 0x00  0xa0 0x01 0x00 0x00 0x03 0x01 0x00 0x00 0x00 0x00 0xa5 s    # turn off power LED
$ i2cset -y 0x01 0x3a 0x00  0xa0 0x01 0x00 0x00 0x03 0x00 0x00 0x00 0x00 0x00 0xa4 s    # turn on power LED
```