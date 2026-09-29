# nerves_system_core3566

Nerves system for the Core3566 SoM (Rockchip RK3566, quad-core Cortex-A55,
CM4-compatible DF40 edge connector).

This is a from-scratch port, adapted from
[`nerves_system_radxa_cm3`](https://github.com/sportalliance/nerves_system_radxa_cm3) --
an existing Nerves system for the Radxa CM3, which uses the *same* RK3566 SoC
and the *same* CM4-form-factor DF40C-100DS connector as the Core3566. That
made it a much closer starting point than a generic Buildroot board file
would have been: the SoC-level bring-up (Buildroot toolchain, kernel/U-Boot
pinning, GPIO bank layout) carries over directly, and only the
module/carrier-specific device tree bits needed rework.

## Status: builds the Elixir/Buildroot scaffold, hardware bring-up incomplete

This has not been booted on real Core3566 hardware. Treat the confidence
notes below as a map of what's been mechanically adapted vs. what still
needs verification against real hardware or vendor schematics.

**Carried over from the working `nerves_system_radxa_cm3` base (high
confidence -- same SoC):**

* Buildroot/toolchain plumbing, Linux 6.1.39 kernel pin, custom U-Boot
  (`pgwipeout/u-boot-quartz64.git`) pin, `rkbin` package for DDR init/ATF
  blobs
* GPIO bank layout (`gpio0`-`gpio4`, `gpio-ranges`), USB host/OTG PHY wiring,
  eMMC (`sdhci`) 8-bit HS200 configuration
* Ethernet: RTL8211F RGMII wiring via `gmac1`/`mdio1` -- the Core3566
  datasheet confirms an on-board RTL8211F, matching the CM3 base as-is

**Adapted using data straight from the Core3566 datasheet's pinout table
(medium-high confidence -- module pins are documented, but unverified on
hardware):**

* WiFi/BT enable pinctrl: `WL_nDisable` -> GPIO2_B1 (module pin 89),
  `BT_nDisable` -> GPIO2_B7 (module pin 91). Neither is wired to an
  `mmc-pwrseq`/bluetooth node yet -- see "Known gaps" below.
* Activity LED: `Pi_nLED_Activity` -> GPIO0_C5 (module pin 21)

**Known gaps (need a real schematic or hardware to close):**

* **Power/PMIC.** The datasheet documents the module's 200-pin edge
  connector, not the on-board "Power IC" visible in the board photo. Its
  identity and real regulator rails are unknown, so `linux/rk3566-core3566.dts`
  stands in with `regulator-fixed` nodes at RK3566's nominal voltages
  (`vdd_cpu`/`vdd_gpu`/`vdd_logic` at 0.9V, etc.) instead of a PMIC driver
  copied from the CM3's RK817. This should boot, but CPU/GPU voltage will
  **not** scale with frequency until the real PMIC is identified and wired
  up -- don't assume full 1.8GHz is safe under this config without checking.
* **WiFi/BT driver stack.** AP6256 (per datasheet) needs `brcmfmac`
  firmware + an `mmc-pwrseq-simple`/bluetooth node pointed at the pins
  above. Not wired up yet.
* **Carrier-board specifics** (buttons, extra LEDs, display/camera wiring)
  are entirely carrier-dependent and not covered here -- the CM3 base's
  `pi-led-green`/`pwr-led-red` nodes were dropped rather than carried over
  unverified.

## Building

```sh
mix deps.get
mix compile
```

## Using

The most common way of using this Nerves System is to create a project with
`mix nerves.new` and export `MIX_TARGET=core3566`. See the [Getting started
guide](https://hexdocs.pm/nerves/getting-started.html#creating-a-new-nerves-app)
for more information.

## Flashing

*These instructions have to be executed in the project folder, **not** the
nerves_system folder!*

1. Clone, build and install [rkdeveloptool](https://github.com/rockchip-linux/rkdeveloptool.git)
   (or install `rkflashtool` from your Linux distro).
1. Compile the project (`mix firmware`).
1. Create the image file:
   `fwup -a -d _build/<target name>/nerves/images/image.img -i _build/<target name>/nerves/images/<project name>.fw -t complete`
1. Put the board in MASKROM mode and connect it via USB to the PC. Per the
   Core3566 datasheet: hold `nRPIBOOT` (module pin 93) low at power-on, or
   -- if no SD card is connected -- hold the carrier board's boot button
   (tied to `eMMC_D0`, module pin 68) low at power-on. The exact mechanism
   depends on your carrier board.
1. `sudo rkdeveloptool db support_files/rk356x_spl_loader_ddr1056_v1.10.111.bin`
1. `sudo rkdeveloptool wl 0 _build/<target name>/nerves/images/image.img`
1. `sudo rkdeveloptool rd`

## GPIO Numbering

The RK3566 has multiple IO banks, GPIO0 to GPIO4, with each bank having 32
pins. These banks are further divided into 4 sub-banks, GPIOx_A to GPIOx_D:
```
GPIO0_A0 - GPIO0_A7
GPIO0_B0 - GPIO0_B7
GPIO0_C0 - GPIO0_C7
GPIO0_D0 - GPIO0_D7
GPIO1_A0 - GPIO1_A7
.
.
.
GPIO4_D0 - GPIO4_D7
```

The resulting pin number can be calculated using the following formula:
```
number = <bank>*32 + <sub_bank>*8 + pin
sub_bank = <A = 0, B = 1, C = 2, D = 3>

e.g.
GPIO3_A1 = 97 (3*32 + 0*8 + 1)
GPIO4_D7 = 159 (4*32 + 3*8 + 7)
```

The Core3566 datasheet's Chapter 4 (Pinout) gives the module pin -> GPIO
mapping for all 28 edge-connector GPIO signals.

## Console access

The console is currently not configured. It can be enabled by modifying the
boot arguments in `uboot/vars.txt`:
```
extra_bootargs="earlyprintk console=ttyS2,1500000n8"
```

## Provisioning devices

This system supports storing provisioning information in a small key-value
store outside of any filesystem. Provisioning is an optional step and
reasonable defaults are provided if this is missing.

Provisioning information can be queried using the Nerves.Runtime KV store's
[`Nerves.Runtime.KV.get/1`](https://hexdocs.pm/nerves_runtime/Nerves.Runtime.KV.html#get/1)
function.

Keys used by this system are:

Key                    | Example Value     | Description
:--------------------- | :---------------- | :----------
`nerves_serial_number` | `"12345678"`       | By default, this string is used to create unique hostnames and Erlang node names. If unset, it defaults to part of the module's serial number.

The normal procedure would be to set these keys once in manufacturing or
before deployment and then leave them alone.

For example, to provision a serial number on a running device, run the
following and reboot:

```elixir
iex> cmd("fw_setenv nerves_serial_number 12345678")
```

Serial numbers are stored on the MicroSD card so if the MicroSD card is
replaced, the serial number will need to be reprogrammed. The numbers are
stored in a U-boot environment block. This is a special region that is
separate from the application partition so reformatting the application
partition will not lose the serial number or any other data stored in this
block.

Additional key value pairs can be provisioned by overriding the default
provisioning.conf file location by setting the environment variable
`NERVES_PROVISIONING=/path/to/provisioning.conf`. The default
provisioning.conf will set the `nerves_serial_number`; if you override the
location to this file, you will be responsible for setting this yourself.

## Linux, U-Boot and Toolchain versions

This Nerves system uses Linux 6.1.39 and a custom U-Boot repository
(`pgwipeout/u-boot-quartz64.git`) with patches for RK3566 chips.

Don't update the toolchain for now -- newer toolchains cause a kernel panic
because the glibc version is too high (carried over from the CM3 base's
`NOTES.md`).

## Device tree

`linux/rk3566-core3566.dts` is a custom out-of-tree device tree, built via
`BR2_LINUX_KERNEL_CUSTOM_DTS_PATH`. See the "Status" section above for what's
verified vs. still a placeholder.
