# Palo Alto Networks Firewall: Factory Reset (Admin Password Unknown)

Procedure for resetting a Palo Alto Networks firewall (PA-220, PA-400, PA-800 series) to factory defaults when the **admin password is unknown**, using the Maintenance Recovery Tool. Requires physical console access only.

Written for lab environments where firewalls are reset between semesters.

> [!NOTE]
> **Verified on:** PA-440, PAN-OS 10.1.9-h3, October 2026.

> [!IMPORTANT]
> On the **PA-440**, maintenance mode is **not** reached by typing `maint` at a text prompt. The unit shows a graphical boot menu where you select `PANOS (maint-sysroot0)`. See [step 5](#5-select-the-maintenance-partition). The `maint` text-prompt method applies to older platforms like the PA-220, kept in the table there for reference.

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
  - [5. Select the maintenance partition](#5-select-the-maintenance-partition)
  - [6. Wait for the maintenance image](#6-wait-for-the-maintenance-image)
  - [7. Continue past the welcome screen](#7-continue-past-the-welcome-screen)
  - [8. Select Factory Reset](#8-select-factory-reset)
  - [9. Confirm the reset](#9-confirm-the-reset)
  - [10. Reboot from the status screen](#10-reboot-from-the-status-screen)
  - [11. Exit ZTP mode](#11-exit-ztp-mode)
  - [12. Wait for autocommit](#12-wait-for-autocommit)
  - [13. Log in with the defaults](#13-log-in-with-the-defaults)
  - [14. Set the new admin password](#14-set-the-new-admin-password)
  - [15. Verify](#15-verify)
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
> SSH will not work for this procedure. The boot menu and maintenance mode are only reachable on the console.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 5 | About 2 minutes |
| Step 6 maintenance image load | 2 to 3 minutes |
| Steps 7 to 9 | Under 1 minute |
| Step 10 reset and reboot | 8 to 15 minutes |
| Steps 11 to 12 autocommit | 2 to 5 minutes |
| Steps 13 to 15 | About 2 minutes |
| **Per firewall total** | **20 to 30 minutes** |

Measured on a PA-440 running 10.1.9-h3.

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

The PA-440 boots through an InsydeH2O BIOS first, then shows a PAN-OS boot menu. Watch the terminal.

### 5. Select the maintenance partition

The firewall shows a PAN-OS boot menu with four entries:

```
PANOS (maint-sysroot0)
PANOS (maint-sysroot1)
PANOS (sysroot0)
PANOS (sysroot1)
```

Arrow to **`PANOS (maint-sysroot0)`** and press <kbd>Enter</kbd>. The `maint-` entries are maintenance mode. `sysroot0` is the active partition, so use that first; if it fails to load, retry with `maint-sysroot1`.

> [!NOTE]
> This menu appears on its own during boot. You do not type `maint`, and you do not need to press Esc for BIOS boot options to reach it.

> [!TIP]
> Click inside the PuTTY window first so arrow keys and <kbd>Enter</kbd> reach the firewall.

> [!NOTE]
> **Older platforms (PA-220 and similar) use a text prompt instead.** They show `Enter 'maint' to boot to maint partition. Entry:` and you type `maint` within about 5 seconds. If you see that prompt rather than the four-entry menu, type `maint` and press <kbd>Enter</kbd>.

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

- The PAN-OS image that will be used (for example `panos-10.1.9-h3`)
- A warning that all logs and configuration will be removed
- A **Scrub** option, with a scrub type selector (`nnsa` / `dod`)

**Leave Scrub unchecked.** Scrub iteratively overwrites the disk and can take several hours to days. It is for decommissioning or RMA, not semester resets. The scrub type selector below it is irrelevant while Scrub is off.

Arrow down to the **Factory Reset** entry below the Scrub section and press <kbd>Enter</kbd>.

> [!IMPORTANT]
> If the screen shows `No current active image found`, the PAN-OS install is damaged and Factory Reset cannot run. See [Troubleshooting](#troubleshooting).

### 10. Reboot from the status screen

The reset runs, then shows:

```
Factory Reset Status: Success
```

Below it is a `Reboot` entry, and you may see `Bootstrapping [plugin] into partition "panrepo"`. That line is normal post-reset cleanup.

Arrow to **Reboot** and press <kbd>Enter</kbd>.

> [!WARNING]
> Do not power cycle during the reset. Interrupting it can damage the PAN-OS install and leave you reimaging the unit.

### 11. Exit ZTP mode

After the reboot the firewall comes up in Zero Touch Provisioning (ZTP) mode and will **not** accept a normal login until you exit it. You will see:

```
Do you want to exit ZTP mode and configure your firewall in standard mode (yes/no)[no]?:
```

Type `yes` and press <kbd>Enter</kbd>.

> [!IMPORTANT]
> Until you answer this, any `admin` / `admin` you type goes into the ZTP prompt, not a login, and logs as `Login incorrect`. That is not a failed reset.

### 12. Wait for autocommit

**The login prompt appearing does not mean the firewall is ready.** PAN-OS runs an autocommit job after boot that loads the default configuration. Until it finishes, `admin` / `admin` is rejected and the console may repeat `Login incorrect` on its own.

Wait 2 to 5 minutes, hands off the keyboard, until a clean login prompt with the **default hostname** settles:

```
PA-440 login:
```

The model name in place of the old student hostname is your sign the reset took.

> [!IMPORTANT]
> Repeated `Login incorrect` right after boot is autocommit still running, **not** a failed reset. This is the single most common false alarm in this procedure. Wait it out before doing anything else.

### 13. Log in with the defaults

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

### 14. Set the new admin password

On **PAN-OS 9.0.4 and later**, the first login forces a password change. It cannot be skipped.

```
Enter old password :
Enter new password :
Confirm password   :
```

The old password is `admin`. Enter the shared lab password decided before the batch.

> [!NOTE]
> Record the new password somewhere the next lab admin can find it. Losing it means repeating this entire procedure.

### 15. Verify

```
show system info
```

Expected results:

| Field | Expected value |
| --- | --- |
| `hostname` | Model name, e.g. `PA-440` |
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
| Hostname | Model name (e.g. `PA-440`) |
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
| `Login incorrect` repeating right after boot | ZTP prompt not yet answered, or autocommit still running. Answer ZTP with `yes`, then wait. |
| `admin` / `admin` rejected immediately after boot | Autocommit still running. Wait and retry. |
| `Bootstrapping [plugin] into partition "panrepo"` | Normal post-reset cleanup on the status screen. |
| `FIPS-CC Plugin Self-Tests ... passed` during boot | Normal boot self-test. Not an indication the unit is in FIPS-CC mode. |
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
| Boots straight to a login prompt, no boot menu seen | Menu scrolled past, or you were not watching | Power cycle and watch from `Booting from SPI0`. The four-entry menu follows shortly after. |
| Boot stops at an InsydeH2O BIOS "Front Page" asking for a password | You pressed Esc into BIOS boot options | Do not use BIOS boot options. Those entries are password protected and are a dead end. Reboot and select `PANOS (maint-sysroot0)` from the PAN-OS menu instead. |
| `Login incorrect` on every `admin` / `admin` right after boot | ZTP prompt not answered yet | Answer the exit-ZTP prompt with `yes`, then wait for autocommit |
| `maint-sysroot0` fails to load | Active partition is sysroot1 | Reboot and select `PANOS (maint-sysroot1)` |
| `No current active image found` on the reset screen | PAN-OS install is damaged | Use **Disk Image** in the maintenance menu to reinstall PAN-OS, then run Factory Reset |
| Firewall drops back into maintenance mode on every boot | Repeated boot failures, PAN-OS damaged | Run **FSCK**, then Factory Reset. If it persists, reinstall via **Disk Image**. |
| `admin` / `admin` rejected after answering ZTP and waiting 10 minutes | Reset did not complete | Check the hostname at the login prompt. If it is the old hostname, repeat from step 4. |
| New password rejected | Does not meet complexity rules | Read the error message and adjust |
| Cannot reach `https://192.168.1.1` | Laptop on the wrong subnet, or cable in a data port | Confirm static IP, confirm cable is in **MGT** |

---

## Batch Workflow

This procedure is mostly waiting. With multiple console cables, overlap the units.

1. Test on one unit first to confirm your cable, COM port, the 9600 setting, and the boot-menu behavior for your model.
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
