# tui

Terminal user interfaces in Avin: raw mode and the alternate screen, a grid of cells drawn by
difference, keys, mouse and paste read from raw bytes, and a few small widgets. Everything is Avin
code except six calls into Sysl's library: whether standard input is a terminal, its size, raw mode
on and off, waiting for input with a time limit, and reading it.

The module is `dev.avinlang.tui`.

```hocon
dependencies {
  tui { git = "github.com/avin-lang/tui", version = "0.1.0" }
}
```

## A program

```text
import avin.time
import dev.avinlang.tui.{Char, KeyPress, Options, Rect, Stop, Style, Terminal, Yellow}

main() -> int = Terminal.use(Options(), run) match
    Ok(()) -> 0
    Err(e) ->
        eprintln(e.message())
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

## What is in it

| | |
|---|---|
| `Terminal` | `open` and `close`, or `use(options, body)`; `frame()`, `present(grid)`, `poll(timeout)`, `width()`, `height()`, `redraw()`. Raw mode, the alternate screen, the cursor hidden, the mouse and bracketed paste on, all put back by `close`. |
| `Event` | `KeyPress(key, mods)`, `Mouse(MouseEvent)`, `Paste(text)`, `Resize(cols, rows)`, `Stop(signal)`. |
| `Key` | `Char(c)`, `Enter`, `Tab`, `BackTab`, `Backspace`, `Escape`, the arrows, `Home`, `End`, `PageUp`, `PageDown`, `Insert`, `Delete`, `F(n)`; modifiers `Shift`, `Alt`, `Ctrl` as bits. |
| `Grid`, `Cell` | Cells of one cluster of text and a `Style` each. A two-column character is held by its left cell; writing over either half blanks the other. |
| `Style`, `Color` | Foreground, background, and `Bold`, `Dim`, `Italic`, `Underline`, `Reverse`. Colours are `Default`, `Basic(0..15)`, `Indexed(0..255)` or `Rgb(r, g, b)`, sent as the nearest the terminal shows (`TrueColor`, `Palette256`, `Basic16`). |
| Drawing | `g.text`, `g.textIn` (padded, aligned, cut with `…`), `g.box` (four borders, a title), `g.meter` (a bar in eighths of a cell), `g.fill`, each clipped to a `Rect`. |
| `Table` | Headings, rows given at each `draw`, a selected row kept in view by scrolling. |
| `Decoder` | Bytes to events, in pieces of any size; `Renderer`, grids to bytes. Both work with no terminal, which is how they are tested. |
| Width | `charWidth`, `textWidth`, `clusters`: two columns for East Asian wide characters and emoji, none for combining marks and joiners, from the Unicode 16.0.0 tables in `width_table.avin`. |

## On the way out

`close` (and `use`, when its body returns) leaves the alternate screen, shows the cursor, turns the
mouse and paste off and restores the terminal's mode. A `Terminate`, `Hangup` or `Interrupt` sent to
the program arrives as a `Stop` event, so the program closes the terminal as on any other way out.
In raw mode Ctrl-C is a key like any other.

A program that ends without closing gets its terminal mode back, but stays on the alternate screen;
one that traps gets neither, and `reset` puts the terminal right.

## Tests

```text
avin test .
```

The renderer is tested byte for byte, the decoder over byte sequences split at every point, and the
widgets drawn into grids; nothing needs a terminal.

The width table is generated from the Unicode Character Database:

```text
avin run tools/widthgen -- <dir holding DerivedGeneralCategory.txt, EastAsianWidth.txt, emoji-data.txt> dev/avinlang/tui/width_table.avin
```

The choices behind all this are in [docs/decisions.md](docs/decisions.md).
