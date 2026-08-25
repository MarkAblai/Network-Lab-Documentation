# Cisco 1100 Series ISR: Password Recovery and Config Wipe

Procedure for regaining access to a Cisco 1000/1100 Series Integrated Services Router when the enable password and startup configuration are unknown, then erasing the configuration for reuse.

Written for lab environments where routers are reset between semesters. Requires physical access only.

> [!NOTE]
> **Verified on:** C1111-8P, IOS XE 16.10.1b (`c1100-universalk9_ias.16.10.01b.SPA.bin`), August 2026. Full run took about 16 minutes including both boot cycles.

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
- [Expected End State](#expected-end-state)
- [Config Register Reference](#config-register-reference)
- [Benign Messages](#benign-messages)
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

Measured on a C1111-8P running 16.10.1b.

| Phase | Duration |
| --- | --- |
| Steps 1 to 7 | About 1 minute |
| Step 8 boot to `Router>` | 7 to 8 minutes |
| Steps 10 to 13 | Under 1 minute |
| Step 14 reload | 4 to 5 minutes |
| **Per router total** | **13 to 16 minutes** |

Boot on this platform is slower than typical IOS. Budget 8 minutes per boot cycle, not 3.

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

```
confreg 0x2142
```

That is a zero, then the letter x, then `2142`.

Expected response:

```
You must reset or power cycle for new config to take effect
```

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

## Expected End State

| Property | Value |
| --- | --- |
| Hostname | `Router` |
| Startup config | Not present |
| Enable password | None. `enable` works immediately. |
| Local users | None |
| Interfaces | All GigabitEthernet present, no IPs, administratively down |
| Config register | `0x2102` |
| SSH | Self-signed key regenerated automatically on boot |

> [!NOTE]
> On IOS XE **17.5.1 and later**, the first boot after a wipe forces you to set an enable secret and cannot be skipped. Passwordless is only achievable on releases before 17.5.1. Confirm your version with `show version` before planning a batch around passwordless routers.

---

## Config Register Reference

| Value | Behavior |
| --- | --- |
| `0x2102` | Default. Boot IOS, load startup config from NVRAM, ignore Break after boot. |
| `0x2142` | Boot IOS, **ignore** startup config. Recovery only. |
| `0x2100` | Boot directly to ROMmon. |

Check the current value at any time with `show version | include register`.

---

## Benign Messages

These appear during a normal run and do not indicate a problem.

| Message | Meaning |
| --- | --- |
| `%PNP-6-PNP_DISCOVERY_STOPPED: (Startup Config Present)` | Config exists in NVRAM but is being ignored. Expected during the bypass boot. |
| `% Failed to initialize backup nvram` | Normal immediately after `write erase`. Rebuilds on first save. |
| `%FLASH_CHECK-3-DISK_QUOTA: Flash disk quota exceeded` | Bootflash filling up. Not fatal. Clean up with `dir bootflash:` and delete old files, leaving the `.bin` image alone. |
| `%PKI-4-NOCONFIGAUTOSAVE: Configuration was modified` | Self-signed cert regenerated. Do **not** run `write memory` unless you want to save it. Leaving it unsaved keeps the router blank. |
| `%OSPF-4-NORTID: OSPF process 1 failed to allocate unique router-id` | Leftover from the old config during the pre-wipe boot. Gone after the erase. |
| `%PKI-2-NON_AUTHORITATIVE_CLOCK` | No NTP source. Harmless on a lab bench. |
| `%SMART_LIC-3-EVAL_EXPIRED` | Evaluation license expired. Does not affect lab functionality. |

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal stays blank after power-on | Wrong COM port or wrong baud rate | Verify port in Device Manager, confirm speed is 115200 |
| Break never interrupts boot | Adapter does not pass the signal | Use PuTTY's Special Command > Break, or switch to the mini-USB console port |
| No `STARTUP_CONFIG_IGNORED` line, old hostname appears | `confreg` did not take, or you power cycled instead of using `reset` | Return to step 4 and redo the ROMmon steps |
| `PASSWORD RECOVERY IS DISABLED` | `no service password-recovery` was set | Answer `y` at the prompt. The config is erased automatically. |
| Router boots blank every time after recovery | Config register left at `0x2142` | Set `config-register 0x2102`, then reload |
| Reset button on chassis appears to do nothing useful | The button is not a config bypass on this platform | See [reset-button-recovery.md](./reset-button-recovery.md) |

---

## Batch Workflow

For 10 to 20 routers, boot time is the bottleneck. Two boot cycles at 7 and 5 minutes means roughly 13 to 16 minutes of wall clock per router, of which only about 2 minutes is typing.

1. Run the full procedure on one router first to confirm your cable and Break method work.
2. Prepare 2 to 3 console cables and one PuTTY window per cable.
3. Stagger the units. While router 1 is in its step 8 boot, start the ROMmon steps on router 2.
4. Do a single verification pass at the end rather than verifying each unit inline.

| Cables | 20 routers |
| --- | --- |
| 1 | About 5 hours |
| 3 | About 1.5 to 2 hours |

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
