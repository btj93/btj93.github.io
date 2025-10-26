---
title: pr.nvim, part 2 — drafts, suggestions, and diagnostics
date: 2025-10-26
permalink: /pr-nvim-drafts-and-suggestions
---

# pr.nvim, part 2 — drafts, suggestions, and diagnostics

[Last month](/why-i-built-pr-nvim) I wrote about why I started building pr.nvim and the early shape of it: pickers, provider abstraction, inline diagnostics. The plugin could *read* a PR. This month it learned to *review* one.

The headline features that shipped in October:

- Persistent drafts.
- Suggestion blocks — rendered as readable diffs, applyable in place.
- Review submission with batched pending comments.
- Status counters in the winbar.
- `:PRQuickfix` and proper `vim.diagnostic` integration.

This post is about the design decisions behind those, because every one of them had at least one rabbit hole I had to climb out of.

<div class="github-card" data-github="btj93/pr.nvim" data-width="400" data-height="" data-theme="default"></div>

## Drafts: the requirement is "make me trust you"

A "draft" is a comment that's been typed but not posted. Sounds simple. The non-trivial part is that I need to *trust* the plugin with the draft.

If I type a paragraph, accidentally close the popup, and the paragraph is gone — I will go back to the browser. That's the bar. Drafts have to outlive:

- closing the popup window,
- closing the buffer,
- closing Neovim entirely,
- a Neovim crash.

The shape that satisfied me:

```
~/.local/share/nvim/pr.nvim/drafts/
└── <provider>/
    └── <repo>/
        └── <pr_id>/
            └── <thread_id_or_position_hash>.md
```

One file per draft. Markdown body. The path tells you exactly which thread it belongs to. If a draft becomes a posted comment, the file is deleted. If you close everything and open it again tomorrow, the popup pulls the draft back from disk.

Two non-obvious decisions:

1. **The draft file is the source of truth, not a Neovim buffer variable.** A Neovim buffer can disappear; a file in `~/.local/share` does not.
2. **Editing happens in a real buffer with `filetype=markdown`.** That gives me all my normal markdown editing keybindings, completion, treesitter highlighting, and so on, for free.

For the second one, I added `@user` and `#issue` omnifunc completion, because review comments are at least 30% mentions and issue references.

```lua
-- in the draft buffer
vim.bo.omnifunc = "v:lua.require'pr.completion'.complete"
```

The completion function shells out to `gh api search/issues` or the equivalent. It's slow on the first invocation per session — a few hundred ms — and instant from then on, because the results are cached per buffer.

### Conflict handling on save

A subtle problem: what if you opened the same thread in two Neovim instances, and edited the draft in both?

The way pr.nvim handles it now:

- When the popup opens, it reads the draft file and records its `mtime` and content hash.
- When the popup saves, it re-reads the file before writing and compares hashes.
- If the file changed under us, show a buffer-local message: *"Draft was modified externally. Run `:PRDraftMerge` to view both."*

It's not as elegant as a CRDT, but in practice the user opens both instances, sees the warning, picks one, copy-pastes the other in if they want, and moves on. Engineering-wise: cheap and predictable.

## Suggestion blocks: render as a diff, apply in place

GitHub-style suggestion blocks look like this in a comment body:

````
The variable name should be more descriptive:

```suggestion
const completedTaskCount = tasks.filter(t => t.done).length;
```
````

In the browser, that renders as a colored diff with an "Apply suggestion" button. The author of the PR clicks it; the suggestion lands as a commit on the branch.

I wanted the same workflow in pr.nvim. Two pieces:

1. **Render the suggestion as a diff** — not as a raw fenced code block — when the popup displays the comment.
2. **Make it applyable** with a single keymap, by writing the suggested content back to the original file.

For rendering, I extract the suggestion from the comment body, look up the *original* lines from the diff (so I can show a real diff, not just the replacement), and render side-by-side using treesitter ranges within the popup's markdown buffer.

For applying:

```lua
-- pseudo
local original_range = thread.diff_position
local replacement = suggestion.body
vim.api.nvim_buf_set_lines(
  bufnr,
  original_range.start_line - 1,
  original_range.end_line,
  false,
  vim.split(replacement, "\n", { plain = true })
)
```

It's deliberately a *local* edit. No commits, no pushes. The user reviews the change in their working tree like any other diff, then decides what to do with it. If they want a commit, they make one. If they want to amend, they amend.

This took one cycle of getting it wrong before I landed on it. My first version auto-committed the suggestion. People (correctly) hated this. Apply ≠ commit.

### Authoring suggestions from visual mode

The other half — *creating* a suggestion — is a small affordance that has paid off:

```lua
vim.keymap.set("v", "<leader>ps", function()
  require("pr").suggest()
end, { desc = "Author suggestion from visual selection" })
```

You select a block in visual mode, hit the keymap, and the plugin opens a draft popup pre-populated with:

````
```suggestion
<the lines you selected>
```
````

You edit the lines, save, and pr.nvim posts it as a review comment on those lines. The thing that took the longest here was not the keymap — it was figuring out what "those lines" map to in the *PR's* diff coordinates, because the buffer line numbers are local to the working tree.

The trick: when you check out a PR via pr.nvim's picker, the plugin remembers the head SHA and the diff hunks. Local edits invalidate the mapping, but as long as you suggest against unmodified lines, the position is unambiguous.

## Review submission: batched, not per-comment

GitHub (and friends) have two modes for posting review comments:

- **Single comment** — fires off immediately, no review context.
- **Pending comment inside a review** — staged, attached to a review, posted as a group when you "submit" the review with `Approve` / `Request changes` / `Comment`.

For anything more than a one-line "looks good" I want the latter, because it batches the email notifications, communicates intent (approve/RC/comment), and lets me move comments around before they go out.

In pr.nvim that surfaces as:

```
:PRReview start
   - opens a review session for this PR
   - any subsequent draft you post becomes a pending comment

:PRReview submit
   - prompts: approve / request changes / comment + optional body
   - posts the review with all pending comments attached
```

The internal state machine has two states: *no active review* and *active review for PR X*. Drafts know which state to post into. Status counter (more below) shows how many pending comments are stacked, so you remember to submit.

## Status counters + winbar

Once review submission existed, I needed a way to know "am I currently in a review, and how many pending comments do I have?".

A status counter in the winbar:

```
my-pr-branch │ pr.nvim: ✓ active review, 4 pending │ ...
```

It updates whenever a draft moves from "drafting" to "pending", or a pending comment is posted, or a review is submitted. The whole thing is a single autocmd on a custom user event that pr.nvim fires when state changes.

```lua
vim.api.nvim_create_autocmd("User", {
  pattern = "PRStateChanged",
  callback = function() update_winbar() end,
})
```

This pattern — fire a `User` event, let consumers re-render — keeps the winbar decoupled. Statusline plugins can hook the same event without any pr.nvim-specific glue.

## `vim.diagnostic` integration, properly this time

Last month's post described publishing threads as diagnostics in a custom namespace. That was the right call but the integration was incomplete: `:PRQuickfix` was a custom command that mimicked the quickfix list, and severity tuning was a hardcoded mapping.

The October pass aligned things properly:

```lua
opts = {
  diagnostics = {
    severity_active   = vim.diagnostic.severity.WARN,
    severity_outdated = vim.diagnostic.severity.HINT,
    severity_resolved = vim.diagnostic.severity.INFO,
  },
}
```

`:PRQuickfix` became one line:

```lua
vim.api.nvim_create_user_command("PRQuickfix", function()
  vim.diagnostic.setqflist({ namespace = require("pr").ns() })
end, {})
```

…and `]d` / `[d` jumping through threads now works exactly like LSP diagnostic jumping — including when filtered by severity (e.g. only active threads).

## What's next

The shape of the plugin is now usable enough that I'm starting to use it for non-trivial PRs at work — not just toy ones. The next stretch of work is:

- **Better thread navigation across files.** Walking thread-by-thread, not just diagnostic-by-diagnostic.
- **CI checks integration.** Show the failing job and let me jump to the log.
- **Permalink yanking** for sharing a specific commented line.

If there's a part 3, that's probably what it covers. For now the goal of "don't switch tabs to read a comment" has stretched to "don't switch tabs to *do a whole review*".

Closer to the goal every month.

Happy reviewing!
