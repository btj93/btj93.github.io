---
title: Reel — writing a scrollable tiling window manager for macOS
date: 2026-04-19
permalink: /reel-scrollable-window-manager
---

# Reel — writing a scrollable tiling window manager for macOS

I've been quietly building a window manager for the last couple of months. I called it **Reel** — because the windows live on an infinite horizontal strip, and you scroll left and right through them like film through a projector.

<div class="github-card" data-github="btj93/reel" data-width="400" data-height="" data-theme="default"></div>

This post is the "what is it, why does it exist, and what does it actually do" introduction. The hard architectural bits — coordinate systems, animation primitives, multi-monitor merging — deserve their own post next month.

## The pitch

Every tiling window manager I've used on macOS divides the screen into rectangles and stuffs windows into them. Yabai, Amethyst, Aerospace — all flavors of "split the available pixels".

This is a perfectly reasonable thing to do, and it's wrong for me.

What I want is closer to how I work on paper. I have one window I'm focused on, a couple I might glance at, and a stack of "I'll get back to these" that don't need to be visible right now. I don't want them tiled into smaller and smaller boxes; I want them off-screen, waiting, available with one keystroke.

Inspired by [niri](https://github.com/YaLTeR/niri) on Linux, Reel arranges windows on an infinite horizontal strip — one column per window — and the screen is a *viewport* onto that strip. You scroll the strip left or right to navigate. The focused window stays prominent. Its neighbors are visible at the edges of the screen. Everything else is parked off-screen at a position you can return to instantly.

The result feels a lot more like flipping through a film reel than tiling a grid. Hence the name.

## What it actually does

**Infinite horizontal strip.** Windows lay out left-to-right. There's no limit. You scroll the strip rather than splitting the screen.

**Spring-based scrolling.** Pressing the scroll key kicks off a spring animation that decelerates onto the target column. Repeated keypresses compound velocity — hold down "focus right" and the strip flies past, then catches.

**Trackpad, keyboard, and mouse.** Reel doesn't pick a favorite input device. Swipe with a modifier to scroll the strip. Hotkeys to jump columns. Click a window's title bar to focus it; drag with a modifier to reorder columns.

**Multi-monitor.** Horizontally-aligned monitors share one continuous strip — columns flow across the seam. Stacked or offset monitors get their own independent strips. (The merging logic is one of the things I want to write a whole post about; it took several iterations to get right.)

**Space-aware.** Switching macOS Spaces saves the strip's state and restores it when you come back.

**Position memory.** Reopen Chrome and it returns to where it was in the strip. The mapping survives across app restarts.

**Configurable focus indicator.** A subtle ring, a vertical raise, or a brief flash on the focused window. Pick what you can ignore.

**Floating windows.** Toggle any window out of the strip. Some apps (system dialogs, settings panels) auto-float by rule.

**IPC.** A `reel-msg` CLI sends commands over a Unix socket. Hook it up to whatever scripting layer you like.

No SIP disable. Pure Swift + the macOS Accessibility API.

## Why Swift, and why no SIP

A non-trivial chunk of macOS window-manager prior art relies on disabling System Integrity Protection or using private frameworks. I didn't want either.

- **SIP disable is a non-starter for me.** I want this on my main laptop. I want full disk encryption, MDM compatibility, all the protections. If a tool's prerequisite is "first, weaken your OS", that tool is not landing on my machine.
- **Private framework usage is fragile.** It works until the next macOS minor release ships a different ABI and your window manager segfaults on boot.

That leaves the documented Accessibility API as the way windows get moved and resized, and a single (one!) carefully-chosen private symbol — `_AXUIElementGetWindow`, which maps `AXUIElement` → `CGWindowID` — that's been stable for a decade and is used by every other AX-based WM in the ecosystem.

Swift is the natural language. Foundation gives you the data types, AppKit gives you windows and screens, `CGEventTap` gives you global hotkeys and gestures, `CADisplayLink` gives you a frame loop. No bridging headers, no Objective-C++, no Electron.

## The layering

The codebase splits into five library modules, with strict layering:

```
Reel (app entry) ──→ WindowManager ──→ Platform ──→ Core
                              │                         ↑
                              ├──→ Config (TOMLKit) ────┘
                              └──→ IPC ─────────────────┘
```

- **Core** — pure layout logic. Foundation + CoreGraphics only. No AppKit, no AX calls. The strip data structures, spring math, snap-point logic. Fully testable without a window in sight.
- **Platform** — macOS API wrappers. AX, hotkeys, gestures, displays. Every "talk to the OS" call goes through here.
- **WindowManager** — orchestration. The thing that listens for hotkeys, asks Core for new layouts, dispatches to Platform.
- **Config** — TOML parsing for `~/.config/reel/config.toml`.
- **IPC** — Unix socket server, `reel-msg` CLI client.

The most important boundary is the Core / Platform one. Core is a pure-function package: give it a strip and a timestamp, get back a list of target frames. That single design choice lets me unit-test almost every behavior without ever opening a window. Layout, animation, snap-on-flick, multi-monitor flow — all of it lives in a module that doesn't know macOS exists.

## The core data structure

The strip itself is, at heart, an ordered list of columns:

```swift
struct Column {
    var data: ColumnData
}

struct ColumnData {
    var tile: TileID            // identifies a window
    var width: CGFloat          // explicit width
    // ...
}

struct Strip {
    var columns: [Column]
    var focusedIndex: Int
    var viewOffset: ViewOffset
    // ...
}
```

The thing I'm most pleased with: column X-positions are *derived*, not stored. Given a list of widths and a gap, the X of column `i` is `sum(widths[0..i]) + i*gap`. Any mutation — insert, remove, reorder, resize — is a mutation of widths. Positions emerge.

This sounds obvious. It's not, because the temptation when you're first sketching the model is to store positions on each column "for performance". The moment you do that, every layout mutation has to keep N positions in sync, and your bug surface explodes. Derived positions = single source of truth.

The other Core type worth mentioning is `ViewOffset`:

```swift
enum ViewOffset {
    case static_(Double)
    case animation(SpringAnimation)
    case gesture(GestureState)

    func current(at time: TimeInterval) -> Double { ... }
}
```

It's a state machine for "where the viewport is on the strip". A keyboard scroll → `.animation`. A trackpad swipe in progress → `.gesture`. Idle → `.static`. The rest of the engine asks `viewOffset.current(at:)` and gets a single Double back; it doesn't have to care which mode the viewport is in.

That single function — "give me the current scroll position as of timestamp T" — is the heart of the whole animation pipeline. Once that exists, the frame loop is "every tick, ask Core for target frames, dispatch them to AX".

## The hardest bug so far

For posterity, the bug that ate the most days: setting an AX window's frame is not atomic.

`AXUIElement` exposes a `frame` attribute (kind of) but in practice you set the size and position separately. If you set position first and then size, the window often clips to the old position's screen bounds and gets repositioned to make the new size fit, which manifests as windows "drifting" off the strip during certain layout sequences.

The fix is a three-step `size → position → size` dance: nudge the size, set the position, then set the size again. The second size set is the one that "sticks" without macOS's per-screen clamping fighting you. It looks ridiculous. It works.

```swift
ax.setSize(target.size)
ax.setPosition(target.origin)
ax.setSize(target.size)
```

I'd love to find a cleaner approach. I have not. This dance is in production.

## Where it stands

The current state: Reel is in active daily use on my laptop. Most of the obvious features are in place. The next big push is around the parts that have been hard:

- The multi-monitor strip-merging logic, particularly when monitors are hot-plugged or rearranged mid-session.
- The focus-indicator vertical "raise" animation, which has to thread through the per-column Y offset without breaking the rest of the animation pipeline.
- A `pip` mode for things like the Picture-in-Picture window that wants to behave differently from tiled windows.

Next month I want to write specifically about coordinate systems and the kinds of bugs you only find when you have three of them in the same source file. Until then — if you're a Mac user who's been looking for a Linux-style scrollable WM, the binary is up and the source is open.

Happy scrolling.
