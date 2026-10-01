# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A personal fork of suckless **st** (simple terminal, v0.9.3), an X11 terminal emulator in C. Upstream style applies: tabs, K&R braces, function return type on its own line, minimal comments, no external dependencies beyond Xlib/Xft/fontconfig/freetype.

## Build

```sh
make            # builds ./st (requires X11, Xft, fontconfig, freetype2 via pkg-config)
make clean      # remove st and *.o
make clean install   # install to /usr/local and compile terminfo (tic -sx st.info)
./st            # run the freshly built binary for manual testing
```

There is no test suite or linter. Key handling can be verified headlessly: start `Xvfb :99`, run `DISPLAY=:99 ./st -e sh -c 'printf "\033[>4;2m"; stty raw -echo; exec cat > out'`, inject keys with `xdotool key ctrl+semicolon` and inspect the bytes in `out`. The same approach works with `emacs -nw` or `vim` recording keys via `read-key-sequence-vector` / `getcharstr()`.

### config.h vs config.def.h

- `config.def.h` is the tracked source of truth for all configuration (fonts, colors, shortcuts, key tables).
- `config.h` is gitignored and is only created from `config.def.h` by `make` **if it does not exist**. After editing `config.def.h`, run `cp config.def.h config.h` (or delete `config.h`) before rebuilding, otherwise the change is silently not compiled in. Keep the two identical.
- `config.h` is `#include`d only by `x.c` and defines `static` data, not just macros. The non-static globals in it that `st.c` needs are declared `extern` in `st.h`.

## Architecture

- **`st.c`** — terminal core, independent of X: PTY creation/fork/exec (`ttynew`, `execsh`, `sigchld`), reading/writing the tty, the escape-sequence parser (`tputc` → CSI/STR/ESC handlers like `csihandle`, `strhandle`), the screen model (`Term` with `Line`/`Glyph` arrays, alt screen, dirty-line tracking), and selection logic.
- **`x.c`** — X11 frontend: window/font setup (`xinit`, Xft font cache), rendering (`xdrawline`, `xdrawglyphfontspecs`, `xdrawcursor`), event loop (`run`) and event handlers, keyboard input, mouse reporting, clipboard.
- **`st.h`** — shared types (`Glyph`, `Arg`, attribute/selection enums) and the API `st.c` exports to the frontend; also `extern`s for config globals defined in `config.h`.
- **`win.h`** — the API the frontend exports back to `st.c` (`x*` functions, `win_mode` flags).
- **`st.info`** — terminfo entry; update it if adding new terminal capabilities.

## Keyboard input

`kpress()` tries, in order: `shortcuts[]` → `kmodother()` → `kmodfunc()` → `kmap()` over `key[]` → the text from `XLookupString` (with Alt as an ESC prefix).

- **`kmodfunc()`**: any cursor/editing/F1–F12 key (and keypad equivalents, table `modfuncs[]`) pressed with Shift/Alt/Ctrl/Super is always sent xterm-style, `CSI 1;<mod>X` or `CSI n;<mod>~`, with mod = 1 + shift 1 | alt 2 | ctrl 4 | super(Mod4) 8. Modified rows for these keys were therefore removed from `key[]`; it only holds the unmodified/legacy sequences (incl. application cursor/keypad variants).
- **`kmodother()`** implements xterm *modifyOtherKeys* in the `CSI <code>;<mod>u` format. It is off until an application sends `CSI > 4 ; N m` (handled in `csihandle()`, stored via `xsetmodkeys()`, queried with `CSI ? 4 m`, reset by RIS). Level 1 only encodes combos the legacy encoding cannot express (Super, Ctrl+Shift+letter, Ctrl+digit/punctuation, modified Return/Tab/BackSpace, Alt+`[`/`O`); level 2 encodes every modified key except Shift alone on printables (Shift+Tab becomes `CSI 9;2u`; below level 2 it is `CSI Z`). tmux with `extended-keys` requests level 2 from st and misparses a plain `CSI Z` in that mode.
- Emacs (`term/st.el`) requests level 1 by itself; Vim needs `keyprotocol` to include `st:mok2`. Plain shells never enable it, so they get the legacy bytes (e.g. Ctrl+\ stays 0x1c).
- `csiparse()` stores the private-parameter prefix (`?` or `>`) in `csiescseq.priv`; `>` sequences other than `CSI > 4 ; N m` are rejected before the main switch so they never reach the `?` handlers.

## Fork-specific changes (vs upstream)

- **Centered content / no resize increments** (`x.c`): `win.hborderpx`/`win.vborderpx` are computed in `cresize()` to center the cell grid in the window, and WM size hints use increment 1. All pixel↔cell conversions must use `win.hborderpx`/`win.vborderpx`, not `borderpx` directly.
- **Keyboard**: see above. modifyOtherKeys is new; upstream only had a partial, hand-written table of modified function keys (with some st-specific sequences such as Shift+Home → `\033[2J`).
- **No keyboard shortcuts; clipboard and zoom are on the mouse** (`config.def.h`, `x.c`): `shortcuts[]` is empty so every key combination reaches the application. `mousesel()` copies every completed selection to CLIPBOARD as well as PRIMARY (the two are treated as one); Shift+Button3 pastes CLIPBOARD, Ctrl+Shift+wheel zooms, Ctrl+Shift+Button2 resets the zoom. There is no middle-click paste. Because `forcemousemod` is Shift, these work even when an application (tmux, Vim, Emacs) has enabled mouse reporting. `mshortcuts[]` is matched in order and Button2 shortcuts fire on release, so keep the Ctrl+Shift entries before the generic wheel entries.
- **Appearance** (`config.def.h`): own color scheme (`colorname[0..15]`, background/foreground at 256/257), font, `borderpx = 0`, `worddelimiters`, `minlatency`, mouse shape.
- `config.mk` adds `-O2`.
