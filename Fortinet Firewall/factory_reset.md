# Fortinet FortiGate Firewall: Factory Reset

Procedure for resetting a FortiGate firewall (40F, 60F, 100F class) to factory defaults, with and without admin credentials. Requires physical console access.

Written for lab environments where firewalls are reset between semesters.

> [!CAUTION]
> **Console speed is 9600, not 115200.** FortiGate uses 9600-8-N-1. If you have been working on Cisco gear, change the PuTTY speed before you connect or the terminal will show garbage.

---

## Table of Contents

- [Requirements](#requirements)
- [Which Path to Use](#which-path-to-use)
- [Time Estimate](#time-estimate)
- [Path A: With Admin Credentials](#path-a-with-admin-credentials)
- [Path B: No Credentials (Maintainer Account)](#path-b-no-credentials-maintainer-account)
- [Path C: BIOS Format (Last Resort)](#path-c-bios-format-last-resort)
- [Factory Defaults](#factory-defaults)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Console cable | RJ-45 to serial, or USB depending on model. A Cisco light blue console cable works on most units. |
| Terminal emulator | PuTTY or equivalent. |
| Device serial number | **Required for Path B.** Printed on the chassis label and shown in the boot output. |
| Physical power access | Paths B and C require a reboot you can catch. |

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
| You do not know the admin password | [Path B](#path-b-no-credentials-maintainer-account) |
| Path B fails and the unit is unrecoverable | [Path C](#path-c-bios-format-last-resort) |

## Time Estimate

| Path | Duration |
| --- | --- |
| Path A | 3 to 5 minutes |
| Path B | 5 to 10 minutes, plus retries |
| Path C | 15 to 25 minutes |

Path A on FortiGate is the fastest reset of any platform in this repo.

---

## Path A: With Admin Credentials

### 1. Console in and log in

Connect at 9600-8-N-1, log in as `admin`.

### 2. Run the reset

```
execute factoryreset
```

Confirm with `y`.

The unit reboots immediately and comes back at factory defaults. About 3 to 5 minutes.

> [!TIP]
> Use `execute factoryreset2` instead if you want to keep the installed firmware images and only wipe the configuration. For a lab reset either is fine.

---

## Path B: No Credentials (Maintainer Account)

FortiGate has a built-in `maintainer` account that works over the console for a short window after boot. It is enabled by default.

### 1. Get the serial number

Read it off the chassis label, or console in and power cycle. The serial appears in the first few lines of boot output:

```
Serial number: FGT60FTK20012345
```

### 2. Build the password ahead of time

The password is the literal string `bcpb` followed by the serial number **in uppercase**, no spaces.

```
bcpbFGT60FTK20012345
```

> [!IMPORTANT]
> Type this into Notepad first. You will have **14 seconds or less** from the login prompt to enter both the username and the password, and there is no countdown shown. Have it ready to paste.
>
> In PuTTY, right-click pastes from the clipboard.

### 3. Power cycle

Unplug the unit, wait 10 seconds, plug it back in.

### 4. Log in as maintainer

When the boot completes you will see:

```
System is started.
login:
```

Enter immediately:

| Prompt | Value |
| --- | --- |
| `login:` | `maintainer` |
| `Password:` | `bcpb` + serial in uppercase |

Expect to need more than one attempt. The window is tight and there is no error message when it expires, the prompt just returns.

### 5. Reset

Once logged in:

```
execute factoryreset
```

Confirm with `y`. The unit reboots to factory defaults.

> [!NOTE]
> If you would rather keep the configuration and just reset the admin password, run this instead:
>
> ```
> config system admin
> edit admin
> set password <newpassword>
> end
> ```

### 6. Limitations

- Does not work on FortiGate VMs, only physical appliances.
- Requires a genuine power cycle. A soft `execute reboot` does not open the maintainer window on all models.
- Can be disabled by a previous admin with `set admin-maintainer disable`. If it was, Path B will not work and you need Path C.

---

## Path C: BIOS Format (Last Resort)

Use only if Path B fails. This wipes the boot device, so you must reload firmware over TFTP afterward.

### 1. Prepare

Put a FortiOS firmware image on a TFTP server running on your laptop. Connect the laptop to **port1** and set a static IP on the same subnet.

### 2. Interrupt boot

Power cycle. When you see:

```
Press any key to display configuration menu...
```

Press any key within about 3 seconds.

### 3. Format the boot device

Select `[F]: Format boot device.`

Confirm. This erases everything.

### 4. Load firmware over TFTP

Select `[G]: Get firmware image from TFTP server.`

Enter your laptop's IP, the unit's temporary IP, and the firmware filename when prompted.

The unit downloads, installs, and boots at factory defaults.

---

## Factory Defaults

| Setting | Value |
| --- | --- |
| Username | `admin` |
| Password | Empty. Press <kbd>Enter</kbd>. |
| Management IP | `192.168.1.99 / 24` on `port1` (or `internal` on some models) |
| Management access | HTTPS, HTTP, SSH, ping on that interface |

> [!WARNING]
> **FortiOS 7.0 and later force a password change on first login.** You cannot leave these at a blank admin password. Pick a lab password before starting the batch.
>
> Some builds also enforce complexity rules. Have a password ready that meets length, case, and digit requirements.

---

## Verification

Connect a laptop to **port1**. Set a static address, for example `192.168.1.100 / 255.255.255.0`.

Browse to:

```
https://192.168.1.99
```

Accept the self-signed certificate. Log in as `admin` with a blank password, then set the new one when prompted.

From the CLI:

```
get system status
```

Confirms firmware version, serial number, and that the config is at defaults.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal shows garbage characters | Speed set to 115200 | Change to 9600 |
| Maintainer login rejected | Serial typed in lowercase, or `bcpb` mistyped | Serial letters must be UPPERCASE, `bcpb` stays lowercase |
| Maintainer login window expires | Slower than 14 seconds | Paste from Notepad, retry. Several attempts is normal. |
| Maintainer login never works after many tries | `admin-maintainer` was disabled, or unit is a VM | Use Path C |
| No boot menu on keypress | Missed the 3 second window | Power cycle and press a key immediately |
| Cannot reach `192.168.1.99` | Laptop on wrong subnet, or cable in the wrong port | Confirm static IP, confirm cable is in **port1** |

---

## Batch Workflow

1. Collect every unit's serial number first. Build the full `bcpb` + serial string for each one in a text file before you touch any hardware. This is the single biggest time saver for Path B.
2. Test on one unit to confirm cable, COM port, and the 9600 setting.
3. Decide the lab password before starting.
4. Path A units are fast enough that you should do all of those first.

| Path | 20 firewalls, 1 cable |
| --- | --- |
| A | About 1.5 hours |
| B | 5 to 8 hours |

With 3 console cables, Path B for 20 units drops to roughly 2 hours.

---

## Related

- [Palo Alto Networks Firewall: Factory Reset](../Palo-Alto-Firewall/factory-reset.md)
- [Cisco 1100 Router: Password Recovery via ROMmon](../Cisco-1100-Router/password-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
