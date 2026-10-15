# Jellyfin-styled mpv OSC — integration contract

A drop-in mpv Lua script that replaces mpv's built-in OSC with one modeled on
the jellyfin-web video OSD: Material vector icons, action-sheet popups for
subtitle / audio / settings selection, a floating Skip Intro / Skip Credits
button, and thumbnail seek previews.

It was written for [jellyfin-mpv-shim](https://github.com/jellyfin/jellyfin-mpv-shim)
and retired there on 2026-07-19 (commit `0e438101`) once the in-window mpvtk HUD
reached parity. Nothing about it is shim-specific: it talks to its host over seven
`script-message` names and degrades to a working generic OSC if nobody answers.

**It is a good fit for a host that launches mpv as a separate process.** All of
the platform-specific hard parts stay inside mpv — it owns its window, swapchain,
and `gpu-next` pipeline, so HDR and high-quality scaling are unaffected. The
script adds zero platform surface: it is the same Lua on every OS.

## Contents

```
trickplay-jf-osc.lua        the OSC (4771 lines)
gen_osc_icons.py            regenerates the embedded icon table from Material SVGs
tools/osc-test/             headless screenshot + click-routing harness (no host needed)
```

## Loading it

Pass it as a script and make sure mpv's own OSC is off:

```
mpv --script=/path/to/trickplay-jf-osc.lua --osc=no --script-opts=osc-visibility=auto ...
```

`--osc=no` matters even if the user's `mpv.conf` says otherwise, or both OSCs
draw at once.

If you also ship [thumbfast](https://github.com/po5/thumbfast), load it too and
seek previews light up automatically — see *Optional integrations* below.

With no host talking to it at all, the script still works: the sheets fall back
to mpv's own `track-list`, and the Jellyfin-specific buttons (favorite,
next/prev item, skip segment, SyncPlay) hide themselves. The back arrow and
window close quit mpv directly, the gear sheet offers a local Screenshot and a
"Quit Without Saving Position" (suppresses mpv's watch-later write and drops the
saved one), and the subtitle sheet's Size/Position/Color rows edit mpv's own
`sub-scale`/`sub-pos`/`sub-color`.

---

## Adoption tiers

Each tier is independently useful, and nothing above a tier is required to use
it. There is no point on this curve where partial adoption looks broken — the
script hides what it has no data for.

### Tier 0 — load it, push nothing

An argv change. You get the whole visual restyle: the jellyfin-web-shaped bar,
Material icons, working seek/volume/chapter controls, and action sheets
populated from mpv's own `track-list`. Every Jellyfin-specific button hides
itself. **Zero host code.**

For most projects this is the bulk of the visible win. Everything below is
optional refinement.

### Tier 1 — name the tracks

Push `shim-jf-osc-state` with four keys: `strings: {}`, `has_media`,
`subtitles`, `audio`. Wire `shim-close` if you want a close-time hook — the
OSC quits mpv itself either way, so the buttons work at Tier 0.

**This is the highest value-per-line item in the contract**, because track
naming is the one thing the Tier 0 fallback genuinely cannot do. Jellyfin serves
external subtitles as separate URLs, and the display titles — language, codec,
Forced, SDH, "External", "Transcode" — live in `MediaStreams`. mpv only sees
`sub 3, srt, und` where Jellyfin knows `English (Full) (ASS)`. The fallback path
gives you a track list nobody can choose from.

Two arrays of `{id, label, selected}` fixes that, and the `id` values are
whatever your host already uses — the OSC never interprets them.

### Tier 2 — skip segments

`shim-jf-osc-skip` in, `skip-segment` out.

If your client already detects intro/outro segments (most Jellyfin clients do,
via the media segments API), the detection half is done — this only moves the
prompt into the video window, where the user is already looking. Small delta,
disproportionately visible.

### Tier 3 — session state

`queue` + `next-item` / `prev-item`, `favorite` + `toggle-favorite`, `quality` +
`set-quality`, `allow_screenshot` + `screenshot`, `unwatched-quit`, and
`set-fullscreen`.

Each of these is a piece of live Jellyfin session state the host must own and
keep fresh across item changes. This is the tier where the OSC stops being a
skin and starts being a client surface — which is worth it only if you want that.

### Tier 4 — deep integration

`profiles` with scopes, `syncplay`, `sub_style`, `secondary_subtitles`.

These assume the host owns transcode negotiation, shader-profile management, and
SyncPlay group membership. Without a host, `sub_style` still appears and edits
mpv's own `sub-scale`/`sub-pos`/`sub-color`. Sensible for a client built around mpv; probably not
for a client that wants to stay thin.

### The one architectural question

The state blob is a **push** model: it assumes the host knows when something
changed and has somewhere to call `send_state()` from.

If your client already runs a playback event loop — reporting progress, pause
state, and resume position back to Jellyfin — that loop is the hook, and Tier 1
is an afternoon's work. If mpv is launched fire-and-forget and only polled
occasionally over IPC, then Tier 1 means building the push path first. That, not
the JSON, is the real cost. Answer this before estimating anything above Tier 0.

---

## Host → OSC

### `shim-jf-osc-state` — the state blob

One argument: a JSON object, as a string. Push it whenever anything in it
changes; it is idempotent and cheap. An open action sheet rebuilds in place, and
the OSC re-runs layout (track counts, `favorite`, and SyncPlay availability
decide which buttons exist at all).

```jsonc
{
  "strings": { /* see below */ },
  "has_media": true,             // false => everything below may be absent
  "allow_screenshot": true,      // false hides the screenshot row (it exists with a local action when no state is pushed)
  "favorite": false,             // heart button fill state
  "queue": { "has_prev": true, "has_next": true },

  // Option lists. `aside` is optional right-aligned secondary text.
  "subtitles": [
    { "id": -1, "label": "None", "selected": false },
    { "id": 2,  "label": "English (SRT)", "selected": true },
    { "id": 5,  "label": "Signs PGS", "aside": "Transcode", "selected": false }
  ],
  "secondary_subtitles": [ /* same shape */ ],
  "audio": [ /* same shape */ ],

  // Grouped options: `current` is the label shown on the collapsed row.
  "quality":  { "current": "No Transcode", "options": [ /* {id,label,selected} */ ] },
  "profiles": { "current": "None", "options": [ … ], "scopes": [ … ] },
  "sub_style": {
    "size":     { "current": "Normal", "options": [ … ] },
    "position": { "current": "Bottom", "options": [ … ] },
    "color":    { "current": "White",  "options": [ … ] }
  },

  "syncplay": { "enabled": false, "current": "None (Disabled)", "groups": [ … ] }
}
```

Notes that will save you time:

- **Every option object is `{id, label, selected}`**, plus optional `aside`.
  `id` is echoed back verbatim in the matching action verb, so it can be
  whatever your host wants — the OSC never interprets it.
- **Omit a key to remove its UI.** Leaving out `profiles` drops the profile row;
  leaving out `syncplay` (rather than sending an empty one) removes the SyncPlay
  button entirely. That is the right shape when the feature is unavailable to
  this user, because a visible button that can only fail is worse than no button.
- **`has_media: false` is the only guarantee** when nothing is playing. Send just
  `{"strings": {...}, "has_media": false}`.
- **`strings` carries every localized fixed string**, so the Lua ships no
  translations. Keys: `off`, `none`, `unknown`, `ends_at` (a format string whose
  `{0}` has already been replaced by `%s`), `quality`, `speed`, `aspect`,
  `aspect_auto`, `profile`, `stats`, `screenshot`, `unwatched`, `sub_size`,
  `sub_position`, `sub_color`, `sp_disabled`, `sp_new`, and the no-host
  fallback labels `quit_nosave`, `tiny`, `small`, `normal`, `large`, `huge`,
  `top`, `middle`, `bottom`, `white`, `yellow`, `green`, `cyan`, `red`. Send
  `{}` and it falls back to English.
- Option **labels are pre-formatted by the host** — the OSC does no string
  building beyond `strings` lookups.

### `shim-jf-osc-skip` — floating skip button

One argument: the button label, or `""` to hide it.

```
script-message shim-jf-osc-skip "Skip Intro"
script-message shim-jf-osc-skip ""
```

Send only on transitions — the shim diffed this on every pump and pushed nothing
when the label was unchanged. The OSC clears it automatically on mpv's
`start-file`, so a stale button cannot survive into the next item.

Pressing it emits the `skip-segment` action; the host performs the actual seek.

### `shim-jf-osc-menu` — open a sheet programmatically

One argument: `sub`, `audio`, or `settings`. Shows the OSC and toggles that
action sheet. Intended for host key bindings and remote-control commands.

### `shim-jf-osc-close` — close from outside the OSC

No arguments. Runs the same path as the close button and back arrow: quits mpv
and emits `shim-close`. Intended for host key bindings and remote-control
commands.

---

## OSC → Host

### `shim-jf-osc-action` — the verb channel

`script-message shim-jf-osc-action <verb> [arg]`. Every interactive control
routes through here. The OSC changes no playback state itself (two exceptions,
noted below) — it reports intent and waits for the next state push.

| Verb | Arg | Meaning |
|---|---|---|
| `set-sub` | stream id | select subtitle track (`-1` = off) |
| `set-secondary-sub` | stream id | select secondary subtitle track |
| `set-audio` | stream id | select audio track |
| `set-quality` | option id | `none`, `max`, or a bitrate |
| `set-sub-size` | option id | subtitle scale |
| `set-sub-position` | option id | `bottom` / `top` / `middle` |
| `set-sub-color` | option id | subtitle color |
| `set-profile` | `"<scope>\|<profile>"` | shader/profile select; bare value = default scope; `unset` clears |
| `profiles-scopes` | — | warm hint: profile sheet opened, resolve anything expensive now |
| `syncplay-refresh` | — | warm hint: SyncPlay sheet opened, go discover groups |
| `syncplay-join` | group id | join a group |
| `syncplay-new` | — | create a group |
| `syncplay-disable` | — | leave |
| `toggle-favorite` | — | flip the heart |
| `next-item` / `prev-item` | — | queue navigation |
| `skip-segment` | — | the skip button was pressed |
| `set-fullscreen` | `yes` / `no` | **notification, not a request** (see below) |
| `screenshot` | — | take a screenshot |
| `unwatched-quit` | — | quit and mark unwatched (leave no resume position) |

Unknown verbs should be logged and ignored; the OSC never waits for a reply.

The two `-refresh` / `-scopes` verbs exist because those two menus need data that
costs a network request. They fire when the sheet *opens*, so the host can fetch
off the render path and let the result arrive in a later state push.

**`set-fullscreen` is one of two exceptions to "the OSC changes nothing"** (the
other is `shim-close`, below). It
toggles mpv's `fullscreen` property locally first — so the button still works
with no host attached — and then reports the new state. Treat it as recording
user intent: without it, a host with auto-fullscreen keeps re-fullscreening every
episode after the user has deliberately left it.

### `shim-jf-osc-ui-seek` — "that seek was mine"

No arguments, throttled to at most once per second. Emitted before seeks that
originate from the OSC's own controls (seekbar scrub, rewind/forward, chapter
buttons).

The shim used it to exempt those seeks from seek-to-skip-intro: a user dragging
the seekbar through an intro should not trip the skip handler. Ignore it if you
have no such logic; keep the throttle window open a couple of seconds per notice
if you do.

### `shim-close` — user asked to close

No arguments. Fired by the window-controls close button and the back arrow. The
OSC quits mpv itself — the buttons must work with no host attached — and emits
this first, so the host can do close-time bookkeeping (stop playback, report
watched state, release device sessions). Treat it as a notification like
`set-fullscreen`: the exit is already underway when it arrives.

---

## Optional integrations

**thumbfast (seek previews).** If `thumbfast.lua` is loaded, the OSC listens for
`thumbfast-info` and drives it with `script-message-to thumbfast thumb <offset>
<x> <y>` / `clear`. Requests are deduplicated by `offset:x:y` — the render loop
runs on every mouse move and at 10 Hz while visible, so undeduplicated requests
flood the IPC channel whenever the cursor rests on the seekbar while paused.

Nothing is needed from the host **for local files**. For Jellyfin, read the next
section before promising this feature to anyone.

### Trickplay is not free

Stock thumbfast works by running a hidden second mpv that decodes the media
itself. Jellyfin media is remote, authenticated, and may be transcoded, so that
path is unavailable — the host has to become a thumbfast-compatible *provider*
backed by the server's trickplay tiles (or a BIF file). This is an orthogonal
subsystem, not a tier of the contract above: the OSC only ever speaks
thumbfast's protocol and does not care where the pixels came from.

It is very doable, and jellyfin-mpv-shim's `trickplay.py` / `bifdecode.py` are a
working reference. Budget for it honestly:

- **jellyfin-web's trickplay implementation is of no use here.** It renders into
  the DOM for the web player. With playback in a separate mpv process there is
  nothing to reuse, so this is new code regardless of how thin the client is.
- **You need image decoding in the host process** to crop the sprite grid —
  Pillow, `sharp`, `canvas`, something. For a project avoiding native
  dependencies, this is the one place that gets expensive.
- **mpv's JSON IPC is text.** Raw BGRA cannot travel through it. `overlay-add`
  takes a *file path* or a memory address, so a separate-process host gets the
  file path and inherits everything below.

The shape that performs well is one flat file per video holding every frame as
concatenated raw BGRA; mpv mmaps it and seeks to `frame * width * height * 4`,
so hovering the seekbar becomes offset arithmetic with no per-hover IPC and no
per-hover file write. Writing a temp file per hover instead is the obvious first
implementation and it feels bad enough that you will end up here anyway.

**The mmap is a loaded gun.** Because mpv maps the file rather than reading it,
three ordinary-looking mistakes are a SIGBUS *in the mpv process*, with a stack
trace that says nothing about your code:

1. **Rewriting a frame file in place.** `open(path, "wb")` truncates the inode
   while mpv still holds the mapping, which then extends past EOF. Use a fresh
   path per generation and never reuse one.
2. **A per-instance generation counter.** A new worker is typically built each
   time mpv is re-created, so a per-instance counter restarts at 0 and the second
   generation of the second worker reuses the first worker's path — the exact
   case (1) forbids. Keep the counter module-level / process-global.
3. **Reporting the manifest's frame count instead of frames actually written.**
   A 404 on a late tile, a truncated body, or a server still generating trickplay
   leaves you short, and mpv then seeks past EOF. Report what you wrote.

Also order teardown as *renderer first, file second*: `overlay-remove` has to
land before the file is unlinked.

**Inherited mpv OSC messages.** The upstream contract still works, since this is
a fork of mpv's `osc.lua`: `osc-message`, `osc-visibility`, `osc-idlescreen`,
`osc-chapterlist`, `osc-playlist`, `osc-tracklist`.

**`user-data/osc-margins`.** Set to `"l,r,t,b"` while the bar is visible, cleared
otherwise — the standard property other scripts read to avoid drawing underneath
the OSC.

---

## Trying it without a host

`tools/osc-test/` runs the real mpv binary under Xvfb, injects **real X11 input**
with xdotool, and asserts over the JSON IPC socket. Needs `mpv`, `Xvfb`,
`xdotool`, `socat`, `python3`.

```bash
tools/osc-test/osc-test.sh shot out.png            # screenshot the bar
tools/osc-test/osc-test.sh shot out.png settings   # with the gear sheet open
tools/osc-test/osc-test.sh clicks                  # click routing
tools/osc-test/osc-test.sh restart                 # idle -> new file -> clicks
tools/osc-test/osc-test.sh flicker                 # trickplay overlay churn
```

`KEEP_LOG=1` keeps the mpv log; the OSC logs every input event it processes when
`JMS_JF_OSC_DEBUG=1` (the harness sets it).

`drivers/shot-driver.lua` is also the **shortest worked example of the contract** —
a complete fake state blob with every field populated, pushed from Lua. If you
implement against one thing, implement against that file.

Two things the harness learned the hard way, preserved here so you don't rediscover
them:

- Synthetic `mouse` / `keypress` mpv commands do **not** traverse the same input
  path as real X11 events — they miss section and mouse-area routing entirely.
  Test with real input or you are testing nothing.
- Under Xvfb/XTEST, mpv can deliver a quick click's *release* to Lua noticeably
  late. The OSC judges releases by the press position so actions still land; if a
  click test behaves oddly, read the event timeline before blaming the script.

---

## Provenance and licensing

Read this before vendoring — the chain matters and it is not all one license.

- The base is mpv's `player/lua/osc.lua` at
  `bca516bd2c282670aa2c92663329e7d5ddf978e0`. Check that file's header upstream
  for its terms.
- Thumbnail support came via `trickplay-osc.lua`; those hunks are still marked
  `BEGIN patch` / `END patch` in the source.
- The Jellyfin layout, icons, action sheets, and the entire contract above are
  from jellyfin-mpv-shim, which is **GPL-3.0-or-later**.
- The icon path data is derived from Google's Material Design icons,
  **Apache-2.0**, compiled to ASS drawing commands on a 24×24 canvas by
  `gen_osc_icons.py`. The generated table in the Lua file is marked
  `BEGIN generated icons` — regenerate rather than hand-editing.

Confirm compatibility with your own project's license before adopting.
