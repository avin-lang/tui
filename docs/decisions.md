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

The decisions below were **settled on the user's delegation 2026-11-09** (built on branch `tui-020`). Review with the user: the
widget API as a whole, once marshal's `top` has been ported onto it.

### Mouse reporting is off unless asked for

`Options.mouse` defaults to `false`; bracketed paste stays on. `examples/hello` and `examples/top`
ask for the mouse, since they show it.

- **Why:** while a terminal reports the mouse, its own selection and copy need a modifier held, which
  surprises anyone who did not ask for clicks; marshal's `top` already turned it off by hand. Paste
  does not take anything from the terminal, so it stays on.
- **Alternatives:** keep it on (the program that does not use the mouse pays for the one that does).

### Splits are `Fixed`, `Fraction` and `Fill`, served in order

`splitSizes` gives fixed parts and fractions what they ask, first come first served, and shares what
is left among the fills by weight, cut at the running total so the last fill takes the remainder.
Margins and padding are both `Rect.shrink`/`pad`: there is no box model, only rectangles.

- **Why:** one pass, predictable, and the parts add up to the whole whenever there is a fill.
  A fraction is two integers, so `Fraction(1, 3)` is exact with no real rounding.
- **Alternatives:** a constraint solver with `Min`/`Max` (ratatui's Cassowary; much more code for
  layouts this size); a fraction as a `real`; percentages.

### Table columns: `min` and `flex` on `Column`; dropped columns take their gap

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

### `ScrollView` is top-anchored with a follow flag

The view keeps the top line and `follow`; a draw while following shows the end; moving up stops
following, reaching the end starts it again. Like `Table`, it holds no lines; `LogView` is the one that
does, the last `limit` in a `Deque`, moving a non-following view up as old lines are dropped.

- **Why:** while reading back, new lines must not move what is on the screen; anchoring at the top does
  that, where counting back from the end (what marshal's `top` does) shifts on every line added.
- **Alternatives:** an offset from the end; following only by an explicit key.

### Wrapping breaks at spaces, a long word between clusters

`wrap` breaks each `\n`-separated line at spaces, keeps spaces between two words of a row and an
indent at the start of the first row, drops them at a break, and breaks a word wider than the row
between clusters (a two-column cluster in a width of 1 gets a row of its own). A text block that does
not fit ends its last row with `…`.

- **Why:** what a reader expects of prose in a help screen or a command line; measuring clusters keeps
  CJK, emoji and combining marks whole.
- **Alternatives:** Unicode line breaking (UAX #14; breaks between CJK characters too, much more
  table); hyphenation; character wrap only (what marshal's `top` does).

### Modals are a trait over a stack; Resize and Stop pass through

`Modal` has `title`, `measure`, `draw` and `handle(e) -> bool` (stays open). `ModalStack` draws each
centred and bordered, bottom first, and gives keys, mouse and paste to the top alone; `Resize` and
`Stop` are never taken, so the program still resizes and quits. `MessageBox` and `Confirm` are the
two built in.

- **Why:** the help screen and the kill question are the two modals marshal's `top` hand-draws; a
  stack lets a confirm sit over a help. Passing `Stop` keeps a signal from being swallowed by a dialog.
- **Alternatives:** one modal slot (no confirm over a help); modals returning an enum of outcomes
  (the callback in `Confirm` keeps the trait to one bool).

### Gauge segments round at cumulative boundaries, in whole cells

Each boundary is placed at the nearest cell to where the segments up to it end; what passes the
budget is cut off. Every cell is one column: a wide character for a segment is drawn as `█`.
Brackets take two of the given width and are left off under three columns.

- **Why:** rounding each segment alone can lose or gain a cell; rounding the running total cannot.
  Eighths of a cell (as `meter` uses) cannot show two colours meeting inside one cell.
- **Alternatives:** eighths for the last segment only; width given without the brackets (what
  marshal's bars take).

## 0.2.1: closing the gaps marshal's `top` drew by hand

Each of these was **settled on the user's delegation 2026-11-09** (branch `tui-021`): built so that a port of marshal's `top` can draw
its memory bar, footer, log header, LABEL column and log pane with tui and look byte for byte as it
does now. Every new parameter defaults to the 0.2.0 behaviour.

### Gauge boundaries: the product before the quotient (bug fix)

`gaugeCells` rounded `sum / budget * width + 0.5`, so 15 of 22 in 11 cells (exactly 7.5) came out
7.4999… and rounded down. It now rounds `sum * width / budget + 0.5`: with whole amounts the product
is exact, and a quotient that is exactly a half is representable, so it rounds up.

- **Why:** an exact half must round the same way whatever the budget; top's `unitBar` already
  avoids the quotient by counting in integers.
- **Alternatives:** integer amounts (`Segment.amount` is a `real`, and memory is not whole cells);
  rounding with an epsilon (moves the error rather than removing it).

### Gauge eighths: `eighths = true`; an overlay is two segments end to end

`gauge(..., eighths = true)` and `gaugeEighths` place each boundary at the nearest eighth of a
cell. A cell is drawn by the segment its left edge falls in: whole in the segment's character, or a
left block of the eighths it covers, in the segment's colour over `empty`'s background (as `meter`
draws its last cell). A segment beginning inside a cell shows from the next one. top's memory bar,
held in blue with the reserved part in cyan over it, is `[reserved, held - reserved]`: the negative
second amount when nothing is held beyond the reservation takes no cells, as top's `if held >
reserved` does. A test draws top's two-meter bar and the gauge side by side for every case of five
budgets, twelve widths and every reserved/held pair, and compares every cell.

- **Why:** the overlay top draws is exactly the end-to-end layout; a separate overlay segment would
  be a second way to say the same thing. Drawing a part cell over `empty`'s background (not the next
  segment's) is what top's bar shows, and what a terminal can show: one cell has one foreground.
- **Alternatives:** an `Overlay(amount, style)` segment kind; the part cell's background taken from
  the next segment (two colours in one cell, but not top's look); partial blocks only for `█`
  segments. Exact halves of an eighth now round up where top's quotient-first rounding could round
  down, so the port differs from today's top only there, in the fix's favour.

### Key hints: `sep`, `gap` and `cut`

`keyHints(..., sep = 1, gap = 2, cut = false)`: the blanks after a key and between hints, and
whether the last hint that does not fit is cut at the edge (every hint drawn, clipped, the answer
counting all of them, as `text` does) instead of left out. top's footer is `sep = 0, gap = 0, cut =
true`, the key bold and the word with its trailing space black on cyan.

- **Why:** three parameters say everything top's loop does; the blanks are not drawn, so a bar like
  top's carries its spaces in the words, in their style.
- **Alternatives:** a style for the gaps (top needs none); a `Span` list per hint (more general,
  but `spans` already is that).

### `wrapChars` and `Overflow.WrapChars`

Breaks each `\n`-separated line between any two clusters, filling each row, dropping and adding
nothing, so the rows of a line joined are that line: top's command and label in the log header.
Measured in columns by cluster, so a wide character is never split; for ASCII it is top's
`wrapText` exactly. Width 0 or less gives no lines, as `wrap` does (top passes at least 1).

- **Why:** a break at a space in a command line reads as part of the command; top relies on the
  rows rejoining to the text.
- **Alternatives:** a `breakAnywhere` flag on `wrap`; counting characters rather than columns (what
  top does; wrong for CJK). Adding a variant to `Overflow` breaks an exhaustive `match` on it in a
  program; none is known, and the brief named this shape.

### Column `max`, `fit` and `keep`

`max` caps a flexible column (what it gives up is shared again among the others, or left blank when
all are capped) and a fit column. `fit = true` sizes a column to its widest cell in the rows drawn,
between `min` (at least 1) and `max`, ignoring `width` and `flex`. Fit columns give way to the
flexible ones: the flexible columns are laid out as if no fit column were there, then each fit
column, the last first, is narrowed toward its `min` until every flexible column has `max(min,
keep)`, and left out (with its blank column) where that would take it under its `min`; a fit column
with nothing in it is left out. `keep` is separate from `min` because a flexible column is dropped
for want of its `min`, while top's COMMAND is never dropped but has LABEL leave it 20 cells. A test
compares the widths with top's `labelWidth` + a fixed LABEL for every width up to 150 and every label
length up to 31. `widths(width, rows)` is new, beside `widths(width)`.

- **Why:** top's LABEL rule (as wide as the longest label, 5 to 24, dropped when it would leave
  COMMAND under 20, absent when no job has a label) becomes three declarations.
- **Alternatives:** a priority per column for dropping; fit counting the heading (top sets `min`
  to the heading's width instead); `keep` on the fit column as "leave this much" (it is the flexible
  column's need, so it sits there).

### ScrollView: `fromEnd`, `back`, `backLimit`, `overlap`

With `fromEnd = true` the position is `back`, lines back from the end, kept as lines are added;
`follow` is `back == 0`; Home goes as far back as allowed. `back` is held to the lines at the last
draw, or with `backLimit >= 0` to that alone, because top reads only the last `rows + back` lines and
must be able to ask for more than it has read. `overlap` is what a page leaves in view (1 by default;
top pages by the whole pane, 0). A test drives top's hand-written pane and the view through the same
keys with lines arriving between them and compares every frame and `back`.

- **Why:** top counts "N lines back" because it tails a file; anchoring at the top would need the
  whole log's length.
- **Alternatives:** a separate `TailView` class (a second pager with the same keys); a page size in
  lines rather than an overlap (wrong once the pane is resized).
