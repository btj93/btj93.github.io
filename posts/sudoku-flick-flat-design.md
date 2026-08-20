---
title: Sudoku Flick — flat design, timing animations, and the rules I won't break
date: 2026-02-22
permalink: /sudoku-flick-flat-design
---

# Sudoku Flick — flat design, timing animations, and the rules I won't break

A month after [the first post](/designing-sudoku-flick), Sudoku Flick is *playable*. There's a real solver, a real generator, a real undo stack, and the flick gesture stopped fighting me weeks ago. I've put about two hundred commits into it since the new year.

But the part I want to write about today is design discipline: the constraints I wrote down on day one and how they've held up under contact with real features.

<div class="github-card" data-github="btj93/sudoku-flick" data-width="400" data-height="" data-theme="default"></div>

## The rule sheet I started with

Day one rules:

1. **Flat design only.** No shadows, no gradients, no blur, no glassmorphism, no 3D.
2. **Timing-based animations only.** `withTiming` with cubic easing. Never `withSpring`. No bounce, elastic, or overshoot.
3. **Zero hardcoded colors.** Every color goes through a theme token. The same component renders correctly in any palette.
4. **Strategic negative space.** Decoration is a defect.
5. **The grid is the interface.** The Sudoku board is the hero. Everything else exists to serve solving.

Each one of these has been tested at least once.

## Rule 2 has been the hardest

The rule I get tempted to break the most is "no springs."

Springs are *fun*. A button that gives a little bounce on tap, a cell that wiggles when you place a wrong digit, a pop on success. The vocabulary of mobile delight is, almost entirely, written in spring physics.

But springs do two things I don't want:

1. **They overshoot.** The whole point of Sudoku Flick's aesthetic is calm. A spring that overshoots and settles is the visual equivalent of a small percussive sound. Fine once, exhausting after the seventieth placement.
2. **They have non-deterministic duration.** A [`withSpring(target, { damping: 15, stiffness: 80 })`](https://docs.swmansion.com/react-native-reanimated/docs/animations/withSpring) can take anywhere from 200ms to 1.2s depending on the velocity it starts with. If two animations need to land at the same time (say, a digit appearing in a cell and a conflict highlight in a peer cell), [`withTiming`](https://docs.swmansion.com/react-native-reanimated/docs/animations/withTiming) lines them up cleanly. `withSpring` doesn't.

So: `withTiming` with cubic easing, every time. The constraint forces me to design transitions in terms of *duration* and *easing*, which are properties I can reason about, instead of *physics*, which are properties that emerge.

In Reanimated terms (with [`Easing`](https://docs.swmansion.com/react-native-reanimated/docs/animations/withTiming) for the curve):

```ts
const CAMERA_EASING = Easing.bezier(0.25, 0.1, 0.25, 1);

opacity.value = withTiming(1, { duration: 200, easing: CAMERA_EASING });
```

That `CAMERA_EASING` curve is the one I use for almost everything. It's a slight ease-out: fast to start, gentle to settle. The first time it appears in the file it feels like a deliberate choice. By the seventieth time, you stop noticing animations are happening, which is the goal.

### The wrong-digit shake

The hardest case I hit was "what happens when the player places a digit that conflicts with the rules?"

The instinct from mobile-game muscle memory is: shake the cell. A horizontal shake done with springs is a one-liner. I wrote it. It worked. It felt wrong inside the app.

What I ended up with instead:

- The cell flashes the conflict color (`semantic.error`) for 120ms.
- The digit is *not* placed.
- The peer cells that caused the conflict flash the same color for the same duration.

No movement at all, just a brief color event. The information transfer is the same (*this digit doesn't fit here, and these are why*), but the page feels still.

The spring impulse came from "movement = feedback". But color is also feedback, and color obeys the same rules as the rest of the UI.

## Rule 3 forced me to build a real theme system

"No hardcoded colors" sounds boring in a rule list. In practice it shaped most of the app's structure.

The shape:

```ts
const colors = useTheme(); // hook returns the active theme's tokens

<Cell
  borderColor={colors.gridBorder}
  background={isSelected ? colors.cellSelected : colors.cellBackground}
  textColor={isGiven ? colors.textGiven : colors.textUser}
/>
```

Every theme is an object that fills in *the same set of keys*. The component doesn't know which theme it's drawing in. Switching the theme is a single state change at the root; every component re-renders against the new tokens with no per-component code.

Two things I got out of this:

1. **Colorblind themes for free.** Two of the themes use a blue/yellow palette instead of red/green. I didn't have to find every conflict-color use site and change it; I had to define the tokens once. Components don't care.
2. **Dark mode for free, also.** Each theme has a light and dark variant. The "follow system" toggle just picks one of the two at render time.

The first time I tried to do a quick visual hack and hardcoded `#FF6B6B` for an error flash, I had to back it out within a day because it didn't work in the dark variant of one of the themes. After that I just stopped trying. Every color goes through the tokens.

## Rule 5 is the rule I forget

"The grid is the interface" is the easiest rule to write and the hardest to enforce.

Every "wouldn't it be cool if..." feature I think of starts with adding something to the board area. A hint badge in the corner of cells with hint-worthy peers, a timer at the top, a streak counter, a bonus animation when you complete a row.

Every one of those, when I sit with it for a day, gets cut. The principle is *the player should be able to look at the board and only see what's needed to solve the puzzle*. Hints belong on demand, not always-on. Time elapsed belongs in a screen the player navigates to, not always-on. Streaks don't belong at all.

The single visual element on the board that *isn't* the puzzle: a small indicator above the active cell showing which digit is about to be placed when the flick lands. It's there for a third of a second, it's a thin outline, and it disappears the moment the gesture ends. Anything more permanent gets cut.

## Things I added that don't break the rules

A few features I shipped this month that survived contact with the rule sheet:

- **Practice mode for new players.** A four-cell exercise where the player learns the flick gesture in isolation, before the real puzzle. Uses timing-based fades for cell transitions, no springs. The "all correct!" message fades in over 200ms, and a *single* "one more round" button fades in 150ms later. The player can replay as many times as they want; the Next button stays enabled after the first success.

- **Draggable bottom nav.** The settings/themes/about row at the bottom can be dragged side-to-side. You don't have to drag, you can tap, and the drag uses `withTiming` snap-to-position rather than springs. It works because Sudoku Flick has a deliberately small set of top-level destinations. The drag is faster than the tap for me, but the tap is the obvious thing to do for anyone else.

- **Animation toggle in settings.** This one was required. The setting reads from the system's "Reduce Motion" preference (via [`AccessibilityInfo`](https://reactnative.dev/docs/accessibilityinfo)) on first launch and can be overridden by the player. Every `withTiming` call in the codebase checks this value:

```ts
const duration = animationsEnabled ? 200 : 0;
opacity.value = withTiming(1, { duration, easing: CAMERA_EASING });
```

`duration: 0` means the transition is instant but the *callback structure* is the same. State machines that wait for an animation to finish still work; they just don't wait.

## Animation as a layer, not a feature

Across all of these, animation is a *layer* in the architecture, not a feature.

Every component renders the same regardless of whether animations are on. The state model doesn't change with the animation toggle. The toggle changes one thing: the `duration` argument to every transition.

It makes "what does this look like with animations off?" a one-line check. I don't have a separate accessibility-rendering path. I have *one* rendering path, parameterized by duration, where duration can be zero.

People turn animation off for good reasons: vestibular issues, low-power mode, sensory sensitivity. For an app that wants to ship to those people, that's the only design that's honest.

## Where it stands

The puzzle generator and solver are working. Difficulty levels are stubbed in. The flick gesture is at a place where I trust it on a moving train (literally, I commute with it every day).

What I'm building next month: an onboarding flow that doesn't suck. Sudoku Flick is asking the player to learn a new input method, and "tap-to-place" is the muscle memory we have to overwrite. The plan is a few-screen, in-context tutorial that *only* teaches the flick. No "welcome to Sudoku Flick" marketing slide. No "here are our features" carousel. Just: here's a cell, do this, you did it.

That's the topic for next time. Spring-free, of course.

## References

- [Reanimated `withTiming`](https://docs.swmansion.com/react-native-reanimated/docs/animations/withTiming): also where the `Easing` module is documented
- [Reanimated `withSpring`](https://docs.swmansion.com/react-native-reanimated/docs/animations/withSpring)
- [easings.net](https://easings.net/): visualization of common easing curves
- [React Native `AccessibilityInfo`](https://reactnative.dev/docs/accessibilityinfo): for reading the system Reduce Motion preference
- [Part 1 — Designing Sudoku Flick](/designing-sudoku-flick)
