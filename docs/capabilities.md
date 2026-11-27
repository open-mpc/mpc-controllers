# What an MPC 3.10 controller script can do

MPC 3.10 has an official controller-script layer: **Lua 5.4 (sol2)** inside the MPC binary. Akai ships example scripts
for its MPK mini (`/usr/share/Akai/SME0/Devices/Akai/`). Source: registered names and docstrings in the binary, then
device tests. Verified on 3.10.0.49 and 3.10.1.8 unless noted.

## Installing a script
- A script is a `.ini` + `.lua` pair. The `.ini` has `input=` / `output=` port-name matches (order = port index),
  `script=` and `checksum=`.
- Search dirs: `/usr/share/Akai/SME0/Devices` and `<user root>/MPC Documents/Devices` (user root is `/storage`,
  `/data/UserData`, `/nvme` or `/sdcard` depending on the device).
- **Checksum = sha256(lua bytes || 36-char salt)**. The salt lives in the MPC binary; read it from your own copy, never publish it.
  The checksum check is skipped when the hardware product code is 0 (desktop software).
- The same salt worked on 3.10.0.49 and 3.10.1.8 (FINDINGS 30), and on a Force running 3.10.1.12 (COMMUNITY C9).
- The `.ini` loader refuses **silently**: first line must be `[info]`, and `scriptVersion` must be `1.0` (semver like
  `1.2.0` is skipped with no log). `input=`/`output=` match MPC's ALSA sequencer port name (`cat /proc/asound/seq/clients`),
  not the USB name, and may be a comma list of candidates. A wrong name means `OnLoad` never fires (COMMUNITY C5, C9).
- `OnLoad` runs when the device connects and can run more than once; guard your registrations.
- Scripts are read from internal storage only, once at startup: restart MPC after copying. MPC 3.10.1's developer
  mode (Preferences > Enhancements) gives SSH on a Live II (COMMUNITY C17).
- **Native code:** `io` and `os` are nil, but `package.loadlib(path, symbol)` loads a native library into MPC and
  calls it; a script with no `input=`/`output=` lines still runs at every MPC start, so it can be a boot hook (FINDINGS 35).
- **Cost:** each native function call takes ~2.9 ms; property reads are free. Budget calls per tick (COMMUNITY C12).

## API by task
| Want to | Use |
|---|---|
| Clip grid (launch, state, colours) | `clipMatrixAperture` (`SetColumns/SetRows` up to 16x16, `TriggerClip`, `TriggerRow`, `StopClipsOnTrack`, `SelectClip`, `SelectTrack`, `MoveToColumn`), `clipProperties[c][r].Colour/State/Selected` (listenable), `sceneProperties[row].State`. State enum: 0 NoSlot, 1 Empty, 2 Ready, 3 WillPlay, 4 Playing, 5 WillStopPlaying, 6 WillRecord, 7 Recording, 8 WillStopRecording, 9-11 overdub (COMMUNITY C2) |
| Mute / solo / rec arm / pan / volume / sends | `mpc.tracks` and `tracks.normalTracks[i]` (colour, name, mute, solo, rec arm, pan, volume, `VolumeDB`, sends 1-4, type). The `mpc.tracks.Get*/Set*` functions take the track **name**, called with a dot: `SetTrackRecordArmed(name, true)` (COMMUNITY C13). 128 slots always: count with `SizeNormalTracks()`. `currentTrack.Index` and `SetCurrentTrack(i)` use the **global** index, which differs from the `normalTracks` index when non-mixable tracks exist (COMMUNITY C7) |
| Q-links | `qlinks.AddInfiniteCCToQLink` (1-63 cw, 64-127 ccw; verify with `IsQLinkMappedInfinite`), `SetQLinkFineAdjust`, `qlinks[i].GetPosition/TargetName/ValueString/IsTouched/Touch/Twist`. **Not `SetPosition`**: it changes the displayed value only (COMMUNITY C3) |
| Route MIDI to a track (input port, channel, monitoring) | `mpc.tracks.SetTrackInputPort(name, n)`, `SetTrackInputChannel(name, ch)`, `SetTrackMonitorState(name, m)` and the `Get*` forms; also `GetTrackOutputPort/Channel`, `GetTrackVolume/Pan/Colour`, `IsTrackMidiTriggered/Mixable` (FINDINGS 34, device 3.10.1.12). Port values are opaque IDs, stable across restarts: read one from a track set by hand |
| Pad colours, cheaply | `pads.virtual[vp].Colour.value` (0-127, every bank, a property: no 2.9 ms call, COMMUNITY C12, C14) |
| Pads | `pads.AddNoteToMpcPad` (pad <= 15, current bank), `AddNoteToForcePad` (<= 63), `AddNoteToVirtualPad` (<= 127, absolute program pad); `pads.Tap/Press/ReleaseVirtual` (1-64); `pads.LightRGBColor/LightBrightness`; `pads.CurrentBank` |
| Press Force buttons from a script | `mpc.force.<name>` (every Force button: `Press/Release/Hold/DoublePress/Help`) |
| Copy / delete clips and scenes | `clipMatrix.CopyClip(sc, sr, dc, dr)`, `DeleteClip(c, r)`, `InsertRows(r, n)` + `CopyRow(src, dst)` (COMMUNITY C16) |
| Edit notes | `selectedClip.events`: `CreateNoteEvent`, `DeleteEvent`, `SetNote`, `SetNoteVelocity`, `MoveEvent`, `SetNoteRatchet` |
| Transport / undo | `sequencer.*`, `commandManager.Undo/Redo`, `clipMatrix.StopAllClips` |
| Send MIDI out (LEDs, sysex) | `midiOut.SendMIDINoteOn/CC/Sysex(outputIndex)` |
| Receive MIDI | callbacks `HandleMIDINoteOn/Off`, `HandleMIDICC`, `HandleMIDISysex`, `HandleTimer` |
| Show feedback | `ui.ShowTUIFeedback`, `popups.*`, aperture `Show()/SetLabel()` |
| Switch views | `engineMode.RequestModeChange(name)`; names follow the menu labels (`"Clip Editor"`, `"List Edit"`), list them with `GetAvailableModes()` (COMMUNITY C11) |
| Hand a button to a native control at runtime | `RemoveCCToScript`, then `mappings.controlSurface.AddCCToControl(ch, cc, "Tap Tempo")`; reverse on release (COMMUNITY C8) |
| Quantise, loop, automation | `sequencer.quantiser.{Get,Set}{Swing,Strength,TimeDivision,...}`, `sequencer.SetLoop`, `project.AdvanceGlobalAutomationFilterType()` |
| Run native code (files, commands, services) | `package.loadlib(dir .. "lib.so", "fn")()`; find `dir` with `debug.getinfo(1, "S").source`. Start long-lived work with `systemd-run`, or it dies with MPC (FINDINGS 35) |
| Everything else | [api-reference.md](api-reference.md): every name on 3.10.1.8, from fuzboxz's table dump (COMMUNITY C1) |

## Useful facts
- **Q-link numbering is column-major** on the Live III: knob j (0-3, left to right) in bank b is Q-link `4*(3-j)+b` (FINDINGS 26).
- Drum NOTE layout on a 64-pad Force: banks A-D are 4x4 quadrants, A bottom-left, B bottom-right, C top-left, D top-right (FINDINGS 20).
- Pads, step sequencer, Force buttons, per-track state are all reachable; see [recipes](recipes/).
