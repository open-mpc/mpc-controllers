# Community findings

Findings from controller scripts other people built on top of this repo, checked against our own notes. Each entry
credits its source and says how it was verified there. Where a finding corrects or extends one of ours, it says which.
Firmware is **MPC 3.10.1.8 (build 38fdde70994), Live III** unless noted.

## Sources
| Key | Project | Author | Verified |
|---|---|---|---|
| **LPP3** | [mpc-launchpad-pro-mk3](https://github.com/fuzboxz/mpc-launchpad-pro-mk3) (Novation Launchpad Pro MK3) | [@fuzboxz](https://github.com/fuzboxz) | device, 2026-11-17/18 |
| **LCXL3** | [mpc-launchcontrolxl-mk3](https://github.com/fuzboxz/mpc-launchcontrolxl-mk3) (Novation Launch Control XL 3) | [@fuzboxz](https://github.com/fuzboxz) | device, 2026-11-18 (XL 3 fw 1.1.11) |
| **LPMK3** | [mpc-launchpad-mini-mk3](https://github.com/fuzboxz/mpc-launchpad-mini-mk3) (Novation Launchpad Mini MK3) | [@fuzboxz](https://github.com/fuzboxz) | device, 2026-11-18 |
| **NK** | [KorgNanoKontrolMk1_4_AkaiForce](https://github.com/kagusta1/KorgNanoKontrolMk1_4_AkaiForce) (Korg nanoKONTROL Mk1) | [@kagusta1](https://github.com/kagusta1) | device, **Akai Force, MPC 3.10.1.12**, 2026-11-16/17 |

Thanks to fuzboxz for publishing the Novation scripts, with device-test plans that make it clear which claims were
checked on hardware, and to kagusta1 for the first script we know of running on a Force (and on a newer build than
ours). LPP3 also used NK as a reference.

---

## C1. A full dump of the Lua API on 3.10.1.8 (LPP3)
LPP3's [`docs/LUA-API-DUMP.md`](https://github.com/fuzboxz/mpc-launchpad-pro-mk3/blob/main/docs/LUA-API-DUMP.md)
walks `mpc` and the device table to depth 4 from inside a running script and logs every key, type and property value.
It's the most complete map of the script-visible API we know of, and it fills gaps in ours. Namespaces we had not
listed: `mpc.masterClipMatrix`, `mpc.song` (step list editing), `mpc.exporter` (`ExportClip/Sequence/Song`),
`mpc.tui` (`MouseDown/Drag/Up`), `mpc.features`, `mpc.filesystem`, `mpc.samples`, `mpc.modeMenu`,
`mpc.stepSequencerControl`, `mpc.IsDesktop/IsStandalone`. Also `mpc.clipMatrix` has `CopyClip`, `DeleteClip`,
`CopyRow/Track`, `Create*Track`, `SetClipName`, `SetClipStart/End/LoopStart/LoopEnd`, `FlattenClip`; `currentSequence.events`
mirrors `selectedClip.events`. Presence only: most of these have not been called on hardware.

## C2. Clip `State` enum (LPP3, from the APC mini mk2 script and the Gen 1 enum)
```
0 NoSlot, 1 Empty, 2 Ready, 3 WillPlay, 4 Playing, 5 WillStopPlaying,
6 WillRecord, 7 Recording, 8 WillStopRecording, 9 WillOverdub, 10 Overdubbing, 11 WillStopOverdubbing
```
LPP3 drives its clip LEDs from this on device. **Corrects** our FINDINGS 18 note "0 = Stopped" and force-link's shorter
`stateName` guess. `aperture.sceneProperties[row].State` uses the same enum and covers the whole scene row, including
columns outside the window.

## C3. `qlinks[i].SetPosition` does not change the parameter (LPP3, device 2026-11-17)
`SetPosition` updates the Q-link's `ValueString` but the MPC's edit value doesn't move; an LPP3 page built on it was
reverted. Drive Q-links with `Touch()` + `Twist(delta)` (delta = target − `GetPosition()`, at most 2 calls because
`GetPosition` lags one change, FINDINGS 26), or better, hand the CC to MPC with `AddInfiniteCCToQLink` (LCXL3 routed
16/16 encoders this way on device). **Extends** our capabilities table, which listed `SetPosition` without this caveat.

## C4. Script-sent MIDI does not loop back through `dev.mappings` (LPP3, LCXL3)
Hardware input does reach MPC through `mappings.*` (pads, Q-links, controls), but MIDI the script emits does not. So a
script can't synthesise CCs to drive a mapped Q-link; it has to call the control API directly.

## C5. The `.ini` loader refuses silently (LPP3, device)
- `scriptVersion` must be `MAJOR.MINOR`. `1.2.0` (semver) is skipped with no log line. `1.2` loaded on one OS state and
  was later refused the same way. Use `1.0`, as Akai's own scripts do.
- The first line must be `[info]` (matches our FINDINGS 18).
- `input=`/`output=` may be a comma list of candidates; Akai's `mpkMiniMkIV.ini` uses
  `input=MPK mini IV DAW Port,MIDIIN2 (MPK mini IV)` and LCXL3 loads with `LCXL3 1 DAW In,MIDIIN2 (LCXL3 1)`.
  (LPP3 recommends one exact name per line; both load.)
- LPP3's [`tools/check-ini.sh`](https://github.com/fuzboxz/mpc-launchpad-pro-mk3/blob/main/tools/check-ini.sh) is a
  pre-commit gate for these.

## C6. A popup can abort the app under project-scoped calls (LCXL3, device 2026-11-18)
While a dialog owns the UI (e.g. New Project), project-scoped native calls can abort MPC. LCXL3 checks
`mpc.engineMode.CurrentPopup` before every such call; on device the guard fired and refused cleanly.

## C7. Track indices: global vs normal (LCXL3, LPP3)
- `normalTracks` always has 128 slots; use `SizeNormalTracks()` / `Count.value` for the real count.
- `currentTrack.Index` is a **global** index (normal tracks first, then master). The MPC UI numbers every track, while
  `normalTracks` holds only the mixable ones, so with a non-mixable track (e.g. MIDI) in the project the two differ.
  LCXL3 learns the offset as `currentTrack.Index − position in normalTracks` whenever the current track is a normal
  one, debounced because `Index` and `Name` update asynchronously (an undebounced version oscillated on device).
- `mpc.tracks.SetCurrentTrack(i)` takes the **global** index and returns nothing: gate on the function existing,
  not on its return value.

## C8. Smaller verified facts
| Fact | Source |
|---|---|
| `normalTracks[i].VolumeDB` exists beside `Volume`; MPC's fader curve is `dB = 40·log10(gain) + 6` | LPP3 (measured) |
| Pad colours have no listener: poll `pads` `GetColour()` / `Colour.value` (~100 ms works) | LPP3 |
| Dynamic re-routing: `RemoveCCToScript` then `mappings.controlSurface.AddCCToControl(ch, cc, "Tap Tempo")` hands a button to a native MPC control by name; reverse it on release (used for Shift + Tap) | LPP3 |
| `dev.timer.Now()` returns milliseconds | LPP3 |
| `mpc.features.UseClipWorkflow.value` tells you whether the clip matrix is in use; gate the clip surface on it | LPP3 (from the APC mini mk2 script) |
| `sequencer.quantiser.{Get,Set}{Swing,Strength,TimeDivision,Window,Shift,Enabled}`, `sequencer.SetLoop`, `OverdubState`, `project.AdvanceGlobalAutomationFilterType()` | LPP3, LCXL3 |
| `currentTrack.Next/PrevPluginPreset`, `Next/Previous` | LPP3, LCXL3 (present; log if missing) |
| No readable "selected Q-link" (`qlink.Selected` / last-touched) on 3.10.1 | LCXL3 (dump) |
| The 64-pad (8x8) view is a screen setting with no script API; drum programs have no 8x8 view | LPP3 |
| Hardware pad rows count from the bottom, aperture rows from the top | LPP3 (APC pattern) |
| `pads.Tap` may ignore velocity; use `Press` + timed `Release` for velocity | LPP3 (from force-link) |

## C9. Scripts on the Force, and on 3.10.1.12 (NK)
- A signed script loads on an **Akai Force running MPC 3.10.1.12**, with the checksum made from the **3.10.0.49** salt.
  So the salt is unchanged from 3.10.0.49 through 3.10.1.12, and is the same on the Force as on the Live III.
  **Extends** FINDINGS 30 (same salt on 3.10.0.49 and 3.10.1.8, Live III).
- `input=`/`output=` match the **port name of MPC's own ALSA sequencer client**, which can differ from the USB product
  name and from `amidi -l`. On the Force the nanoKONTROL is `nanoKONTROL _ CTRL`. Check with
  `cat /proc/asound/seq/clients` (client `MPC 3`).
- `OnLoad` runs when the device **connects**, not when the script is evaluated. With a wrong port name the script loads
  but `OnLoad` never fires, so there's no error to see. It can also run more than once: guard against registering twice.

## C10. Solo vs cue on the Force (NK)
There is no scriptable cue. `normalTracks[i].Solo` only does something while the Force's SOLO preference is
"solo tracks"; in "CUE tracks" mode the only route is to emulate the buttons (`mpc.force.solo`, then that track's
`trackAssign<n>` / `trackSelect<n>`). Pressing a mode button through `mpc.force.*` changes the Force's lower-row mode,
which a script can't read back (FINDINGS 30), so NK returns to a fixed mode afterwards (`mpc.force.recArm` etc).

## C11. Engine mode names follow the menu labels (NK)
`mpc.engineMode.RequestModeChange(name)` takes names like `"Clip Editor"`, `"List Edit"`; they're undocumented and
differ by firmware. NK tries a list of candidates per view and logs `GetAvailableModes()` when none works.
Also: `mpc.sequencer.TogglePlayStart`, `mpc.sequencer.JumpToPulse(0)` (works during playback),
`mpc.force.stop.DoublePress()`. `mpc.tracks.Get/SetTrackMonitorState` error in every call form (matches FINDINGS 33).

## Still unconfirmed in those projects
`AddCCToScript` on MIDI channels other than 0 was confirmed by LCXL3 for channels 1, 7 and 16 (1-based) on device.
The `AddInfiniteCCToQLink` argument order: LCXL3 tries `(ch, cc, qlink)` then `(qlink, ch, cc)` and checks with
`IsQLinkMappedInfinite`; 16/16 mapped on device, but the log doesn't say which order won.
`clipMatrix.CopyClip/DeleteClip` and the plugin-preset calls are listed as "confirm on hardware".
