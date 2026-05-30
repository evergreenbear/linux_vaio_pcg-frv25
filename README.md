# linux_vaio_pcg-frv25
Tools for running Linux on the Sony Vaio PCG-FRV25 laptop

## Preamble
I have not found any content online of this machine running Linux at all. I cannot guarantee that you will face the same roadblocks on an identical model, nor can I claim that what works for my Vaio is guaranteed to work for yours, but I am sharing what has worked for me in hope that it will help another FRV25 owner down the line. They're neat little machines! ("little" very much being relative here)

The FRV25 has the following hardware:
- CPU: Intel Pentium 4 2.66GHz (Northwood)
- GPU: ATI Radeon IGP 340M (RS200), 64MB VRAM
- Chipset: ALi M1671 / M1535+
- PMU: ALi M7101 (PCI 0:6.0)
- Display: 15" XGA (1024x768) CCFL panel

## Backlight
The backlight dies immediately and completely on any Linux distribution that initializes ACPI. The image remains and can be seen when shining a flashlight at the display, but the CCFL inverter is disabled.
This is due to the ALi PMU. At bit 6 of the PCI register `0x9D` is the GPIO14 pin which directly enables or disables the CCFL inverter. When ACPI initializes and enumerates the PMU as a PCI device, something in that sequence clears bit 6 of this register, thereby flipping the GPIO pin and turning off the backlight. Presumably, Sony's Windows driver is intended to restore this bit, however I have not yet tested if this is the case.
The brightness level (PWM) is a separate concern from the enable signal and is controlled through the `sony_laptop` kernel module's sysfs interface. The enable pin being low overrides any brightness setting — even at maximum brightness the backlight will be off if GPIO14 is not asserted.



### Fix
The fix is a udev rule that fires the moment the `sony_laptop` backlight interface appears. It asserts GPIO14 via `setpci` to enable the inverter, then immediately sets maximum brightness via the sysfs interface.
Install the rule, then reload:

```bash
cp 99-vaio-backlight.rules /etc/udev/rules.d/
udevadm control --reload-rules
```

Requirements:
- `setpci` from the `pciutils` package
- `sony_laptop` kernel module (likely loaded automatically on your distro)


### Manual Recovery
If you're on a live system with a dead backlight and do not have the udev rule installed, you can restore the backlight by issuing the same commands manually:

```bash
# Enable the inverter
setpci -s 0:6.0 0x9D.B=40

# Set maximum brightness (requires sony_laptop to be loaded)
echo 7 > /sys/class/backlight/sony/brightness
```


### Attempted Fixes
The following approaches were tested and were not found to restore the backlight. I list them here so you don't waste time repeating them, or in the case that you can prove me wrong:

- `acpi=off` works (backlight survives) but disables all ACPI functionality including battery status, thermal management, and power states
- `acpi=noirq` does not help; the GPIO is cleared before EC event processing begins
- `nomodeset` does not help; the backlight dies before any display driver loads
- `radeon.backlight=0` does not help; the radeon driver is not involved
- Blacklisting `acpi_video`, `sony_laptop`, `radeon`, `radeonfb`, or `drm` does not help; the GPIO is cleared during PCI device initialization which predates any of these drivers
- `acpi_backlight=native` or `acpi_backlight=vendor` do not help
- `video=LVDS-1:e` does not help
- DSDT patching will not help; the FRV25's DSDT tables do not contain any hardware manipulation of the backlight

### Scope
On my machine, this affects every operating system which uses ACPI. I was able to boot Haiku and install plain NetBSD without losing backlight, but (as of now) I belive these did not initialize ACPI. 
The udev rule has been tested on Alpine linux, but should work on any distro using udev.
The ULi M7101, a later rebranding of the same chip, uses the same registers and the same fix should apply.
