# Palo Alto Networks Firewall: Factory Reset (Admin Password Unknown)

Procedure for resetting a Palo Alto Networks firewall (PA-220, PA-400, PA-800 series) to factory defaults when the **admin password is unknown**, using the Maintenance Recovery Tool. Requires physical console access only.

Written for lab environments where firewalls are reset between semesters.

> [!NOTE]
> **Verified on:** _Not yet verified. Fill in model, PAN-OS version, and date after the first lab run._

> [!IMPORTANT]
> There is **no password recovery** on Palo Alto firewalls. Unlike the Cisco 1100, there is no config register trick that bypasses the startup config and lets you in with the existing configuration intact. Maintenance mode Factory Reset is the only route in without credentials, and it wipes everything.
>
> If you **do** know the admin password, use [factory-reset-known-password.md](./factory-reset-known-password.md) instead. It is faster and needs no boot interrupt.

> [!CAUTION]
> **Console speed is 9600, not 115200.** Palo Alto uses 9600-8-N-1. If you have been working on Cisco gear, change the PuTTY speed before you connect or the terminal will show garbage.

> [!WARNING]
> A factory reset removes all configuration **and all logs**. There is no undo, and without credentials there is no way to back up the config first.

---

## Table of Contents

- [Requirements](#requirements)
- [Time Estimate](#time-estimate)
- [Procedure](#procedure)
  - [1. Connect the console cable](#1-connect-the-console-cable)
  - [2. Identify the COM port](#2-identify-the-com-port)
  - [3. Open a serial session](#3-open-a-serial-session)
  - [4. Power cycle the firewall](#4-power-cycle-the-firewall)
  - [5. Type maint at the boot prompt](#5-type-maint-at-the-boot-prompt)
  - [6. Wait for the maintenance image](#6-wait-for-the-maintenance-image)
  - [7. Continue past the welcome screen](#7-continue-past-the-welcome-screen)
  - [8. Select Factory Reset](#8-select-factory-reset)
  - [9. Confirm the reset](#9-confirm-the-reset)
  - [10. Wait for the reset and reboot](#10-wait-for-the-reset-and-reboot)
  - [11. Wait for autocommit](#11-wait-for-autocommit)
  - [12. Log in with the defaults](#12-log-in-with-the-defaults)
  - [13. Set the new admin password](#13-set-the-new-admin-password)
  - [14. Verify](#14-verify)
- [Expected End State](#expected-end-state)
- [Maintenance Menu Reference](#maintenance-menu-reference)
- [Benign Messages](#benign-messages)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Console cable | Palo Alto ships an RJ-45 to DB9 cable. A Cisco light blue console cable also works. |
| USB-to-serial adapter | Only if your laptop has no serial port. |
| Terminal emulator | PuTTY or equivalent. |
| Physical power access | Power cable or switch. The procedure requires a cold boot you can interrupt. |

Console settings:

| Field | Value |
| --- | --- |
| Speed | `9600` |
| Data bits | `8` |
| Parity | `None` |
| Stop bits | `1` |
| Flow control | `None` |

> [!NOTE]
> SSH will not work for this procedure. The boot interrupt only happens on the console.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 5 | About 2 minutes |
| Step 6 maintenance image load | 2 to 3 minutes |
| Steps 7 to 9 | Under 1 minute |
| Step 10 reset and reboot | 8 to 15 minutes |
| Step 11 autocommit | 2 to 5 minutes |
| Steps 12 to 14 | About 2 minutes |
| **Per firewall total** | **20 to 30 minutes** |

> [!NOTE]
> These are estimates. Replace with measured times after the first lab run.

---

## Procedure

### 1. Connect the console cable

Plug the RJ-45 end into the port labeled **CONSOLE**. Do not confuse it with the **MGT** port, which is also RJ-45 and sits next to it on most models.

Connect the other end to your laptop, through the USB-to-serial adapter if needed.

### 2. Identify the COM port

Press <kbd>Win</kbd> + <kbd>X</kbd> and open **Device Manager**. Expand **Ports (COM & LPT)**.

Look for an entry such as `USB Serial Port (COM4)`. Note the number.

If no entry appears, the adapter driver is missing. Install it before continuing.

### 3. Open a serial session

In PuTTY:

| Field | Value |
| --- | --- |
| Connection type | `Serial` |
| Serial line | Your port, e.g. `COM4` |
| Speed | `9600` |

Click **Open**. Press <kbd>Enter</kbd> once. If the firewall is running, a login prompt with the old hostname appears. That confirms the console works before you commit to a power cycle.

### 4. Power cycle the firewall

Unplug the power cable, wait 5 seconds, plug it back in.

Keep your eyes on the terminal. The next step has a short window.

### 5. Type maint at the boot prompt

Watch for the bootloader prompt. On PA-220 and similar units it looks like this:

```
Welcome to the PanOS Bootloader.

Autoboot to default partition in 5 seconds.
Enter 'maint' to boot to maint partition.
Entry:
```

Type `maint` and press <kbd>Enter</kbd>. You have about 5 seconds.

| What you see | What it means | Action |
| --- | --- | --- |
| Maintenance image starts loading | `maint` landed | Continue to step 6 |
| Normal PAN-OS boot continues, login prompt with old hostname | Missed the window | Power cycle, retry step 5 |
| A `CHOOSE PANOS` or GRUB-style menu | Older PAN-OS or a different platform | Arrow to `PANOS (maint)` and press <kbd>Enter</kbd> |
| A prompt asking you to press `m` | Older PAN-OS | Press `m` repeatedly until it loads |

> [!TIP]
> Start typing `maint` as soon as `Entry:` appears. Do not wait to read the whole screen. Click inside the PuTTY window before power cycling so it has keyboard focus.

### 6. Wait for the maintenance image

The maintenance image loads separately from PAN-OS. Allow 2 to 3 minutes.

Leave the session open and do not press keys while it loads.

### 7. Continue past the welcome screen

The Maintenance Recovery Tool opens with a welcome screen. **Continue** is selected by default.

Press <kbd>Enter</kbd>.

> [!NOTE]
> The menu is a text UI. Use the arrow keys to move and <kbd>Enter</kbd> to select. The mouse does nothing.

### 8. Select Factory Reset

The main menu lists the recovery options. Arrow down to **Factory Reset** and press <kbd>Enter</kbd>.

See [Maintenance Menu Reference](#maintenance-menu-reference) for what the other entries do. You do not need any of them for a lab reset.

### 9. Confirm the reset

A confirmation screen appears showing:

- The PAN-OS image that will be used for the reset
- A warning that all logs and configuration will be removed
- A **Scrub** option

**Leave Scrub unchecked.** Scrub securely overwrites the disk and takes much longer. It is for decommissioning or RMA, not semester resets.

Arrow down to **Factory Reset** at the bottom of the screen and press <kbd>Enter</kbd>.

> [!IMPORTANT]
> If the screen shows `No current active image found`, the PAN-OS install is damaged and Factory Reset cannot run. See [Troubleshooting](#troubleshooting).

### 10. Wait for the reset and reboot

Progress displays as a percentage. When it finishes, the firewall reboots into normal PAN-OS on its own.

> [!WARNING]
> Do not power cycle during the reset. Interrupting it can damage the PAN-OS install and leave you reimaging the unit.

Boot is complete when the login prompt returns with the **default hostname**:

```
PA-220 login:
```

The model name in place of the old student hostname is your first sign the reset took.

### 11. Wait for autocommit

**The login prompt appearing does not mean the firewall is ready.** PAN-OS runs an autocommit job after boot that loads the default configuration. Until it finishes, `admin` / `admin` may be rejected.

Wait 2 to 5 minutes after the prompt appears before logging in.

> [!IMPORTANT]
> If `admin` / `admin` is rejected right after boot, **do not assume the reset failed.** This is the most common false alarm in this procedure. Wait and retry before doing anything else.

### 12. Log in with the defaults

| Prompt | Value |
| --- | --- |
| `login:` | `admin` |
| `Password:` | `admin` |

Confirm the system is fully up:

```
show chassis-ready
```

You want:

```
yes
```

Confirm autocommit finished:

```
show jobs all
```

Look for a job of type `AutoCom` with status `FIN` and result `OK`.

### 13. Set the new admin password

On **PAN-OS 9.0.4 and later**, the first login forces a password change. It cannot be skipped.

```
Enter old password :
Enter new password :
Confirm password   :
```

The old password is `admin`. Enter the shared lab password decided before the batch.

> [!NOTE]
> Record the new password somewhere the next lab admin can find it. Losing it means repeating this entire procedure.

### 14. Verify

```
show system info
```

Expected results:

| Field | Expected value |
| --- | --- |
| `hostname` | Model name, e.g. `PA-220` |
| `ip-address` | `192.168.1.1` |
| `netmask` | `255.255.255.0` |
| `default-gateway` | Empty |

Optionally confirm from a browser. Connect a laptop to the **MGT** port, set a static IP of `192.168.1.100 / 255.255.255.0`, and browse to:

```
https://192.168.1.1
```

Accept the self-signed certificate warning and log in with the new lab password.

---

## Expected End State

| Property | Value |
| --- | --- |
| Hostname | Model name (e.g. `PA-220`) |
| Admin credentials | `admin` / your lab password |
| Management IP | `192.168.1.1 / 24` on MGT |
| Management access | HTTPS and SSH on MGT |
| Security policies, NAT, VPN, zones | Defaults only |
| Logs | Cleared |
| PAN-OS version | The image selected on the Factory Reset screen |
| Licenses | Check under **Device** > **Licenses**. Re-retrieve if missing. |

> [!NOTE]
> If licenses are missing after the reset, connect the firewall to the internet and use **Device** > **Licenses** > **Retrieve license keys from license server**. For a lab with no active subscriptions this has no effect on basic firewalling.

---

## Maintenance Menu Reference

Only Factory Reset is needed for a lab reset. The others are listed so you know what not to touch.

| Option | What it does |
| --- | --- |
| **Factory Reset** | Wipes config and logs, reinstalls the selected PAN-OS image. **The one you want.** |
| Set FIPS-CC Mode | Switches the firewall into FIPS-CC mode. Do not select for a lab. It changes default credentials and restricts features. |
| FSCK (Disk Check) | Checks and repairs the file system. Useful if the unit fails to boot normally. |
| Log Files | View system logs from the maintenance image. |
| Disk Image | Advanced options for reinstalling or reverting PAN-OS images. Use if Factory Reset reports no active image. |
| Select Running Partition | Choose which partition to boot. |
| Content Rollback | Revert to a previous content version. |
| Reboot | Exits maintenance mode and boots normally. Use this if you entered maintenance mode by mistake. |

> [!WARNING]
> Do not select **Set FIPS-CC Mode**. Leaving FIPS-CC mode requires another factory reset.

---

## Benign Messages

| Message | Meaning |
| --- | --- |
| `admin` / `admin` rejected immediately after boot | Autocommit still running. Wait and retry. |
| `show chassis-ready` returns `no` | System still initializing. Wait a minute and run it again. |
| Long pause with no output after selecting Factory Reset | Normal. Progress percentage may update slowly. |
| Certificate warning at `https://192.168.1.1` | Self-signed certificate regenerated by the reset. Expected. |
| License or content update warnings in the system log | No internet connection to the update server. Harmless on a lab bench. |

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal shows garbage characters | Speed set to 115200 | Change to 9600 |
| Terminal stays blank | Wrong COM port, or cable in MGT instead of CONSOLE | Verify the port in Device Manager, confirm the cable is in CONSOLE |
| Normal boot continues, `maint` ignored | Missed the 5 second window, or PuTTY window did not have focus | Click into PuTTY, power cycle, type `maint` the moment `Entry:` appears |
| Keystrokes do nothing at `Entry:` | Adapter or cable issue on transmit | Confirm you can type at a normal login prompt first. Try another cable. |
| `No current active image found` on the reset screen | PAN-OS install is damaged | Use **Disk Image** in the maintenance menu to reinstall PAN-OS, then run Factory Reset |
| Firewall drops back into maintenance mode on every boot | Repeated boot failures, PAN-OS damaged | Run **FSCK**, then Factory Reset. If it persists, reinstall via **Disk Image**. |
| `admin` / `admin` rejected after waiting 10 minutes | Reset did not complete, or the boot was normal and `maint` never landed | Check the hostname at the login prompt. If it is the old hostname, repeat from step 4. |
| New password rejected | Does not meet complexity rules | Read the error message and adjust |
| Cannot reach `https://192.168.1.1` | Laptop on the wrong subnet, or cable in a data port | Confirm static IP, confirm cable is in **MGT** |

---

## Batch Workflow

This procedure is mostly waiting. With multiple console cables, overlap the units.

1. Test on one unit first to confirm your cable, COM port, the 9600 setting, and the `maint` timing.
2. **Try known credentials first.** If any unit still has a documented password, reset it with [factory-reset-known-password.md](./factory-reset-known-password.md). It roughly halves the time for that unit.
3. Record each unit's serial from the chassis label, since you cannot run `show system info` until after the reset.
4. Decide the lab password before starting. Any unit on 9.0.4 or later will demand one.
5. Prepare 2 to 3 console cables and one PuTTY window per cable.
6. Stagger the units. Once firewall 1 is in step 10, start the power cycle on firewall 2.
7. Do a single verification pass at the end rather than verifying each unit inline.

| Cables | 20 firewalls |
| --- | --- |
| 1 | 7 to 10 hours |
| 3 | 2.5 to 3.5 hours |

Pull any unit that reports `No current active image found` into a separate pile rather than blocking the batch on it.

---

## Related

- [Palo Alto Networks Firewall: Factory Reset (Admin Password Known)](./factory-reset-known-password.md)
- [Palo Alto Networks Firewall: Factory Reset (all paths)](./factory-reset.md)
- [Fortinet FortiGate: Factory Reset](../Fortinet-Firewall/factory-reset.md)
- [Cisco 1100 Router: Password Recovery via ROMmon](../Cisco-1100-Router/password-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
- [Repository index](../README.md)
