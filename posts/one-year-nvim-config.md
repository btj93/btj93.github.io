---
title: One year in — the Neovim config patterns I won't give up
date: 2025-07-19
permalink: /one-year-nvim-config
---

# One year in — the Neovim config patterns I won't give up

It's been roughly a year since I migrated to Neovim. The `.config/nvim` folder has gone through plugin sprees, deletion sprees, refactoring sprees, and the occasional "I'm sure this colorscheme is the one" phase.

I figured a quieter month was a good excuse to look back and write down the patterns that have *survived*. Not the plugins themselves — those still rotate. The patterns. The conventions. The little structural decisions that keep the config from collapsing into a 2,000-line `init.lua`.

## Pattern 1: one file per plugin

Easily the biggest one. My old setup had a single `plugins.lua` with every plugin spec, options, keys, and dependencies inline. After about thirty plugins it became unworkable.

The split that stuck for me, borrowed almost verbatim from [LazyVim](https://www.lazyvim.org/):

```
lua/
├── config/
│   ├── autocmds.lua
│   ├── keymaps.lua
│   ├── lazy.lua
│   └── options.lua
└── plugins/
    ├── harpoon.lua
    ├── mini-files.lua
    ├── treesitter.lua
    └── ...
```

Every plugin gets a file. The file returns a spec table. That's it.

```lua
-- lua/plugins/harpoon.lua
return {
  "ThePrimeagen/harpoon",
  branch = "harpoon2",
  dependencies = { "nvim-lua/plenary.nvim" },
  opts = { settings = { sync_on_ui_close = true } },
  keys = { ... },
}
```

Why this matters:

- **Disable a plugin by renaming the file.** `harpoon.lua` → `harpoon.lua.disabled`. No commenting-out, no `enabled = false` graveyard.
- **Diffs are clean.** A change to `harpoon.lua` only touches `harpoon.lua`. Code review (even of yourself, six months later) is dramatically easier.
- **You can search for "where is this plugin configured" by literally `:find <plugin>.lua`.**

I will never go back.

## Pattern 2: keymaps live with the plugin they belong to

This one took me a while to internalize. The temptation is to throw all keymaps into `lua/config/keymaps.lua` so you can "see them in one place".

The cost is that the plugin file no longer tells you what the plugin does on your machine. To answer "what's `<leader>ha`?", you have to grep across the config.

The convention I settled on:

- **General editing keymaps** (movement, escape, save, etc.) → `lua/config/keymaps.lua`.
- **Plugin-specific keymaps** → live inside the plugin's spec under `keys = { ... }`.

The lazy.nvim `keys` field is doubly nice because it triggers plugin loading. Half my plugins are now lazy-loaded purely because the only entry point is a keymap.

```lua
return {
  "echasnovski/mini.files",
  keys = {
    { "<leader>e", function() require("mini.files").open() end, desc = "Open mini.files" },
  },
  -- no need for `event = "VeryLazy"` — the keymap loads it
}
```

## Pattern 3: leader-prefixed keymaps over chord-style

I went through a phase of trying to be efficient by binding things to `<C-S-Whatever>`. It does not work for me. I can never remember which `<C-S-...>` does what, and tooling assumes too many of those for itself.

What stuck instead: nearly everything I do interactively goes through `<leader>`, with a one- or two-letter mnemonic.

```
<leader>1..4   -- harpoon slots
<leader>ha     -- harpoon add
<leader>hl     -- harpoon list
<leader>r      -- replace word under cursor
<leader>dd     -- toggle diff
<leader>v      -- select to end of line
```

When I forget what the second key is, [which-key.nvim](https://github.com/folke/which-key.nvim) shows me a menu. That's it. That's the system.

It's also a lot kinder to people who pair on my machine — they can read the popup and intuit the structure instead of memorizing what `<C-S-M-F4>` means.

## Pattern 4: don't fight LazyVim — extend it

I started with [LazyVim](https://www.lazyvim.org/) as a distro. Then I went through the "I want to build it all myself" phase. Then I came back.

The realization: LazyVim isn't an opinionated wall, it's a baseline. Every plugin it ships can be:

- **Replaced** by returning the same spec name with `enabled = false`.
- **Tweaked** by returning the same spec name with my own `opts`.
- **Extended** by adding more `keys` or dependencies.

So the "do it all myself" phase mostly cost me time. The thing I actually want is "LazyVim's defaults, but with my opinions layered on top." That's exactly what the distro is designed for.

Trust the path. Then deviate where you genuinely care.

## Pattern 5: capture experiments in `lua/user/` (or wherever) — not in `init.lua`

When I want to try a new keymap, plugin idea, or Lua snippet, the temptation is to wedge it directly into `init.lua` "just to see". Three weeks later there are eight half-formed experiments competing for attention there.

I now have a `lua/user/` folder that's essentially a scratch space:

```
lua/user/
├── jump_to_func.lua
├── neotest_to_tmux.lua
└── README.md
```

Things in `user/` are required from somewhere intentionally (`require("user.neotest_to_tmux").setup()` in a plugin spec, for example). If a snippet earns its keep, it moves into the plugin file it belongs to. If it doesn't, it stays where it is, and `git log` tells me when I stopped touching it.

It's a small thing. It dramatically reduces the half-life of a "quick test".

## Pattern 6: lock files in git, plugin data out

`lazy-lock.json` is in git. The plugin install directory is not.

This sounds obvious, but I had to learn it the hard way after losing my config to a corrupted plugin install. The lock file gives you reproducibility. The install directory gives you 200 MB of submodules and headaches.

```
.gitignore:
    /plugin/
    /spell/*.spl
    /spell/*.sug
```

```
git add:
    lazy-lock.json
```

A `:Lazy restore` after `git clone` puts the world back together exactly as it was. No regrets.

## Pattern 7: when in doubt, write it small and inline

I went through a phase of trying to extract every helper into modules. Beautiful from a Java perspective. Painful from a Vim perspective.

The thing about Lua-in-Neovim is that the API surface is the config. You're not writing a library; you're writing instructions for an editor. A five-line autocmd inline is more discoverable than a `require("user.utils").attach_autocmd(...)` with the same body hidden one file away.

Now my rule is: extract when the same logic shows up in three places. Not before.

## What I'm still figuring out

The next thing I want to crack is: a clean way to share keymap definitions between *modes*. I have `gh`/`gl` bound to start/end of line in both normal and visual via `{"n", "v"}`, but more complex maps (like motion repeats, or operator-pending custom motions) don't generalize as cleanly.

I don't have an answer yet. Probably a topic for a future post.

For now though — the structure above has survived a year of churn, and I can find anything in my config in under five seconds. That's the only metric I really care about.

Happy hacking!
