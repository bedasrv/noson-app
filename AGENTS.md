# AGENTS.md — noson-app + libnoson (PipeWire low-latency branch)

## Branches

- lib: `bedasrv/noson:pipewire-low-latency` (worktree: `/tmp/opencode/noson-work`)
- app: `bedasrv/noson-app:pipewire-low-latency` (this repo)
- Full findings: `PIPEWIRE-FINDINGS.md` in this directory. Read it first.

## Build

- No system cmake: use `/tmp/opencode/cmake-3.30.5-linux-x86_64/bin`.
- After lib edits, sync `backend/lib/noson` from `/tmp/opencode/noson-work`
  (`noson/` + `CMakeLists.txt`) for app builds, then
  `git checkout -- backend/lib/noson` to restore.
- Install prefix is `/usr/local` (`/usr/bin/noson-app` is the distro build;
  mind which one launches). Our branch builds with
  `HAVE_PIPEWIRE=1 HAVE_PULSEAUDIO=1`.

## Audio test safety (Bedroom Sonos @ 192.168.100.155)

- Laptop sink stays MUTED during every audio test.
- Mic tests use raw `alsa_input.pci-0000_00_1f.3.analog-stereo` at 30%,
  never `easyeffects_source`. Keep tests to short windows (~30-45s).
- Ask before touching Bedroom playback if the user is listening to
  something; restore prior state when done.

## Runtime rules (each cost real debugging time)

- ONE `noson` instance at a time. Duplicate sink names cause silent
  misroutes. Always `pgrep -x noson-gui` / `pgrep -x noson-cli` first.
  Never `pkill -f "pattern"` when your own command line contains the
  pattern (self-match SIGTERM); use `pkill -x`.
- The GUI hides to tray on close — closing the window does NOT quit it.
- GUI logs go to the journal, and DBG macros are off there: diagnose with
  `noson-cli --debug` runs, not the GUI.

## PipeWire traps

- Never pump the graph yourself alongside hardware (`PW_STREAM_FLAG_DRIVER`
  + `trigger_process` double-drives → measured 2.1x). Behave like
  `easyeffects_sink`: plain node, no triggers.
- `pw_stream` param_changed does NOT deliver Props. Subscribe on your own
  node via core → registry → bind `pw_stream_get_node_id` (invalid until the
  server creates it; retry lazily). Volume is scalar Float `SPA_PROP_volume`.
- Native sink monitors are PRE-fader (full-scale regardless of slider).
  Never apply 1/v gain to monitor audio. Tap buffers are post-fader.
- Verify rates by measurement (pull N seconds, count audio seconds), never by
  reasoning about pacing. Content analysis (gaps, clipping fraction,
  autocorrelation) beats throughput graphs.
- `wpctl`/`pactl` need a session bus and flake out headless; prefer `pw-dump`.
  Never feed unchecked multiline IDs into `set-volume`.
- pipewire-pulse null-sink monitors are SILENT (even for `paplay`); muted-sink
  monitors are silent too. Generate test audio in-process instead.

## Working agreements

- No autostart, no background services, no settings changes without asking.
- Confirm listening verdicts by ear for audio quality; numbers alone missed a
  2x-speed bug and an 80%-clipped stream.
- Commit + push both repos when asked; upstream PRs stay unopened until
  requested.
