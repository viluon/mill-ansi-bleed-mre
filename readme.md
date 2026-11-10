> [!WARNING]
> MRE done by Copilot, Claude Opus 4.8.

# ANSI state bleeds across task log lines

`foo` prints `ESC[1mfoo` (bold, never reset). `bar` depends on `foo` and prints two plain lines.

```sh
./mill bar | cat -v
```

```
1] foo ^[[1mfoo
^[[0m^[[1mbar-1
^[[0m^[[1mbar-2
```

Expected: `bar-1` and `bar-2` plain, prefixed with `2] bar`.

Fails on Mill 1.3.1.

## Cause

`PromptLogger.logPrefixedLine`:

- `endOfLastLineColor` is shared by all tasks, so `foo`'s leftover bold is replayed on `bar`'s lines.
- Lines ending in a color get `fansi.Color.Reset` (`ESC[39m`) before `\n`. That resets only the foreground, so `\n` stays bold and `stripAnsi`'s `fansi.Str(...).render` moves the closing `ESC[0m` after it. The line no longer ends in `\n`, so `incompleteLine` stays set and the prefix is never printed again. `fansi.Attr.Reset` (`ESC[0m`) would keep `\n` last.
