---
title: Why I started writing pr.nvim
date: 2025-09-14
permalink: /why-i-built-pr-nvim
---

# Why I started writing pr.nvim

I've been a happy Neovim user for a while, and I review a lot of pull requests. The two activities don't combine very well.

The usual flow looks like this:

1. PR notification → open the browser.
2. Read the diff in the browser, scroll up and down trying to remember the file structure.
3. Want to check something three files away from the diff → switch to terminal, `nvim`, find the file, look at it, switch back to browser.
4. Want to comment → click in the diff line, type, submit.
5. Want to draft several comments and review them together → start a review on GitHub. Now the comments are in a third state ("pending"), and I can only see them by going back to a specific tab in the UI.
6. CI failed → click the checks tab. Read the log. Curse a little.

Multiply by ten PRs and a day is gone.

I wanted the whole thing to live in the editor. So I started a plugin.

<div class="github-card" data-github="btj93/pr.nvim" data-width="400" data-height="" data-theme="default"></div>

## What I actually wanted

Before I wrote a single line of plugin code, I made myself write down what I was solving for:

1. **Inline review threads, in the buffer.** I want to open a file that has unresolved review comments and *see them as virtual text or signs* on the relevant lines. Same UX as LSP diagnostics — because that's what they are, conceptually.
2. **A floating window to read and reply to threads.** Not a popup that disappears the moment you move the cursor. A real, focusable, scrollable window I can navigate with normal Vim motions.
3. **Drafts that survive.** If I open a thread, type half a reply, accidentally close it — the draft is still there next time.
4. **One picker to find PRs.** I shouldn't have to leave the editor to switch to a different PR.
5. **Provider-agnostic.** I work in GitHub mostly, but not exclusively. I don't want this rewritten if I ever land on a GitLab project.

That last one turned out to drive most of the architecture.

## The "shell out to a CLI" decision

The most consequential decision was made very early: **do not call provider APIs directly from Lua.**

Reasons:

- I would have to reimplement auth flows for every provider — GitHub PATs, GitHub Apps, GitLab tokens, Bitbucket app passwords.
- I would have to track and update every provider's REST/GraphQL schema.
- I would have to handle rate limits, retries, pagination — all the messy bits.
- And then I'd have a Neovim plugin that does HTTP. There's already a tool that does HTTP per provider, and *I already have it installed*: `gh`, `glab`, `curl`.

So the plugin shells out. Provider abstraction becomes a tiny strategy interface:

```lua
-- lua/pr/providers/init.lua
local M = {}

local providers = {
  github    = require("pr.providers.github"),
  gitlab    = require("pr.providers.gitlab"),
  bitbucket = require("pr.providers.bitbucket"),
}

function M.get(name)
  return providers[name] or error("unknown provider: " .. tostring(name))
end

return M
```

Each provider implements a small set of operations:

```
list_prs(opts, cb)
get_pr(id, cb)
get_threads(pr_id, cb)
post_comment(pr_id, body, line, cb)
submit_review(pr_id, event, cb)
... and a few more
```

The implementation of each is "build the right CLI arg list, run it async via plenary, parse the JSON output". That's it.

This buys me a few things I didn't fully appreciate at first:

- **Auth is solved.** Whatever the user did to set up `gh auth login`, that's what the plugin uses.
- **Rate limits are solved.** The CLI handles them.
- **I can debug by running the same CLI command myself.** The plugin output is literally a JSON dump of `gh api ...`. I can copy the command from the plugin's log and run it in a terminal.
- **Adding a provider is a few hundred lines, not a few thousand.** GitLab and Bitbucket landed within weeks of GitHub working end-to-end.

## Inline threads as `vim.diagnostic`

This was the second decision that paid off out of all proportion.

Review comments are conceptually just diagnostics. They have a location (file + line), a severity, and a message. Neovim already has a beautifully designed API for displaying those: `vim.diagnostic`.

So pr.nvim publishes review threads as diagnostics in a dedicated namespace:

```lua
local ns = vim.api.nvim_create_namespace("pr.nvim/threads")

vim.diagnostic.set(ns, bufnr, {
  {
    lnum = thread.line - 1,
    col = 0,
    message = thread.body,
    severity = severity_for(thread),
    source = "pr.nvim",
  },
})
```

The instant payoff:

- Threads show inline with whatever the user has already configured for diagnostics — virtual text, signs, virtual lines, underlines, whatever.
- `]d` / `[d` jumps between threads, because to Neovim they *are* diagnostics.
- `:PRQuickfix` is literally just `vim.diagnostic.setqflist({ namespace = ns })`.
- Themes and colorschemes already handle the styling.

I get most of an inline review UX for free by being a good citizen on top of `vim.diagnostic`.

## Outdated and resolved threads — what to show by default?

This one was tricky.

A review thread can be:

- **active** — the line still exists at this position in the current commit.
- **outdated** — the line numbers no longer match the current commit's view of the file.
- **resolved** — the review participants explicitly marked the conversation as done.

Showing all three by default makes the buffer noisy. Showing only active hides important context — sometimes you *want* to see what was said on a line that has since been refactored.

The compromise I landed on:

- Active threads → inline by default.
- Outdated → off by default inline. Accessible via the picker and via filter toggles.
- Resolved → off by default inline. Toggleable to a low-severity (`INFO`) virtual line so the user's existing diagnostic theme can dim them visually.

```lua
opts = {
  show_outdated_inline = false,
  show_resolved_inline = false,
  diagnostics = {
    severity_resolved = vim.diagnostic.severity.INFO,
  },
}
```

Both toggles are reachable from the picker via `<C-f>` to cycle filters in place, so a user who *does* want to see all the context can flip them on without leaving the workflow.

## The popup is its own headache

The other piece is the floating window — the one that opens when you hit `<CR>` on a thread.

I tried implementing it from scratch with `nvim_open_win`. Bad time. There's the popup itself, but then there's a header, a body, a reply input, scrolling, focus management between header and body, key dispatch, layout on resize, layout on small screens, layout on multiple monitors...

The thing that saved me was [`nui.nvim`](https://github.com/MunifTanjim/nui.nvim). It's a UI-primitives library — popups, layouts, menus — and it does the chore work I really did not want to do. After switching to it, the popup code roughly halved and the bug count fell off a cliff.

## Picker as control surface

The other thing that has scaled well: pickers (snacks / telescope / fzf) as the central control surface.

Rather than have a custom UI for "list PRs", "list threads", "switch filters" — every list-shaped thing is a picker entry. Users pick the picker they already use. The plugin defines actions on entries; the picker handles fuzzy matching, preview, multi-select, the works.

```
:PR
   ├── list PRs in this repo
   ├── filter: status (open/closed/all)
   ├── filter: author
   ├── action: checkout
   ├── action: view info
   └── action: pick a thread
```

If I add a feature, it usually appears as a new action on an existing picker, not a new command. Fewer entry points to remember.

## Where it stands today

Today, pr.nvim:

- Lists PRs across GitHub, GitLab, and Bitbucket Cloud.
- Shows review threads inline as diagnostics.
- Has a popup for reading threads and posting replies.
- Has a checkout action so I can land on a PR's branch without leaving the editor.

There's a *lot* more I want to do — drafts that survive across sessions, full review submission with pending comments, suggestion blocks rendered as applyable diffs, status counters, CI checks. Some of that is already in progress. The next post will cover those pieces once they've shipped.

For now, the headline is: I haven't switched tabs to read a review comment in six weeks. That alone has been worth the build.

Happy reviewing!
