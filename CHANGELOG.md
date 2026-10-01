# Changelog

All notable changes to this project are documented here. Versioning follows [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`), kept in sync with the service worker's cache name in `sw.js`, the version shown in the app's Settings sheet, and an annotated git tag `vMAJOR.MINOR.PATCH` — pushing that tag is what triggers the GitHub Pages deploy (see the README's [Deploy](README.md#deploy-github-pages) section).

## 0.2.1

### Changed

- **Renamed the app to Tickr** (was "Metronome") and replaced the icon with a new monogram mark, to reflect that the app is now a metronome *and* tuner rather than metronome-only. Updated everywhere the old name/icon appeared: `<title>`, `manifest.webmanifest` (`name`/`short_name`), the version line in the Settings sheet, and `README.md`. The "Metronome" and "Tuner" labels on the mode switcher itself are unchanged — those name the two tools inside the app, not the app.
- Bumped this version a second time within the same unreleased cycle specifically to force the service worker's cache to invalidate: `icon.svg` is a precached asset (see `sw.js` `PRECACHE_URLS`), and its content changed without the cache name changing, so any client that had already installed the service worker under the `0.2.0` cache name — including a dev session that had loaded the app earlier the same day — kept serving the old icon from cache indefinitely under the cache-first strategy, regardless of reloading or opening a fresh private window on an already-open private session. Bumping `VERSION` is what actually busts it (see `README.md`'s "Releasing a change" steps).

## 0.2.0

### Added

- **Tuner mode**: a second mode next to the metronome, switched via a segmented control in the header, designed in the same Claude Design canvas as the metronome itself and implemented 1:1 from that design (see `DECISIONS.md` in the project's Obsidian docs). Two independent tools:
  - **Reference tone generator**: a note grid (four quick-access notes — C4, E4, G4, A4 — expandable to two full chromatic octaves, C3–B4) that plays a sustained sine tone through the existing Web Audio engine — no microphone or permission needed, works even where microphone access is unavailable or denied.
  - **Pitch detection**: requests microphone access (`echoCancellation`/`noiseSuppression`/`autoGainControl` disabled for signal fidelity), analyzes the live input with `AnalyserNode` + an autocorrelation algorithm (silence-trimmed window, parabolic interpolation, cents-only exponential smoothing while a note is held) in a dedicated `requestAnimationFrame` loop. Shows the nearest note name + accidental + octave, a cents-deviation meter (rail, in-tune zone, tick marks, moving marker), the live frequency in Hz, and a five-state status line (idle/pending/listening/detecting/denied).
  - Tuner and metronome are mutually exclusive (playing a tone, listening, or starting the metronome stops whichever of the others was running) to avoid the microphone picking up the metronome's own clicks or two audio sources overlapping.
  - Microphone access is feature-detected; on browsers without `getUserMedia`, the "Listen" control doesn't render at all (only the tone generator, which needs nothing) — consistent with how the vibration toggle already hides on iOS instead of showing disabled.
  - Status announcements are deliberately throttled: the note display only pushes into the shared screen-reader announcer when the note identity changes, not on every small cents correction while a note is held — otherwise a screen reader would get re-announcements up to ~16 times per second.

### Fixed

- The HTML `hidden` attribute doesn't hide inline `<svg>` elements in Chromium (confirmed by testing — an `<svg hidden>` stays rendered; a `<p hidden>` doesn't). This silently broke the Start/Stop button's icon swap since the very first release: the play triangle and the stop square rendered side by side at all times ("▶ ■ Start"/"▶ ■ Stop") instead of swapping one for the other. Found while building the Tuner's status icon, which had the same problem. Fixed with one explicit `svg[hidden]{display:none}` rule covering every icon toggle in the app.

## 0.1.0

Initial release.

### Added

- Metronome core: 30–240 BPM, meters 2/4, 3/4, 4/4, 5/4, 6/8, 7/8 with visual LED grouping for odd meters, subdivisions (none/eighths/triplets/sixteenths), three sounds synthesized with the Web Audio API (click/wood/hi-hat).
- Tap tempo, keyboard shortcuts (`Space`, `↑`/`↓`, `←`/`→`, `T`), ±1/±5 step buttons.
- Settings: volume, accent volume, sound, vibration on beat (Android/Chrome only), screen wake lock, appearance (system/light/dark).
- PWA support: offline via service worker (cache-first, same-origin GET), installable manifest, icon.
- PL/EN localization based on system language.
- Accessibility: ARIA roles/labels on all controls, `aria-live` start/stop announcements, full keyboard navigation, WCAG AA contrast in both themes.
- GitHub Pages deploy workflow via GitHub Actions, no build step.
- MIT license.

### Fixed

- Background-tab audio scheduling: a scheduler tick that fired late (browser timer throttling) could schedule notes at a time already in the past — Web Audio doesn't drop those, it clamps them to fire immediately, bursting several missed notes into one clashing instant. The scheduler now silently fast-forwards through anything already behind `audioCtx.currentTime` before scheduling new notes, instead of letting stale notes collide.
- Metronome going completely silent when the browser window loses OS-level visibility (e.g., another app covering it fullscreen) for long enough that the browser throttles the tab's timers to a near-stop. Added a steady, effectively inaudible keep-alive tone plus a Media Session `playbackState` declaration so the tab is recognized as actively playing audio and throttled less aggressively.
- The beat LED and vibration were firing on every subdivision tick instead of only on the main beat, and the LED was being cleared before the beat visually finished.
