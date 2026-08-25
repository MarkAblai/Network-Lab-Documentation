# Cisco 1100 Series ISR: Reset Button Behavior

> [!CAUTION]
> **The chassis Reset button does not reset the configuration on the Cisco 1100.**
>
> It does not bypass the startup config, does not clear NVRAM, and does not get you past an unknown enable password. Use [password-recovery.md](./password-recovery.md) instead.
>
> This document exists so nobody retries the button procedure expecting it to work.

**Tested on:** C1111-8P, IOS XE 16.10.1b, August 2026. Result: failed to bypass the configuration.

---

## Table of Contents

- [What the Button Actually Does](#what-the-button-actually-does)
- [Observed Test Output](#observed-test-output)
- [Why the Common Instructions Are Wrong](#why-the-common-instructions-are-wrong)
- [Where the Button Is](#where-the-button-is)
- [When the Button Is Useful](#when-the-button-is-useful)
- [Use This Instead](#use-this-instead)

---

## What the Button Actually Does

Holding Reset during power-on triggers a **golden image boot attempt**, not a configuration bypass.

The router looks for a recovery image at `bootflash:golden.bin`. If that file exists, it boots from it. If it does not exist, the router falls through and autoboots the normal IOS image with the existing startup configuration fully loaded.

Either way, the configuration in NVRAM is untouched and the enable password still applies.

---

## Observed Test Output

Boot log from a C1111-8P with the Reset button held through power-on:

```
Reset button push detected
unable to open bootflash:golden.bin (14)

.......

no valid BOOT image found
Final autoboot attempt from default boot device...
Located c1100-universalk9_ias.16.10.01b.SPA.bin
```

The router detected the button press, looked for the golden image, did not find one, and booted normally. Later in the same log:

```
%SYS-5-CONFIG_I: Configured from memory by console
%PNP-6-PNP_DISCOVERY_STOPPED: PnP Discovery stopped (Startup Config Present)
```

The configuration loaded from NVRAM as usual. The router came up with its original hostname and prompted for the enable password:

```
TestPod>en
Password:
Password:
Password:
% Bad passwords
```

No access gained. Procedure failed.

---

## Why the Common Instructions Are Wrong

A procedure like this circulates widely:

> 1. Turn off the router
> 2. Turn on the router while pushing in on the reset button
> 3. If console window is open it will display "factory reset" on the CLI
> 4. Verify the running config file is clean from any custom configuration

Every step after the first has a problem:

| Step | Problem |
| --- | --- |
| 2 | Triggers a golden image lookup, not a config reset. Config survives. |
| 3 | The console never prints "factory reset" from a button press. That text only comes from the `factory-reset all` CLI command. What actually prints is `Reset button push detected`. |
| 4 | `show running-config` cannot confirm a wipe. A router with a bypassed config shows a clean running config while NVRAM is still fully loaded. The correct check is `show startup-config`. |

The instructions likely originate from Cisco SMB switch platforms, where a button hold genuinely does perform a factory reset. See [the Catalyst 1200 procedure](../Cisco-1200-Switch/factory-reset.md), where the button method **is** correct. It does not carry over to ISR routers.

---

## Where the Button Is

Rear panel, far left side, near the ground lug and the LED cluster. Recessed, so a paperclip or pen tip is required.

Do not confuse it with:

- The RJ-45 console port
- The mini-USB console port
- The USB host ports

Per the Cisco Hardware Installation Guide for the 1000 Series, the Reset button is callout **7** on the rear panel diagram.

---

## When the Button Is Useful

The button has one legitimate use on this platform: booting a pre-staged recovery image.

If you place a known-good IOS image at `bootflash:golden.bin`, holding Reset during power-on boots from it instead of the configured boot image. This helps when the primary image is corrupt or a bad `boot system` statement is preventing startup.

For a lab environment being reset between semesters, this is not relevant. You want the config gone, not a different image.

---

## Use This Instead

[password-recovery.md](./password-recovery.md) — ROMmon Break method. Verified working on this hardware.

Summary of that procedure:

1. Console in at 115200
2. Power cycle, send Break within 60 seconds to reach `rommon 1 >`
3. `confreg 0x2142` then `reset`
4. Look for `%SYS-6-STARTUP_CONFIG_IGNORED` in the boot log
5. `enable`, `write erase`, `config-register 0x2102`, `reload`

About 13 to 16 minutes per router.

---

## Related

- [Cisco 1100 Router: Password Recovery via ROMmon](./password-recovery.md)
- [Cisco Catalyst 1200 Switch: Factory Reset](../Cisco-1200-Switch/factory-reset.md)
