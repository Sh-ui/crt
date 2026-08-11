# crt-filter -- the global CRT screen filter (clockwork-crt)

`clockwork-crt` puts a CRT look over the entire desktop -- scanlines, an optional phosphor mask, vignette, rounded tube corners -- as a fullscreen click-through ARGB overlay window that mutter (always compositing) alpha-blends over everything: the same architecture as the warmth tint in [[handoff-display-warmth]], with a patterned pixmap where warmth has a solid pixel. Every gamma/LUT path is a silent no-op on the CM4 (HVS5 gamma block disabled in vc4, see [[handoff-display-warmth]] and [[lessons]]), and DRM overlay planes are master-locked under a running X server, so a compositor-blended window is both the pragmatic path and, on this stack, the performant ceiling. Filter toggles live -- no reboot, no rebuild.

## Why it costs (almost) nothing

The pattern is rendered ONCE per config change (numpy, in a warm `--render-worker` child that imports numpy once, turns a re-render around in ~0.2 s on the CM4, and exits after 60 s idle so nothing stays resident) and uploaded to a server-side pixmap as the window background; after that the daemon blocks on select(X socket, signal pipe) exactly like the warmth overlay -- zero polling, zero per-frame work (house rule, [[lessons]]). mutter composites every frame it paints anyway, so the overlay adds one textured blend only to frames something else already damaged: an idle screen costs exactly nothing extra. The honest costs, both shared with an active warmth tint: an always-on-top translucent window blocks mutter's fullscreen unredirect (fullscreen apps keep paying compositing they could otherwise skip), and dark patterns eat effective brightness (the tube preset's period-3 lines at 55% cost ~18% light -- compensate with backlight at battery cost, or run the `subtle` preset). The hardware cursor is composited above everything, so the pointer escapes the filter (also true of warmth).

## Fit to this exact panel

720 landscape rows = exactly 3 x 240, so `scanline_period=3` is an integer-scaled 240-line CRT: no moire (the classic crt-pi shader failure at non-integer scales), and a 3 px period at this panel's ~294 PPI is ~20 cycles/degree at 30 cm -- the same angular line pitch as a real desktop CRT. The panel's RGB stripe runs along its native-portrait short axis, which lands VERTICALLY through each landscape pixel, so a landscape aperture grille cannot align with hardware subpixels: `mask_type=grille` is a whole-pixel approximation (translucent primary stripes) that hazes dark content, and defaults off. Panel facts live in [[reference-clockwork-pi]]; research sources in [[meta/references|references]].

## Config port

`~/.config/clockwork/display-crt` -- key=value, `#` comments, fail-soft (a bad value keeps the compiled default and warns, never crashes), validated by `clockwork-crt --check` (resolved keys + warnings + a timed test render). Any set/apply re-reads it live.

| Key | Meaning |
|---|---|
| `enabled` | 0/1 master switch |
| `strength` | 0..100 master dial over all layer opacities (corners exempt: geometry, not intensity) |
| `scanline_period` / `scanline_thickness` | px per line cycle (3 = 240-line integer scale) / dark px per cycle |
| `scanline_opacity` / `scanline_softness` | 0..100 darkness / 0..100 edge feather |
| `convergence` | 0..100 red/blue fringing on scanline edges -- CRT gun-misconvergence, the chromatic-aberration look a static layer CAN fake (panel label: `chroma ab`) |
| `glow` | 0..100 warm glass glow over the whole stack (capped at a 15% wash) -- halation, the static face of bloom: lifts blacks toward tube-glass grey, softens lines into their gaps |
| `mask_type` | `none` \| `grille` (RGB stripe approximation) \| `grid` (black pixel lattice) |
| `mask_period` / `mask_opacity` | px per mask cycle / 0..100 |
| `vignette_opacity` / `vignette_size` | corner darkening 0..100 / reach toward center 0..100 |
| `corner_radius` | px rounded tube corners |

Presets (`--preset tube|subtle|grille|terminal`) are compiled starting points that write into the config file; the file stays the one editable port. A demo render is available off-device: `clockwork-crt --render-demo out.ppm 1280x720` composites the pattern over synthetic color bars.

What a static layer CANNOT do: content-driven bloom, true chromatic aberration, and curvature all sample and displace the screen CONTENT, which needs a per-frame shader in the compositor -- mutter 3.38 standalone has no shader hook, so that path means a mutter MetaPlugin in C or a compositor swap (xfwm4+picom), GPU-affordable at 720p per the crt-pi numbers but desktop surgery with a standing per-frame cost -- parked as a decision, [[tasks]] T71. Two knobs are the honest static approximations: `convergence` (real CRT color fringing came from misconverged guns tracing each scanline offset by ~1 px -- scanline-locked, not content-locked, so a static layer reproduces it faithfully as red-above/blue-below line-edge fringes) and `glow` (halation: the uniform glass-glow component of bloom, minus the bright-content flare).

## Surfaces

- **GUI** `clockwork-crt` -- faux-TUI control panel in the house style ([[tui-visual-style]]), centered on open, keyboard-first: one focus bar walks every row (on/off, preset selector, each knob) with Up/Down, Left/Right changes the focused row, Enter/Space activates, and the slate footer key-hints are themselves the mouse targets (no separate button row). Mouse works everywhere (click focuses/activates, drag scrubs a gauge). The panel ground follows the desktop color mode via `~/.config/clockwork/color-mode`, read at launch (the flit exception extended here by user preference 2026-07-12; relaunch after a mode flip) -- the light-side chrome hexes come from the [[tui-visual-style]] table. Commits are debounced 150 ms and the warm worker puts a tweak on glass in ~0.4 s. Launcher `config/desktop/CRT-Filter.desktop` with a bespoke tube-TV icon (`config/desktop/icons/clockwork-crt.svg`, Tabler-style cream stroke; both icon sets rendered by the clockwork-lab installer's targeted rsvg pass, and the panel wears the mode-matching set).
- **CLI** `--on` / `--off` / `--toggle` / `--set k=v ...` / `--preset NAME` / `--status` / `--check` / `--apply` (login hook).
- **Mac** `cw crt` opens the panel on the Pi; `cw crt --toggle` etc. pass through to the CLI.
- **Login** `clockwork-ensure` runs `clockwork-crt --apply`; clockwork-lab's `bin/install-pi.sh` symlinks the script and launcher out of this repo's clone at `~/vault/dev/crt`. Pi dependencies: `python3-xlib` (required), `python3-numpy` (present on this image; without it a slow pure-python fallback renders scanlines/mask only).

## Sibling protocol (shared with clockwork-warmth)

mutter composites override-redirect windows in map order ([[handoff-display-warmth]]), so each persistent overlay remaps itself on foreign MapNotify to stay on top -- and two such daemons would remap-war forever ([[lessons]]). Siblings are detected two ways, because a pre-upgrade daemon carries no class: the shared WM_CLASS class `ClockworkOverlay`, or the shape of a screen filter itself (override-redirect, depth 32, covering the root). Both daemons also check their signal pipe once per event (an event storm must never starve SIGTERM), rate-limit remaps to 8 per 2 s so any future storm degrades to "not on top" instead of a war, and install their signal handlers BEFORE the slow startup work (a launch-time poke used to hit the default action and kill a daemon mid-first-render). Relative order between the two filters is last-mapped-wins, which is visually immaterial: a uniform low-alpha tint and a dark pattern near-commute under src-over.

Durability rests on three symlinks and one login hook: `/usr/local/bin/clockwork-crt` and `~/.local/share/applications/CRT-Filter.desktop` point at this repo's clone, the tube-TV icon is rendered into both PiXombre sets, and `clockwork-ensure` runs `clockwork-crt --apply` at login so the overlay respawns with whatever `~/.config/clockwork/display-crt` holds. **On the Pi today those symlinks still point into the clockwork-lab clone: deployment from this repo is applied on the Mac side only, reboot-persistence UNVERIFIED, blocked on the device being offline since 2026-08-06.** What is proven is the filter itself -- render engine, overlay daemon, and control panel were exercised live on-device 2026-07-12/13 (tube preset in `cw shot`, clean `--off` exit, no war against the old warmth daemon, panel driven by the user with settings persisting to config) and it came up on every reboot from the old path, confirmed by the user, last checked 2026-07-22.

## Open threads

- T71 -- real CRT shader pass (bloom / chromatic aberration / curvature), tracked in [[tasks]] while the monorepo still holds the task hub.

## Linked from

The pages that link here live in [[clockwork-lab]], and their `[[crt-filter]]` links land on that repo's moved-out pointer rather than this page: [[handoff-display-warmth]], [[lessons]], [[meta/references|references]], [[reference-clockwork-pi]], [[tui-visual-style]], and the docs index. Two pages of the same basename is the cost of keeping those links resolvable; the pointer names this page by path.
