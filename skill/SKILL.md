---
name: mpc-controller-scripts
description: Write or debug MPC 3.10 Lua controller scripts (Akai MPC Live III / Force surfaces, MIDI controllers): the available API, what it cannot do, and verified quirks. Use when the user mentions MPC controller scripts, the Devices folder, script .ini/.lua, Q-links, clip matrix, aperture, mpc.force, or driving an MPC from a hardware controller.
---

# MPC 3.10 controller scripts

Start from the docs in this repo (paths relative to the repo root):
1. `docs/capabilities.md`: the API by task and install rules.
2. `docs/api-reference.md`: every API name on 3.10.1.8 with notes; a name being present doesn't mean it works.
3. `docs/limitations.md`: check this **before** promising a feature (no state dump, no non-current-track editing; per-track MIDI input *is* settable by track name, FINDINGS 34; `package.loadlib` loads native code, FINDINGS 35).
4. `docs/findings/FINDINGS.md`: dated evidence; cite the finding number and firmware build.
5. `docs/findings/COMMUNITY.md`: findings from other people's scripts (Launchpad Pro MK3, Launch Control XL 3, nanoKONTROL on Force, Ableton Move on Live II), incl. silent `.ini` failures, the `SetPosition` trap, the ~2.9 ms cost of native calls, and name-keyed `mpc.tracks` functions.
6. `examples/force-link/`, and the community scripts linked in `README.md`: full working controllers to copy patterns from.

Rules: claims are firmware-specific (3.10.0.49 / 3.10.1.8); say which build you are relying on. Never publish or
ask for the signing salt or Akai binaries. Scripts need a valid checksum on hardware.
