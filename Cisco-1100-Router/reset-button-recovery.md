# Cisco 1100 Series ISR: Reset Button Recovery and Config Wipe

Fast path for wiping a Cisco 1000/1100 Series Integrated Services Router using the chassis Reset button instead of a ROMmon break. Requires physical access only.

Use this when you have no configs and no passwords and just need a blank router for student use.

> [!NOTE]
> This is **password recovery**, not a factory reset. It bypasses the startup configuration on boot so you can erase NVRAM. It does not sanitize bootflash, ROMmon variables, or TAM Flash. For RMA or post-compromise data destruction, use `factory-reset all secure` instead.

> [!IMPORTANT]
> The Reset button only works if `service password-recovery` is still enabled on the router. It is on by default, but a previous user may have disabled it. If the button does nothing, fall back to the [ROMmon break method](./cisco-1100-password-recovery.md).

---

## Table of Contents

- [Requirements](#requirements)
- [Time Estimate](#time-estimate)
- [Procedure](#procedure)
  - [1. Connect the console cable](#1-connect-the-console-cable)
  - [2. Identify the COM port](#2-identify-the-com-port)
  - [3. Open a serial session](#3-open-a-serial-session)
  - [4. Power off the router](#4-power-off-the-router)
  - [5. Hold Reset and power on](#5-hold-reset-and-power-on)
  - [6. Skip the setup wizard](#6-skip-the-setup-wizard)
  - [7. Enter privileged EXEC mode](#7-enter-privileged-exec-mode)
  - [8. Erase the startup configuration](#8-erase-the-startup-configuration)
  - [9. Reload](#9-reload)
  - [10. Verify](#10-verify)
- [Comparison to the Break Method](#comparison-to-the-break-method)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Console cable | RJ-45 (light blue) or mini-USB. Either works. Break signal is not needed here. |
| USB-to-serial adapter | Only if using the RJ-45 cable on a laptop without a serial port. |
| Terminal emulator | PuTTY or any serial terminal. |
| Paperclip or pen tip | The Reset button is recessed. |
| Physical power access | Power cable or switch. Requires a cold boot. |

Console baud rate is **115200**. The Cisco 1100 console port supports no other rate.

> [!CAUTION]
> Do not plug a PoE-enabled cable into the console port. Cisco documents that this can damage the port.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 5 | About 1 minute |
| Step 5 boot | 3 to 5 minutes |
| Steps 6 to 8 | Under 1 minute |
| Step 9 reload | 3 to 5 minutes |
| **Per router total** | **7 to 11 minutes** |

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

### 4. Power off the router

Unplug the power cable. Wait 5 seconds.

### 5. Hold Reset and power on

Locate the recessed **Reset** button on the chassis. Press and hold it with a paperclip or pen tip.

While still holding, plug the power cable back in. Keep holding through the boot.

The router boots ignoring its startup configuration.

> [!TIP]
> Cisco does not publish a hold duration for the 1100. Test on one router first. If it boots into a normal login prompt, try holding longer, or try releasing shortly after the front panel port LEDs come on.

### 6. Skip the setup wizard

Boot text scrolls for 3 to 5 minutes. When prompted:

```
Would you like to enter the initial configuration dialog? [yes/no]: no
```

If asked to terminate autoinstall, answer `yes`.

You should land at:

```
Router>
```

No password required. If you get a login prompt instead, see [Troubleshooting](#troubleshooting).

### 7. Enter privileged EXEC mode

```
enable
```

The prompt changes to `Router#`. The `#` indicates privileged mode.

> [!NOTE]
> IOS has no `sudo`. `enable` is the equivalent.

### 8. Erase the startup configuration

```
write erase
```

Press <kbd>Enter</kbd> at the confirmation prompt.

This clears NVRAM, which holds the enable secret, line passwords, local usernames, and all interface and routing configuration.

Optionally, clear any SSH keys a previous user generated:

```
crypto key zeroize rsa
```

> [!NOTE]
> There is no config register step in this method. The Reset button does not modify the register, so there is nothing to restore. This is the main advantage over the break method.

### 9. Reload

```
reload
```

| Prompt | Answer |
| --- | --- |
| System configuration has been modified. Save? | `no` |
| Proceed with reload? | <kbd>Enter</kbd> |

### 10. Verify

After boot, answer `no` to the setup dialog, then:

```
enable
show startup-config
show version | include register
```

Expected results:

- `show startup-config` reports that the startup config is not present, or returns an empty config.
- `show version` reports `Configuration register is 0x2102`.

> [!WARNING]
> On IOS XE 17.5.1 and later, the first boot after a wipe forces you to set a new enable password using `enable secret`. It cannot be skipped. If you want passwordless routers, confirm your IOS XE version is older than 17.5.1 before planning the batch.

---

## Result

After step 10 the router is a blank slate:

- Empty startup config, hostname is `Router`
- All GigabitEthernet interfaces present, no IP addresses, administratively down
- No VLANs, no DHCP, no routing, no ACLs, no local users
- No enable password (pre-17.5.1) or your chosen one (17.5.1 and later)
- Config register at `0x2102` so student configuration persists across reloads

Optionally set a hostname per unit so students know which console they are on:

```
configure terminal
hostname R1
end
write memory
```

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Terminal stays blank after power-on | Wrong COM port or wrong baud rate | Verify port in Device Manager, confirm speed is 115200 |
| Router boots to a login prompt | Button press did not register, or `no service password-recovery` was set | Retry with a longer hold. If it fails twice, use the [break method](./cisco-1100-password-recovery.md) |
| `PASSWORD RECOVERY IS DISABLED` prompt appears | `no service password-recovery strict` was set | Answer `y`. The router erases its own config. Skip to step 9. |
| Button appears to do nothing at all | Feature disabled, or router already past the boot window | Power cycle and hold the button before applying power, not after |
| Config wiped but student configs vanish on reload later | Config register left at `0x2142` from a prior recovery | Set `config-register 0x2102`, then reload |

---

## Batch Workflow

For 10 to 20 routers, boot time is the bottleneck, not typing.

1. Run the full procedure on one router first to confirm the button works and to find the right hold duration.
2. Prepare 2 to 3 console cables and one PuTTY window per cable.
3. Stagger the units. Start router 2 while router 1 is booting in step 5.
4. Do a single verification pass at the end rather than verifying each unit inline.

With one cable, budget roughly 1.5 hours for 20 routers. With three cables in parallel, roughly 35 minutes.

Pull any router where the button fails into a separate pile and run the break method on those as a second batch.

---

## When to Use Factory Reset Instead

This procedure is for reuse of trusted hardware. Use `factory-reset all secure` instead when:

- Returning hardware to Cisco under RMA
- The router was compromised by a malicious attack
- Transferring hardware outside your organization

Be aware that `factory-reset all secure` erases the boot image and can take hours per unit. The router will not boot IOS afterward without TFTP or USB recovery.
