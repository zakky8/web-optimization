# Web Audio Autoplay Policy — Validation Record
**Validator:** Senior Research Analyst
**Date:** 2026-05-26
**Folder:** 47-validation-round2

---

## Summary

Chrome's autoplay policy for Web Audio has been in effect since Chrome 71. AudioContext
created outside a user gesture starts in `suspended` state and requires an explicit
`resume()` call. A user gesture does NOT automatically resume a context — the call must
be made programmatically. Safari adds a fourth state (`interrupted`) for system-level
interruptions. MDN Best Practices doc confirms context created *inside* a gesture
starts in `running` without a resume() call.

---

## 1. Chrome Autoplay Rules

**Source:** https://developer.chrome.com/blog/autoplay/ (tier 1, accessed 2026-05-26)

### Media element autoplay (video/audio tags)

Chrome permits autoplay when any of the following is true:
1. Content is **muted** — always allowed regardless of user interaction
2. **User interaction** with the domain has occurred (click, tap)
3. Desktop only: **Media Engagement Index (MEI)** threshold crossed
4. Mobile: site added to **home screen**; Desktop: **PWA installed**
5. **Cross-origin iframes**: parent can delegate via Permissions Policy

### Media Engagement Index (MEI) criteria
All of the following must be met for a session to count as a "significant media playback event":
- Media playback exceeds **7 seconds**
- Audio present and **unmuted**
- Tab is **active**
- Video dimensions exceed **200×140 pixels**

MEI is a per-origin ratio of visits to significant media playback events.

### Web Audio API (AudioContext) rules
Effective since **Chrome 71**.

Key rules from the Chrome blog:
> "AudioContext state: Created contexts start in 'suspended' state without prior user gesture"
> "Resume requirement: Call context.resume() after user interaction"
> "Detection: Check AudioContext.state property; it switches to 'running' if playback is permitted"
> "Don't assume a video will play, and don't show a pause button when the video is not actually playing."
> "wait for a user interaction before starting audio playback so that users are aware of something happening."

```yaml
- claim: "Chrome has enforced AudioContext autoplay restrictions since Chrome 71"
  exact_quote: "AudioContext state: Created contexts start in 'suspended' state without prior user gesture"
  category: web_audio.autoplay.chrome_policy
  sources:
    - url: "https://developer.chrome.com/blog/autoplay/"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Web Audio API section"
  status: verified
  notes: "Policy active since Chrome 71; applies to AudioContext constructor"
```

---

## 2. Safari Autoplay Rules

**Sources:** MDN Web Audio Best Practices (tier 2, accessed 2026-05-26),
MDN AudioContext.state (tier 2, accessed 2026-05-26)

Safari enforces autoplay restrictions similar to Chrome. Key distinctions:
- Safari introduced its own autoplay blocking earlier than Chrome's formal policy
- Safari adds a **fourth AudioContext state not present in Chrome**: `"interrupted"`

The `interrupted` state occurs when audio is paused by an **external system event**
(e.g., phone call, Siri activation, hardware audio route change) rather than by
the web app itself. MDN documents this state specifically with an iOS Safari example:

```javascript
function play() {
  if (audioCtx.state === "interrupted") {
    audioCtx.resume().then(() => play());
    return;
  }
  // rest of the play() function
}
```

This state must be handled explicitly — `resume()` is required here too.

```yaml
- claim: "Safari adds an 'interrupted' AudioContext state not present in other browsers"
  exact_quote: "The interrupted state indicates that the audio context was paused by an
                external event outside the web app's control"
  category: web_audio.autoplay.safari_interrupted_state
  sources:
    - url: "https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Value descriptions table — 'interrupted'"
  status: verified
  notes: "Specific to iOS/Safari; code example in MDN docs targets this state"
```

---

## 3. AudioContext.state Values

**Source:** https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state
(tier 2, accessed 2026-05-26)

| Value | Meaning |
|---|---|
| `"running"` | Audio context is operating normally; audio processing active |
| `"suspended"` | Paused by user action inside the web app (`suspend()` called), or by browser autoplay policy at creation time |
| `"closed"` | Context has been closed via `AudioContext.close()`; cannot be restarted |
| `"interrupted"` | Paused by an external system event outside the web app's control (Safari-specific) |

```yaml
- claim: "AudioContext.state has four possible values: running, suspended, closed, interrupted"
  exact_quote: null
  category: web_audio.audiocontext.state_values
  sources:
    - url: "https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Value descriptions table"
  status: verified
  notes: "'interrupted' is Safari-specific; W3C spec defines only running/suspended/closed"
```

---

## 4. Does AudioContext Resume Automatically on User Gesture?

**Claim being verified:** "AudioContext resumes automatically on user gesture"
**Finding:** CORRECTED — This is partially true but requires precision.

**Scenario A — AudioContext created *inside* a user gesture handler:**
Per MDN Web Audio Best Practices (accessed 2026-05-26):
> "Create or resume context from inside a user gesture"

If the `new AudioContext()` constructor is called inside a click event handler,
the context **starts in `"running"` state automatically** — no `resume()` call needed.

**Scenario B — AudioContext created *outside* a user gesture (at page load):**
This is the most common pattern and the source of the policy restriction.
The context starts in `"suspended"` state. A user gesture does **not automatically**
resume it. The developer must explicitly call `context.resume()` inside a user gesture
handler.

MDN example:
```javascript
const audioCtx = new AudioContext(); // suspended — created outside gesture

button.addEventListener("click", () => {
  // Must explicitly call resume()
  if (audioCtx.state === "suspended") {
    audioCtx.resume();
  }
});
```

**Conclusion:** The statement "AudioContext resumes automatically on user gesture" is
FALSE for the typical use case. Explicit `resume()` is required. Only when the context
is constructed inside the gesture callback does it start running without `resume()`.

```yaml
- claim: "AudioContext does NOT resume automatically on user gesture; explicit resume() is required when context was created outside a gesture"
  exact_quote: "Create or resume context from inside a user gesture"
  category: web_audio.autoplay.resume_behavior
  sources:
    - url: "https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices"
      tier: 2
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Key Principle section"
    - url: "https://developer.chrome.com/blog/autoplay/"
      tier: 1
      accessed_at_utc: "2026-05-26T00:00:00Z"
      archive_url: null
      quote_location: "Web Audio API section — 'Resume requirement: Call context.resume() after user interaction'"
  status: verified
  notes: "Exception: context created inside gesture handler starts in 'running' without resume(). Typical pattern (context at page load) always requires explicit resume()."
```

---

## Overall Assessment

| Claim | Status |
|---|---|
| Chrome autoplay policy active since Chrome 71 | VERIFIED |
| AudioContext starts suspended without user gesture | VERIFIED |
| Safari adds `interrupted` state | VERIFIED |
| AudioContext auto-resumes on user gesture | CORRECTED — false for typical pattern; resume() is explicit |
| AudioContext state values: running / suspended / closed / interrupted | VERIFIED |

**Sources used:**
- https://developer.chrome.com/blog/autoplay/ (tier 1, accessed 2026-05-26)
- https://developer.mozilla.org/en-US/docs/Web/API/AudioContext/state (tier 2, accessed 2026-05-26)
- https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices (tier 2, accessed 2026-05-26)
