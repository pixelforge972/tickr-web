# Metronome

A minimal metronome PWA. No install required, no tracking, works offline after the first load.

## Features

- Tempo 30–240 BPM: slider, ±1/±5 step buttons, keyboard (`↑`/`↓` ±1, `←`/`→` ±10, `Space` start/stop, `T` tap tempo), tap tempo.
- Meter: 2/4, 3/4, 4/4, 5/4, 6/8, 7/8 (with visual LED grouping for odd meters).
- Subdivisions: none / eighths / triplets / sixteenths.
- Three sounds synthesized with the Web Audio API (click, wood, hi-hat) — no audio files.
- Vibration on beat (Android/Chrome; unavailable on iOS Safari) and screen wake lock while running.
- Dark/light theme following `prefers-color-scheme`, with `prefers-reduced-motion` support.
- PL/EN based on system language.
- Works as an offline PWA (service worker, manifest, icon).

## Stack and architecture decisions

Plain HTML/CSS/JS — a single `index.html` file (inline CSS and JS), no bundler, no `package.json`, no `node_modules`. At this page size, gzip/brotli served by GitHub Pages compresses about as well as manual minification would, so a build step wouldn't add value — only extra complexity and harder source debugging.

- **Audio timing**: a lookahead scheduler on the `AudioContext.currentTime` clock (the "A Tale of Two Clocks" pattern), not `setInterval` — a driving timer (~25 ms) schedules notes about 100 ms ahead; when the tab is hidden or occluded by another window, the lookahead grows to ~1 s to absorb browser timer throttling. A metronome's clicks are brief and mostly silence, so browsers don't reliably flag the tab as "playing audio" the way continuous media does, and an unflagged tab gets its timers throttled hard once backgrounded. A steady, effectively inaudible 20 Hz tone plays alongside the clicks (plus a Media Session `playbackState` declaration) purely to keep the tab flagged as active audio, so the scheduler keeps running on time.
- **Changing BPM while playing** doesn't reset phase — notes already scheduled finish at the old tempo, the new tempo applies from the next step onward. Changing the meter or subdivision restarts the beat counter at the nearest bar boundary, with no audible jump.
- **Sound**: three sounds synthesized on the fly (Web Audio API: oscillators + filtered noise), no audio files.
- **iOS Safari**: `AudioContext` can only resume inside a user gesture; the physical mute switch blocks Web Audio (there's no API to detect that state) — hence the in-UI hint. `navigator.vibrate` isn't supported by WebKit (Apple removed it in 2017), so the vibration toggle is disabled on iOS.
- **Wake Lock**: feature-detected, the request is wrapped in `try/catch` and re-acquired when the tab becomes visible again (the browser releases the lock automatically when a tab loses visibility).
- **Layout**: a single grid, with portrait/landscape/desktop breakpoints handled entirely by CSS container queries (not media queries) — the same file behaves identically in a desktop window and on a phone.

## Running locally

```bash
python3 -m http.server
```

Open `http://localhost:8000`. The Service Worker and Wake Lock APIs require a secure context — `localhost` qualifies.

While iterating locally, check **"Update on reload"** in devtools → Application → Service Workers. Without it, the service worker's cache-first strategy means edits to `index.html` can silently keep showing the old cached version until the cache name in `sw.js` changes and the new worker takes over — which normally takes an extra reload or two to notice.

## Structure

```
index.html                  # the whole app: inline CSS + inline JS
sw.js                        # service worker (cache-first, same-origin GET)
manifest.webmanifest
icon.svg
LICENSE
CHANGELOG.md
.github/workflows/deploy-pages.yml
.github/dependabot.yml
```

## Deploy (GitHub Pages)

The `.github/workflows/deploy-pages.yml` workflow publishes the repo with no build step, via `actions/upload-pages-artifact` + `actions/deploy-pages`. In the repo settings: **Settings → Pages → Source: GitHub Actions**.

**The workflow runs on version tag pushes (`v*`), not on every push to `main`.** This keeps deploys deliberate: you can push work-in-progress commits to `main` without shipping them, and a deploy always corresponds to a specific, named, reproducible version. It can also be run manually from the Actions tab (`workflow_dispatch`) if a tag push is ever missed.

### Version tags

Every released version is marked with an annotated git tag named `vMAJOR.MINOR.PATCH` (e.g. `v0.1.0`), following [Semantic Versioning](https://semver.org/) — the same number shown in `CHANGELOG.md` and in the app's Settings sheet, just with the conventional `v` prefix that distinguishes a version tag from a branch name at a glance. Pushing a tag in this format is what triggers the deploy:

```bash
git tag -a v0.1.0 -m "v0.1.0"
git push origin v0.1.0
```

Bump the cache version in `sw.js` (`VERSION`) on every meaningful static-asset release, otherwise users stay stuck on the old cached version.

### Action pinning

The four GitHub Actions the workflow uses are pinned to a full commit SHA, not a version tag (`actions/checkout@11d5960a... # v4.4.0`, etc.) — a tag can be moved to point at different code if an action's maintainer account is ever compromised, a SHA can't. The repo setting **Settings → Actions → General → Require actions to be pinned to a full-length commit SHA** is enabled, so an unpinned action reference in this or any future workflow fails CI outright rather than silently running.

## Maintenance

There are no dependencies to update — the project has no `package.json`/`node_modules` by design, so there's nothing for `npm audit` or Dependabot to scan beyond the four pinned GitHub Actions below. Dependabot security updates are enabled for those, so an advisory against one of the pinned SHAs shows up as an automatic PR bumping it.

**Releasing a change:**

1. Edit `index.html` (or `sw.js`/`manifest.webmanifest`/`icon.svg`) locally.
2. Test locally with `python3 -m http.server`, open `http://localhost:8000`, and check: no console errors, both themes (toggle via the in-app selector or your OS setting), portrait/landscape/desktop layouts (resize the devtools viewport — the breakpoints are container queries, so resizing the browser window is enough, no device emulation needed), and `prefers-reduced-motion` (devtools → Rendering → Emulate CSS media feature).
3. If you changed `index.html`, `manifest.webmanifest`, or `icon.svg` (anything the service worker precaches), bump the version in three places, kept identical: `VERSION` in `sw.js`, the `.sheet-version` text in `index.html` (shown at the bottom of the in-app Settings sheet), and a new entry at the top of `CHANGELOG.md`. Skipping the `sw.js` bump leaves returning users on the stale cached version indefinitely, since the old service worker keeps serving its own cache until a version bump triggers `activate` to clear it; the other two are just so anyone (including future you) can tell which version is actually running or was shipped when.
4. Commit and push to `main`. This alone does **not** deploy — it just gets the change onto the branch.
5. Tag the release and push the tag — this is what actually triggers the deploy (see [Version tags](#version-tags)): `git tag -a v0.1.0 -m "v0.1.0" && git push origin v0.1.0`.
6. Check the Actions tab for a green run, then open the live URL and confirm the update actually landed (hard-refresh, or check the version text in Settings / `navigator.serviceWorker.getRegistration()` in devtools) — a stale `VERSION` is the most common reason a deployed change doesn't show up.

**Periodic checks** (browser support for these APIs has shifted before and can shift again):

- Vibration API on Android browsers — re-check current support on [caniuse](https://caniuse.com/vibration-api) before assuming the feature-detect branch (`"vibrate" in navigator`) still degrades correctly; iOS Safari support is not expected to change (WebKit removed it in 2017).
- Screen Wake Lock API in installed/standalone PWAs, especially on iOS — support was broken by a WebKit bug until iOS 18.4; re-verify after major iOS releases via [caniuse](https://caniuse.com/wake-lock).
- The pinned GitHub Actions SHAs (`actions/checkout`, `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages`) — Dependabot opens a PR automatically when one needs bumping; merging it is enough, since Dependabot resolves the new SHA itself.

## Known limitations

- Vibration is unavailable on iOS (Safari/WebKit doesn't support `navigator.vibrate`).
- Wake Lock in an installed (`standalone`) PWA on older iOS may fail on WebKit's side — handled gracefully (`try/catch`, the screen may simply turn off), it doesn't block the metronome from running.
- There's no programmatic way to detect the iPhone's physical mute switch — hence the in-UI hint while playing.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).
