# Changelog

PS Vita port of [mstan's SuperMetroidRecomp](https://github.com/mstan/SuperMetroidRecomp).
All credit for the recompilation, the framework and the widescreen renderer goes to the
original authors — see [CREDITS.md](CREDITS.md).

## r28 (SuperSlopified fork) — 2026-11-10

### Sound emulation: whole-instruction stepping
- The SPC700/DSP emulator advanced one clock cycle per call (~17,000 calls a
  frame), re-checking the port queue and counting down three timers each time,
  although an instruction's work happens entirely at its first cycle.
- `apu_runToGuestCycle` now steps a whole instruction at once when no CPU port
  write is due within 24 cycles and the run target is at least that far away:
  same opcode order, DSP samples at the same cycle numbers, timers advanced by
  arithmetic. Anything closer uses the unchanged per-cycle path.
- Verified bit-identical against the per-cycle path on the host: 40 randomized
  programs with random port writes, comparing SPC state, timers, all 64 KB of
  APU RAM and every output sample. `-DSNESRECOMP_APU_NO_BATCH` restores the old
  behaviour.
- Expected effect on the Vita: less CPU for the background sound thread (the
  `apu bg=` figure in the perf line), so it falls behind less often.

### Sound thread on its own user core (core 2)
- The background sound thread is pinned to core 2 at priority 159, just above
  the drawing worker that shares that core. Previously it scavenged idle time
  on cores 0-2 at idle priority.
- Tried and measured first: the system core (core 3, with CapUnlocker). The
  thread's cost roughly doubled there (about 5 ms to 9-10 ms a frame) because
  core 3 also carries the picture compositor and the kernel's own work, so it is
  no longer the default. No plugin is needed for the new placement.
- Empty files in `ux0:data/SuperMetroidRecomp/` select the other placements for
  comparison: `apu_core3` (system core; needs the plugin) and `apu_idle` (the
  r27 placement). These replace `apu_no_core3` and `apu_old_affinity`.
- `log.txt` reports where the threads landed (`[apu] placement: ...`,
  `[compose] placement: ...`), and each perf line now ends with
  `audio[under= drop= minocc=]`: buffer underruns, samples dropped to a full
  buffer, and the lowest buffer level the audio device found in that window.

## r27 (SuperSlopified fork) — 2026-11-09

Fork of DarthSully's r21 with Vita performance work. Build from source (see [SETUP.md](SETUP.md));
macOS now works as a build host too.

### Performance
- **Leaner interpreter on the Vita.** Code that still runs in the 65816 interpreter skips the PC build's
  report-only bookkeeping (per-page tallies, the edge map), reads WRAM through a direct fast path, and keeps
  its hot bus-path counters 32-bit (one load/add/store each on the Vita's 32-bit CPU instead of two).
  Gameplay went from ~58.7 to ~59.5 fps in profiled runs. The PC build is unchanged
  (`SNESRECOMP_LEAN_INTERP`, Vita only).
- **Steadier frame pacing.** The frame wait sleeps most of the time and spins the last ~2 ms: a Vita thread
  sleep can wake several ms late when the cores are busy, which was the cause of 47-52 fps windows in which
  the game thread was mostly idle. An empty `sleepwait` file in `ux0:data/SuperMetroidRecomp/` restores the
  old sleep-only wait.

### Fixes
- **No more audio dropouts while profiling.** `log.txt` was flushed line by line, so a log line meant a
  memory-card write on the thread that printed it (game thread included) — those writes can block for
  20-80 ms. The log is now buffered and written by a low-priority background thread four times a second.
  A crash can lose the last ~0.25 s of log; an empty `synclog` file restores line-buffered logging.
- **Build script works on macOS** (`shasum` fallback for `sha256sum`, `sysctl` for the core count) and finds
  the VPK by its real name.

### Tip: 16:9 vs Fit
- **16:9 costs about 1.5 ms per frame** on the Vita (extra widescreen drawing and compositing) compared with
  Fit (4:3). It holds 60 fps in normal play, but leaves little headroom in heavy rooms (lava, many enemies,
  power bombs). If a spot dips, switch to Fit: double-tap the left half of the touch screen. The choice is
  saved to `sm-video.ini`.

### Known issues
- Power bombs and elevators can still crackle; the background APU thread runs at idle priority and is
  starved in heavy frames. Setting `AudioSamples = 2048` in config.ini helps.

## r21 — 2026-10-27

Download: [downloads/SuperMetroid-Vita-r21.vpk](downloads/SuperMetroid-Vita-r21.vpk)

### Performance
- **Door transitions are much faster.** The game decompresses each room's graphics
  through an entry point (`$80:B0FF`) that takes its source address from the bytes after
  the call. That entry was never declared to the recompiler, so all 63 call sites ran the
  whole decompression in the slow interpreter — roughly 1.3 s of CPU per door on the
  Vita. It is now native code. Over 43 door transitions on the desktop build, interpreted
  work dropped by 82%. This removes most of the door-transition slowdown and the audio
  cuts that came with it.
- `LoadLevelScrollAndCre` (room loading) is now native code too; its starting register
  widths were declared wrong, so the recompiler had always skipped it.
- About 100 more routines compile as a side effect: the recompiler had been misreading
  the argument bytes after those 63 calls as instructions.
- **Full speed again with current VitaSDK.** Newer VitaSDK builds of SDL2 default to an
  OpenGL ES renderer (through vitaGL) that cost about 12 ms per frame and several seconds
  at start-up. The port now asks SDL2 for its native GXM renderer. The log lists the
  available renderers and the one in use.

### Fixes
- **16:9: junk beside the pause map.** While the map faded out after unpausing, the
  widescreen renderer treated the frame as gameplay and drew room tiles in the side
  areas. Fixed (the 4:3 view was never affected).

### Build
- Links with current VitaSDK: `vitaGL`, `vitashark`, `SceShaccCgExt`, `taihen_stub`,
  `mathneon`, `SceShaccCg_stub` and `stdc++` are named explicitly, in dependency order.
- Desktop builds: `SM_PIPELINE=1` reproduces the Vita's pipelined compose thread, for
  testing the Vita rendering path on a PC.

### Known issues
- The room-object handler (`PlmHandler_Async`) still runs in the interpreter on purpose:
  item and save message boxes pause inside it. Rooms with many objects can still dip
  below 60 fps.
- In 16:9, the side areas show enemies the game keeps in memory. Enemies that only spawn
  or move once they enter the original 4:3 view (for example Zebs coming out of pipes)
  appear when they reach it — that is how the game itself works.
- The door-transition speed-up was measured on the desktop build; the Vita gain is
  expected to be larger (about 3 µs per interpreted instruction there).
- A full playthrough (Mother Brain, escape, credits) has not been verified on the Vita.

## r18 — 2026-10-20

First public release.
- Door-transition freeze fixed (an enemy routine returning through an uncompiled
  function leaked stack bytes).
- Saves persist when the game is closed with the PS button (battery SRAM is written
  shortly after every in-game save).
- Skipping the new-game intro no longer gives 900 missiles or changes the controls.
- The game's NMI handler, sound queue and Samus routines run as native code.
- Faster 16:9 drawing (vectorized layer blending) and the audio thread at idle priority
  across cores 0–2.
- Runs at 444 MHz.
