# Pulse

Interval training timer built as an installable iOS web app. Robinhood-style dark
UI, a portfolio of 100 equipment-paired workouts, and a session view built around
three things that have to work in a gym.

## Session structure

Every session is **exactly 45 minutes**, played in **four quarters**:

```
Quarter 1   station A (superset pair)  ─ work · rest · work · rest ─ × rounds
            station change 30s          ← this IS the rest, never stacked with one
            station B                  ─ … ─                        ends on work
CARDIO      randomized 1–5 min
            station change 30s
Quarter 2 … Quarter 3 … Quarter 4                      ends on work
```

- Two stations (superset pairs) per quarter, both exercises shown on screen with
  the live one lit.
- A randomized cardio block between quarters — different machine and length each
  session — followed by one 30s station change.
- **No two rest periods ever run back to back.** A superset's trailing rest is
  simply not emitted; whatever follows it (station change, or the cardio block)
  *is* that rest. Enforced structurally, and asserted by the test suite across
  300 generated sessions.
- Work stays 45s and rest 15s. Rounds per station flex (typically 2–3) to fill
  the time left after cardio, and the residual is absorbed back into the cardio
  blocks so the total lands on 45:00 exactly — verified over 50,000 random rolls.

### Session screen

The timer is **pinned** — it never scrolls away. The complete plan scrolls
underneath it: every interval, grouped under sticky quarter headers, completed
rows struck through, the current one lit.

It follows along on its own, centring each interval as it starts. Scroll manually
and following pauses, with a **Jump to now** pill to get back; it also resumes by
itself after a few seconds idle. Scrolling is contained to the plan, so the page
behind it never rubber-bands.

The session chart draws the whole workout as a pulse waveform with the cardio
blocks highlighted.

## The three gym requirements

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
explicitly.

**The policy is: never dull your music.** Cues layer on top of Spotify at its own
volume and nothing dips.

| Mode | `audioSession.type` | Behaviour |
|---|---|---|
| **Mix** (default) | `ambient` | Cues layer on top at music volume. Spotify is **never** ducked. Requires the ring/silent switch to be **off**, or iOS mutes the cues. |
| **Duck** | `transient` | Cues play over the silent switch, but Spotify dips for each one. Opt in only if you want to train with the phone silenced. |

Switch between them in **Settings → Sound → Music behaviour**.

The tradeoff is inherent, not an oversight: the Audio Session API exposes no type
that both refuses to duck *and* ignores the silent switch. `ambient` never ducks
but obeys the switch; `transient` ignores the switch but ducks; `playback` and
`transient-solo` interrupt other audio outright. Pulse takes `ambient` and
surfaces the consequence in the UI rather than hiding it.

Because ducking used to be the default, a one-time migration flips existing
installs to Mix on first load, leaving every other saved preference untouched. A
later deliberate choice of Duck is never overridden.

### Sound engine

Cues play through a pool of HTML `<audio>` elements by default, not raw Web
Audio. Two reasons, both learned the hard way on a real phone:

- iOS exempts media elements from the ring/silent switch.
- An `AudioContext` can be **suspended by iOS at any time** (an interruption,
  another app taking the session). The `resume()` that would revive it runs from
  the cue scheduler — a timer, not a user gesture — which iOS refuses. The
  result was a context stuck suspended and *every remaining cue silent, with no
  error*. That failure mode is why elements are now the default.

Elements must still be unlocked by a user gesture, so the pool is primed (played
and immediately paused) inside the first tap. Without that the element path is
silent too.

**Precise** mode in Settings switches back to Web Audio, which schedules cues
ahead against the `AudioContext` clock — a 100 ms scheduler looking 1.5 s into
the future — so main-thread jank can't smear a beep. Tighter timing, but exposed
to the suspension problem above.

Whichever engine is live, a failed cue now **says so**: the Audio chip turns to a
warning, a toast fires once, and the Audio check names the error. Tap the **Audio
chip in the session header** to fire a real cue without leaving the workout.

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
| 1 | Start Spotify, return to Pulse, start a session | Music keeps playing at full volume, **never dips** |
| 2 | With the ring/silent switch **off**, tap Test | Cues clearly audible over the music |
| 3 | Leave a session running ~2 min untouched | Screen never dims; "Screen on" chip stays lit |
| 4 | Background the app, wait 30 s, return | Timer still correct; wake lock re-acquires |
| 5 | Enable Airplane Mode, relaunch from Home Screen | App still loads (service worker) |

If the music dips in test 1, check **Settings → Sound → Music behaviour** reads
**Mix**, and that the Audio check shows `audioSession ambient`.

If cues go silent in test 2, the ring/silent switch is on — that is `ambient`
behaving as designed. Either flip the switch off, or accept ducking by choosing
**Duck**.

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
