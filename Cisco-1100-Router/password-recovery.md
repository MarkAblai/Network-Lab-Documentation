# Cisco 1100 Series ISR: Password Recovery and Config Wipe

Procedure for regaining access to a Cisco 1000/1100 Series Integrated Services Router when the enable password and startup configuration are unknown, then erasing the configuration for reuse.

Written for lab environments where routers are reset between semesters. Requires physical access only.

> [!NOTE]
> **Verified on:** C1111-8P (IOS XE 16.10.1b), C1111X-8P (IOS XE 17.x), and C1111-8P running the UC image in SD-WAN controller mode. August and September 2026.

> [!IMPORTANT]
> This is the **working method** for the C1100. The chassis Reset button does **not** bypass the startup configuration on this platform. See [reset-button-recovery.md](./reset-button-recovery.md) for what the button actually does and why it is not a substitute.

> [!NOTE]
> This is password recovery, not a factory reset. It clears the startup configuration in NVRAM. It does not sanitize bootflash, ROMmon variables, or TAM Flash. For RMA or post-compromise data destruction, use `factory-reset all secure` instead.

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
  - [9. Confirm the bypass worked](#9-confirm-the-bypass-worked)
  - [10. Skip the setup wizard](#10-skip-the-setup-wizard)
  - [11. Enter privileged EXEC mode](#11-enter-privileged-exec-mode)
  - [12. Erase the startup configuration](#12-erase-the-startup-configuration)
  - [13. Restore the config register](#13-restore-the-config-register)
  - [14. Reload](#14-reload)
  - [15. Verify](#15-verify)
- [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units)
- [Expected End State](#expected-end-state)
- [Config Register Reference](#config-register-reference)
- [Benign Messages](#benign-messages)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)
- [When to Use Factory Reset Instead](#when-to-use-factory-reset-instead)

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

Measured on a C1111-8P running 16.10.1b.

| Phase | Duration |
| --- | --- |
| Steps 1 to 7 | About 1 minute |
| Step 8 boot to `Router>` | 7 to 8 minutes |
| Steps 10 to 13 | Under 1 minute |
| Step 14 reload | 4 to 5 minutes |
| **Per router total** | **13 to 16 minutes** |

Boot on this platform is slower than typical IOS. Budget 8 minutes per boot cycle, not 3.

Add 15 to 20 minutes for any unit in [SD-WAN controller mode](#sd-wan-controller-mode-units).

---

## Procedure

### 1. Connect the console cable

Plug the cable into the port labeled **CONSOLE** on the router. The C1100 has two: one RJ-45, one mini-USB. Use whichever cable you have.

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
| `PASSWORD RECOVERY IS DISABLED` prompt | `no service password-recovery` was configured | Answer `y`. The router erases its own config. Skip to step 12. |

### 7. Set the config register to bypass NVRAM

Type this **once**:

```
confreg 0x2142
```

That is a zero, then the letter x, then `2142`.

Response when the value changes:

```
You must reset or power cycle for new config to take effect
```

> [!IMPORTANT]
> ROMmon prints that line **only when the register value actually changes**. If the register was already `0x2142` from an earlier attempt, the command returns silently with no output at all.
>
> **Silence is not failure.** Do not retype the command, and do not "correct" it by entering a different value. Running `confreg 0x2102` afterward overwrites `0x2142` and the bypass will not happen. Only the last value entered counts.
>
> To read the current value without changing it, run `confreg` with no argument.

### 8. Restart from ROMmon

```
reset
```

Expected output begins:

```
Resetting .......

Rom image verified correctly

System Bootstrap, Version 16.9(1r), RELEASE SOFTWARE
...
Last reset cause: LocalSoft
```

`Last reset cause: LocalSoft` confirms the reset came from your command rather than a power event.

Boot takes 7 to 8 minutes. Leave the session open.

### 9. Confirm the bypass worked

Watch for this line in the boot log:

```
%SYS-6-STARTUP_CONFIG_IGNORED: System startup configuration is ignored
based on the configuration register setting.
```

**That single line is your confirmation.** If you see it, the bypass worked and the old password is irrelevant.

> [!WARNING]
> You will also see `%PNP-6-PNP_DISCOVERY_STOPPED: PnP Discovery stopped (Startup Config Present)`. **Ignore it.** The config is still in NVRAM, it is simply not being loaded. This message is not a sign of failure.
>
> If you see `%SYS-5-CONFIG_I: Configured from memory by console` and the hostname is anything other than `Router`, the bypass did **not** work. Return to step 4.

> [!NOTE]
> If you get `STARTUP_CONFIG_IGNORED` but a `Username:` prompt still appears, the unit is in SD-WAN controller mode. See [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units).

### 10. Skip the setup wizard

When prompted:

```
Would you like to enter the initial configuration dialog? [yes/no]: no
Would you like to terminate autoinstall? [yes]:
```

Answer `no` to the first. Press <kbd>Enter</kbd> at the second to accept the `yes` default.

Press <kbd>Enter</kbd> again to get a prompt:

```
Router>
```

No password required.

> [!TIP]
> If the dialog loops with `% Please answer 'yes' or 'no'.`, type the full word `no` rather than `n` or a bare <kbd>Enter</kbd>. Click inside the PuTTY window first so it has keyboard focus.

### 11. Enter privileged EXEC mode

```
enable
```

The prompt changes to `Router#`. The `#` indicates privileged mode.

> [!NOTE]
> IOS has no `sudo`. `enable` is the equivalent.

### 12. Erase the startup configuration

```
write erase
```

Expected output:

```
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm]
[OK]
Erase of nvram: complete
%SYS-7-NV_BLOCK_INIT: Initialized the geometry of nvram
```

Press <kbd>Enter</kbd> at the confirm prompt.

This clears NVRAM, which holds the enable secret, line passwords, local usernames, and all interface and routing configuration.

Optionally, clear any SSH keys a previous user generated:

```
crypto key zeroize rsa
```

> [!NOTE]
> If `write erase` returns `This command is not supported in Controller mode.`, stop here and see [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units).

### 13. Restore the config register

```
configure terminal
config-register 0x2102
end
```

> [!WARNING]
> Do not skip this step. Leaving the register at `0x2142` causes the router to ignore its startup configuration on **every** future boot. Student configurations will silently vanish on reload, which presents as a broken router.

### 14. Reload

```
reload
```

| Prompt | Answer |
| --- | --- |
| System configuration has been modified. Save? | `no` |
| Proceed with reload? | <kbd>Enter</kbd> |

Answering `no` is correct here. The config register change is stored outside NVRAM and survives regardless.

Boot takes 4 to 5 minutes. Answer `no` and <kbd>Enter</kbd> to the setup dialog prompts again.

> [!IMPORTANT]
> On IOS XE **17.5.1 and later**, this boot presents a forced enable secret prompt. It cannot be skipped, and pressing <kbd>Enter</kbd> produces a `% No defaulting allowed` loop. Enter a password meeting the stated rules: 10 to 32 characters, at least one uppercase, one lowercase, one digit, and it cannot contain the string `cisco`.

### 15. Verify

```
enable
show startup-config
show version | include register
```

Expected results:

```
Router#show startup-config
startup-config is not present

Router#show version | include register
Configuration register is 0x2102
```

Both must match. If the register still reads `0x2142`, repeat step 13 and reload.

---

## SD-WAN Controller Mode Units

Some C1100 units ship or get configured in **SD-WAN controller mode** rather than autonomous mode. The standard procedure does not work on them until the mode is changed.

**Verified on:** C1111-8P running `c1100-ucmk9.16.10.3a`, and C1111X-8P. September 2026.

### How to recognize one

Any of these:

| Signal | Where |
| --- | --- |
| Image name contains `ucmk9` instead of `universalk9` | Boot log, `show version` |
| `Router operating mode: Autonomous` is **absent** | Boot log |
| `%Cisco-SDWAN-RP_0-CFGMGR` messages repeating | Boot log |
| Daemons `confd`, `cfgmgr`, `vdaemon`, `ompd`, `ftmd` starting | Boot log |
| `write erase` returns `This command is not supported in Controller mode.` | After login |
| `conf t` returns `Please use the equivalent command - config-transaction` | After login |

### Why the standard procedure fails

The config register bypass works. You will still see `%SYS-6-STARTUP_CONFIG_IGNORED`. But in controller mode, authentication is handled by the SD-WAN daemon stack rather than the IOS startup config, so **the login prompt does not go away**. The register has no effect on it.

### Getting in

At the `Username:` prompt, try the SD-WAN defaults:

| Username | Password |
| --- | --- |
| `admin` | `admin` |

On some units this forces an immediate password change. Set your lab password and continue. You should land at `Router#` directly, with no `enable` step required.

> [!WARNING]
> If `admin` / `admin` fails and you reach only user EXEC (`Router>`) with no way to escalate, there is no software path forward. The unit needs a TFTP reimage from ROMmon with a `universalk9` image. Budget 45 to 60 minutes and set it aside rather than blocking the batch.

### Leaving controller mode

From `Router#`:

```
controller-mode disable
```

Confirm when it warns that the configuration will be lost. That is the intent.

The router reboots into autonomous mode, drops the SD-WAN daemons, and comes back as a normal IOS router. Allow 10 to 15 minutes.

> [!NOTE]
> Do not try to work around this with `config-transaction`. That is the SD-WAN configuration syntax and it cannot perform the wipe you need. Changing the operating mode is the fix.

### After the mode change

Confirm with:

```
show version | include operating mode
```

You want `Router operating mode: Autonomous`.

Then run the standard procedure from [step 11](#11-enter-privileged-exec-mode). `write erase` and `conf t` now work normally.

### Repeating warning during controller mode

`%Cisco-SDWAN-RP_0-CFGMGR-4-WARN-300005: New admin password not set yet, waiting for daemons to read initial config.` repeats every 15 seconds while in controller mode. It is noise, not an error. It stops once you leave controller mode.

---

## Expected End State

| Property | Value |
| --- | --- |
| Hostname | `Router` |
| Startup config | Not present |
| Enable password | None on pre-17.5.1. Your chosen password on 17.5.1 and later. |
| Local users | None |
| Operating mode | Autonomous |
| Interfaces | All GigabitEthernet present, no IPs, administratively down |
| Config register | `0x2102` |
| SSH | Self-signed key regenerated automatically on boot |

> [!NOTE]
> On IOS XE **17.5.1 and later**, the first boot after a wipe forces you to set an enable secret and cannot be skipped. A truly passwordless router is only achievable on releases before 17.5.1. Confirm your version with `show version` before planning a batch around passwordless routers, and expect a mixed rack: C1111-8P units on older releases come up passwordless, C1111X-8P units on newer releases do not.

---

## Config Register Reference

| Value | Behavior |
| --- | --- |
| `0x2102` | Default. Boot IOS, load startup config from NVRAM, ignore Break after boot. |
| `0x2142` | Boot IOS, **ignore** startup config. Recovery only. |
| `0x2100` | Boot directly to ROMmon. |

Check the current value from IOS with `show version | include register`, or from ROMmon by running `confreg` with no argument.

---

## Benign Messages

These appear during a normal run and do not indicate a problem.

| Message | Meaning |
| --- | --- |
| `%PNP-6-PNP_DISCOVERY_STOPPED: (Startup Config Present)` | Config exists in NVRAM but is being ignored. Expected during the bypass boot. |
| `% Failed to initialize backup nvram` | Normal immediately after `write erase`. Rebuilds on first save. |
| `%FLASH_CHECK-3-DISK_QUOTA: Flash disk quota exceeded` | Bootflash filling up. Not fatal. Clean up with `dir bootflash:` and delete old files, leaving the `.bin` image alone. |
| `%PKI-4-NOCONFIGAUTOSAVE: Configuration was modified` | Self-signed cert regenerated. Do **not** run `write memory` unless you want to save it. Leaving it unsaved keeps the router blank. |
| `%OSPF-4-NORTID: OSPF process failed to allocate unique router-id` | Leftover from the old config during the pre-wipe boot. Gone after the erase. |
| `%PKI-2-NON_AUTHORITATIVE_CLOCK` | No NTP source. Harmless on a lab bench. |
| `%SMART_LIC-3-EVAL_EXPIRED` | Evaluation license expired. Does not affect lab functionality. |
| `%Cisco-SDWAN-RP_0-CFGMGR-4-WARN-300005` repeating | Controller mode noise. See [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units). |
| `unable to open bootflash:golden.bin (14)` | Reset button was held during power-on. No recovery image staged. Harmless. |

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal stays blank after power-on | Wrong COM port or wrong baud rate | Verify port in Device Manager, confirm speed is 115200 |
| Break never interrupts boot | Adapter does not pass the signal | Use PuTTY's Special Command > Break, or switch to the mini-USB console port |
| `confreg 0x2142` produces no output | The register was already `0x2142` | Normal. Do not retype it or enter a different value. Proceed to `reset`. |
| `monitor: command "congreg" not found` | Typo. The command is `confreg` | Retype carefully. Read the response before running `reset`. |
| No `STARTUP_CONFIG_IGNORED` line, old hostname appears | `confreg` did not take, or a later `confreg` overwrote it | Return to step 4. Enter `confreg 0x2142` once, then `reset`. |
| `PASSWORD RECOVERY IS DISABLED` | `no service password-recovery` was set | Answer `y` at the prompt. The config is erased automatically. |
| Setup dialog loops with `% Please answer 'yes' or 'no'.` | Keystrokes not reaching the router, or partial input | Click into the PuTTY window, type the full word `no` |
| `% No defaulting allowed` loop at enable secret prompt | IOS XE 17.5.1+ forced password, <kbd>Enter</kbd> pressed | Enter a valid password: 10 to 32 chars, upper, lower, digit, no `cisco` |
| Login prompt persists despite `STARTUP_CONFIG_IGNORED` | SD-WAN handles auth outside the IOS config | Try `admin` / `admin`, then see [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units) |
| `write erase` returns `not supported in Controller mode` | Unit is in SD-WAN controller mode | See [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units) |
| `conf t` returns `use the equivalent command - config-transaction` | Unit is in SD-WAN controller mode | See [SD-WAN Controller Mode Units](#sd-wan-controller-mode-units) |
| Router boots blank every time after recovery | Config register left at `0x2142` | Set `config-register 0x2102`, then reload |
| Reset button on chassis appears to do nothing useful | The button is not a config bypass on this platform | See [reset-button-recovery.md](./reset-button-recovery.md) |

---

## Batch Workflow

For 10 to 20 routers, boot time is the bottleneck. Two boot cycles at 7 and 5 minutes means roughly 13 to 16 minutes of wall clock per router, of which only about 2 minutes is typing.

1. Run the full procedure on one router first to confirm your cable and Break method work.
2. **Survey the rack before starting.** Note which units run `universalk9` versus `ucmk9`, and which run IOS XE 17.5.1 or later. Those two facts determine whether a unit needs the controller mode detour and whether it can be left passwordless.
3. Decide your lab password up front. A mixed rack means some units will demand one whether you wanted it or not.
4. Prepare 2 to 3 console cables and one PuTTY window per cable.
5. Stagger the units. While router 1 is in its step 8 boot, start the ROMmon steps on router 2.
6. Do a single verification pass at the end rather than verifying each unit inline.

| Cables | 20 routers |
| --- | --- |
| 1 | About 5 hours |
| 3 | About 1.5 to 2 hours |

Pull any unit that cannot be recovered into a separate pile rather than blocking the batch on it.

---

## When to Use Factory Reset Instead

This procedure is for reuse of trusted hardware. Use `factory-reset all secure` instead when:

- Returning hardware to Cisco under RMA
- The router was compromised by a malicious attack
- Transferring hardware outside your organization

Be aware that `factory-reset all secure` erases the boot image and can take hours per unit. The router will not boot IOS afterward without TFTP or USB recovery.

---

## Related

- [Cisco 1100 Router: Reset Button Behavior](./reset-button-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
- [Palo Alto Networks Firewall: Factory Reset](../Palo-Alto-Firewall/factory-reset.md)
- [Fortinet FortiGate: Factory Reset](../Fortinet-Firewall/factory-reset.md)
- [Repository index](../README.md)
