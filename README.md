# Dell Laptop Fans Not Spinning on Linux — Fix

## TL;DR

If your Dell laptop fans **never spin in Linux** but work fine in Windows, add this to your GRUB config:

```bash
sudo nano /etc/default/grub
```

Change:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```
To:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_osi=!\ acpi_osi='Windows 2022'"
```

Then:
```bash
sudo update-grub
sudo reboot
```

That's it. No extra software needed.

---

## The Problem

Fans **never spin under Linux** — not during POST, not in GRUB, not on the desktop. The CPU hits 100°C and thermal throttles constantly. Meanwhile, the exact same fans work perfectly when booting into Windows.

```
$ sensors
Package id 0: +100.0°C  (high = +100.0°C, crit = +100.0°C)  ALARM (CRIT)
Core 0:        +96.0°C   ALARM (CRIT)
Core 8:       +100.0°C   ALARM (CRIT)
fan1:           0 RPM
```

### After the fix:

```
$ sensors
Package id 0:  +64.0°C
fan1:        4784 RPM
```

---

## Why This Happens

Modern Dell laptops use **Intel DPTF** (Dynamic Platform and Thermal Framework) to control the fans. On Windows, the **Intel DTT** (Dynamic Tuning Technology) driver manages this.

When the BIOS detects that the OS is **not Windows**, it simply **does not initialize the thermal table that controls the fans**. The fans stay off permanently — not because they're broken, but because the BIOS never tells the Embedded Controller to turn them on.

The parameter `acpi_osi='Windows 2022'` tells the BIOS "I am Windows", which makes it activate the full thermal framework including automatic fan control.

---

## Confirmed Hardware

This fix was diagnosed and confirmed on the following system:

| Component | Detail |
|---|---|
| **Laptop** | Dell 15 DC15250 |
| **SKU** | 0D77 |
| **CPU** | 13th Gen Intel Core i5-1334U (Raptor Lake) |
| **Motherboard** | 0MWM8R |
| **BIOS** | 1.6.0 (2025-12-02) |
| **Chipset** | Intel Raptor Lake LPC/eSPI Controller |
| **Vendor** | Dell Inc. |
| **Chassis** | Notebook (type 10) |

### Tested on:

| OS | Kernel | Status |
|---|---|---|
| Linux Mint 22.3 (Zena) | 6.17.0-14-generic | **Working** |

### Likely also applies to:

- Other **Dell Inspiron / Vostro / Latitude** models from 2023+ with Intel 13th Gen (Raptor Lake)
- Dell laptops where `i8kutils`, `dell_smm_hwmon`, and `smbios-thermal-ctl` all fail
- Dell laptops that use **Intel DPTF/DTT** for thermal management
- Any Dell where fans work in Windows but are completely dead in Linux

> **If this fix worked for your Dell model, please open an issue or PR to add it to the confirmed list above.** This helps other users find this solution.

---

## How We Found This

The diagnosis took 10 different approaches before finding the root cause. This section documents every method attempted, in order, so anyone debugging a similar issue can skip straight to the solution or use this as a reference.

### Method 1 — i8kutils / dell_smm_hwmon (SMM Legacy)

The traditional approach for Dell fan control on Linux.

```bash
sudo apt install i8kutils
sudo modprobe dell_smm_hwmon restricted=0 force=1 ignore_dmi=1
```

**Result:** The module loaded but `/proc/i8k` returned `-22` for all fields. `i8kfan` returned `-1 -1`. The DC15250 is not in the driver's whitelist and legacy SMM calls don't work with this EC.

```
$ cat /proc/i8k
1.0 1.6 CKHZHD4 -22 -22 -22 -22 -22 -1 -22

$ i8kfan
-1 -1
```

Kernel log:
```
dell_smm_hwmon: Forcing legacy SMM calls on a possibly incompatible machine
```

### Method 2 — dell-smbios / smbios-thermal-ctl (WMI)

The modern WMI-based interface for Dell thermal management.

```bash
sudo modprobe dell_smbios
sudo smbios-thermal-ctl --set-thermal-mode=performance
```

**Result:** `ERROR: Could not execute SMI.` Both the legacy (SMM) and modern (WMI/SMBIOS) interfaces were blocked. Neither could communicate with the EC to control the fans.

### Method 3 — NBFC-linux (Embedded Controller direct access)

NoteBook FanControl uses direct EC access via `/dev/port`.

```bash
sudo apt install ./nbfc-linux_0.3.19_amd64.deb
sudo nbfc config --set auto
```

**Result:** `No config found to apply automatically`. Only 4 Dell configs existed (Inspiron 7348, 7375, Vostro 3350, XPS M1530), none compatible. However, `ec_probe` could read the EC successfully:

```bash
$ sudo ec_probe dump
# Register 0x34 = 0x63 = 99 decimal = real CPU temperature
```

The classic Dell registers (0x93/0x94) were in the EC's zero zone — they don't exist on this model.

### Method 4 — ec_probe write (direct EC register manipulation)

Attempted writing to known Dell fan registers.

```bash
sudo ec_probe write 0x93 0xFF  # Enable manual control
sudo ec_probe write 0x94 0x64  # Fan to max
```

**Result:** No effect. Registers 0x93/0x94 used by classic Dells don't exist in this model's EC map. Also tried registers 0x44, 0x46, 0x06, 0x2F — none worked.

### Method 5 — ec_probe monitor (register discovery)

Monitored all EC registers in real-time to find fan-related ones.

```bash
sudo ec_probe monitor
```

**Result:** Identified active registers:
- `0x34`: CPU temperature (confirmed, matched `sensors` output)
- `0x40, 0x42`: Rapidly fluctuating values (not fan tachometers as initially suspected)
- `0x16`: Slowly decreasing value (likely battery-related)

The fans physically did not spin despite EC activity.

### Method 6 — PWM via sysfs (dell_smm hwmon)

After a reboot, `dell_smm` started reading real temperatures:

```bash
$ cat /sys/class/hwmon/hwmon4/pwm1
128

$ echo 255 | sudo tee /sys/class/hwmon/hwmon4/pwm1
```

**Result:** PWM values could be written but `pwm1_enable` **did not exist** — there was no way to put the fan in manual mode via sysfs. Fan stayed at 0 RPM.

### Method 7 — dell-bios-fan-control

```bash
sudo dell-bios-fan-control 1  # Enable BIOS control
sudo dell-bios-fan-control 0  # Disable BIOS control
```

**Result:** Reported `BIOS CONTROL ENABLED/DISABLED` but had zero effect on the fans.

### Method 8 — acpi_call with AMWW/AMW3 (wrong ACPI paths)

Tried ACPI methods documented for Dell G15 gaming laptops.

```bash
echo "\_SB.AMWW.WMAX 0 0x15 {1, 0xab, 0x00, 0x00}" | sudo tee /proc/acpi/call
```

**Result:** `Error: AE_NOT_FOUND`. These ACPI paths are for Dell G15 series. This model doesn't have `AMWW` or `AMW3`.

### Method 9 — DSDT decompilation (led to the solution)

Decompiled the ACPI firmware to find the actual methods.

```bash
sudo cat /sys/firmware/acpi/tables/DSDT > /tmp/dsdt.dat
sudo apt install acpica-tools
iasl -d /tmp/dsdt.dat
grep -n "FANN\|WMBB\|ECRB\|ECWB" /tmp/dsdt.dsl
```

**Discoveries:**
- This laptop uses `\_SB.AMW0` and `\_SB.AMW5` (not AMWW/AMW3)
- The correct method is `WMBB` (not WMAX)
- An object `FANN` exists that reads fans via EC registers `0x6F`, `0x70`, `0x71`
- The EC uses an indirect protocol: `ECWB(0x03, data)` followed by `ECRB(register)`

```bash
$ echo "\_SB.AMW5.WMBB 0 0x01 {0x00}" | sudo tee /proc/acpi/call
$ sudo cat /proc/acpi/call
[0x0, 0x1, 0x1]  # <-- EC responded! But fans still didn't spin
```

This confirmed the EC was responsive but the thermal policy was never activated.

### Method 10 — thermald

```bash
sudo systemctl start thermald
```

**Result:** Ran but reported `sensor id 17 : No temp sysfs for reading raw temp`. Did not control fans.

### The Breakthrough — dell_ddv (Dell Data Vault)

After a reboot, the `dell_ddv` driver appeared and could read fan state:

```
dell_ddv-virtual-0
CPU Fan:        0 RPM
CPU:          +69.0°C
Ambient:      +52.0°C
SODIMM:       +45.0°C
```

This confirmed three things:
1. The fan hardware **works** (also confirmed by Windows)
2. The EC **responds** to read requests
3. The problem is that **the BIOS never tells the EC to turn on the fans** when it detects Linux

The solution was as simple as telling the BIOS "I am Windows" with `acpi_osi='Windows 2022'`.

---

## Important Notes

- **After kernel or GRUB updates**, verify that `acpi_osi` is still in `/etc/default/grub` and run `sudo update-grub`
- **No daemons needed** — fans are controlled automatically by the BIOS thermal policy, just like in Windows
- **Secure Boot** was disabled during testing; this fix has not been tested with Secure Boot enabled
- If `Windows 2022` doesn't work for your model, try `Windows 2020` or `Windows 2015`

## Contributing

If this fix worked (or didn't work) for your Dell model, please open an issue with:

1. Your laptop model (`cat /sys/class/dmi/id/product_name`)
2. Your BIOS version (`cat /sys/class/dmi/id/bios_version`)
3. Your Linux distribution and kernel version (`uname -r`)
4. Whether the fix worked or not

---

*Diagnosed and fixed on March 1, 2026*

*Licensed under [MIT](LICENSE)*
