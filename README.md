# Photo-studio

A single-file, browser-based photo editor. Open `index.html` in a modern browser — no build step, no
install. (It pulls the Tailwind CSS runtime from a CDN, so the first load needs internet access.)

## Fix log

**2026-11-08 — `TypeError: t.key.toUpperCase is not a function` (app dead on startup)**

Four tool definitions — `polylasso`, `crop`, `pcrop` and `pen` — declared the `key` property **twice in
the same object literal**: once as the keyboard-shortcut string (`key: 'shift+l'`) and once as the
`key(e)` key-event handler method. In JavaScript the later member wins, so `tool.key` was a function, and
`Tools.buildToolbar()` calling `t.key.toUpperCase()` threw. Because that happens inside `boot()`, the
whole startup aborted: the toolbar stopped after three tools and the canvas, panels, keyboard shortcuts
and welcome screen were never initialised.

Fix: the shortcut is now stored as `shortcut` (on all 25 tools, including the `paintTool`, `retouchTool`
and `shapeTool` factories), `key` is reserved for the optional key-event handler, `buildToolbar()` and
`initKeys()` read `shortcut`, and `initKeys()` only calls `Tools.cur.key` when it is really a function.
21 lines changed, no behaviour change other than the app starting.
