# Pulse

Interval training timer built as an installable iOS web app. Robinhood-style dark
UI, a portfolio of 100 equipment-paired workouts, and a session view built around
three things that have to work in a gym:

1. **Cues are audible** — even with the iPhone's physical silent switch on.
2. **Spotify keeps playing** — the music ducks for each cue, then comes back.
3. **The screen stays awake** — for the whole session.

## How those three actually work

### Sound + Spotify

On iOS, raw Web Audio output is treated as an *ambient* audio session by default.
Two consequences bite a gym timer:

- The **hardware ring/silent switch mutes it**, silently and with no error. This
  is the single most common reason a web-based timer "has no sound".
- Safari can escalate the page to an exclusive `playback` session once it starts
  producing audio, which **interrupts other apps** — Spotify stops.

Pulse sets [`navigator.audioSession.type`](https://developer.mozilla.org/docs/Web/API/AudioSession/type)
explicitly:

| Mode | `audioSession.type` | Behaviour |
|---|---|---|
| **Duck** (default) | `transient` | Playback-class session that mixes and ducks. Cues play **over** the silent switch, Spotify dips briefly then recovers. |
| **Mix** | `ambient` | Cues layer on top at music volume, nothing dips — but iOS mutes them when the silent switch is on. |

Switch between them in **Settings → Sound → Music behaviour**.

Cues are **scheduled ahead of time against the `AudioContext` clock**, not fired
from a JS timer, so main-thread jank can't drop or smear a beep. A 100 ms
scheduler looks 1.5 s into the future and books each cue at an exact audio-clock
time.

`navigator.audioSession` requires **iOS 16.4+**. On older versions Pulse falls
back to a pool of `<audio>` elements fed by a WAV it synthesises at runtime —
media elements are exempt from the silent-switch restriction, so cues stay
audible. The Settings screen reports which path is live.

**Deliberately not implemented:** the Media Session API. Registering one would
seize the lock-screen transport controls away from Spotify, which is the opposite
of what this app is for.

### Screen wake

Uses the [Screen Wake Lock API](https://developer.mozilla.org/en-US/docs/Web/API/Screen_Wake_Lock_API),
acquired when a session starts and released on pause or exit. iOS drops the lock
whenever the page hides, so Pulse re-acquires it on every `visibilitychange` —
without that it dies the first time you glance at another app.

Wake Lock needs a **secure context (HTTPS)**. Opening `index.html` from the
filesystem will not keep the screen on. It also needs **iOS 16.4+**; where it is
unavailable the status chip says so rather than pretending.

### Timing

Interval ends are tracked as monotonic `performance.now()` deadlines, and each
new deadline is derived by *adding* to the previous one — so rounding never
accumulates across a 44-minute session. Display updates run on
`requestAnimationFrame`, decoupled from timekeeping, and the timer re-syncs after
background throttling on `visibilitychange`.

## Running it

No build step, no dependencies. Any static host works.

```bash
python3 -m http.server 8000    # then open http://localhost:8000
```

Note that on `localhost` the wake lock works (localhost counts as a secure
context) but `navigator.audioSession` is Safari-only — desktop Chrome will report
it as unsupported, which is expected.

## Deploying to your iPhone

1. Push to GitHub, then enable **Settings → Pages → Deploy from branch**, pointing
   at this branch, folder `/ (root)`. You need the HTTPS URL Pages gives you —
   Wake Lock will not work over plain HTTP.
2. Open that URL in **Safari** on the iPhone (not Chrome — the relevant APIs are
   WebKit's).
3. **Share → Add to Home Screen.**
4. Launch from the **Home Screen icon**, not from Safari. Standalone mode gives
   you the fullscreen app chrome and a more reliable audio session.

## Verifying on-device

The three hard requirements can only be confirmed on a real iPhone. Go to
**Settings → Sound → Audio check** — it prints the live engine, session type,
context state, wake-lock state, secure-context and standalone status.

Then:

| # | Test | Expected |
|---|---|---|
| 1 | Start Spotify, return to Pulse, start a session | Music keeps playing; dips briefly on each cue |
| 2 | **Flip the physical silent switch on**, tap Test | Cues still audible |
| 3 | Leave a session running ~2 min untouched | Screen never dims; "Screen on" chip stays lit |
| 4 | Background the app, wait 30 s, return | Timer still correct; wake lock re-acquires |
| 5 | Enable Airplane Mode, relaunch from Home Screen | App still loads (service worker) |

If test 2 fails, check that **Music behaviour** is set to **Duck** — `ambient`
mode is silenced by the silent switch by design.

## Layout

```
index.html                 whole app (markup, styles, logic)
manifest.webmanifest       PWA metadata
sw.js                      offline shell cache
icons/logo.svg             logo, traced from the source artwork
icons/icon-*.png           app icons (180 / 192 / 512 / maskable)
```

Settings and stats are stored in `localStorage` on the device only. Nothing is
sent anywhere; the app makes no network requests after load.
