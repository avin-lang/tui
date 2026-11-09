# Decisions

Each choice the library makes that its users may want to know about, why it was made and what else
was weighed. All are **settled on the user's delegation 2026-11-06** (branch `core`). Review with
the user: the width rules, and frames on standard output, refused when it is not a terminal.

## Events come from a blocking `poll(timeout)` on the caller's thread

`Terminal.poll(timeout)` waits on standard input with `sysl.io.wait.ready`, in slices of at most 50
ms, looking at the resize and stop signals between slices, and answers the first event or `None`.

- **Why:** a full-screen program is a loop of "draw, then wait for input or the next tick", which is
  exactly one call. No thread is started, nothing is shared, and the decoder and terminal state stay
  owned by one thread. A program that wants events on a channel spawns a thread that polls and sends.
- **Alternatives:** a reader thread feeding a `Channel[Event]` (Avin's `Channel` has no timed
  receive, so the program's tick would need a second thread or a timer event); a callback per event
  (inverts the program's control flow).

## Frames are written to standard output, with `flush()`

**Supersedes** the earlier decision to write frames to `/dev/tty`. Each frame is `print`ed and
`flush()`ed; input is still read from standard input.

- **Why:** Avin now flushes on request, so the reason for opening `/dev/tty` is gone, and one stream
  is simpler: no file to open or close, and the exit hook can write the restore sequence with the same call.
- **Redirected output:** `open` fails with "standard output is not a terminal" when standard output is
  not a tty (`isTty`-style check on descriptor 1), as it does for standard input. Escape sequences in a
  file or pipe are never wanted, and drawing on a terminal behind the program's back would surprise.
- **Alternatives:** keep `/dev/tty` (lets `tool | less` still draw, but needs a second file and
  ignores that the program's own printing goes elsewhere); draw regardless of what stdout is.

## Escape is told from a sequence by a 25 ms pause

A lone escape byte is the Escape key when nothing follows it within 25 ms; an escape before any other
key is that key with `Alt`. A sequence cut short is read as Escape and the keys of its bytes. A
bracketed paste waits for its end however long it takes.

- **Why:** terminals send a sequence's bytes in one write; 25 ms is far below what a person notices
  and above what a local or ssh terminal takes to deliver one write.
- **Alternatives:** 50–100 ms (what some editors use; Escape feels slow), or no delay (breaks
  sequences split by a slow link).

## Keys: what the control bytes are

`0x0d` and `0x0a` are both `Enter` (the terminal's input translation may turn one into the other);
`0x09` is `Tab`; `0x7f` is `Backspace`; `0x08` is Ctrl-h, not Backspace; `0x00` is Ctrl-Space; the
other control bytes are Ctrl with the lowercase letter, and `0x1c`–`0x1f` Ctrl with `4`–`7`. Shift-Tab
is `BackTab` with `Shift`. Sequences the decoder does not know (focus reports, device attributes,
keypad modes it has no key for) are dropped.

- **Why:** what the common terminals send by default; dropping unknown sequences keeps a stray report
  from arriving as keys.
- **Alternatives:** an `Unknown(bytes)` event (a program would have to ignore it everywhere).

## Ctrl-C is a key; outside signals are `Stop` events

Raw mode turns the terminal's signal keys off, so Ctrl-C, Ctrl-\ and Ctrl-Z arrive as keys and the
program decides. `Terminate`, `Hangup` and `Interrupt` sent from outside are watched and arrive as
`Stop(signal)`, as does the end of input. The window size is watched (`WindowChange`) and arrives as
`Resize` once the new size has been read.

- **Why:** every way out then goes through the program's own `close`, which is the only way the
  alternate screen is left (see the last entry).
- **Alternatives:** keep `ISIG` on (Ctrl-C would kill the program with the screen not put back).

## Character widths

From the Unicode Character Database 16.0.0 (`tools/widthgen` writes `width_table.avin`):

- two columns for East Asian Wide and Fullwidth; one for Ambiguous (as a terminal in a Western locale
  shows them) and for everything unassigned;
- none for `Mn`, `Me` and `Cf`, except the soft hyphen and the prepended concatenation marks, which
  are shown; none for the Hangul vowels and finals U+1160–U+11FF and U+D7B0–U+D7FF;
- control characters are drawn as U+FFFD, one column, and never sent;
- a cluster is a character and the zero-width characters after it; a zero-width joiner also takes the
  next character, so a joined emoji sequence is one cluster **as wide as its first character**; a
  variation selector 16 after an emoji beyond ASCII makes the cluster two columns; two regional
  indicators are one two-column flag; zero-width characters with nothing to join are dropped.

- **Why:** this is how the terminals that render emoji sequences as one glyph (iTerm2, kitty,
  WezTerm, Ghostty, foot) lay them out.
- **Alternatives:** `wcwidth` per code point, summing the parts of a joined sequence (matches older
  terminals, which draw each part; a family emoji then takes six columns); full UAX #29 grapheme
  clustering (more tables, and terminals do not follow it either).

## The diff renderer

- A frame starts where the last left the terminal and ends with the style reset, so text written
  between frames and the shell afterwards start from the terminal's own colours.
- A style change is the shorter of the changes alone and a reset with everything; a tie takes the
  changes.
- A cursor move is the shortest of none, a relative move on the same row, and an absolute move; a tie
  takes the absolute one. Where the cells between the cursor and the next change are one column each
  and in the current style, writing them again is used when it is shorter than any move.
- After the last column the cursor's place is not trusted (terminals differ on the pending wrap), so
  the next change moves absolutely.
- The first frame, and a frame of another size than the last, clears the screen and draws every cell
  that is not blank.

- **Alternatives:** redrawing whole lines that changed (simpler, more bytes); scroll regions for
  moved content (not needed by the programs this serves yet).

## Colours

Colours are matched in RGB by squared distance against xterm's default palette. In 256 colours an RGB
colour is the nearer of the cube's colour (each channel to its nearest level) and the nearest grey.
The mode is `TrueColor` where `COLORTERM` is `truecolor` or `24bit`, `Palette256` where `TERM` names
`256color`, otherwise `Basic16`; `Options(colors = Some(mode))` overrides it. `NO_COLOR` is not
consulted: a full-screen program's colours are part of what it draws.

- **Alternatives:** a perceptual distance (CIELAB; more code for differences few see in a terminal);
  reading `terminfo` (a C library and a database for what two variables answer).

## The table keeps no rows

`Table` holds the columns, the selection and the scroll; the rows are passed to each `draw`.

- **Why:** a process list changes every second; the program owns it and the table never shows a stale
  copy.
- **Alternatives:** a table holding its rows with `setRows` (two copies of the data to keep in step).

## The terminal is put back by `close`, and by an exit hook when the program ends

`open` registers `process.onExit` with a body that holds only the leave sequence (a string, so
`@sendable`): it writes it, flushes and calls Sysl's cooked-mode restore, which keeps the saved mode
itself. `close` cancels the hook and does the same. `exit` and a trap (after its message) restore the
terminal; checked in a pty: the output ends with the leave sequence and `stty -a` shows `icanon echo`.

- **Why:** the exit hook is run however the program ends, so nothing is left on the alternate screen
  with the mouse reporting on. The saved mode needed no capture, so there was no gap to report.
- **Alternatives:** leave it to the program (what it did before the hook existed).

## `examples/top` reads the system through commands, behind a `Source` trait

**Settled on the user's delegation 2026-11-06** (branch `htop-demo`). The example runs `sysctl hw.ncpu vm.loadavg kern.boottime
hw.memsize vm.swapusage`, `vm_stat` and `ps -axo pid=,ppid=,user=,pcpu=,pmem=,rss=,time=,utime=,state=,comm=`
at every look (every 1.5 s, and 0.3 s after the first) and parses what they print. Everything else
sees only `Sample`s from a `Source`.

- A process's CPU% is the growth of its `time` (user and system) between two looks over the time
  that passed, as a percent of one processor; with one look there is nothing to compare, and `ps`'s
  own decaying `pcpu` is shown. A pid whose time went down or whose command changed is a new process.
- No command on macOS prints each processor's ticks (`kern.cp_time` does not exist there; `iostat`
  and `top` give one average), so the command source has no per-processor meters: one `CPU` meter
  shows all processors, its user part the growth of every process's `utime`, its system part the
  rest of `time`, over the time passed times the processor count. Processes that ended between two
  looks are left out of it. The header draws one meter per processor whenever a source gives ticks.
- Memory `used` is active plus wired pages, then the compressor's pages, then inactive (cached)
  pages, each its own colour; the figure is used plus compressed over `hw.memsize`.
- `sysctl` is asked without `-n`, so each value is found by its name rather than its line.

- **Why:** Sysl's library has no calls for the process list, per-process times and memory,
  per-processor ticks or the VM statistics, and this is a demo. The trait keeps the commands in one
  class (`CommandSource`), so a native source replaces it without touching the rest.
- **Alternatives:** Sysl bindings for `proc_listpids`/`proc_pidinfo`/`host_processor_info`/
  `host_statistics64` (the real answer; needs Sysl to grow them); `top -l 2` for the total (blocks a
  second per look); `iostat` (whole percents since boot unless given an interval, which blocks).
- **Review with the user:** the native calls Sysl would need are listed in the branch's report.

## 0.2.0 widgets

The decisions below are **Proposed** (branch `tui-020`), awaiting the user's review.

### Mouse reporting is off unless asked for — Proposed

`Options.mouse` defaults to `false`; bracketed paste stays on. `examples/hello` and `examples/top`
ask for the mouse, since they show it.

- **Why:** while a terminal reports the mouse, its own selection and copy need a modifier held, which
  surprises anyone who did not ask for clicks; marshal's `top` already turned it off by hand. Paste
  does not take anything from the terminal, so it stays on.
- **Alternatives:** keep it on (the program that does not use the mouse pays for the one that does).

### Splits are `Fixed`, `Fraction` and `Fill`, served in order — Proposed

`splitSizes` gives fixed parts and fractions what they ask, first come first served, and shares what
is left among the fills by weight, cut at the running total so the last fill takes the remainder.
Margins and padding are both `Rect.shrink`/`pad`: there is no box model, only rectangles.

- **Why:** one pass, predictable, and the parts add up to the whole whenever there is a fill.
  A fraction is two integers, so `Fraction(1, 3)` is exact with no real rounding.
- **Alternatives:** a constraint solver with `Min`/`Max` (ratatui's Cassowary; much more code for
  layouts this size); a fraction as a `real`; percentages.

### Table columns: `min` and `flex` on `Column`; dropped columns take their gap — Proposed

`Column` gained `min = 0` and `flex = 1`, so every 0.1.0 table keeps its widths. When the flexible
columns' minimums do not fit, the last flexible column with a minimum is left out (with the blank
column before it) and the rest are tried again. Styled cells are `Span(text, style: Option[Style])`
given to `drawSpans`; `draw` with strings wraps them. On the selected row a cell keeps its own style
with `selectedStyle`'s attributes added.

- **Why:** marshal's `top` drops its LABEL column when the screen is narrow and repaints STATE and
  DEADLINE in colour over the plain table; both become declarations. Adding the selected attributes
  (Reverse by default) to a cell's colours keeps the colour visible on the highlighted row.
- **Alternatives:** a `max` per column (not needed yet); a priority per column for dropping;
  replacing a selected cell's style with `selectedStyle` (loses the colour).

### `ScrollView` is top-anchored with a follow flag — Proposed

The view keeps the top line and `follow`; a draw while following shows the end; moving up stops
following, reaching the end starts it again. Like `Table`, it holds no lines; `LogView` is the one that
does, the last `limit` in a `Deque`, moving a non-following view up as old lines are dropped.

- **Why:** while reading back, new lines must not move what is on the screen; anchoring at the top does
  that, where counting back from the end (what marshal's `top` does) shifts on every line added.
- **Alternatives:** an offset from the end; following only by an explicit key.

### Wrapping breaks at spaces, a long word between clusters — Proposed

`wrap` breaks each `\n`-separated line at spaces, keeps spaces between two words of a row and an
indent at the start of the first row, drops them at a break, and breaks a word wider than the row
between clusters (a two-column cluster in a width of 1 gets a row of its own). A text block that does
not fit ends its last row with `…`.

- **Why:** what a reader expects of prose in a help screen or a command line; measuring clusters keeps
  CJK, emoji and combining marks whole.
- **Alternatives:** Unicode line breaking (UAX #14; breaks between CJK characters too, much more
  table); hyphenation; character wrap only (what marshal's `top` does).

### Modals are a trait over a stack; Resize and Stop pass through — Proposed

`Modal` has `title`, `measure`, `draw` and `handle(e) -> bool` (stays open). `ModalStack` draws each
centred and bordered, bottom first, and gives keys, mouse and paste to the top alone; `Resize` and
`Stop` are never taken, so the program still resizes and quits. `MessageBox` and `Confirm` are the
two built in.

- **Why:** the help screen and the kill question are the two modals marshal's `top` hand-draws; a
  stack lets a confirm sit over a help. Passing `Stop` keeps a signal from being swallowed by a dialog.
- **Alternatives:** one modal slot (no confirm over a help); modals returning an enum of outcomes
  (the callback in `Confirm` keeps the trait to one bool).

### Gauge segments round at cumulative boundaries, in whole cells — Proposed

Each boundary is placed at the nearest cell to where the segments up to it end; what passes the
budget is cut off. Every cell is one column: a wide character for a segment is drawn as `█`.
Brackets take two of the given width and are left off under three columns.

- **Why:** rounding each segment alone can lose or gain a cell; rounding the running total cannot.
  Eighths of a cell (as `meter` uses) cannot show two colours meeting inside one cell.
- **Alternatives:** eighths for the last segment only; width given without the brackets (what
  marshal's bars take).
