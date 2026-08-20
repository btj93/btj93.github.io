---
title: Neovim Custom Keymap Essentials
date: 2025-04-05
permalink: /nvim-keymap-essentials
---

# Neovim Custom Keymap Essentials

Some keymap remappings I use in Neovim, with the Lua snippets to set them up.

## Understanding Keymaps in Neovim

Neovim (and Vim) is built around **modal editing**, where different modes (normal, insert, visual, etc.) have distinct key behaviors. Keymaps let you customize or extend these behaviors.

For example:

- In **normal mode**, pressing `h` moves the cursor left.
- In **visual mode**, `h` extends the selection left.

Custom keymaps can optimize these interactions to fit your habits and needs.

## Quality-of-Life Keymaps

### Keep the Cursor Centered with `zz`

Commands like `gg` (go to top) and `G` (go to bottom) place the cursor at the screen edge, forcing your eyes to hunt for it.

Instead, remap them to recenter the cursor with `zz`:

```lua
vim.keymap.set("n", "G", "Gzz", { noremap = true, desc = "Go to bottom and center" })
```

Similarly, remap search motions so the match is always centered:

```lua
vim.keymap.set("n", "n", "nzz", { noremap = true })
vim.keymap.set("n", "N", "Nzz", { noremap = true })
vim.keymap.set("n", "*", "*zz", { noremap = true })
vim.keymap.set("n", "#", "#zz", { noremap = true })
vim.keymap.set("n", "g*", "g*zz", { noremap = true })
vim.keymap.set("n", "g#", "g#zz", { noremap = true })
```

### Duplicate a Line and Comment the Original

When experimenting, it's handy to duplicate a line and comment out the original:

```lua
vim.keymap.set("n", "yc", "yy<cmd>normal gcc<CR>p", { noremap = true, desc = "Duplicate line and comment original" })
```

For **visual mode**, duplicate and comment a selection:

```lua
local function duplicate_and_comment()
  -- Exit visual mode
  local esc = vim.api.nvim_replace_termcodes("<Esc>", true, false, true)
  vim.api.nvim_feedkeys(esc, "x", false)

  -- Get selection range
  local start_line = vim.fn.line("'<")
  local end_line = vim.fn.line("'>")

  -- Yank and paste below
  vim.cmd(start_line .. "," .. end_line .. "yank")
  vim.cmd((end_line + 1) .. "put")

  -- Reselect pasted block
  vim.api.nvim_feedkeys("gv", "n", false)

  -- Comment the original selection
  vim.api.nvim_feedkeys("gc", "v", false)
end

vim.keymap.set("v", "yc", duplicate_and_comment, { noremap = true, desc = "Duplicate selection and comment original" })
```

### Select to the End of Line

We have:

- `Y` to yank to end of line
- `D` to delete to end of line
- `C` to change to end of line

Why not a **select to end of line**?

```lua
vim.keymap.set("n", "<leader>v", "vg_", { noremap = true, desc = "Select to last non-blank character" })
```

### Move to Start/End of Line with Home Row Keys

`_` and `$` move to start/end of line, but they're awkward to reach. Try:

```lua
vim.keymap.set({ "n", "v" }, "gh", "_", { noremap = true, desc = "Go to start of line" })
vim.keymap.set({ "n", "v" }, "gl", "$", { noremap = true, desc = "Go to end of line" })
```

### Easier Escape from Insert Mode

Pressing `<Esc>` is disruptive. Instead, remap quick sequences:

Common candidates includes:

- `<C-c>`
- `jj`
- `jk`
- `kj`
- `kk`

```lua
vim.keymap.set("i", "jk", "<Esc>", { noremap = true, desc = "Exit insert mode with jk" })
vim.keymap.set("i", "JK", "<Esc>", { noremap = true, desc = "Exit insert mode with JK" })
```

> **The only drawback is that now I am constantly inserting excessive `jk` at the end of sentences all over the place outside of NeoVim.jk**

Alternatively, remap `CapsLock` to `Esc` at the OS or firmware level.

#### Bonus Tip

If you did remap `jk` to `Esc`, you will find that there is a small delay between pressing `j` and the character showing up.

This is because NeoVim is waiting for the next keypress to know whether you are try to use the `jk` sequence or not.

After [a brief configurable delay](https://neovim.io/doc/user/options.html#'timeoutlen'), NeoVim inserts the character anyway.

To fix this, you can install the following plugin, designed specifically for this purpose:

<div class="github-card" data-github="max397574/better-escape.nvim" data-width="400" data-height="" data-theme="default"></div>

## Feature-Specific Keymaps

### Toggle Diff Mode Side-by-Side

Built-in diff mode is handy for comparing files. To toggle it:

```lua
local function toggle_diff()
  if vim.wo.diff then
    vim.cmd("diffoff!")
  else
    vim.cmd("windo diffthis")
  end
end

vim.keymap.set("n", "<leader>dd", toggle_diff, { noremap = true, desc = "Toggle diff mode" })
```

### Search and Replace Word Under Cursor

Replace all instances of the word under the cursor:

```lua
vim.keymap.set("n", "<leader>r", [[:%s/\<<C-r><C-w>\>//g<Left><Left>]], { desc = "Search and replace word under cursor" })
```

> Source: [Vim Tips Wiki](https://bit.ly/4eLAARp)

### Move the Cursor in Insert Mode

Sometimes you just want to nudge the cursor without leaving insert mode:

```lua
-- Move by character
vim.keymap.set("i", "<C-n>", "<Down>", { noremap = true })
vim.keymap.set("i", "<C-p>", "<Up>", { noremap = true })
vim.keymap.set("i", "<C-b>", "<Left>", { noremap = true })
vim.keymap.set("i", "<C-f>", "<Right>", { noremap = true })
vim.keymap.set("i", "<C-e>", "<C-o>$", { noremap = true })

-- Move by word
vim.keymap.set("i", "<M-f>", "<C-o>w", { noremap = true })
vim.keymap.set("i", "<M-b>", "<C-o>b", { noremap = true })
```

These mimic common shortcuts in terminals and macOS apps.

### Send Search Results to the Quickfix List

After searching (`/` or `*`), list all matches in the quickfix window:

```lua
vim.keymap.set("n", "g/", ":vimgrep /<C-R>//j %<CR>|:cw<CR>", { noremap = true, silent = true, desc = "Populate quickfix with search results" })
```

### Yank, Delete, or Change Up to the Next Quote

NeoVim lacks a simple `yq` (yank to next quote). Here's a workaround:

```lua
vim.keymap.set("n", "dq", "v/[\"'`]<CR><Left>d<cmd>nohlsearch<CR>", { noremap = true, desc = "Delete up to next quote" })
vim.keymap.set("n", "yq", "v/[\"'`]<CR><Left>y<cmd>nohlsearch<CR>", { noremap = true, desc = "Yank up to next quote" })
vim.keymap.set("n", "cq", "v/[\"'`]<CR><Left>di<cmd>nohlsearch<CR>", { noremap = true, desc = "Change up to next quote" })
```

It's by no means the most elegant solution, but it works!

## Final Thoughts

Start small, remap what annoys you, and build up from there.

Happy hacking! 🚀
