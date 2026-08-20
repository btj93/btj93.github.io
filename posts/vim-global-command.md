---
title: Mastering the global Command in Vim
date: 2025-05-10
permalink: /vim-g-command
---

# Mastering the global Command in Vim

The `global` command in Vim runs a command on every line that matches a pattern.

The `global` command has the following syntax:

```
:g/pattern/command
```

You can use it for substitutions, deletions, and other actions.

## Basic Syntax

The basic syntax of the `global` command is as follows:

```
:g/pattern/command
```

- `:g` indicates that you are using the `global` command.
- `pattern` is the text you want to search for.
- `command` is the action you want to perform on the matching lines.

### Example

To delete all lines containing the word "error", you would use:

```
:g/error/d
```

This command will remove all lines that match the pattern "error" from the current buffer.

## Viewing Grep Results

After running a `global` command, you can see the results directly in the buffer. You can also combine it with other Vim commands to act on the matching lines.

## Chaining Commands with Global and Substitute

It is not a coincidence that the [last post](https://btj93.github.io/nvim-substitute-command) is about another command in vim.

Vim lets you chain commands together. You can use the `global` command with the `substitute` command to find and replace text across all matching lines.

### Grep and substitute synergy

Suppose you want to find all lines that has the word "foo" in your file and replace the whole line with "bar". You can do this in one command:

```
:g/foo/s/.*/bar/g
```

This is particularly useful when you want to have the condition and the replacement string be different.

- `:g` is the command to execute on all lines that match the pattern.
- `foo` searches for the lines that contains the word "foo".
- `s/.*/bar/g` is the command to execute on the lines that match the pattern.
  - `.*` matches any character
  - `bar` is the replacement string

### Real world example

This example is based on a [reddit post](https://www.reddit.com/r/neovim/comments/1jy5gyf/how_would_you_go_about_editing_this/).

initial text:

```
line of code 1
# comment 1
line of code 2
# comment 2
line of code 3
# comment 3
```

result:

```
line of code 1
line of code 2
line of code 3
# comment 1
# comment 2
# comment 3
```

#### Solution 1

With the `global` command, we can use the following command:

```
:g/#/m$
```

- `g` is the command to execute on all lines that match the pattern.
- `#` searches for the lines that contains the `#` character.
- `m$` is the command to execute on the lines that match the pattern. In this case, it moves the line to the end of the file, denoted by `$`.

#### Solution 2

Another solution is to use this command:

```
:g/^#/norm ddGp
```

- `g` is the command to execute on all lines that match the pattern.
- `^#` searches for the lines that starts with `#`.
- `norm ddGp` is the command to execute on the lines that match the pattern.
  - `norm` executes a normal mode command
  - `dd` deletes the line
  - `G` moves the cursor to the end of the file.
  - `p` pastes the deleted line below the current line.

In this solution, `norm` lets us execute a normal mode keymap, so we can use normal mode keymaps to process the line.

## Advanced global Options

The `global` command also supports several options to refine your actions:

- `!`: Negate the pattern, applying the command to lines that do not match the pattern.
- `c`: Confirm each action before it is executed.

You can use these options to customize how the `global` command behaves. For example, to confirm each substitution, you can use:

```
:g/foo/s//bar/gc
```

## Conclusion

The `global` command runs an action on every line that matches a pattern. Chaining it with the `substitute` command lets you find and replace across those lines.

Happy editing!
