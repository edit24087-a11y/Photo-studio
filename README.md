# Photo Studio Pro

A single-file, browser-based photo editor — layers, masks, adjustment layers, paths, text and shape
layers, 20 filters, 14 non-destructive adjustments, history, autosave and `.pspro` projects.
Open `index.html` in a modern browser: no build step, no install, no server.
(It loads the Tailwind CSS runtime from a CDN, so the first paint needs internet access.)

## What's new in 2.0

**Background pixel engine (`Pix`).** Filters used to run on the main thread and freeze the UI on large
images. They now run in a pool of Web Workers sized to the machine. The worker source is generated at
runtime from the very same kernel functions the main thread uses (`Function.prototype.toString`), so the
two paths cannot drift apart. Wide images are split into horizontal slabs — with a halo margin for
shift-invariant kernels such as blur, so seams are impossible — and processed in parallel. If workers are
unavailable, switched off, or a kernel fails or times out, the job re-runs on the main thread
automatically: identical output, only slower. The filter preview, the Apply action and the Filter Gallery
thumbnails all go through it (the gallery now renders on several lanes at once).

**Histogram panel.** RGB / luminosity / single channel, log or linear scale, measured on the composite or
the active layer, with mean, σ, median, level range and clipped-pixel percentages. Cached by layer
revision so it costs nothing when the image has not changed.

**Performance HUD.** The status bar shows live render cost (`ms/frame · fps · ⚙ workers`); click it for
Help ▸ **Performance…** — engine state, jobs offloaded vs. run on the main thread, average filter time,
document size, DPR, hardware concurrency and heap. Workers can be switched in Preferences and restarted
from that dialog.

**Automation surface.** `window.PhotoStudio` exposes `App`, `Docs`, `Tools`, `Pix`, `FX`, `View`,
`Panels`, `Dialog`, `Hist` and friends, so the editor can be scripted from the console:

```js
PhotoStudio.Docs.create({ w: 800, h: 600, bg: 'white' });
const out = await PhotoStudio.Pix.render(PhotoStudio.FX.byId.gaussian, imageData, 1);
```

## Verification

Every change is checked by running the real page in jsdom (`smoke`, `deep`, `worker` suites): boot and
toolbar, all 25 tools and their shortcuts, 168 menu items including submenus, 11 dock panels, the main
dialogs, plus the pixel engine — where the generated worker source is parsed and really executed, and
each kernel's off-thread output is compared byte-for-byte with the main-thread implementation, including
tiled slabs. See the fix log below for the startup crash this branch started from.

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
