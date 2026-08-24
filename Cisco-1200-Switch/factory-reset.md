# Cisco Catalyst 1200 Series Switch: Factory Reset

Procedure for resetting a Cisco Catalyst 1200 Series switch to factory defaults using the chassis Reset button. Requires physical access only.

Use this when you have no configs and no credentials and just need a blank switch for student use.

> [!IMPORTANT]
> The Catalyst 1200 is **not** an IOS device. It runs Linux-based software with a web UI and its own CLI. None of the IOS router procedures apply here. There is no ROMmon, no `write erase`, no config register, and no console Break needed.

---

## Table of Contents

- [Requirements](#requirements)
- [Time Estimate](#time-estimate)
- [Procedure](#procedure)
  - [1. Disconnect all Ethernet cables](#1-disconnect-all-ethernet-cables)
  - [2. Locate the Reset button](#2-locate-the-reset-button)
  - [3. Hold Reset for 16 to 20 seconds](#3-hold-reset-for-16-to-20-seconds)
  - [4. Release and wait](#4-release-and-wait)
  - [5. Verify](#5-verify)
- [LED Timing Reference](#led-timing-reference)
- [Factory Defaults](#factory-defaults)
- [Troubleshooting](#troubleshooting)
- [Batch Workflow](#batch-workflow)

---

## Requirements

| Item | Notes |
| --- | --- |
| Physical access | The Reset button is on the chassis. |
| Paperclip or pinky finger | The button is small and recessed on some models. |
| Laptop with Ethernet port | Only for the verification step. |
| Power | Leave the switch **powered on** throughout. Unlike the routers, this is not a cold boot procedure. |

No console cable required. No terminal emulator required.

> [!NOTE]
> Reset button location varies by model. On most Catalyst 1200 units it is on the front panel. On some it sits on the rear next to the USB-C port.

## Time Estimate

| Phase | Duration |
| --- | --- |
| Steps 1 to 3 | About 30 seconds |
| Reset and reboot | 2 to 4 minutes |
| Verification | About 1 minute |
| **Per switch total** | **3 to 5 minutes** |

Considerably faster than the router procedure. No console session to set up.

---

## Procedure

### 1. Disconnect all Ethernet cables

Unplug every cable from the switch ports. Leave the power connected.

This matters for two reasons. A live uplink can hand the switch a DHCP address that masks whether the reset actually took, and if several factory-default switches sit on the same network without DHCP they will all claim `192.168.1.254` and collide.

### 2. Locate the Reset button

Small recessed button on the chassis. Check the front panel first, then the rear near the USB-C port.

### 3. Hold Reset for 16 to 20 seconds

Press and hold. Watch the **system LED** while you count.

| Elapsed | System LED | Release here means |
| --- | --- | --- |
| 1 to 5 sec | Solid green | Nothing happens |
| 6 to 10 sec | Slow flashing green | Reboot only, config preserved |
| 11 to 15 sec | Solid green | Nothing happens |
| **16 to 20 sec** | **Rapid flashing green** | **Factory reset** |

> [!WARNING]
> Releasing during the slow flash at 6 to 10 seconds reboots the switch without erasing anything. The config survives and the switch looks unchanged. Count past it and wait for the rapid flash.

### 4. Release and wait

Release the button while the LED is rapidly flashing.

The switch reloads to factory defaults. Give it 2 to 4 minutes.

### 5. Verify

Connect your laptop directly to any switch port with an Ethernet cable.

Set your laptop to a static address in `192.168.1.0/24`, for example `192.168.1.100 / 255.255.255.0`.

Browse to:

```
https://192.168.1.254
```

Log in with:

| Field | Value |
| --- | --- |
| Username | `cisco` |
| Password | `cisco` |

Both are case-sensitive. If those credentials work, the reset succeeded.

> [!TIP]
> A continuously flashing System LED means the switch is on its factory default IP. Steady green means it took a DHCP or static address instead, which is why step 1 says to unplug the uplinks.

---

## LED Timing Reference

Full documented Reset button behavior:

| Hold duration | LED behavior | Action on release |
| --- | --- | --- |
| Under 1 sec | Port LED solid amber for 5 sec | Displays PoE status, no reset |
| 1 to 5 sec | System LED solid green | None |
| 6 to 10 sec | System LED slow flashing green | Reload, config preserved |
| 11 to 15 sec | System LED solid green | None |
| 16 to 20 sec | System LED rapid flashing green | Reload to factory defaults |

---

## Factory Defaults

After the reset the switch comes up with:

| Setting | Value |
| --- | --- |
| Management IP | `192.168.1.254 / 24` on VLAN 1, if no DHCP server responds |
| DHCP client | Enabled. A DHCP lease overrides the default IP. |
| Username | `cisco` |
| Password | `cisco` |
| VLANs | Default VLAN 1 only |
| Port config | All ports default, no VLANs assigned, no ACLs |

> [!CAUTION]
> **You cannot leave these switches passwordless.** On first login the switch forces you to replace both the default username and password, and it rejects `cisco` as the new password. Password complexity rules are strict.
>
> For a student lab this means one of two things:
> - Leave the switches at factory default and let students hit the forced-change prompt themselves. Give them the `cisco` / `cisco` starting credentials.
> - Set one lab credential on all units yourself and tape it to the rack.
>
> The first option is closer to a true blank slate and saves you the per-switch work.

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Switch reboots but config is still there | Released during the slow flash at 6 to 10 sec | Retry, hold past 16 sec for the rapid flash |
| `cisco` / `cisco` rejected after reset | Reset did not take, or credentials were changed post-reset | Retry the reset |
| Cannot reach `192.168.1.254` | Switch took a DHCP lease from a live uplink | Unplug all Ethernet cables, reset again, connect only your laptop |
| Multiple switches unreachable at once | Several factory-default units on the same segment claiming the same IP | Configure them one at a time |
| No LED response to the button | Wrong button, or switch not powered | Confirm power, check both front and rear panels |

---

## Batch Workflow

For 10 to 20 switches this goes much faster than the routers.

1. Test on one switch first to confirm the button location and LED timing on your specific model.
2. Leave all units powered. Unplug every Ethernet cable in the rack.
3. Walk the rack holding Reset on each unit for 16 to 20 seconds. No waiting for boots, no console cables.
4. Come back in 5 minutes and spot check a few units with a laptop.

Budget roughly 20 to 30 minutes for 20 switches including verification.

> [!NOTE]
> Verify one at a time when checking `192.168.1.254`. Factory-default switches on the same segment will collide on that address.

---

## Related

- [Cisco 1100 Router: Reset Button Recovery](../Cisco-1100-Router/reset-button-recovery.md)
- [Cisco 1100 Router: Password Recovery via ROMmon](../Cisco-1100-Router/password-recovery.md)
- [Repository index](../README.md)
