# Before / After — Telegram iOS voice-message waveform

**Target:** `submodules/TelegramUI/Components/AudioWaveformComponent/Sources/AudioWaveformComponent.swift`
(the component that draws the waveform inside every voice message bubble, and in the
recording preview and the media scrubber).

**Method:** read the shipped source at `f1dd7a2d` (master, 8 Oct 2026), verified every
claim with `grep` against the repo. No assumptions — line numbers and evidence below.

---

## What the feature is

A voice message draws 48-ish vertical bars. `draw(_:)` runs **once per display-link frame
while the message is playing** (`ConstantDisplayLinkAnimator` in `updatePlaybackAnimation()`
calls `setNeedsDisplay()` every tick). Each frame it:

1. scans every sample to find the peak (`maxSample`),
2. bins the samples into bars (`adjustedSamples`),
3. computes `playbackProgress` from the player status,
4. draws the bars, blending background→foreground up to the playhead.

This is a 60 fps hot path for a component that is on screen for the entire duration of
playback, on the most-used media type in the app.

---

## Finding 1 — dead work on every frame (performance)

**Before** (`draw(_:)`, lines 553–576):

```swift
var bins: [UInt16: Int] = [:]
for i in 0 ..< maxReadSamples {
    let index = i * numSamples / maxReadSamples
    let sample = samples[i]
    if adjustedSamples[index] < sample { adjustedSamples[index] = sample }

    if let count = bins[sample] { bins[sample] = count + 1 } else { bins[sample] = 1 }  // build histogram
}

var sortedSamples: [(UInt16, Int)] = []
var totalCount: Int = 0
for (sample, count) in bins {
    if sample > 0 { sortedSamples.append((sample, count)); totalCount += count }
}
sortedSamples.sort { $0.1 > $1.1 }   // sort it…
```

**Evidence it is dead:** `grep -n "sortedSamples\|totalCount\|bins" AudioWaveformComponent.swift`
returns **only these construction lines** — the histogram, the array and the sort are never
read again in this file.

This is a leftover of the `drawFakeSamplesIfNeeded` feature that still lives in the older
`AudioWaveformNode.swift`: there, the histogram fed `topCountPercent`, which decided whether
to synthesize fake samples for low-variety waveforms. The modern component dropped that
behaviour but kept the computation that supported it. So the app builds a `[UInt16: Int]`
dictionary (up to `maxReadSamples` insertions) **and sorts it, 60 times a second, forever,
and throws it away.**

**After:**

```swift
for i in 0 ..< maxReadSamples {
    let index = i * numSamples / maxReadSamples
    let sample = samples[i]
    if adjustedSamples[index] < sample { adjustedSamples[index] = sample }
}
```

The binning that the drawing actually uses is untouched. Pure deletion — no behaviour change,
nothing to get wrong.

---

## Finding 2 — the reveal animation ignores Reduce Motion (accessibility)

**Before** (`animateIn()`, line 344):

```swift
public func animateIn() {
    if self.animator == nil {
        self.revealProgress = 0.0
        self.setNeedsDisplay()
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.08) {
            self.animator = DisplayLinkAnimator(duration: 0.8, from: 0.0, to: 1.0, update: { … })
        }
    }
}
```

When you send a voice message, `ChatMessageInteractiveFileNode.animateSent()` calls this and
the waveform grows from nothing over **0.8 s**. It never checks Reduce Motion.

**Evidence it is an oversight, not a policy:** `UIAccessibility.isReduceMotionEnabled` is
already the established pattern in this codebase — **64 call sites**, including
`guard !UIAccessibility.isReduceMotionEnabled else { return }` in the same family of
animation code. This one component simply missed it.

**After:**

```swift
public func animateIn() {
    if UIAccessibility.isReduceMotionEnabled {
        self.animator?.invalidate()
        self.animator = nil
        self.revealProgress = 1.0
        self.setNeedsDisplay()
        return
    }
    …
}
```

Under Reduce Motion the waveform appears at its final state instantly — no scale/grow motion —
which is exactly what the setting asks for. Users who enabled it get a still waveform.

---

## The patch

`7 insertions(+), 17 deletions(-)`, one file. See `voice-waveform.patch`.

```
submodules/TelegramUI/Components/AudioWaveformComponent/Sources/AudioWaveformComponent.swift
```

---

## Honest status

- **Verified against real source** at `f1dd7a2d`: both findings, with line numbers and grep
  evidence.
- **Not compiled.** This machine has no Xcode/macOS, and Telegram-iOS builds only through
  Bazel + codesigning on a Mac. The change is a deletion plus a guard on an API the codebase
  already uses 64 times, so it is low-risk, but it has **not** been built or run here.
- **Scope:** this is a correctness/perf/a11y fix, not a visual redesign. It improves battery
  and frame budget and fixes a real accessibility gap, but it does not change how the
  waveform *looks*. If the goal is a Design-Contest "noticeably better-looking" entry, that is
  a different, bigger change — see the notes in the chat.
