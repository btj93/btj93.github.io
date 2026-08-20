---
title: The Powerful Substitute Command in Neovim/Vim
date: 2025-05-03
permalink: /nvim-substitute-command
---

# The Powerful Substitute Command in Neovim/Vim

The substitute command is how you search and replace text in Neovim and Vim. Here is the syntax and the options it takes.

## Basic Syntax

The basic syntax of the substitute command is as follows:

```
:s/pattern/replacement/
```

- `:s` indicates that you are using the substitute command.
- `pattern` is the text you want to search for.
- `replacement` is the text you want to replace the pattern with.

### Example

To replace the first occurrence of "foo" with "bar" in the current line, you would use:

```
:s/foo/bar/
```

## Global Replacement

To replace all occurrences of a pattern in the current line, you can add the `g` flag:

```
:s/foo/bar/g
```

## Replacing in the Entire File

To perform a substitution across the entire file, you can use the `%` symbol before the substitute command:

```
:%s/foo/bar/g
```

## Confirming Each Replacement

If you want to confirm each replacement before it is made, you can add the `c` flag:

```
:%s/foo/bar/gc
```

Vim will prompt you for confirmation before replacing each occurrence of "foo".

## Case Sensitivity

By default, the substitute command is case-sensitive. To make it case-insensitive, you can add the `i` flag:

```
:%s/foo/bar/gi
```

This command will replace "foo", "Foo", "FOO", etc., with "bar".

## Using Regular Expressions

The substitute command also supports regular expressions, so you can write more complex search patterns. To replace any occurrence of "cat" or "dog" with "pet", you can use:

```
:%s/cat\|dog/pet/g
```

## Specifying the Search and Replacement Range

In Vim, you can specify the range of lines to search and replace within. This can be done using the `,` symbol:

```
<start>,<end>s/pattern/replacement/
```

### Absolute Line Numbers

For example, to replace "foo" with "bar" in lines 2 to 5, you would use:

```
2,5s/foo/bar/
```

### Relative Line Numbers

You can also use relative line numbers to specify the range:

```
+2,+5s/foo/bar/
```

Assuming the current line is line 1, this command will replace "foo" with "bar" in lines 3 to 6.

You can also use the `$` symbol to specify the end of the range:

```
2,$s/foo/bar/
```

This command will replace "foo" with "bar" in lines 2 to the end of the file.

The `.` symbol is also useful for selecting the current line:

```
.,$s/foo/bar/
```

This command will replace "foo" with "bar" from the current line to the end of the file.

## Selecting the whole word

To match the whole word, you can use the `\<` and `\>` symbols:

```
:%s/\<foo\>/bar/g
```

This command will replace "foo" with "bar" in the current line, but not in "foobar".

## Tips

- **Be cautious with global replacements**: Always double-check your patterns to avoid unintended changes.
- **Use the `c` flag for confirmation**: This can help prevent mistakes, especially in large files.
- **Practice with regular expressions**: The more regex you know, the more you can do with the substitute command.

## Conclusion

That covers the syntax and the flags. The rest is practice.

Happy editing!
