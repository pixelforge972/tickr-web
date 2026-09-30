# Changelog

All notable changes to this project are documented here. Versioning follows [Semantic Versioning](https://semver.org/) (`MAJOR.MINOR.PATCH`), kept in sync with the service worker's cache name in `sw.js`, the version shown in the app's Settings sheet, and an annotated git tag `vMAJOR.MINOR.PATCH` — pushing that tag is what triggers the GitHub Pages deploy (see the README's [Deploy](README.md#deploy-github-pages) section).

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
