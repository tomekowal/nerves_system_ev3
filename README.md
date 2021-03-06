# Lego Mindstorms EV3

[![CircleCI](https://circleci.com/gh/nerves-project/nerves_system_ev3.svg?style=svg)](https://circleci.com/gh/nerves-project/nerves_system_ev3)
[![Hex version](https://img.shields.io/hexpm/v/nerves_system_ev3.svg "Hex version")](https://hex.pm/packages/nerves_system_ev3)

This is the base Nerves System configuration for the Lego Mindstorms EV3 brick.

![EV3 brick image](assets/images/lego-mindstorms-ev3.jpg)
<br><sup>[Image credit](#wikipediaref)</sup>

| Feature              | Description                     |
| -------------------- | ------------------------------- |
| CPU                  | 300 MHz ARM926EJ-S              |
| Memory               | 64 MB DRAM                      |
| Storage              | 16 MB Flash and MicroSD         |
| Linux kernel         | 4.4 w/ ev3dev patches           |
| Display              | 178x128 LCD - /dev/fb0          |
| IEx terminal         | USB Gadget port or port 1 (see below) |
| GPIO, I2C, SPI       | Yes - ev3dev drivers            |
| ADC                  | Yes                             |
| PWM                  | Yes, but no Elixir support      |
| UART                 | 4 available - ttyACM0           |
| Speakers             | Built-in speaker - ALSA         |
| Camera               | None                            |
| Ethernet             | Requires USB Ethernet dongle    |
| WiFi                 | Requires USB WiFi dongle        |
| Bluetooth            | Not supported                   |

## Using

The most common way of using this Nerves System is create a project with `mix
nerves.new` and to export `MIX_TARGET=ev3`. See the [Getting started
guide](https://hexdocs.pm/nerves/getting-started.html#creating-a-new-nerves-app)
for more information.

If you need custom modifications to this system for your device, clone this
repository and update as described in [Making custom
systems](https://hexdocs.pm/nerves/systems.html#customizing-your-own-nerves-system)

I recommend creating a project that uses
[nerves_init_gadget](https://github.com/nerves-project/nerves_init_gadget) to
get started on the EV3. This makes using the EV3 similar to the Raspberry Pi
Zero so more help will be available on the Nerves slack channel and forum.

Ethernet for uploading new firmware images is provided through the USB gadget
port (labelled "PC" on the EV3). If you connect this via a USB cable to your
laptop and turn on the EV3, you should eventually see a new USB and Ethernet
port. (The first boot of Nerves on the EV3 takes some time.)

Unfortunately, the console doesn't appear to work via the Gadget USB port. I
haven't had time to debug this, so I left it going through `ttyS1` which comes
out port 1. This is very sad. If you have a clue on how to fix this, see
`rootfs_overlay/etc/erlinit.config` to route the console through `ttyGS0` and
let me know.

The Lego device kernel modules are not built into the kernel so they need to be
loaded by your application at initialization time. To do this, run the following
manually or add them to your application:

```elixir
iex> :os.cmd('modprobe suart_emu')
iex> :os.cmd('modprobe legoev3_ports')
iex> :os.cmd('modprobe snd_legoev3')
iex> :os.cmd('modprobe legoev3_battery')
```

When Nerves supports Bluetooth, you'll want to run the following line as well:

```elixir
iex> :os.cmd('modprobe legoev3_bluetooth')
```

Once you get to the console and can update firmware, you should be able to
follow ev3dev instructions for reading and writing to files to control motors,
read sensors, and everything else that the EV3 can do.

## Example projects

Since the documentation is sparse here, you may find some example projects
helpful:

* [nerves_ev3_example](https://github.com/fhunleth/nerves_ev3_example)

If you have a project to share, please help us by adding it to the list and
sending a pull request. Thanks!

## Port 1 console access

The EV3 supports a special UART output on port 1. Nerves can use this output for
Linux kernel debug messages and an IEx prompt. If you plan on doing any Linux
kernel, driver, or boot related work with the EV3, this console is
indispensable.

You will either need to buy a [console
adapter](http://www.mindsensors.com/ev3-and-nxt/40-console-adapter-for-ev3) or
build [one](http://botbench.com/blog/2013/08/15/ev3-creating-console-cable/) to
use this port.

Loading the `legoev3_ports` driver automatically disables the console port.
Since we're working on the EV3 and it's not as easy to use as it should be,
we've told the `legoev3_ports` module to not touch it. If you're on the EV3,
you'll see the following line in `/etc/modprobe.d/ev3dev.conf`:

    options legoev3_ports disable_in_port=1

If you want to use port 1, you'll need to disable this. To do this, add a
`/etc/modprobe.d/ev3dev.conf` to your project's `rootfs_overlay`. If an empty
file exists, it will override this default one, but I usually create a file with
the line commented out so that I remember what the special line is.

## Supported USB WiFi devices

The base image includes drivers and firmware for Ralink RT53xx (`rt2800usb`
driver), MediaTek MT7601U (`mt7601u`), and Edimax EW-7811Un (`8192cu`) devices.
One option for these devices is to get a Tenda W311MI Wireless USB Adapter.

We are still working out which subset of all possible WiFi dongles to support in
our images. At some point, we may have the option to support all dongles and
selectively install modules at packaging time, but until then, these drivers and
their associated firmware blobs add significantly to Nerves release images.

If you are unsure what driver your WiFi dongle requires, run Raspbian and
configure WiFi for your device. At a shell prompt, run `lsmod` to see which
drivers are loaded.  Running `dmesg` may also give a clue. When using `dmesg`,
reinsert the USB dongle to generate new log messages if you don't see them.

## Wired Ethernet

If you have a USB Ethernet adapter, find the driver for it on your PC. For
example, plug it in and check `dmesg` and `lsmod` to see which driver it loads.
In my case, I have an adapter that loads the `asix` driver. Make sure that your
driver is compiled in as a module to the Linux kernel in Nerves and then
manually load the driver via `modprobe asix`.

## Power down, halt, reboot

On most platforms, the default behavior of Nerves is to hang if something goes
wrong.  This is good for debugging since rebooting makes it easy to lose console
messages or it might hide the issue completely. On the EV3 hanging requires a
slightly more complex restarting process.

1. Hold down the Back, and center buttons on the EV3 Brick
1. When the screen goes blank, release the Back button
1. When the screen says “Starting,” release the center button

If you would like to power down instead you will need to change `erlinit.config`
from `--hang-on-exit` to `---poweroff-on-exit`.

If you're attached to the console, you may see a kernel panic when you run power
off. From what I can tell, this panic happens after the important parts of
shutting down gracefully have completed and does not cause a problem.

## SDCard vs. internal NAND Flash notes

The EV3 brick has a 16 MB NAND Flash inside it that's connected to SPI bus 0.
It doesn't look like the ev3dev project has included support for it yet except
in their version of u-boot. The means that it can only be programmed using the
Lego supplied tools. The 16 MB NAND Flash also has a couple other issues. First,
it appears to be super slow. This leads to them copying the whole image to DRAM
instead of reading it as needed. It appears that this uses up 10 MB of DRAM
compared to running off the SDCard. This is significant when you consider that
the board only has 64 MB total DRAM. On the other hand, programming the internal
NAND Flash is cool and the direction that we'd prefer to go on production
systems.

Currently, the u-boot in the internal NAND Flash that's supplied by Lego and the
ev3dev project expects the `uImage` in the first VFAT partition. Ideally, it
would extract it out of the rootfs so that we could implement more atomic
firmware updates. To avoid the need to reflash the firmware to use Nerves, I'm
staying with the existing mechanism.

## ev3dev

This port draws substantially on the [ev3dev](http://www.ev3dev.org/) project.
In general, if there's a way to do something in ev3dev, it can be made to work
in Nerves. Nerves uses the same Linux kernel from the ev3dev project and enables
the same set of custom drivers that were created by the ev3dev developers.

## Provisioning devices

This system supports storing provisioning information in a small key-value store
outside of any filesystem. Provisioning is an optional step and reasonable
defaults are provided if this is missing.

Provisioning information can be queried using the Nerves.Runtime KV store's
[`Nerves.Runtime.KV.get/1`](https://hexdocs.pm/nerves_runtime/Nerves.Runtime.KV.html#get/1)
function.

Keys used by this system are:

Key                    | Example Value     | Description
:--------------------- | :---------------- | :----------
`nerves_serial_number` | "1234578"`        | By default, this string is used to create unique hostnames and Erlang node names. If unset, it defaults to part of the EV3's device ID.

The normal procedure would be to set these keys once in manufacturing or before
deployment and then leave them alone.

For example, to provision a serial number on a running device, run the following
and reboot:

```elixir
iex> cmd("fw_setenv nerves_serial_number 1234")
```

This system supports setting the serial number offline. To do this, set the
`NERVES_SERIAL_NUMBER` environment variable when burning the firmware. If you're
programming MicroSD cards using `fwup`, the commandline is:

```sh
sudo NERVES_SERIAL_NUMBER=1234 fwup path_to_firmware.fw
```

Serial numbers are stored on the MicroSD card so if the MicroSD card is
replaced, the serial number will need to be reprogrammed. The numbers are stored
in a U-boot environment block. This is a special region that is separate from
the application partition so reformatting the application partition will not
lose the serial number or any other data stored in this block.

Additional key value pairs can be provisioned by overriding the default provisioning.conf
file location by setting the environment variable 
`NERVES_PROVISIONING=/path/to/provisioning.conf`. The default provisioning.conf
will set the `nerves_serial_number`, if you override the location to this file,
you will be responsible for setting this yourself.

[Image credit](#wikipediaref): By Klaus-Dieter Keller - Own work, CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=29156877
