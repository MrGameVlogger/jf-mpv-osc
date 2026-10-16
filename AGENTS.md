# jf-mpv-osc

## Project
- Standalone OSCs for mpv: `trickplay-jf-osc.lua` (Jellyfin-styled — the
  README is the integration contract of record) and `trickplay-osc.lua`
  (vanilla-styled stock osc.lua with thumbnail support)
- Fork of https://github.com/iwalton3/jf-mpv-osc — `gh` defaults to the
  upstream repo; use `-R MrGameVlogger/jf-mpv-osc` or
  `gh repo set-default MrGameVlogger/jf-mpv-osc` once

## Keeping copies in sync
- `MrGameVlogger/mpv-config` vendors `trickplay-jf-osc.lua` byte-identically
  as `scripts/trickplay-jf-osc.lua`. After changing it here: copy the file
  there (PR), verify with `diff`, and pull the live config
  (`git -C ~/.config/mpv pull`)
- The README's Contents block quotes the script line counts — update them
  in the same PR when either file changes

## Testing
- `tools/osc-test/` — headless harness (real mpv under Xvfb + xdotool);
  `OSC_SCRIPT` selects which script it drives
- Portable alternative (works on macOS): open sheets with the
  `shim-jf-osc-menu` script message, navigate with `keypress` (sheets bind
  UP/DOWN/ENTER/ESC), assert on JSON IPC properties and files — this is how
  the hostless fallbacks were verified

## Releases
- Cut a GitHub release when the scripts change; include both
  `trickplay-jf-osc.lua` and `trickplay-osc.lua`
- `gh release create vX.Y.Z --repo MrGameVlogger/jf-mpv-osc --title "vX.Y.Z" --notes-file <notes> trickplay-jf-osc.lua trickplay-osc.lua`
  The release mints the tag; a bare `git tag` fails in non-interactive
  terminals (use `git tag -a vX.Y.Z -m "vX.Y.Z"` to tag first)
- Semantic versioning: patch for fixes, minor for features, major for
  breaking contract changes (the README contract is the compatibility surface)

## Contract changes
- The README's adoption tiers and message tables ARE the API. Changing a
  `shim-*` message name, the state-blob shape, or the no-host fallbacks is
  a breaking change — update the README and bump the major version
