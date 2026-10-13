# dwmstatus

## Overview

dwmstatus is a small X11 program that writes a one-line status into the root window name (`WM_NAME`). dwm reads that name and shows it in the bar.

This repository is a configured copy of that program, version 1.0, copyright 2011-2018 Christoph Lohmann. The shipped status line is load average, the battery at `/sys/class/power_supply/BAT1`, local time in `Europe/Copenhagen`, and UTC. The program builds and runs. It has no automated tests and no CI. `dwmstatus-temperature.c` and `new-acpi-battery.c` are unbuilt snippets.

## Repository layout

| Path | Role |
| --- | --- |
| `dwmstatus.c` | The program. The only C file the Makefile compiles. |
| `config.mk` | Program name, version, prefix, compiler, and X11 flags. |
| `Makefile` | Targets `all`, `options`, `clean`, `dist`, `install`, and `uninstall`. |
| `dwmstatus-temperature.c` | Snippet of `gettemperature`. Not linked into the binary. |
| `new-acpi-battery.c` | Alternate battery-reader snippet. Not linked into the binary. |
| `LICENSE` | MIT/X Consortium License. |

`config.mk` sets `NAME = dwmstatus` and `VERSION = 1.0`. The Makefile compiles `${NAME}.c` and nothing else. `MANPREFIX` is defined and unused. No manual page is installed.

## Setup

Prerequisites:

- `make` and a C99 compiler on `cc`.
- X11 headers and `libX11`.
- A running X server to run the program. Compiling does not need a display.
- Linux sysfs for a battery reading. The path in `main` is `/sys/class/power_supply/BAT1`.

On Debian or Ubuntu:

```
sudo apt-get install build-essential libx11-dev
```

`make` completed successfully in this tree with those packages installed. `cc` was Clang 18. `config.mk` still sets `X11INC` and `X11LIB` to `/usr/X11R6`, and those directories were absent. The build still found headers via `-I/usr/include` and `libX11` on the default linker path.

## Build, run and test

Build `./dwmstatus`:

```
make
```

Print `CFLAGS`, `LDFLAGS`, and `CC` without compiling:

```
make options
```

Run it on the current display:

```
./dwmstatus
```

The loop body runs, then `sleep(60)` runs, and that repeats. The first update is immediate. The process stays in that loop until it receives a signal such as SIGINT or SIGTERM.

Read the string it wrote:

```
xprop -root WM_NAME
```

A run against a display with an empty `/sys/class/power_supply` produced:

```
WM_NAME(STRING) = "L: 0.00 0.02 0.00 B:  T: KW 41 Tue 13 Oct 20:19 CEST 2026 U:18:19"
```

With the display cleared, `env -u DISPLAY ./dwmstatus` exits 1 and prints `dwmstatus: cannot open display.`

Install and remove. The default `PREFIX` is `/usr`, so the binary is copied to `/usr/bin`. `DESTDIR` is prepended to that path. Both of these were run against a temporary `DESTDIR`:

```
make install DESTDIR=/tmp/dwmstatus-root PREFIX=/usr
make uninstall DESTDIR=/tmp/dwmstatus-root PREFIX=/usr
```

Source tarball:

```
make dist
```

This writes `dwmstatus-1.0.tar.gz` containing `Makefile`, `config.mk`, `dwmstatus.c`, and `LICENSE`. The `dist` target depends on `clean`, so a local binary is removed first.

Remove build output:

```
make clean
```

This deletes `dwmstatus`, `dwmstatus.o`, and `dwmstatus-1.0.tar.gz`.

There is no test suite. `make test` stops with `No rule to make target 'test'`. The check before a commit is `make`, which must exit 0.

## Code style and conventions

- C99, with the active flags `-std=c99 -pedantic -Wall`. `config.mk` selects a debug build (`-g -O0`). The `-Os` CFLAGS and the stripped LDFLAGS are comments.
- C sources use two-space indentation, with the opening brace on the same line as the function or statement.
- The program is one translation unit. It has no project header.
- Status behaviour (timezones, battery path, format strings, sleep interval) is in `dwmstatus.c`. Build and install settings are in `config.mk`.
- The Makefile includes `config.mk`, prints a short label per recipe, and declares `.PHONY: all options clean dist install uninstall`.
- `dwmstatus-temperature.c` and `new-acpi-battery.c` are fragments. `cc -std=c99 -Wall -c` fails on each of them. Leave them out of the Makefile unless the status line is meant to change.

## CI and tooling

Not applicable for continuous integration. The repository has no `.github/workflows` directory and no other CI configuration.

Local tooling is `make` plus `config.mk`. The compiler command is `CC = cc`.

## Working rules for agents

- Keep changes small and focused.
- Run the tests before committing. This repository has no test target, so run `make` and require exit 0. Leave the binary, object files, and `dwmstatus-1.0.tar.gz` untracked. There is no `.gitignore`.
- Never commit secrets.
- Update the docs when behaviour changes, including `README.md` when the change is user-visible, and this file when agent instructions change.
- Use conventional commit messages.
- Keep `AGENTS.md`, `CLAUDE.md`, and `.github/copilot-instructions.md` in sync. `AGENTS.md` is the source of truth. The other two copy it and may add only the one-line canonical-source header.

## Gotchas

- The status format string is `"L: %s B: %s T: %s U:%s"`. The UTC field is introduced by `U:` with no space before the time. `L` is the 1, 5, and 15 minute load averages from `getloadavg`. `B` is the battery. `T` is local time. `U` is UTC as `%H:%M`.
- Local time uses the global `tzlocal` (`Europe/Copenhagen`) and `"KW %W %a %d %b %H:%M %Z %Y"`. `KW` is literal text, and `%W` is the week number. UTC uses the global `tzutc` (`UTC`).
- The battery directory is hardcoded to `/sys/class/power_supply/BAT1`. A missing directory yields an empty battery field, which is the `B:` in the example above. A host that exposes `BAT0` shows that same empty field until the path is edited and the program is rebuilt.
- The percentage is remaining charge divided by design capacity: `charge_now` or `energy_now`, over `charge_full_design` or `energy_full_design`. It does not use `charge_full` or `energy_full`.
- A missing `present`, design, or remaining file becomes an empty battery field. `present` other than `1` becomes `not present`. Unparsed capacities become `invalid`. A readable battery is `%.0f` plus `+` for `Charging`, `-` for `Discharging`, or `?` otherwise, such as `76%-`.
- `getbattery` does not test the `status` read. If that file is missing, `strncmp` is called with a null pointer. The `status` allocation is also never freed.
- `gettemperature` is compiled in and never called, so the bar has no temperature. It reads a sysfs millidegree value and formats `°C`. The snippet file shows the call `gettemperature("/sys/class/hwmon/hwmon0/device", "temp1_input")`.
- `main` takes no arguments. `for (;; sleep(60))` never reaches `XCloseDisplay`. Configuration is a source edit followed by `make`.
- `settz` only calls `setenv("TZ", ...)`. It does not call `tzset`.
- `make dist` omits the two snippet files, and it deletes `./dwmstatus` because `dist` depends on `clean`.
- `X11INC` is `/usr/X11R6/include` and `X11LIB` is `/usr/X11R6/lib`. Point those at the local X11 directories when the compiler or linker cannot find X11.
- The compiler is passed `-DVERSION=\"1.0\"`, and the source never reads `VERSION`.
