# crt -- operating rules

`clockwork-crt`, the uConsole's global CRT screen filter. Extracted from
[Sh-ui/clockwork-lab](https://github.com/Sh-ui/clockwork-lab) as the dissolution pilot;
work happens here on `master`, never back in the monorepo.

Cloned at `~/vault/dev/crt` on every machine that touches it. **Mac** authors, **Pi**
runs. Deployed copies (`/usr/local/bin/clockwork-crt`, the launcher, the icons) are
symlinks back into this clone -- **edit here, never the deployed copy**.

## Reaching the Pi

Use the `cw` CLI: `cw sh` / `x` / `shot`. Never `raspberrypi.local`, never a LAN IP.
If the Pi does not answer it is genuinely offline -- report it, do not chase addresses.

## The bar

- **Reboot-durable, reboot-verified.** A live-session test is not proof. Anything you
  cannot reboot-test is "applied, reboot-persistence UNVERIFIED", never "done".
- **Config-first, from v1.** Every knob lives in `~/.config/clockwork/display-crt` with a
  compiled default and fail-soft parsing; `clockwork-crt --check` is the validator entry
  point. A new knob that is not readable from the config file is incomplete.
- **No idle polling loops.** The overlay blocks on `select(X socket, signal pipe)` and does
  zero per-frame work after the one-time render. A reassert loop is banned on this
  handheld; fix the root declaratively or hook the event instead.
- **The sibling protocol is load-bearing.** This daemon and `clockwork-warmth` (still in
  the monorepo) both remap on foreign MapNotify and would remap-war forever without it.
  Sibling detection by WM_CLASS `ClockworkOverlay` AND by shape, the signal pipe checked
  once per event, remaps rate-limited, and signal handlers installed before the slow
  startup work -- all four fixes are scar tissue. Do not simplify them. Any change to
  either daemon's map handling is a change to both.
- **Docs are soft-wrapped.** One logical line per paragraph or list item; `docs/` is read
  in ekphos, which wraps on its own. Wikilinks resolve vault-wide, so links into the
  clockwork-lab docs still work.
- **Never plain `rm`.** `trash`, or `mv` to `~/.local/state/junk/`; git removals use
  `git rm`.

## Ledger

Changes here are logged in the vault ledger the same way as the monorepo:
`~/vault/dev/clockwork-lab/bin/vault-log <area> <action> "<what>" "<why>" <files...>`.
It is append-only.
