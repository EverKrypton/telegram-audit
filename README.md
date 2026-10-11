# Telegram-iOS — code audit

A real audit of Telegram's official iOS client, run against the shipped source.
Every finding is verified in the code (line numbers + `grep` proof), not guessed.

**Base commit:** `f1dd7a2d` (master, 7 Oct 2026)
**Category:** every finding is a *bug fix* or *optimization* — the only category the
official `CONTRIBUTING.md` accepts in pull requests (it rejects new features and new UI).

## Files

- [`AUDIT.md`](AUDIT.md) — the full report: 10 verified findings across
  performance, correctness, crash-safety and accessibility.
- [`BEFORE_AFTER.md`](BEFORE_AFTER.md) — deep dive on the voice-message waveform.
- [`voice-waveform.patch`](voice-waveform.patch) — the patch submitted upstream.

## Submitted PR

- **TelegramMessenger/Telegram-iOS #2368** — `AudioWaveformComponent: drop per-frame dead
  work, honor Reduce Motion`
  → https://github.com/TelegramMessenger/Telegram-iOS/pull/2368

## Summary of findings

| # | Area | File | What | Category |
|---|---|---|---|---|
| A1 | Waveform | `AudioWaveformComponent.swift:553` | builds + sorts a sample histogram every frame, never reads it | perf |
| A2 | Waveform | `AudioWaveformNode.swift:194` | `progress` property is dead (both branches identical) | perf/dead |
| B1 | Gallery | `ChatItemGalleryFooterContentNode.swift:1301` | `isLandscape` if/else has identical arms | correctness |
| B2 | Core | `AccountStateManagementUtils.swift:3507,3653` | `if let _` with identical arms | correctness |
| C1 | Calls | `CallSessionManager.swift:1232,1261,1658` | `connections.first!` with no empty check (server data) | crash |
| D1 | Waveform | `AudioWaveformComponent.swift:344` | reveal animation ignores Reduce Motion | a11y |
| D2 | Location | `ChatMessageLiveLocationPositionNode.swift:16` | infinite pulse ignores Reduce Motion | a11y |
| D3 | Peers | `PeerOnlineMarkerNode.swift:67` | infinite "voice bars" ignore Reduce Motion | a11y |
| D4 | Location | `LiveLocationWavesNode.swift:60` | display-link waves ignore Reduce Motion | a11y |
| D5 | Effects | `ConfettiView.swift:134` | continuous confetti ignores Reduce Motion | a11y |

## Honesty notes

- Not compiled locally: this host has no macOS/Xcode, and Telegram-iOS builds only through
  Bazel + codesigning on a Mac. The submitted change is a pure deletion plus a guard on an
  API already used 64+ times, so the risk is low — but it has **not** been built here.
- Historically the official repo has **0 merged PRs** (it is a mirror of an internal repo),
  so a PR is allowed and welcomed in principle, but acceptance is not guaranteed.

Reported by **@ograinhard**.
