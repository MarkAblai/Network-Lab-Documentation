# Palo Alto Networks Firewall: Factory Reset

Procedure for resetting a Palo Alto Networks firewall (PA-220, PA-400, PA-800 series) to factory defaults, with and without admin credentials. Requires physical console access.

Written for lab environments where firewalls are reset between semesters.

> [!CAUTION]
> **Console speed is 9600, not 115200.** Palo Alto uses 9600-8-N-1. If you have been working on Cisco gear, change the PuTTY speed before you connect or the terminal will show garbage.

> [!WARNING]
> A factory reset removes all configuration **and all logs**. There is no undo. Back up anything worth keeping first.

---

## Table of Contents

- [Requirements](#requirements)
- [Which Path to Use](#which-path-to-use)
- [Time Estimate](#time-estimate)
- [Path A: With Admin Credentials](#path-a-with-admin-credentials)
- [Path B: No Credentials (Maintenance Mode)](#path-b-no-credentials-maintenance-mode)
- [Factory Defaults](#factory-defaults)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Console cable | Palo Alto ships an RJ-45 to DB9 cable. Use a USB-to-serial adapter if your laptop has no serial port. |
| Terminal emulator | PuTTY or equivalent. |
| Physical power access | Path B requires a reboot you can interrupt. |
| Admin credentials | Optional. Determines which path you take. |

Console settings:

| Field | Value |
| --- | --- |
| Speed | `9600` |
| Data bits | `8` |
| Parity | `None` |
| Stop bits | `1` |
| Flow control | `None` |

## Which Path to Use

| Situation | Path |
| --- | --- |
| You know the admin password | [Path A](#path-a-with-admin-credentials) |
| You do not know the admin password | [Path B](#path-b-no-credentials-maintenance-mode) |

There is no way to recover a lost Palo Alto admin password. Maintenance mode is the only route in, and factory reset is the only thing it can do for you short of loading an older config whose password you know.

## Time Estimate

| Path | Duration |
| --- | --- |
| Path A | 10 to 15 minutes |
| Path B | 20 to 30 minutes |

Path B is slower because the maintenance image loads separately, then the device reboots again after the reset. Autocommit after the reset can take several extra minutes before login works.

---

## Path A: With Admin Credentials

### 1. Console in

Connect at 9600-8-N-1 and log in with your admin account.

### 2. Choose your command

Two options depending on what you need.

**Full factory reset** (config, logs, everything):

```
debug system maintenance-mode
```

The firewall reboots into maintenance mode. Continue from [Path B step 4](#4-press-enter-at-the-maintenance-menu).

**Private data reset** (config and logs, keeps PAN-OS version and licenses):

```
request system private-data-reset
```

Confirm with `y`. The firewall reboots on its own and comes back at factory defaults.

> [!NOTE]
> For a lab reset between semesters, `request system private-data-reset` is the better command. It is faster, it leaves the installed PAN-OS version alone, and it does not disturb licensing.

### 3. Wait

Reboot plus autocommit takes 10 to 15 minutes. Do not power cycle during this.

---

## Path B: No Credentials (Maintenance Mode)

### 1. Console in

Connect at 9600-8-N-1. The terminal will be blank until the device boots.

### 2. Power cycle

Unplug the firewall, wait 5 seconds, plug it back in.

### 3. Type `maint` at the boot prompt

Watch for a line like:

```
Booting PANOS (sysroot0) after 5 seconds...
Entry:
```

Type `maint` and press <kbd>Enter</kbd>. You have about 5 seconds.

> [!TIP]
> On older PAN-OS you may need to press `m` repeatedly instead. On PAN-OS 7.1 a `CHOOSE PANOS` menu appears with several options. Select `PANOS (maint)`.

The maintenance image takes 2 to 3 minutes to load.

### 4. Press Enter at the maintenance menu

When the Maintenance Recovery Tool welcome screen appears, press <kbd>Enter</kbd> on **Continue**.

### 5. Select Factory Reset

Arrow down to **Factory Reset** and press <kbd>Enter</kbd>.

A confirmation screen appears showing the image that will be used and warning that all logs and configuration will be removed.

Select **Factory Reset** again and press <kbd>Enter</kbd>.

### 6. Wait

Progress displays as a percentage. The firewall reboots when finished.

> [!IMPORTANT]
> After the reboot, autocommit runs before login works. This can take several extra minutes. If `admin` / `admin` is rejected immediately after boot, wait and try again rather than assuming the reset failed.

---

## Factory Defaults

| Setting | Value |
| --- | --- |
| Username | `admin` |
| Password | `admin` |
| Management IP | `192.168.1.1 / 24` on the MGT port |
| Management access | HTTPS and SSH on MGT |

> [!WARNING]
> **PAN-OS 9.0.4 and later force a password change on first login.** You cannot leave these firewalls at `admin` / `admin`. Pick a lab password before you start the batch.
>
> For a student lab, set one shared lab password on every unit and document it, or let each pod set their own on first login and accept that you will need maintenance mode to get back in.

---

## Verification

Connect a laptop directly to the **MGT** port. Set a static address in the same subnet, for example `192.168.1.100 / 255.255.255.0`.

Browse to:

```
https://192.168.1.1
```

Accept the self-signed certificate warning. Log in with `admin` / `admin` and set the new password when prompted.

From the CLI you can confirm the reset with:

```
show system info
```

Look for an empty hostname and the default management IP.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal shows garbage characters | Speed set to 115200 | Change to 9600 |
| Terminal stays blank | Wrong COM port | Verify in Device Manager |
| `maint` prompt never appears | Missed the 5 second window | Power cycle and watch more closely |
| `admin` / `admin` rejected right after reset | Autocommit still running | Wait 5 more minutes and retry |
| `No current active image found` at the reset screen | PAN-OS install is damaged | Use Advanced Options in the maintenance menu to reinstall PAN-OS |
| Cannot reach `192.168.1.1` | Laptop not on the same subnet, or plugged into a data port | Confirm static IP, confirm cable is in **MGT** |

---

## Batch Workflow

1. Test on one unit first to confirm your cable, COM port, and the 9600 setting.
2. Record each unit's PAN-OS version. Anything 9.0.4 or later will force a password change.
3. Decide the lab password before starting so you are not improvising per unit.
4. Path B is mostly waiting. With 2 to 3 console cables you can overlap the maintenance image loads.

| Cables | 20 firewalls, Path B |
| --- | --- |
| 1 | 7 to 10 hours |
| 3 | 2.5 to 3.5 hours |

If you have admin credentials on even some of the units, do those first with `request system private-data-reset`. It roughly halves the time for those.

---

## Related

- [Fortinet FortiGate: Factory Reset](../Fortinet-Firewall/factory-reset.md)
- [Cisco 1100 Router: Password Recovery via ROMmon](../Cisco-1100-Router/password-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
