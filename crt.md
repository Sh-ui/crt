# crt

The uConsole's global CRT screen filter. `clockwork-crt` draws scanlines, an optional
phosphor mask, a vignette, and rounded tube corners as one fullscreen click-through ARGB
overlay window; mutter is always compositing, so it alpha-blends the pattern over the whole
desktop for free. The pattern renders once per config change into a server-side pixmap, so
after that the daemon does no per-frame work at all.

- **kind:** [[app]]
- **runs-on:** the uConsole Pi. Authored on the Mac, deployed to the Pi.
- **repo:** `Sh-ui/crt`, cloned at `~/vault/dev/crt`.
- **reference:** [[crt-filter]] -- architecture, the panel geometry math, the knobs, and what
  a static layer cannot do.

Extracted from [[clockwork-lab]] as the dissolution pilot, the first app to leave the
monorepo and become its own vault citizen. Its sibling overlay `clockwork-warmth` stays in
the monorepo for now, and the two share a map-order protocol that neither can change alone.

The panel is keyboard-first in the house TUI style: one focus bar walks every row, the
footer key hints are themselves the mouse targets, and the ground follows the desktop color
mode. Knobs live in `~/.config/clockwork/display-crt` and `clockwork-crt --check` validates
them.
