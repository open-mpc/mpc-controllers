# mpc-controllers

What an MPC 3.10 **controller script** (the Lua layer) can and cannot do, found by testing on real hardware,
written down so you can build your own controller support without redoing the digging.

- **[docs/capabilities.md](docs/capabilities.md)**: the API surface, by what you want to do.
- **[docs/limitations.md](docs/limitations.md)**: the hard walls, each with the evidence.
- **[docs/api-reference.md](docs/api-reference.md)**: every table, function and property a script can see on 3.10.1.8, with notes on what works.
- **[docs/recipes/](docs/recipes/)**: short how-tos for things that took effort to get right.
- **[docs/findings/](docs/findings/)**: the raw dated research log (firmware 3.9.1 to 3.10.1.8).
- **[skill/](skill/)**: a Claude skill that points an AI agent at the docs above.
- **[examples/force-link](examples/force-link)**: a complete working controller (an Akai Force as a surface for an MPC Live III), as a git submodule.
- **[examples/mpc-launchpad-pro-mk3](examples/mpc-launchpad-pro-mk3)** (v1.3.0): a Novation Launchpad Pro MK3 as a state-aware clip grid + pads surface for an MPC (clip colours, track row, Q-Link pages), as a git submodule.
- **[examples/mpc-launchcontrolxl-mk3](examples/mpc-launchcontrolxl-mk3)** (v1.1.0): a Novation Launch Control XL 3 as a mixer + Q-Link surface driven through its DAW mode (two-way: MPC state painted onto encoder LEDs and OLED), as a git submodule.
- **[examples/mpc-launchpad-mini-mk3](examples/mpc-launchpad-mini-mk3)** (v1.0.0): a Novation Launchpad Mini MK3 as a clip/state grid, mixer and fader-menu surface, as a git submodule.
- **[Community scripts](#community-scripts)**: controller scripts other people have built from these notes.

## Why this exists
Controller support on MPC is thinly documented. Akai ships a handful of MPK mini scripts and no API reference, so
anyone who wants a Push, a Launchpad or a second Force to drive an MPC has to reverse-engineer what a script can reach.
We did that while building [force-link](examples/force-link) (a Force as an extra surface for a Live III), and kept
notes on every wall we hit. This repo publishes those notes so the next person can start from what works and what
doesn't, instead of rediscovering it. force-link is the worked example.

## What changed between 3.9 and 3.10
Both firmwares have a Lua controller-script loader (found by reading the binaries; checked on device for 3.10).
3.10 is where it becomes useful for state-aware surfaces. (FINDINGS 5, 8, 11)

| | 3.9.1 | 3.10 |
|---|---|---|
| Script loader | present, **no checksum** (unsigned scripts should load; untested on 3.9.1) | present, **signed**: `checksum = sha256(lua + salt)` |
| API | mapping only: `AddNoteToMpcPad/ForcePad`, `AddNoteToControl`, `AddInfiniteCCToQLink`, `SendMIDI*`, `timer`, `TapVirtual` | adds everything below |
| Clip grid (`clipMatrixAperture`, clip colour/state) | no | yes |
| Per-track mute / solo / rec arm properties | no (a button press can be mapped, but there's no state back) | yes |
| Q-link names (`TargetName`, `ValueString`) | no, so no way to know what a knob controls | yes |
| Sysex into the script (`HandleMIDISysex`) | no | yes |
| Mode change, TUI feedback, `AddNoteToVirtualPad` | no | yes |
| `mpc.force.*` (one control per Force button) | not seen | yes (3.10.1.8) |
| Note editing on the selected clip | no | yes |

In practice: on 3.9.1 a controller is an **input surface** (pads, buttons, knobs) with no clip colours or knob names.
On 3.10 it can be **state-aware**: a clip grid with live colours, mute/solo/arm rows, named Q-links. Even on 3.10
there are real limits (no state dump, no per-track MIDI input, no non-current-track editing): see
[limitations](docs/limitations.md).

## Community scripts
Controller scripts built on this repo by other people. What we learned from them is in
[docs/findings/COMMUNITY.md](docs/findings/COMMUNITY.md), credited to each project.

| Controller | Script | Author | Tested on |
|---|---|---|---|
| Novation Launchpad Pro MK3 (v1.3.0) | [mpc-launchpad-pro-mk3](https://github.com/fuzboxz/mpc-launchpad-pro-mk3) ([submodule](examples/mpc-launchpad-pro-mk3)): 8x8 clip launcher with live clip states, scenes, mixer pages, Q-link page, MPC pads mode, tempo/swing pages | [@fuzboxz](https://github.com/fuzboxz) | Live III, MPC 3.10.1.8 |
| Novation Launch Control XL 3 (v1.1.0) | [mpc-launchcontrolxl-mk3](https://github.com/fuzboxz/mpc-launchcontrolxl-mk3) ([submodule](examples/mpc-launchcontrolxl-mk3)): faders/sends/pan mixer, encoders routed into the 16 Q-links (dot-form Twist driver), transport, OLED track map | [@fuzboxz](https://github.com/fuzboxz) | Live III, MPC 3.10.1.8, XL3 fw 1.1.11 |
| Novation Launchpad Mini MK3 (v1.0.0) | [mpc-launchpad-mini-mk3](https://github.com/fuzboxz/mpc-launchpad-mini-mk3) ([submodule](examples/mpc-launchpad-mini-mk3)): clip/state grid with live colours, scene column, mixer and fader-menu pages | [@fuzboxz](https://github.com/fuzboxz) | Live III, MPC 3.10.1.8 |
| Korg nanoKONTROL (Mk1) | [KorgNanoKontrolMk1_4_AkaiForce](https://github.com/kagusta1/KorgNanoKontrolMk1_4_AkaiForce): volume, pan, mute, cue, transport, view switching | [@kagusta1](https://github.com/kagusta1) | Force, MPC 3.10.1.12 |
| Akai Force | [force-link](examples/force-link): a Force as an extra surface for a Live III (this repo's worked example) | this repo | Live III, MPC 3.10.0.49 / 3.10.1.8 |

Built one? Open a PR adding a row (and, if you like, your repo as a submodule under `examples/`): the controller, a link, and the device and firmware you tested on.

## Scope
Verified on **MPC 3.10.0.49 and 3.10.1.8** (Live III, Gen 2) unless a note says otherwise. Each claim is tagged with the
build it was checked on; behaviour may change in later firmware. Some findings are about the Force (MPC 3.9.1) and
say so. Re-test before relying on anything here.

## Ground rules
Akai's software is not included and must not be added. Findings describe behaviour and give offsets for people who
own the firmware; they do not redistribute code, and the script-signing salt is deliberately not published.
Use at your own risk: several routes need root and restart MPC (which closes the open project).

## Contributing
Corrections and new findings welcome: say the firmware build, the device, and how you checked.

Licence: documentation CC-BY-4.0; code samples MIT.
