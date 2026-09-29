# MMM-LorenzAttractor — context for Claude sessions

Charles's MagicMirror² module: the Lorenz attractor, three trajectories released 10⁻⁵ apart,
for his hallway mirror, as one page in a rotation of pages. Split out of MMM-ChaosTheory on
2026-09-28 with its history (it was the `lorenz` simulation of that module's chaos page).

## Files

- `MMM-LorenzAttractor.js` — module shell: one canvas plus an HTML caption (equations + live
  readout, updated 2×/s). New sim every `cycleSeconds` (60) and on each `resume()`. Loop:
  `setTimeout` until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a
  sim with `resting = true` is polled only every 500 ms (lorenz never rests); while MagicMirror
  fades the module out (`hidden` is set at the start, `suspend()` comes after), frames draw
  nothing. `turns: { of, at }`: only every nth showing; otherwise the wrapper gets
  `display: none` and nothing starts.
- `simulations/lorenz.js` — the `lorenz` sim on `window.LorenzSimulations`. RK4 at h = 0.004,
  0.55 Lorenz time units per real second (`FixedClock`), a random 20–40 time-unit warm-up from
  (1, 1, 20), three states 10⁻⁵ apart in x, 1400-point trails (every 3rd step), additive
  red/green/blue. `lorenzStyle: "rotate"` (default): the view turns (yaw 0.12 rad/s, tilt 0.35);
  each frame clears only the bounding box of this and the last frame's curves, 1 px lines.
  `"exposure"`: fixed view, only new segments each frame.
- `simulations/common.js` (`window.LorenzCommon`) — `FixedClock`, `Trail`, `palette`, `sci`.
- UMD-style, so the physics runs in Node: `tests/lorenz.test.js` (`node --test`, no
  dependencies; fixed points, bounded attractor, Lyapunov exponent ≈ 0.906, divergence).
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the repo,
  then `/dev/preview.html?lorenzStyle=exposure`).

MMM-ChaosTheory has the same simulation among its eight (`simulations/lorenz.js` there, on
`window.ChaosSimulations`): a fix to the simulation belongs in both.

The shell (`MMM-LorenzAttractor.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is
shared in spirit with the sibling modules (MMM-ChaosTheory, MMM-DoublePendulum,
MMM-FractalBasins, MMM-LogisticMap, MMM-SymmetricIcons, MMM-ThreeBody, MMM-ChaoticBilliards,
MMM-Rule30, and the non-chaos pages: MMM-Atom, MMM-FractalZoom, MMM-Chladni, MMM-SacredGeometry,
MMM-Tilings, MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck), all checked out
side by side in `~/dev/mirror-modules/`: a fix there probably belongs in the siblings too.

As of the split, the mirror's config.js (in the setup repo, being reworked) gives it a page of
its own, `classes: "page-lorenz"`, 900×900 at 20 fps; check there for the current rotation.

## Measured cost on the Pi

900², 20 fps, Electron + cage over 60 s (measured as part of MMM-ChaosTheory): rotating 165% of
a core at 16 fps (the renderer is saturated, so ~150% whatever `fps` is), `lorenzStyle:
"exposure"` 55% at 20+ fps. Hidden: 0.3% (baseline 0.2%). Not yet measured as a module on its
own, nor at 700² / 12 fps.

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame).
- So: draw incrementally (long-exposure trails), keep each frame's changes spatially compact,
  and rest when the picture is static. Opacity and `rAF` vs timer made no difference; for the
  turning view, 1 px lines gave 16 fps against 8 for 1.6 px.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-LorenzAttractor`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
