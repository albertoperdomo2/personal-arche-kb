---
title: Terminal menu activates but external display stays on Arc
date: 2026-09-23
type: incident
status: resolved
---

# 2026-09-23 — Terminal menu activates but external display stays on Arc

## Symptom

Caps Lock+T via Raycast and Command+Tab activated Terminal's menu bar on the laptop, but the external monitor continued showing full-screen Arc. Terminal had its own full-screen Space on that external display, visible in Mission Control. Laptop-only switching worked. The initial report that both configurations failed was corrected by controlled reproduction.

## Environment

- macOS 26.6.2, build 25G83; laptop plus external display.
- Apple Terminal, Arc, Raycast app hotkeys.
- Both captures recorded `AppleSpacesSwitchOnActivate=0`, `mru-spaces=0`, `spans-displays=0`, `GloballyEnabled=0`.

## Root Cause

The disabled app-to-Space switching preference was the actionable configuration issue: enabling it resolved the external-display failure, as confirmed by the user. The internal reason the disabled preference still permitted switching with the laptop alone was not established.

Activation itself succeeded. In the laptop-only capture, WindowServer scheduled a switch to Terminal Space 8, Dock acknowledged it, and the current Space changed 943 → 8. In the external-display capture, Terminal became the foreground app with `err=0/noErr`, but the active display changed and no corresponding switch to Terminal's Space was logged. Screenshots confirmed Terminal's menu on the laptop and Arc remaining on the external display. This was not simply a missed Raycast hotkey.

## Resolution

1. Open **System Settings → Desktop & Dock → Mission Control**.
2. Enable **When switching to an application, switch to a Space with open windows for the application**.
3. Retry switching to Terminal with the external display connected.

The user confirmed it worked immediately; no post-change trace was taken. No restart or other setting change was reported.

## Prevention / Runbook

Keep that option enabled when app activation should follow windows across Spaces. Inspect its UI state before resetting Terminal, changing hotkeys, or restarting Dock.

For recurrence, local capture script: `bash ~/terminal-switch-diagnostics/capture.sh`. It records active-app samples and relevant unified logs for 90 seconds without changing settings. Active-app telemetry alone cannot prove which Space was visible; pair it with observations or screenshots.

## Related

- [Apple's macOS Tahoe Desktop & Dock settings](https://support.apple.com/en-euro/guide/mac-help/mchlp1119/26/mac/26).
- Local findings: `~/terminal-switch-diagnostics/findings-20260923.md`.
- Working baseline: `capture-20260923-085836-zJqSXN`.
- External-display failure: `capture-20260923-090058-eiJg5I`.