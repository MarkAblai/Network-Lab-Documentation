# Network Lab Documentation

Procedures and reference material for resetting and recovering networking lab equipment between classes. Written for a physical lab where the same routers and switches get handed to a new group of students every semester and need to come back to a clean, credential-free state.

Everything here has been run against real hardware in the lab and documents exactly what happened, including the error messages and dead ends, not just the happy path.

## Why this exists

Vendor documentation and generic "how to reset a router" guides online are frequently wrong for a specific platform, or describe a procedure for one device that gets mistakenly applied to another. This repo exists to record, per device, the procedure that is actually verified to work, the one that looks like it should work but doesn't, and why.

## Contents

| Device | Guide | What it covers |
| --- | --- | --- |
| Cisco 1100 Series ISR | [`Cisco-1100-Router/password-recovery.md`](Cisco-1100-Router/password-recovery.md) | Verified ROMmon Break procedure to bypass an unknown enable password and wipe the startup config. |
| Cisco 1100 Series ISR | [`Cisco-1100-Router/reset-button-recovery.md`](Cisco-1100-Router/reset-button-recovery.md) | Why the chassis Reset button does **not** reset the configuration on this platform, with tested boot output, and a pointer to the procedure that does work. |
| Cisco Catalyst 1200 Series Switch | [`Cisco-1200-Switch/factory-reset.md`](Cisco-1200-Switch/factory-reset.md) | Reset-button factory reset, including LED timing and default credentials. |
| Decoy portal | [`decoy-portal/`](decoy-portal/) | A single-file classroom teaching prop for a port-scanning exercise. Not a recovery procedure — see its own README. |

## Guide format

Every device guide follows the same structure so they're quick to scan mid-lab:

- **Header callouts** for platform, firmware, and test date, plus warnings up front about anything that looks like it should work but doesn't
- **Requirements** and a **time estimate**, measured on real hardware rather than estimated
- **Numbered procedure** with expected command output at each step, so you can tell a real failure from a benign message
- **Troubleshooting table** for the failure modes actually hit while writing the guide
- **Batch workflow** notes for resetting a rack of 10-20 units instead of just one
- **Related** links to adjacent or easily-confused procedures on other platforms

## A note on the two Cisco 1100 guides

The Reset button on the Cisco 1100 ISR does **not** perform a configuration reset — it only attempts a golden-image boot. This trips people up because the equivalent button on the Catalyst 1200 switch genuinely does factory-reset it. If you're not sure which guide you need:

- **Router (ISR, runs IOS XE)** → use [`password-recovery.md`](Cisco-1100-Router/password-recovery.md)
- **Switch (Catalyst 1200, web UI)** → use [`factory-reset.md`](Cisco-1200-Switch/factory-reset.md)

[`reset-button-recovery.md`](Cisco-1100-Router/reset-button-recovery.md) documents the router button behavior specifically so nobody loses time retrying it expecting a wipe.



## Contributing

If you verify a procedure on hardware or firmware not yet covered, or find that one of these has stopped working after a firmware update, open a pull request following the existing format: state the exact hardware/firmware tested, include real command output, and note anything that looked like a failure but wasn't.
