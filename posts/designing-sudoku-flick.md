---
title: Designing Sudoku Flick — from idea to first playable
date: 2026-01-25
permalink: /designing-sudoku-flick
---

# Designing Sudoku Flick — from idea to first playable

I have made a small new year resolution: ship a thing that isn't a developer tool.

I read code for a living, and writing libraries and Neovim plugins is fun, but the population of people who care about a Neovim plugin is in the low thousands at best. I wanted to build something my mother could touch.

The thing I picked is a Sudoku app.

<div class="github-card" data-github="btj93/sudoku-flick" data-width="400" data-height="" data-theme="default"></div>

This post is about why it's a Sudoku app, why it has a stupid name like "Sudoku Flick", and what the first three weeks of building it looked like.

## Why Sudoku, when there are a thousand of them?

There's a particular kind of mobile experience I've grown to hate.

I open a Sudoku app on the train. The first thing I see is a banner ad. I dismiss it. There's a streak counter. There's an XP bar. There's a "challenge a friend" button. The cell I want to tap is a 30×30 hit target inside a 40×40 cell with a 2px border. I tap. The keyboard at the bottom is twelve buttons in a row, and they're all 28px tall. I miss the 7 and hit the 8. I undo. I try again. By move five I've already accepted that I'm playing a game made by a marketing team.

The thing I want — the thing I think a lot of people want — is the opposite. Quiet. No streaks. No XP. No ads (or at most, the kind of ad that doesn't move). Big, comfortable touch targets. An input system that doesn't require my eyes to leave the puzzle.

So that's the brief. Quiet Sudoku, played with one thumb, optimized for the train.

## The flick gesture

The naming convention "Sudoku Flick" is on the nose, but the gesture really is the idea.

A traditional Sudoku UI has a number pad — nine buttons, one through nine. You tap a cell, then tap a number. Two taps minimum. If you want notes mode, three taps. The buttons live at the bottom of the screen because your thumb is at the bottom of the screen. Your eyes have to do a *lot* of round trips between the grid and the pad.

The idea behind Sudoku Flick:

- Tap and *hold* a cell. A radial menu appears around your thumb.
- The eight cardinal directions correspond to numbers 1-8.
- The center (release without flicking) is 9.
- Release in a direction → that number is placed.

```
        2
      ╲   ╱
   1 ─ 9 ─ 3
      ╱   ╲
        ...
```

(Yes, 9 in the middle is deliberate — the most common "fall back to center" gesture is the easiest one to do. The mapping of digits to directions is also configurable; I have a hunch right-handers and left-handers want different layouts, and I want to leave room for that.)

The thing this buys, from playtests with myself on the train:

- **Eyes stay on the cell.** The number you're placing appears under your thumb, not at the bottom of the screen.
- **One gesture, not two taps.** A move is select-and-place in one motion.
- **Notes mode is a modifier.** Hold + flick = place. Hold + a modifier + flick = note. (I'm still iterating on what "the modifier" is — long-hold? two-finger? a side button?)
- **Bigger targets.** The radial menu's hit zones are huge — each direction is a 45° wedge of a circle. Compare to a 9-button pad where each button is ~30px wide.

It's not the first time someone has tried gesture-based Sudoku input, but every time I've seen it the gesture was tacked on as a feature next to the number pad. I want to start from the gesture and design everything else around it.

## What the app is built on

A boring choice: React Native + [Expo](https://docs.expo.dev/).

- I want this on iOS and Android.
- I do not want to learn Swift and Kotlin in parallel.
- I want OTA updates for tweaks.
- I want to ship without a Mac in the build path (eventually).

Expo SDK 55 it is. Plus:

- [`react-native-reanimated`](https://docs.swmansion.com/react-native-reanimated/) for animation.
- [`react-native-gesture-handler`](https://docs.swmansion.com/react-native-gesture-handler/) for, well, gestures. Plus a custom PanResponder for the flick logic, because the radial gesture is finer-grained than what stock gesture handlers expose.
- [`react-native-svg`](https://github.com/software-mansion/react-native-svg) for the geometric UI elements.
- [`@expo-google-fonts`](https://github.com/expo/google-fonts) for typography — currently a triplet of Urbanist (titles), DM Sans (body), and IBM Plex Mono (digits).

The big absent friend, deliberately: no game engine. The board is just SVG and React. Sudoku is not a 60fps gameplay loop; it's a puzzle. Reanimated for the few animations that do happen; React for everything else.

## The aesthetic north star

I keep two references open while I work on this:

1. **[Mini Metro](https://dinopoloclub.com/games/mini-metro/).** Transit-map minimalism. Flat shapes. Two-tone color. Confident negative space. The UI gets out of the way of the thing you're doing.
2. **A blank piece of graph paper.** Sudoku is a paper puzzle. The screen should feel like the paper, not like a casino.

That gives me a set of constraints I write down on day one and refuse to break:

- **No shadows.** No drop shadow on cells, on buttons, on dialogs. None.
- **No gradients.** A button is a solid color. So is a background. So is a divider.
- **No blur, no glass, no skeuomorphism.** A button is not a tiny round-rect mountain. It's a rectangle.
- **No bouncy springs.** Animations are linear or cubic-eased, with explicit duration. No "boing".
- **Strategic negative space.** The board is the hero. Everything else exists to serve the board.

Constraints are easier than freedom for a one-person project. "Can I add a drop shadow?" is a question I do not have to answer because the answer is always no.

## What I built in three weeks

A lot of the first three weeks was infrastructure that has nothing to do with the game itself.

- Project scaffolding, navigation, theme system (light + dark + a couple of palettes).
- A `Cell` component that knows how to render given, user-placed, and noted digits with distinct font weights — never with color alone, because of colorblind users.
- A `Board` that lays out 9×9 with the right grid weight (heavier lines on the 3×3 box boundaries, lighter inside).
- A `Game` state model that tracks given digits, user digits, notes, conflicts, undo stack.
- The flick gesture, end-to-end. This took the longest.

The flick implementation deserves a separate post. The short version: I tried three different angle-thresholding schemes before landing on a polar-coordinate hit test with a deadzone radius. The first two felt like fighting the gesture system. The third one felt like the gesture was reading my mind.

I'll cover that in detail next month.

## What's left

For "first playable", I need:

- A puzzle generator (currently a hand-typed test grid).
- A solver, so I can validate the generator's puzzles have unique solutions.
- A real undo stack — current one is a debug placeholder.
- Animations on number entry and conflict feedback that don't break the no-shadows-no-springs rule.
- A settings screen, so the player can pick their theme and toggle animations.

I'm aiming for first playable by end of February. Not "App Store ready" — just "I can hand it to a friend and they can solve a puzzle".

The thing I keep reminding myself: this is a puzzle app. Not a platform. Not a SaaS. Not a side hustle. If I never make a cent on it, I will still have used it on the train every day, and that is good enough.

More soon.

## References

- [Expo](https://docs.expo.dev/) — managed React Native framework
- [react-native-reanimated](https://docs.swmansion.com/react-native-reanimated/) — animation library
- [react-native-gesture-handler](https://docs.swmansion.com/react-native-gesture-handler/) — gesture system
- [react-native-svg](https://github.com/software-mansion/react-native-svg) — SVG primitives
- [@expo-google-fonts](https://github.com/expo/google-fonts) — Google Fonts for Expo
- [Mini Metro](https://dinopoloclub.com/games/mini-metro/) — the design north star
