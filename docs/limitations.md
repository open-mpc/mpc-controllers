# What a controller script cannot do

Each wall lists how we hit it and what (if anything) gets round it.

| Limit | Evidence | Way round |
|---|---|---|
| **No state dump**: `#GetState` / `mpc.GetState` are nil in a script (the registrar exists in the binary but doesn't reach Lua) | FINDINGS 19, 25 (device, 3.10.0.49) | Read MPC's memory from a preloaded shim (needs root), or from a library the script loads with `package.loadlib` (FINDINGS 35) |
| **No LED/light state for Force controls**: scripts get no feedback for `mpc.force.*` buttons | FINDINGS 30 | Derive lights from state you can read, or dim/bright-while-held |
| **Track input-port values are opaque IDs**, not indices or name hashes (0x7FFFFFFD = default/All; e.g. Force Link = -809768406). Stable across a restart and saved in the project; unknown across units or firmware | FINDINGS 34 (device, Live III 3.10.1.12) | Set the port by hand once and read it with `GetTrackInputPort(name)`, or copy it from a track the user set |
| **Cannot drive a non-current track** directly: `aperture.SelectTrack` moves MPC's own current track; there is no separate controller track | FINDINGS 33 | Route by MIDI input instead: set a track's input port/channel/monitoring by name (FINDINGS 34) |
| **Note editing only on the selected clip**: no per-clip events object | FINDINGS 33 | Select the clip first, or native route |
| **Scale/root of pad perform is invisible**: `padPerform`/`keyboardControl` are only the two buttons | FINDINGS 29 | Read from MPC's memory (offsets per build) |
| **`qlinks[i].GetPosition()` lags one change**; `Twist` wraps | FINDINGS 26 | Use MPC's own infinite-CC mapping; cap per-step deltas |
| **`qlinks[i].SetPosition` doesn't reach the parameter**: only `ValueString` changes | COMMUNITY C3 (device, 3.10.1.8) | `Touch()` + `Twist(delta)`, or route the CC with `AddInfiniteCCToQLink` |
| **MIDI the script sends doesn't loop back through `mappings.*`** | COMMUNITY C4 | Call the control API directly |
| **Native calls can abort MPC while a popup is open** | COMMUNITY C6 (device, 3.10.1.8) | Check `engineMode.CurrentPopup` first |
| **The `.ini` loader fails silently** (bad `scriptVersion`, missing `[info]`, wrong port name) | COMMUNITY C5, C9 | `scriptVersion=1.0`; port name from `/proc/asound/seq/clients`; a check like LPP3's `check-ini.sh` |
| **No scriptable cue on the Force**: `Solo` only works in "solo tracks" mode | COMMUNITY C10 (Force, 3.10.1.12) | Emulate `mpc.force.solo` + the track button |
| **Pad colours have no listener**; no readable selected Q-link; no API for the 8x8 pad view | COMMUNITY C8 | Poll pad colours (~100 ms) |
| **No `io` or `os` in the Lua VM**: no files, no commands from Lua itself | FINDINGS 35 (device, Live III) | `package.loadlib` works: put that work in a native library |
| **Each native function call costs ~2.9 ms** (one audio block); 32 per tick stall the screen | COMMUNITY C12 (device, Live II) | Poll properties (`.value`), which are free; keep function calls for actions |
| **Not reachable**: doubling a clip's length, metronome play mode, a plugin's name, retrospective record (except by pressing Shift + Rec) | COMMUNITY C15, C16 | |
| **Q-link bank changes update all 16 at once** | FINDINGS 26 | Follow a bank only when exactly one Q-link changes |
| **No USB access** from Lua (e.g. Push 2 display is bulk USB) | FINDINGS 2 | A bridge device, or native code loaded with `package.loadlib` (FINDINGS 35) |
| **Gen 2 (Live III) has no stock root**: SSH needs an Akai-signed cert, rootfs signed | FINDINGS 2, 7, 30 | Out of scope here |
| **Latency over Wi-Fi** 30-50 ms TCP RTT | FINDINGS 31 | USB networking breaks the Force's own controller (hub mux); USB mode works (FINDINGS 32) |
