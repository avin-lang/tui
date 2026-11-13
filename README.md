# tui

Terminal user interfaces in Avin: raw mode and the alternate screen, a grid of cells drawn by
difference, keys, mouse and paste read from raw bytes, and a few small widgets. Everything is Avin
code except six calls into Sysl's library: whether a descriptor is a terminal, its size, raw mode
on and off, waiting for input with a time limit, and reading it.

The module is `dev.avinlang.tui`.

```hocon
dependencies {
  tui { git = "github.com/avin-lang/tui", version = "0.2.5" }
}
```

## A program

```text
import avin.time
import dev.avinlang.tui.{Char, KeyPress, Options, Rect, Stop, Style, Terminal, Yellow}

main() -> int = Terminal.use(Options(), run) match
    Ok(()) -> 0
    Err(e) ->
        eprintln(e.message)
        1

run(term: Terminal)
    loop
        val g = term.frame()
        g.box(Rect(0, 0, g.width, g.height), Style.fg(Yellow), title = "press q")
        term.present(g)
        term.poll(time.seconds(1)) match
            Some(KeyPress(Char('q'), _)) -> return
            Some(Stop(_)) -> return
            _ -> ()
```

Each frame is drawn into a fresh `Grid` the size of the terminal; `present` sends only the bytes
that change what the terminal shows into it, in one write. `poll` waits for the next event for at
most the time given.

`examples/hello` draws a box, a clock, a meter and the last keys pressed:

```text
avin run examples/hello
```

`examples/top` is a simplified htop for macOS: meters for the processors, memory and swap, the load
and uptime, and every process in a table that sorts (`P` `M` `T` `N`), filters (`/`), shows the tree
of parents (`t`) and sends a process TERM (`k`, then `y`). `h` lists the keys, `q` quits. It reads
the system through `ps`, `sysctl` and `vm_stat`, behind a `Source` trait a native reader can replace.

```text
avin run examples/top
avin test examples/top
```

## What is in it

| | |
|---|---|
| `Terminal` | `open` and `close`, or `use(options, body)`; `frame()`, `present(grid)`, `poll(timeout)`, `width()`, `height()`, `redraw()`. Raw mode, the alternate screen, the cursor hidden, bracketed paste on and the mouse when asked for, all put back by `close`. |
| `Event` | `KeyPress(key, mods)`, `Mouse(MouseEvent)`, `Paste(text)`, `Resize(cols, rows)`, `Stop(signal)`. |
| `Key` | `Char(c)`, `Enter`, `Tab`, `BackTab`, `Backspace`, `Escape`, the arrows, `Home`, `End`, `PageUp`, `PageDown`, `Insert`, `Delete`, `F(n)`; modifiers `Shift`, `Alt`, `Ctrl` as bits. |
| `Grid`, `Cell` | Cells of one cluster of text and a `Style` each. A two-column character is held by its left cell; writing over either half blanks the other. |
| `Style`, `Color` | Foreground, background, and `Bold`, `Dim`, `Italic`, `Underline`, `Reverse`. Colours are `Default`, `Basic(0..15)`, `Indexed(0..255)` or `Rgb(r, g, b)`, sent as the nearest the terminal shows (`TrueColor`, `Palette256`, `Basic16`). |
| Drawing | `g.text`, `g.textIn` (padded, aligned, cut with `…`), `g.box` (four borders, a title), `g.meter` (a bar in eighths of a cell), `g.fill`, each clipped to a `Rect`. |
| Layout | `Rect.rows`, `Rect.columns` by `Fixed`, `Fraction` and `Fill`; `shrink`, `pad`, `centered`. |
| `Table` | Headings, rows given at each `draw`, a selected row kept in view by scrolling; fixed and flexible columns with minimums and weights, cells in their own styles. |
| `ScrollView`, `LogView` | A window over lines that follows the end, with the pager's keys. |
| `g.scrollbar` | A vertical scrollbar, drawn by a table or scroll view on its right edge when asked. |
| Text | `wrap`, `g.textBlock` (wrapped or cut with `…`), `g.spans`, `g.hline`, `g.vline`, `g.keyHints`, `g.labelValues`. |
| `ModalStack` | Boxes over the screen that take the input: `MessageBox`, `Confirm`, or a `Modal` of your own. |
| `g.gauge` | A bar of coloured segments against a budget, between brackets. |
| `Decoder` | Bytes to events, in pieces of any size; `Renderer`, grids to bytes. Both work with no terminal, which is how they are tested. |
| Width | `charWidth`, `textWidth`, `clusters`: two columns for East Asian wide characters and emoji, none for combining marks and joiners, from the Unicode 16.0.0 tables in `width_table.avin`. |

## Layout

A rectangle is cut into rows or columns by `Fixed(n)` cells, a `Fraction(num, den)` of the whole,
or a `Fill(weight)` of what is left; the parts always add up to the whole when one of them fills.
`shrink(top, right, bottom, left)` and `pad(dx, dy)` take margins or padding off it, and
`centered(w, h)` places a box in its middle.

```text
val parts = g.area().rows([Fixed(4), Fill(), Fixed(1)])
val (header, body, footer) = (parts(0), parts(1), parts(2))
val sides = body.columns([Fraction(1, 3), Fill()], gap = 1)
```

## Table

A column of width 0 is flexible: it gets its `min` first, then a share of what is left by its
`flex` weight. A flexible column there is no room for the minimum of is left out, the last first.
`drawSpans` draws cells that carry their own style; on the selected row they keep it, reversed.

```text
val t = Table([Column("ID", 4, 1), Column("STATE", 0, min = 8), Column("COMMAND", 0, flex = 3)])
t.drawSpans(g, body, jobs.map(j -> [Span(j.id.toString()), Span(j.state, Some(Style.fg(Green))), Span(j.cmd)]))
```

`header = false` leaves the headings out; `draw` with rows of strings works as before.

`max` holds a flexible column to at most that many columns, what it gives up going to the others.
`fit = true` sizes a column to the widest of its heading and its cells in the rows drawn, between
`min` and `max`; it gives way to the flexible columns, narrowed down to its `min` so each keeps its
`keep` (or its `min`), then left out, as is a fit column with no heading text and nothing in it. `widths(width, rows)` answers the widths a draw of
those rows would use.

```text
val t = Table([Column("ID", 5, 1), Column("LABEL", 0, min = 5, max = 24, fit = true), Column("COMMAND", 0, keep = 20)])
```

## ScrollView and LogView

A `ScrollView` keeps which line is at the top and whether it follows the end; the lines are given to
each `draw`. Moving up stops following, reaching the end starts it again. `handle(key)` takes the
arrows, Page Up and Down, Home and End. A `LogView` holds the lines too, the last `limit` of them.

```text
val log = LogView(limit = 5000)
log.append(chunk)
log.draw(g, body)
if e is KeyPress(k, _) then log.handle(k)
```

A page is the rows less `overlap` (1 by default; 0 pages by the whole view). With `fromEnd = true`
the view is anchored at the bottom: its position is `back`, the lines from the last shown to the
last, kept as lines are added, so what it shows moves up with them. `back` is held to the lines
there are, or with `backLimit` set to that alone, for a pane that reads only the last `rows + back`
lines of a longer log.

```text
val view = ScrollView()
view.fromEnd = true
view.overlap = 0
view.backLimit = 10000
view.draw(g, pane, readLastLines(log, pane.h + view.back))
```

## Scrollbar

`g.scrollbar(r, total, first, shown)` draws a vertical bar down the first column of `r` for `total`
items of which `shown` are visible from `first` on: a track of `│` and a thumb of `█`, in whole
cells. The thumb is in proportion to what is shown, at least one cell; it touches the top only at
the start and the bottom only at the end. Nothing is drawn while everything fits. `scrollThumb`
answers where the thumb is, and a `Scrollbar` gives the characters and styles.

A `Table` or `ScrollView` (and so a `LogView`, through its `view`) draws one down its right edge
when its `scrollbar` is set and its rows do not fit, taking that column from its width only then;
it is `None` by default, and a table that fits draws exactly as without it.

```text
t.scrollbar = Some(Scrollbar())
log.view.scrollbar = Some(Scrollbar(track = Style.fg(BrightBlack), thumb = Style.fg(Cyan)))
```

## Text

`wrap(text, width)` breaks at spaces and breaks a word wider than the line between characters, all
measured in columns, so CJK and emoji take two. `g.textBlock(r, text)` draws it, the last row ending
with `…` when there is more; `overflow = Ellipsis` cuts each line instead of wrapping.
`wrapChars(text, width)`, and `overflow = WrapChars`, break between any two characters instead,
filling every line, so the lines joined are the text: for command lines and paths.

```text
val rows = g.textBlock(r, description, Style.fg(BrightBlack))
```

## Modals

A `ModalStack` draws each modal centred over the screen in a bordered box, and its `handle` gives
keys, mouse and paste to the top one alone (a `Resize` or `Stop` is never taken). `MessageBox` closes
on any key; `Confirm` calls back with the answer to `y` or `n`. A `Modal` of your own has a `title`,
a `measure`, a `draw` and a `handle` that answers whether it stays open.

```text
modals.push(Confirm("kill", s"Kill job $id?", yes -> if yes then kill(id)))
if !modals.handle(e) then handleBase(e)
modals.draw(g, g.area())
```

## Gauge

```text
g.gauge(6, 1, 20, [Segment(real(ours), Style.fg(Green)), Segment(real(theirs), Style.fg(Magenta), '▒')], real(budget))
```

draws `[█████▒▒▒··········]`: each boundary at the nearest cell to where the segments up to it end
(an exact half rounding up), anything past the budget cut off.

`eighths = true` places the boundaries in eighths of a cell: a cell a segment only partly covers is
a left block (`▏` to `▉`) in its colour over `empty`'s background. One part drawn over the start of
another is two segments end to end, the inner one and then the rest of the outer:

```text
g.gauge(6, 2, 22, [Segment(real(reserved), Style.fg(Cyan)), Segment(real(held - reserved), Style.fg(Blue))], real(budget), rest = ' ', eighths = true)
```

## Small helpers

```text
g.hline(0, 3, g.width, Style.fg(Cyan))
g.keyHints(0, g.height - 1, [("q", "quit"), ("↑↓", "move"), ("h", "help")])
g.labelValues(r, [("kind", job.kind), ("command", job.cmd)])
```

`keyHints` leaves out a hint that does not fit whole; `labelValues` lines the values up after the
widest label and cuts them with `…`. `keyHints` takes `sep` (blanks after a key, 1) and `gap`
(blanks between hints, 2), and `cut = true` cuts the last hint at the edge instead of leaving it out:

```text
g.keyHints(0, y, [("h", "Help "), ("Enter", "Log ")], Style.plain().with(Bold), Style(Black, Cyan, 0), sep = 0, gap = 0, cut = true)
```

## Mouse reporting is opt-in (changed in 0.2.0)

`Options.mouse` is `false` by default: while a terminal reports the mouse, its own selection and copy
need a modifier held. Ask for it with `Terminal.use(Options(mouse = true), run)`. Bracketed paste is
still on by default.

## On the way out

`close` (and `use`, when its body returns) leaves the alternate screen, shows the cursor, turns the
mouse and paste off and restores the terminal's mode. A `Terminate`, `Hangup` or `Interrupt` sent to
the program arrives as a `Stop` event, so the program closes the terminal as on any other way out.
In raw mode Ctrl-C is a key like any other.

A program that ends with the terminal open, by `exit` or by a trap, gets all of it back: `open`
registers an exit hook that writes the leave sequence and restores the mode, and `close` cancels it.
Frames go to standard output, so `open` fails where standard input or standard output is not a
terminal.

## Tests

```text
avin test .
```

The renderer is tested byte for byte, the decoder over byte sequences split at every point, and the
widgets drawn into grids; nothing needs a terminal. The exception is `tests/pty.avin` (and
`examples/top/tests/pty.avin`): they build a probe program (`testdata/ptyprobe`) with the compiler
named by the `AVIN_COMPILER` environment variable (default `avin`) and run it under `script`, which
gives it a pseudo-terminal, to check that `exit`, a trap and `close` all put the terminal back.

The width table is generated from the Unicode Character Database:

```text
avin run tools/widthgen -- <dir holding DerivedGeneralCategory.txt, EastAsianWidth.txt, emoji-data.txt> dev/avinlang/tui/width_table.avin
```

The choices behind all this are in [docs/decisions.md](docs/decisions.md).
