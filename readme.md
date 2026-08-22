# ANSI state from one task bleeds into the log lines of other tasks

Mill 1.1.8. Two tasks write to stdout via `os.call(stdout = os.Inherit)`. `foo` emits a
single line preceded by `ESC[1m` (bold, never turned off). `bar` depends on `foo` and
emits two plain lines.

```
./mill bar
```

Observed (`cat -v`):

```
^[[34m1] foo^[[39m ^[[1mfoo^[[0K
^[[0m^[[1mbar-1^[[0K
^[[0m^[[1mbar-2^[[0K
```

Two defects, both in `PromptLogger.logPrefixedLine`:

`bar`'s lines are rendered bold. `endOfLastLineColor` is one field shared by every
logger key, so the SGR state left over at the end of `foo`'s line is re-emitted at the
start of every subsequent line regardless of which task produced it.

`bar`'s lines also lose their `2] bar` prefix. `stripAnsi` normalises each line through
`fansi.Str(...).render`, which appends the state-terminating escape after the trailing
newline. The resulting byte array no longer ends in `\n`, so `incompleteLine` stays
true and the prefix is suppressed for every following line.

Expected: `bar-1` and `bar-2` are plain and prefixed with `2] bar`.

> [!WARNING]
> MRE done by Copilot, Claude Opus 4.8.
