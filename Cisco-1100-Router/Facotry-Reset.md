# Cisco 1100 Series ISR: Password Recovery and Config Wipe

Procedure for regaining access to a Cisco 1000/1100 Series Integrated Services Router when the enable password and startup configuration are unknown, then erasing the configuration for reuse.

Written for lab environments where routers are reset between semesters. Requires physical access only.

> [!NOTE]
> This is **password recovery**, not a factory reset. It clears the startup configuration in NVRAM. It does not sanitize bootflash, ROMmon variables, or TAM Flash. For RMA or post-compromise data destruction, use `factory-reset all secure` instead.

---

## Table of Contents

- [Requirements](#requirements)
- [Time Estimate](#time-estimate)
- [Procedure](#procedure)
  - [1. Connect the console cable](#1-connect-the-console-cable)
  - [2. Identify the COM port](#2-identify-the-com-port)
  - [3. Open a serial session](#3-open-a-serial-session)
  - [4. Power cycle the router](#4-power-cycle-the-router)
  - [5. Send the Break signal](#5-send-the-break-signal)
  - [6. Confirm ROMmon](#6-confirm-rommon)
  - [7. Set the config register to bypass NVRAM](#7-set-the-config-register-to-bypass-nvram)
  - [8. Restart from ROMmon](#8-restart-from-rommon)
  - [9. Skip the setup wizard](#9-skip-the-setup-wizard)
  - [10. Enter privileged EXEC mode](#10-enter-privileged-exec-mode)
  - [11. Erase the startup configuration](#11-erase-the-startup-configuration)
  - [12. Restore the config register](#12-restore-the-config-register)
  - [13. Reload](#13-reload)
  - [14. Verify](#14-verify)
- [Config Register Reference](#config-register-reference)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Console cable | RJ-45 (light blue) or mini-USB. **Prefer mini-USB.** The Break signal is more reliable over it. |
| USB-to-serial adapter | Only if using the RJ-45 cable on a laptop without a serial port. |
| Terminal emulator | PuTTY, or any emulator that can send a serial Break. |
| Physical power access | Power cable or switch. The procedure requires a cold boot. |

Console baud rate is **115200**. The Cisco 1100 console port supports no other rate.

> [!CAUTION]
> Do not plug a PoE-enabled cable into the console port. Cisco documents that this can damage the port.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 7 | About 1 minute |
| Step 8 boot | 3 to 5 minutes |
| Steps 9 to 12 | Under 1 minute |
| Step 13 reload | 3 to 5 minutes |
| **Per router total** | **8 to 12 minutes** |

---

## Procedure

### 1. Connect the console cable

Plug the cable into the port labeled **CONSOLE** on the router. The 1100 has two: one RJ-45, one mini-USB. Use whichever cable you have.

Connect the other end to your laptop. If using RJ-45, connect through your USB-to-serial adapter.

### 2. Identify the COM port

Press <kbd>Win</kbd> + <kbd>X</kbd> and open **Device Manager**. Expand **Ports (COM & LPT)**.

Look for an entry such as `USB Serial Port (COM4)`. Note the number.

If no entry appears, the adapter driver is missing. Install it from the adapter manufacturer before continuing.

### 3. Open a serial session

In PuTTY:

| Field | Value |
| --- | --- |
| Connection type | `Serial` |
| Serial line | Your port, e.g. `COM4` |
| Speed | `115200` |

Click **Open**. The terminal window will be blank. This is expected until the router boots.

### 4. Power cycle the router

Unplug the power cable, wait 5 seconds, plug it back in.

### 5. Send the Break signal

You have roughly 60 seconds from power-on to interrupt the boot process.

In PuTTY, **right-click the title bar**, then select **Special Command** > **Break**.

Send it several times, a few seconds apart, while boot text is scrolling.

> [!TIP]
> <kbd>Ctrl</kbd> + <kbd>Break</kbd> on the keyboard frequently fails to pass through USB-to-serial adapters. Use the menu option instead, or switch to the mini-USB console port.

### 6. Confirm ROMmon

Success looks like this:

```
rommon 1 >
```

| What you see | What it means | Action |
| --- | --- | --- |
| `rommon 1 >` | Break landed | Continue to step 7 |
| `Router>` or a login prompt | Break missed | Power cycle, retry step 5 |
| `PASSWORD RECOVERY IS DISABLED` prompt | `no service password-recovery` was configured | Answer `y`. The router erases its own config. Skip to step 11. |

### 7. Set the config register to bypass NVRAM

```
confreg 0x2142
```

That is a zero, then the letter x, then `2142`.

This sets the bit that tells the router to ignore the startup configuration on next boot, which is what gets you past the unknown password.

### 8. Restart from ROMmon

```
reset
```

The router reboots. Leave the session open. Boot text will scroll for 3 to 5 minutes.

### 9. Skip the setup wizard

When prompted:

```
Would you like to enter the initial configuration dialog? [yes/no]: no
```

If asked to terminate autoinstall, answer `yes`.

You should land at:

```
Router>
```

No password required.

### 10. Enter privileged EXEC mode

```
enable
```

The prompt changes to `Router#`. The `#` indicates privileged mode.

> [!NOTE]
> IOS has no `sudo`. `enable` is the equivalent.

### 11. Erase the startup configuration

```
write erase
```

Press <kbd>Enter</kbd> at the confirmation prompt.

This clears NVRAM, which holds the enable secret, line passwords, local usernames, and all interface and routing configuration.

Optionally, clear any SSH keys a previous user generated:

```
crypto key zeroize rsa
```

### 12. Restore the config register

```
configure terminal
config-register 0x2102
end
```

> [!WARNING]
> Do not skip this step. Leaving the register at `0x2142` causes the router to ignore its startup configuration on **every** future boot. Saved configurations will silently vanish on reload, which presents as a broken router.

### 13. Reload

```
reload
```

| Prompt | Answer |
| --- | --- |
| System configuration has been modified. Save? | `no` |
| Proceed with reload? | <kbd>Enter</kbd> |

### 14. Verify

After boot, answer `no` to the setup dialog, then:

```
enable
show startup-config
show version | include register
```

Expected results:

- `show startup-config` reports that the startup config is not present, or returns an empty config.
- `show version` reports `Configuration register is 0x2102`.

> [!IMPORTANT]
> On IOS XE 17.5.1 and later, the first boot after a wipe forces you to set a new enable password using `enable secret`. The older behavior of answering `no` and continuing with a blank password was removed. Have your lab standard password ready.

---

## Config Register Reference

| Value | Behavior |
| --- | --- |
| `0x2102` | Default. Boot IOS, load startup config from NVRAM, ignore Break after boot. |
| `0x2142` | Boot IOS, **ignore** startup config. Recovery only. |
| `0x2100` | Boot directly to ROMmon. |

Check the current value at any time with `show version`. The last line reports the register. If it notes that the value will differ on reload, your change is pending and takes effect at next boot.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal stays blank after power-on | Wrong COM port or wrong baud rate | Verify port in Device Manager, confirm speed is 115200 |
| Break never interrupts boot | Adapter does not pass the signal | Use PuTTY's Special Command > Break, or switch to the mini-USB console port |
| `PASSWORD RECOVERY IS DISABLED` | `no service password-recovery` was set | Answer `y` at the prompt. The config is erased automatically. |
| Router boots blank every time after recovery | Config register left at `0x2142` | Set `config-register 0x2102`, then reload |
| Reset button on chassis does nothing | Feature disabled, or router already in IOS/ROMmon mode | Use this console procedure instead |

---

## Batch Workflow

For 10 to 20 routers, the bottleneck is boot time in steps 8 and 13, not typing.

1. Run the full procedure on one router first to confirm your cable and Break method work.
2. Prepare 2 to 3 console cables and one PuTTY window per cable.
3. Stagger the units. Start router 2 while router 1 is in its step 8 boot.
4. Do a single verification pass at the end rather than verifying each unit inline.

With one cable, budget roughly 2 hours for 20 routers. With three cables running in parallel, roughly 45 minutes.

---

## When to Use Factory Reset Instead

This procedure is for reuse of trusted hardware. Use `factory-reset all secure` instead when:

- Returning hardware to Cisco under RMA
- The router was compromised by a malicious attack
- Transferring hardware outside your organization

Be aware that `factory-reset all secure` erases the boot image and can take hours per unit. The router will not boot IOS afterward without TFTP or USB recovery.
