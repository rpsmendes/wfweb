# IC-9700 memory-channel support (groups, recall, MEM mode)

Three related fixes to the web memory browser, driven by IC-9700 repeater
use. Server build only (`resources/web/` + `src/webserver.cpp`); the
Standalone build is untouched.

| Commit | Problem |
|---|---|
| `fix(web): read the rig's real memory groups instead of hardcoded 0` | Memory browser showed nothing / "scanned forever" |
| `feat(web): recall memories on the radio so tone/offset apply` | Clicking a memory set freq + mode only — no tone/offset |
| `fix(web): memory recall now switches the rig to MEM mode` | Recall stayed on VFO A/B, so repeater tone/offset never engaged |

---

## Background: how IC-9700 memories work over CI-V

- The IC-9700 has **three memory banks, one per band** — group `1` = 2 m,
  `2` = 70 cm, `3` = 23 cm (`MemGroups=3`, `MemStart=1` in
  `rigs/IC-9700.rig`; each band's `MemoryGroup=` maps to one).
- **Reading** a channel (`CI-V 1A 00`) is group-addressed: the frame carries
  a group byte (`MemFormat` starts with `%1.1a`). `1A 00 <group> <ch>`.
- **Recalling** a channel (`CI-V 08`) is **not** group-addressed — it acts on
  the *currently selected band's* bank. `08 <ch>` selects a channel; a bare
  `08` (no channel) is what the front-panel **V/M** button sends and flips the
  operating mode to memory.
- A recalled memory channel restores everything the rig stored in it:
  frequency, mode, filter, data mode, duplex direction, offset frequency,
  repeater tone, tone squelch, DTCS.

Other rigs differ: the IC-7300/7610 have no group byte (`MemGroups` is
nominal), the IC-705/905/R8600 have a real group-select command
(`funcMemoryGroup`, `08 A0 <grp>`), and Kenwood/Yaesu use `funcMemorySelect`,
which enters memory mode on its own.

---

## Fix 1 — read the rig's real memory groups

**Was:** the browser always requested `group: 0`. On the IC-9700 group 0 is
invalid, so every one of the 99 channel reads was NAKed; the 500 ms scan
timer ground through all 99 slots (~50 s) and reported "0 found" — the
"scans forever" symptom.

**Now:**

- Backend adds `hasMemories`, `memGroups`, `memStart`, `satMemories` to the
  `rigConnected` push and `buildInfoJson()`.
- The web UI builds the valid group list (`memStart … memGroups`, matching
  desktop wfview's `memories.cpp`) and shows a **Group** selector in the
  memory modal when a rig has ≥ 2 groups. Rigs with no groups keep `group 0`,
  exactly as before.
- `refreshMemories()` and `writeMemory` use the selected group. Late channel
  replies from a group the user has switched away from are dropped.

No CI-V/protocol changes — the backend already passed an arbitrary `group`
through to the codec.

---

## Fix 2 — recall on the radio (new `recallMemory` command)

**Was:** clicking a memory row sent `setFrequency` + `setMode`. The rig
stayed in VFO mode with none of the channel's tone / tone squelch / duplex
offset — useless for working a repeater.

**Now:** a new WebSocket command tells the *radio* to recall the channel, so
its firmware loads the full contents.

```jsonc
{ "cmd": "recallMemory",
  "channel": 5,          // memory channel number
  "group": 2,            // memory group (0 if the rig has none)
  "frequency": 145600000,// the channel's stored freq, in Hz — used for band nudge
  "mode": "FM" }         // only used by the non-Icom fallback
```

Backend sequence (`src/webserver.cpp`, `handleCommand` → `recallMemory`):

1. **Get onto the right band.** If the rig has `funcMemoryGroup`, send it.
   Otherwise (IC-9700 / IC-9100) `setFrequency` to the channel's stored
   frequency — an in-band frequency puts the rig on the band whose memory
   bank `08` will address.
2. **Select the channel** — `funcMemoryMode` with the channel value → `08 <ch>`
   (Kenwood/Yaesu: `funcMemorySelect`).
3. **Enter memory mode** (see Fix 3).
4. `requestVfoUpdate()` so the browser's freq/mode display follows the
   recalled channel.

If the rig has no channel-select command at all, it falls back to loading the
stored frequency + mode (no tone/offset, but better than nothing).

---

## Fix 3 — switch the rig to MEM mode

**Was:** step 2 above pre-selects the channel but leaves the IC-9700 on
VFO A/B, so the recalled tone and offset never take effect.

**Now:** after the channel select, `recallMemory` issues
`funcSelectVFO(vfoMem)` — which `icomcommander` maps to the bare `CI-V 08`
that the front-panel **V/M** button sends. This flips the rig to memory mode
*and* marks the command queue's `rigState.vfo` as `vfoMem`. Icom-only;
`funcMemorySelect` already enters memory mode.

> The CI-V order is **channel-select then bare `08`**, matching desktop
> wfview's memories dialog. If on real hardware a recalled channel doesn't
> "stick" when memory mode engages, swapping the two is a one-line change.

### MEM mode in the UI

- Backend reports `selectedVfo: "MEM"` from `buildStatusJson()` and the
  `funcSelectVFO` cache path whenever `rigState.vfo == vfoMem`.
- The `selectVFO` command accepts `"MEM"` (browser-side V/M).
- The VFO indicator (top-right, above the frequency) shows **MEM**, and
  clicking it **cycles `A → B → MEM → A`** — MEM only appears on rigs that
  expose memory channels. Recalling from the memory browser also flips the
  label to MEM immediately (optimistic; the status poll confirms).
- In MEM mode the frequency display tracks the recalled channel
  (`msg.frequency`) and does not overwrite either VFO's cached frequency, so
  returning to VFO A/B restores the right values.

---

## Not covered / verify on hardware

- **Editing** tone / offset / duplex from the web memory-write form is still
  not supported — only recall of channels already programmed on the radio (or
  in desktop wfview). That's a separate change.
- **Cross-band recall** (recalling a 70 cm memory while on 2 m) relies on the
  `setFrequency` band nudge landing before the `08`. CI-V is serialized by the
  queue, so this should hold, but it wants a real-rig check.
- Whether the IC-9700 broadcasts its VFO/MEM state over CI-V transceive. The
  indicator updates optimistically and via the ~1 s status poll regardless.
- Standalone build — same memory-group bug exists there but is out of scope
  (USB-only; not used for LAN rigs).

---

## Files

| File | Change |
|---|---|
| `src/webserver.cpp` | `recallMemory` command; memory caps in `rigConnected` / `buildInfoJson()`; `selectVFO` accepts `"MEM"`; `selectedVfo: "MEM"` reporting |
| `resources/web/index.html` | Group selector in the memory modal; `recallMemory` on row click; `currentVfo === 'MEM'` state, label, and A→B→MEM cycle |
