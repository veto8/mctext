# AGENTS.md — mctext

## What this is
A local-first desktop text vault built with Common Lisp, McCLIM and SBCL. It stores reusable snippets in groups, shows them as clickable presentations, and copies the selected text to the system clipboard.

The app has a main McCLIM frame with groups, snippets, an editable text pane, buttons and status output, plus a second `clipboard-log-frame` window that lists text seen on the desktop clipboard during the current session. It also has a no-window command line mode.

## Stack
- Common Lisp (SBCL)
- McCLIM via Quicklisp
- CLX on Linux/FreeBSD, SDL2 on macOS and Windows by default (no X server needed on macOS)
- `cl-json` for the local document store
- `bordeaux-threads` for the clipboard watcher
- `anathema` and `anathema-doom` from Quicklisp for the dark, light and high-contrast themes
- No network code in the app

## Structure
- `mctext.asd` — ASDF system; depends on `mcclim`, `cl-json`, `bordeaux-threads`, `anathema` and `anathema-doom`
- `main.lisp` — launcher; loads Quicklisp, selects backend, quickloads `mctext`, activates the Anathema dark theme, handles bundled font, calls `mctext:main`
- `dev.lisp` — development REPL loader with `dev-start`, `dev-reload`, `dev-force-reload`, `dev-watch-start` and `dev-watch-stop`
- `src/package.lisp` — `MCTEXT` package
- `src/model.lisp` — group and snippet structs plus filter helpers
- `src/store.lisp` — JSON document store with `id`, `_rev` and `doc`
- `src/clipboard-log.lisp` — session-only clipboard log, watcher thread and reader commands
- `src/clipboard.lisp` — clipboard shims for wl-copy/xclip/xsel, pbcopy and PowerShell
- `src/theme.lisp` — Anathema theme definitions, activation and theme status
- `src/gui.lisp` — main frame, `prompt-frame` and `clipboard-log-frame`, panes, presentation types, commands and translators
- `src/cli.lisp` — `--list`, `--copy`, `--help` and no-window dispatch
- `sdl2-runtime.lisp` — SDL2 runtime fix; loaded only for SDL2 builds, resets the user-event id, centers managed frames, moves McCLIM's SDL2 loop to the process main thread on macOS SBCL, and destroys the port and closes cl-sdl2's main-thread channel when the app exits
- `sdl2.sh` — pins McCLIM 1.0.0 for SDL2 builds, registers a pinned `cl-sdl2` local project with the Apple Silicon spec on macOS ARM, and downloads `SDL2.dll` on Windows
- `ask.sh` — task runner menu grouped by OS: MAC, LINUX, WINDOWS, then DEVELOPMENT and MAINTENANCE; packaging lives under MAC
- `ask.d/lib.sh` — shared task helpers
- `ask.d/setup-macos.sh` — task 1: macOS setup (SBCL/Quicklisp, SDL2/SDL3 and local project)
- `ask.d/build-mac.sh` — task 2: build the macOS SDL2 executable
- `ask.d/dev-test-mac.sh` — task 3: macOS SDL2 dev test (`-nw` smoke test, then GUI)
- `ask.d/dmg.sh` — task 4: build a macOS `.app` and DMG
- `ask.d/setup-linux.sh` — task 5: Linux/FreeBSD setup (SBCL/Quicklisp and local project)
- `ask.d/build-linux.sh` — task 6: build the Linux CLX executable
- `ask.d/dev-test-linux.sh` — task 7: Linux CLX dev test (`-nw` smoke test, then GUI)
- `ask.d/setup-windows.sh` — task 8: Windows setup (SBCL/Quicklisp, SDL2.dll and local project)
- `ask.d/build-windows.sh` — task 9: build the Windows SDL2 executable
- `ask.d/dev-test-windows.sh` — task 10: Windows SDL2 dev test (`-nw` smoke test, then GUI)
- `ask.d/dev.sh` — task 11: start the GUI and a development REPL with automatic reload on source changes
- `ask.d/run.sh` — task 12: run `sbcl --script main.lisp`
- `ask.d/clean.sh` — task 13: remove build outputs
- `ask.d/build.sh` — shared executable builder used by tasks 2, 6, 9 and CI; hidden from the menu
- `ask.d/dev-test.sh` — shared dev-test helper; hidden from the menu
- `README.md` — project title, logo and commands
- `mctext.svg` — README logo
- `pages/public/` — GitHub Pages site
- `.github/workflows/` — build and release workflows

## Run
```sh
./ask.sh
```

Direct source run:

```sh
sbcl --script main.lisp
```

Built executable:

```sh
./dist/mctext/mctext
./dist/mctext/mctext -- --list
./dist/mctext/mctext -- --copy "Email reply"
```

The user builds and runs all Lisp code. Do not compile or run Lisp code in the agent environment.

## Patterns
- Frame names are `mctext-frame`, `prompt-frame` and `clipboard-log-frame`; each has a command table of the same name.
- Name prompts use `prompt-frame`: one white, single-line text field on gray with default McCLIM Save and Close buttons.
- Both frames use `:reinitialize-frames t` so reloading `src/gui.lisp` rebuilds panes and layouts in running windows.
- Menu bar is `mctext-menu-bar` with File, Edit and Help menus; the log window is opened from the button row.
- Dark, Light and High contrast remain frame commands; startup uses Anathema's Doom One dark theme.
- Presentation types are `mctext-group`, `mctext-snippet` and `clipboard-entry`.
- Left click selects; right click opens commands from `:gesture :menu` translators; left-clicking a log entry saves it to the `Clipboard` group.
- Clipboard copy uses external OS tools; McText does not inject keystrokes into other apps.
- The watcher polls `wl-paste`, X11 CLIPBOARD/PRIMARY via `xclip`/`xsel`, `pbpaste` and PowerShell, deduplicates text, and ignores McText's own writes.
- The log exists only in memory for one GUI session; only clicked entries are written to the JSON store. Wayland needs a compositor/portal helper for reliable background watching.
- Store path is `~/.local/share/mctext/mctext.json` on POSIX and `%APPDATA%\McText\mctext.json` on Windows.
- On macOS SDL2 the SDL loop stays on the process main thread while the McText frame runs in a worker thread; without both halves SDL initializes but no native window appears. When the frame exits, `quit-sdl2-port` destroys the SDL port and calls `sdl2:quit` to close cl-sdl2's main-thread channel; otherwise the process stays alive with no window. Only managed frames are centered; unmanaged menu/popup sheets keep McCLIM's pointer-based placement.
- The macOS builder copies the Homebrew SDL dylibs directly and rewrites them to `@rpath`; SDL is loaded at runtime, so it never appears in the saved executable's `otool -L` output.
- `MCTEXT_BACKEND` overrides backend; `MCCLIM_BACKEND` is accepted as a fallback.
- CI workflows register the checkout in `~/quicklisp/local-projects/mctext` before `ql:quickload :mctext`; the macOS workflows do this through `ask.d/setup-macos.sh`, build through `ask.d/build-mac.sh`, install SDL2/SDL3, set `MCTEXT_BACKEND=sdl2`, bundle `libSDL*.dylib` with `libzstd*.dylib`, and use the macOS main-thread shim.

## Conventions
- Keep the README logo as its first line.
- No comments in code unless asked.
- Do not add networking or telemetry.
- Preserve the document store shape: `docs` entries contain `id`, `_rev` and `doc`.
- Keep the clipboard log session-only; persist an entry only when the user clicks it.
- Start with the dark theme; theme commands switch themes at runtime and repaint the open frames.
- Apply workspace permissions after edits: `chown -R 1000:33 <project>` and `chmod -R g+w <project>`.
