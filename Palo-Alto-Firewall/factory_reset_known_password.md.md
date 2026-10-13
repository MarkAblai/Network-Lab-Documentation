# Palo Alto Networks Firewall: Factory Reset (Admin Password Known)

Procedure for resetting a Palo Alto Networks firewall (PA-220, PA-400, PA-800 series) to factory defaults when you **know the admin password**. Requires console access, or SSH to the management interface.

Written for lab environments where firewalls are reset between semesters.

> [!NOTE]
> **Verified on:** _Not yet verified. Fill in model, PAN-OS version, and date after the first lab run._

> [!IMPORTANT]
> This guide covers the **with-credentials** path only. If the admin password is unknown, use the maintenance mode procedure in [factory-reset.md](./factory-reset.md#path-b-no-credentials-maintenance-mode) instead.

> [!CAUTION]
> **Console speed is 9600, not 115200.** Palo Alto uses 9600-8-N-1. If you have been working on Cisco gear, change the PuTTY speed before you connect or the terminal will show garbage.

> [!WARNING]
> A factory reset removes all configuration **and all logs**. There is no undo. Back up anything worth keeping first.

---

## Table of Contents

- [Requirements](#requirements)
- [Which Command to Use](#which-command-to-use)
- [Time Estimate](#time-estimate)
- [Procedure](#procedure)
  - [1. Connect the console cable](#1-connect-the-console-cable)
  - [2. Identify the COM port](#2-identify-the-com-port)
  - [3. Open a serial session](#3-open-a-serial-session)
  - [4. Log in as admin](#4-log-in-as-admin)
  - [5. Record the current state](#5-record-the-current-state)
  - [6. Back up the configuration (optional)](#6-back-up-the-configuration-optional)
  - [7. Run the reset](#7-run-the-reset)
  - [8. Wait for the reboot](#8-wait-for-the-reboot)
  - [9. Wait for autocommit](#9-wait-for-autocommit)
  - [10. Log in with the defaults](#10-log-in-with-the-defaults)
  - [11. Set the new admin password](#11-set-the-new-admin-password)
  - [12. Verify](#12-verify)
- [Expected End State](#expected-end-state)
- [Benign Messages](#benign-messages)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)
- [When to Use Maintenance Mode Instead](#when-to-use-maintenance-mode-instead)

---

## Requirements

| Item | Notes |
| --- | --- |
| Admin credentials | Username and password for an account with superuser rights. |
| Console cable | Palo Alto ships an RJ-45 to DB9 cable. A Cisco light blue console cable also works. |
| USB-to-serial adapter | Only if your laptop has no serial port. |
| Terminal emulator | PuTTY or equivalent. |

Console settings:

| Field | Value |
| --- | --- |
| Speed | `9600` |
| Data bits | `8` |
| Parity | `None` |
| Stop bits | `1` |
| Flow control | `None` |

> [!TIP]
> If the management IP is reachable, you can run the reset over SSH instead of the console. Use the console anyway. The SSH session drops the moment the firewall reboots, and after the reset the old management IP is gone, so you lose all visibility into the boot.

## Which Command to Use

| Command | What it does | Use it when |
| --- | --- | --- |
| `request system private-data-reset` | Wipes config and logs, restores the default config, reboots. Leaves the installed PAN-OS version in place. | **Default for lab resets.** |
| `debug system maintenance-mode` | Reboots into the Maintenance Recovery Tool, where you run Factory Reset from the menu. | `private-data-reset` fails, or you need the full maintenance mode reset. Continue at [factory-reset.md Path B step 4](./factory-reset.md#4-press-enter-at-the-maintenance-menu). |

The rest of this guide uses `request system private-data-reset`.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 7 | About 2 minutes |
| Step 8 reboot | 5 to 8 minutes |
| Step 9 autocommit | 2 to 5 minutes |
| Steps 10 to 12 | About 2 minutes |
| **Per firewall total** | **10 to 15 minutes** |

> [!NOTE]
> These are estimates. Replace with measured times after the first lab run, the same way the Cisco guides were measured.

---

## Procedure

### 1. Connect the console cable

Plug the RJ-45 end into the port labeled **CONSOLE** on the firewall. Do not confuse it with the **MGT** port, which is also RJ-45 and sits next to it on most models.

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

Click **Open**, then press <kbd>Enter</kbd> once to wake the prompt.

You should see:

```
PA-220 login:
```

The hostname shown will be whatever the previous user configured.

### 4. Log in as admin

Enter the known admin username and password.

Success looks like:

```
admin@StudentFW-03>
```

The `>` indicates operational mode. You do **not** need to enter `configure` for the reset. It is an operational mode command.

### 5. Record the current state

```
show system info
```

Note these fields for the batch log:

| Field | Why |
| --- | --- |
| `hostname` | Confirms which unit you are on |
| `serial` | Identifies the unit after the hostname is wiped |
| `sw-version` | PAN-OS 9.0.4 and later force a password change after the reset |
| `model` | Confirms the platform |

### 6. Back up the configuration (optional)

Skip this for a routine semester reset. Do it if the config has anything worth keeping.

Export the running config to an SCP server on your laptop:

```
scp export configuration from running-config.xml to user@192.168.1.100:/backups/
```

Replace the user, IP, and path with your own. The firewall must be able to reach that IP through the management interface.

### 7. Run the reset

```
request system private-data-reset
```

Expected prompt:

```
Executing this command will remove all logs and configuration and will
reboot the system. Do you want to continue? (y or n)
```

Answer `y`.

> [!WARNING]
> Do not power cycle the firewall from this point until step 10. Pulling power mid-reset can leave the unit needing maintenance mode recovery.

### 8. Wait for the reboot

The firewall reboots on its own. The console shows the full boot sequence. Leave the session open.

Boot is complete when the login prompt returns with the **default hostname**:

```
PA-220 login:
```

The hostname matching the model name, rather than the old student hostname, is your first sign the reset took.

### 9. Wait for autocommit

**The login prompt appearing does not mean the firewall is ready.** After boot, PAN-OS runs an autocommit job that loads the default configuration. Until it finishes, `admin` / `admin` may be rejected or the CLI may behave oddly.

Wait 2 to 5 minutes after the prompt appears before logging in.

> [!IMPORTANT]
> If `admin` / `admin` is rejected right after boot, **do not assume the reset failed.** Autocommit is still running. Wait and retry before doing anything else.

### 10. Log in with the defaults

| Prompt | Value |
| --- | --- |
| `login:` | `admin` |
| `Password:` | `admin` |

Once logged in, confirm the system is fully up:

```
show chassis-ready
```

You want:

```
yes
```

Then confirm autocommit finished:

```
show jobs all
```

Look for a job of type `AutoCom` with status `FIN` and result `OK`.

### 11. Set the new admin password

On **PAN-OS 9.0.4 and later**, the first login forces a password change. It cannot be skipped.

```
Enter old password :
Enter new password :
Confirm password   :
```

The old password is `admin`. Enter the shared lab password decided before the batch.

> [!NOTE]
> Password complexity requirements vary by PAN-OS version. If the new password is rejected, read the error, it states the rule that failed.

### 12. Verify

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
| `serial` | Matches what you recorded in step 5 |

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
| PAN-OS version | Unchanged from before the reset |
| Licenses | Check under **Device** > **Licenses**. Re-retrieve if missing. See note below. |

> [!NOTE]
> Sources disagree on whether `private-data-reset` keeps local license keys. If licenses are missing after the reset, connect the firewall to the internet and use **Device** > **Licenses** > **Retrieve license keys from license server**. For a lab with no active subscriptions this has no effect on basic firewalling.

---

## Benign Messages

| Message | Meaning |
| --- | --- |
| `admin` / `admin` rejected immediately after boot | Autocommit still running. Wait and retry. |
| `show chassis-ready` returns `no` | System still initializing. Wait a minute and run it again. |
| Certificate warning at `https://192.168.1.1` | Self-signed certificate regenerated by the reset. Expected. |
| License or content update warnings in the system log | No internet connection to the update server. Harmless on a lab bench. |

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal shows garbage characters | Speed set to 115200 | Change to 9600 |
| Terminal stays blank | Wrong COM port, or cable in MGT instead of CONSOLE | Verify the port in Device Manager, confirm the cable is in CONSOLE |
| Login with known password fails | Password changed since it was recorded | Use [maintenance mode](./factory-reset.md#path-b-no-credentials-maintenance-mode) |
| `Unknown command: private-data-reset` | Account lacks superuser rights, or very old PAN-OS | Log in as a superuser account, or use `debug system maintenance-mode` |
| Old hostname still present after reboot | Reset did not run, or `n` was entered at the prompt | Log in with the old credentials and repeat step 7 |
| `admin` / `admin` rejected after waiting 10 minutes | Reset did not complete | Run `show system info` with old credentials. If the old config is present, repeat step 7. If neither login works, use maintenance mode. |
| New password rejected | Does not meet complexity rules | Read the error message and adjust |
| Cannot reach `https://192.168.1.1` | Laptop on the wrong subnet, or cable in a data port | Confirm static IP, confirm cable is in **MGT** |

---

## Batch Workflow

Most of the per-unit time is waiting on reboot and autocommit, so overlap units rather than doing them in sequence.

1. Test on one unit first to confirm your cable, COM port, and the 9600 setting.
2. **Survey the rack before starting.** For each unit, record serial, PAN-OS version, and whether you have working credentials. Units without credentials go to the [maintenance mode guide](./factory-reset.md#path-b-no-credentials-maintenance-mode).
3. Decide the lab password before starting. Any unit on 9.0.4 or later will demand one.
4. Prepare 2 to 3 console cables and one PuTTY window per cable.
5. Stagger the units. Run step 7 on firewall 1, then move to firewall 2 while firewall 1 reboots.
6. Do a single verification pass at the end rather than verifying each unit inline.

| Cables | 20 firewalls |
| --- | --- |
| 1 | About 4 to 5 hours |
| 3 | About 1.5 to 2 hours |

If a unit's known password turns out to be wrong, move it to the maintenance mode pile rather than blocking the batch.

---

## When to Use Maintenance Mode Instead

Use the [maintenance mode procedure](./factory-reset.md#path-b-no-credentials-maintenance-mode) when:

- The admin password is unknown or no longer works
- `private-data-reset` fails or the unit does not come back cleanly
- The firewall is stuck in a boot loop or PAN-OS is damaged

---

## Related

- [Palo Alto Networks Firewall: Factory Reset (all paths)](./factory-reset.md)
- [Fortinet FortiGate: Factory Reset](../Fortinet-Firewall/factory-reset.md)
- [Cisco 1100 Router: Password Recovery via ROMmon](../Cisco-1100-Router/password-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
- [Repository index](../README.md)
