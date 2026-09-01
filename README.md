# Network Lab Documentation

Procedures and reference material for resetting and recovering networking lab equipment between classes. Written for a physical lab where the same routers, switches, and firewalls get handed to a new group of students every semester and need to come back to a clean, credential-free state.

Everything here has been run against real hardware in the lab and documents exactly what happened, including the error messages and dead ends, not just the happy path.

## Why this exists

Vendor documentation and generic "how to reset a router" guides online are frequently wrong for a specific platform, or describe a procedure for one device that gets mistakenly applied to another. This repo exists to record, per device, the procedure that is actually verified to work, the one that looks like it should work but doesn't, and why.

## Contents

| Device | Guide | What it covers |
| --- | --- | --- |
| Cisco 1100 Series ISR | [`Cisco-1100-Router/password-recovery.md`](Cisco-1100-Router/password-recovery.md) | Verified ROMmon Break procedure to bypass an unknown enable password and wipe the startup config. |
| Cisco 1100 Series ISR | [`Cisco-1100-Router/reset-button-recovery.md`](Cisco-1100-Router/reset-button-recovery.md) | Why the chassis Reset button does **not** reset the configuration on this platform, with tested boot output, and a pointer to the procedure that does work. |
| Cisco Catalyst 1200 Series Switch | [`Cisco-1200-Switch/factory-reset.md`](Cisco-1200-Switch/factory-reset.md) | Reset-button factory reset, including LED timing and default credentials. |
| Fortinet FortiGate (40F / 60F / 100F class) | [`Fortinet-Firewall/factory-reset.md`](Fortinet-Firewall/factory-reset.md) | Three reset paths: `execute factoryreset` with credentials, the console-only `maintainer` account (`bcpb` + uppercase serial) without them, and BIOS format plus TFTP firmware reload as a last resort. |
| Palo Alto Networks (PA-220 / PA-400 / PA-800 series) | [`Palo-Alto-Firewall/factory-reset.md`](Palo-Alto-Firewall/factory-reset.md) | `request system private-data-reset` with credentials, and the maintenance-mode Factory Reset (`maint` at the boot prompt) without them. Covers autocommit delays after reboot. |
| Decoy portal | [`decoy-portal/`](decoy-portal/) | A single-file classroom teaching prop for a port-scanning exercise. Not a recovery procedure — see its own README. |

## Guide format

Every device guide follows the same structure so they're quick to scan mid-lab:

- **Header callouts** for platform, firmware, and test date, plus warnings up front about anything that looks like it should work but doesn't
- **Requirements** and a **time estimate**, measured on real hardware rather than estimated
- **A "which path to use" table** where a platform has more than one route in, so you can pick based on whether you have credentials before reading the whole guide
- **Numbered procedure** with expected command output at each step, so you can tell a real failure from a benign message
- **Troubleshooting table** for the failure modes actually hit while writing the guide
- **Batch workflow** notes for resetting a rack of 10-20 units instead of just one
- **Related** links to adjacent or easily-confused procedures on other platforms

## Picking the right guide

A button or command that factory-resets one platform often does something completely different on another, so confirm the device before you start.

- **Cisco 1100 ISR router (IOS XE)** → [`password-recovery.md`](Cisco-1100-Router/password-recovery.md)
- **Cisco Catalyst 1200 switch (web UI)** → [`factory-reset.md`](Cisco-1200-Switch/factory-reset.md)
- **Fortinet FortiGate** → [`factory-reset.md`](Fortinet-Firewall/factory-reset.md)
- **Palo Alto Networks** → [`factory-reset.md`](Palo-Alto-Firewall/factory-reset.md)

### The two Cisco 1100 guides

The Reset button on the Cisco 1100 ISR does **not** perform a configuration reset — it only attempts a golden-image boot. This trips people up because the equivalent button on the Catalyst 1200 switch genuinely does factory-reset it. [`reset-button-recovery.md`](Cisco-1100-Router/reset-button-recovery.md) documents the router button behavior specifically so nobody loses time retrying it expecting a wipe.

### Console speed differs by vendor

The Cisco procedures use **115200** baud. Both firewall procedures use **9600** baud. If you move from a Cisco unit to a FortiGate or Palo Alto without changing the speed in PuTTY, the terminal fills with garbage characters and looks like a dead console. Change the speed before you conclude the cable or the unit is bad.

### Modern firmware forces a password change

Neither firewall can be left sitting at its documented default credentials on current firmware — FortiOS 7.0+ and PAN-OS 9.0.4+ both force a change at first login, and some builds enforce complexity rules on top of that. Decide the shared lab password before starting a batch rather than improvising per unit.
