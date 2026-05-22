---
title: Three coordinate systems and a spring — what 0.4.0 of Reel taught me
date: 2026-05-17
permalink: /reel-coordinate-systems
---

# Three coordinate systems and a spring — what 0.4.0 of Reel taught me

[Reel](/reel-scrollable-window-manager) hit 0.4.0 a few days ago. The headline features this release: shared strips across horizontally-aligned monitors, an animated vertical "raise" on focus change, and a snapshot-based replay system for restoring window layouts when you switch back to a macOS Space.

<div class="github-card" data-github="btj93/reel" data-width="400" data-height="" data-theme="default"></div>

But the part I want to write about is the part that nearly broke me: the *three* different coordinate systems that overlay code on macOS quietly lives in, and how they collide.

If you've ever written a macOS app that draws on top of windows and felt like the math should work but doesn't, this is for you.

## A small detour: what a "raise" is

Quick context before the coordinate stuff. Reel 0.4.0 added a focus indicator I call **raise**: when you focus a window, that window's *column* lifts upward by a few pixels (configurable), and the previously-focused column drops back to baseline. It's a soft, peripheral hint that says "this one, right here".

The animation is per-column. Each column has its own Y offset and its own spring animation. When focus moves from column A to column B:

- B's spring retargets toward `raiseHeight`.
- A's spring retargets toward `0`.
- The frame loop ticks both columns each frame until both springs settle.

```swift
struct ColumnData {
    var tile: TileID
    var width: CGFloat
    var raiseAnimation: SpringAnimation?
    var cachedRaiseTarget: CGFloat
    // ...
}
```

Adding `raiseOffset` to the layout function was the easy part. The hard part was that the layout function — `computeTargetFrames(strip:, time:)` — is called from a dozen different code paths, and every one of them had to be updated to pass through the new offset. I missed exactly one. The window that landed in the wrong spot was on a non-primary monitor, of course, because *that's* where the coordinate-system bugs hide.

## The three coordinate systems

Here are the three. They are all valid, all in use somewhere in the macOS toolchain, and at the seams between them everything goes wrong.

### 1. CG coordinates

**Origin: top-left of the primary display. Y grows downward.**

This is the world [`CGEvent.location`](https://developer.apple.com/documentation/coregraphics/cgevent) lives in. It's what AX returns for a window's `position` attribute. It's what `CGRect` measurements in CoreGraphics use.

```
(0, 0) ────────► +X
   │
   │
   ▼ +Y
```

If you're reading the mouse position, you're in CG.

### 2. AppKit global coordinates

**Origin: bottom-left of the primary display. Y grows upward. Spans all displays.**

This is the world [`NSScreen.frame`](https://developer.apple.com/documentation/appkit/nsscreen) lives in. It's the coord space [`NSWindow.frame`](https://developer.apple.com/documentation/appkit/nswindow) is *expressed in*. It's what `NSEvent.locationInWindow` ultimately gets converted from.

```
   ▲ +Y
   │
   │
(0, 0) ────────► +X
```

On a single-monitor setup, the AppKit global y-coord is just `screenHeight - cgY`. Easy. The moment you add a second monitor and stack it above or below the primary, this no longer holds for the secondary monitor — its `frame.origin` is some negative or positive Y offset away from the primary's origin.

### 3. Window-local coordinates

**Origin: bottom-left of the *window*. Y grows upward.**

This is what [`NSView.convert(_:from:)`](https://developer.apple.com/documentation/appkit/nsview) returns. It's the coord space a view's `bounds` lives in.

```
   ▲ +Y inside the window
   │
   │
(0, 0) ────────► +X
   (bottom-left of THIS window)
```

For a window at `(100, 200)` in AppKit global coords with a height of `500`, the window-local coords are `(0, 0)` to `(width, 500)`, and the AppKit-global y-coord `300` corresponds to window-local y-coord `100`.

## The collision

The bug that took the most days in 0.4.0 was this:

> The drag-to-reorder overlay worked perfectly on the primary monitor and silently miscomputed everything by a few hundred pixels on the secondary.

The drag overlay is a fullscreen `NSWindow` covering the entire monitor while you drag a window's title bar. It receives a mouse position (from a `CGEventTap`), converts it into window-local coords to figure out which "drop zone" the cursor is over, and renders a preview thumbnail there.

The code looked roughly like this:

```swift
// Get mouse position from CGEventTap callback.
let cgPosition = NSEvent.mouseLocation  // ⚠️ this is AppKit global

// Convert to overlay-window-local coords.
let localPosition = overlayWindow.contentView!.convert(cgPosition, from: nil)
```

That `convert(_:from: nil)` was the trap. Read it carefully:

> Converts the given point from the **window's** coordinate system to the receiver's coordinate system.

It's converting from the *window's* coords, not from AppKit global coords. The two coincide *only when the window's origin is `(0, 0)`* — which is true on the primary display. On the secondary, the overlay window's origin is `(0, primaryHeight + somePadding)` or wherever the secondary monitor sits in AppKit global space. Feeding global coords into `convert(_:from: nil)` silently subtracts a zero where it should be subtracting `screen.frame.origin`.

So the math was off by exactly the secondary monitor's frame origin.

The fix is to route AppKit-global inputs through `convertFromScreen(_:)` first:

```swift
let screenRelative = overlayWindow.convertPoint(fromScreen: cgPosition)
let localPosition = overlayWindow.contentView!.convert(screenRelative, from: nil)
```

`convertPoint(fromScreen:)` does the screen-frame-origin offset for you. Once you have a point that's expressed in the overlay window's own coord system, `convert(_:from: nil)` does the right thing.

I now have a comment in the file that's roughly:

> `NSView.convert(_:from: nil)` converts from the **window's** coord system, not AppKit-global. They only coincide on the primary display. If you have an AppKit-global point, go through `convertPoint(fromScreen:)` first.

## What I do now

A few practices that have helped after this debacle:

**1. Name the coord system in the variable name.**

Not `point` or `position`. Use `cgPosition`, `appKitGlobalPosition`, `windowLocalPoint`. The verbosity catches mistakes at the assignment site, not at the bug site three modules away.

```swift
let cgMouse = NSEvent.mouseLocation       // wait, this is AppKit-global
// caught at compile time once I renamed: my eye notices the mismatch
```

**2. Centralize the conversions.**

I have a `CoordinateConversions` namespace that exposes the conversions I do most often. Inside it, the asserts are aggressive — non-finite values panic in debug builds. Outside it, callers don't have to remember the spelling of `convertFromScreen`.

**3. Test on a multi-monitor setup, with the secondary above the primary.**

Single-monitor and side-by-side don't trigger the worst bugs. Stacked monitors do, because the secondary's frame origin has a non-zero Y. If your test setup is "side-by-side with the secondary to the right of the primary", you're missing the entire class.

**4. AX frames are top-left.**

Just to round out the dance: the `AXUIElement` `position` and `size` attributes are in CG coords (top-left origin), even though everything else in AppKit is bottom-left. The conversion is per-display: `appKitY = screenHeight - cgY - height`. Get the wrong screen's height in there (e.g. the primary's instead of the display the window is on) and you've reintroduced the bug.

This is anchored at the primary display's height specifically — `CGMainDisplayID()`'s height — because that's the y-origin of AppKit's global coord system. If the primary changes mid-session (yes, this happens — you unplug the laptop's lid, the external becomes primary), you have to re-anchor. That cost me a release.

## The other big 0.4.0 change: shared strips across aligned monitors

Briefly, since this one was *also* a coord-system story.

When two monitors are horizontally aligned — same height, same y-origin, edges touch — Reel now treats them as one logical strip. Columns flow across the seam. Scrolling left/right moves the viewport across both monitors.

The detection logic is the easy part: two displays are "horizontally aligned" if their edges touch within 0.5 px AND they have any Y-overlap.

The fiddly part is *width preset resolution*. If a column says "33% width", 33% of *what*? The display the column currently centers on. So as a column scrolls across the seam, its target width can change. The animation has to handle the width change without snapping, because the user is mid-gesture and a snap would feel terrible.

The other fiddly part: macOS has a "Displays have separate Spaces" setting. When it's on, each display has its own Space and merging strips across them doesn't make sense (a Space change on one display doesn't affect the other). Reel falls back to per-display strips in that mode and surfaces a menu bar warning, because almost no one realizes that setting is the reason their multi-monitor experience is weird.

## Lessons

A handful that I keep coming back to:

1. **There are three coord systems on macOS.** CG (top-left, primary display), AppKit global (bottom-left, primary display), and window-local (bottom-left, per window). The conversion between them is *not* a single subtract; it depends on the display.
2. **`NSView.convert(_:from: nil)` is from the view's *window*, not from screen.** They coincide only on the primary display. This is the single biggest footgun.
3. **Name your coord systems in your variables.** `cgMouse` vs `appKitGlobalMouse` reads as a different thing to your eye than two variables both called `mouse`.
4. **Test on stacked monitors, not side-by-side.** Stacked = different Y origins. Side-by-side = different X origins, but Y matches, so most coord bugs sleep through your test.
5. **AX frames are CG.** The whole rest of AppKit is bottom-left; AX is top-left; you'll convert wrong at least once.

Reel 0.4.0 is a much more solid release than 0.3 was, and most of the work was in this kind of careful boundary-checking. The features look small on a changelog. The patches under them are pages of "and now we also do the conversion through `convertPoint(fromScreen:)` so the secondary monitor stops being haunted."

Worth it. The thing is on my dock, all day, every day.

Happy scrolling — across as many monitors as you like.

## References

**Apple class docs**

- [`NSView`](https://developer.apple.com/documentation/appkit/nsview) — `convert(_:from:)` lives here
- [`NSWindow`](https://developer.apple.com/documentation/appkit/nswindow) — `convertPoint(fromScreen:)` lives here
- [`NSScreen`](https://developer.apple.com/documentation/appkit/nsscreen) — display geometry in AppKit-global coords
- [`CGEvent`](https://developer.apple.com/documentation/coregraphics/cgevent) — `location` is CG (top-left primary display)
- [`AXUIElement`](https://developer.apple.com/documentation/applicationservices/axuielement) — frames are CG

**Related**

- [Part 1 — Reel: writing a scrollable tiling window manager for macOS](/reel-scrollable-window-manager)
