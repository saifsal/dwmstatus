# dwmstatus

dwmstatus writes a one-line status into the X root window name. [dwm](https://dwm.suckless.org/) reads that property and shows it in the bar. This build fills the bar with load average, battery charge, local time, and UTC.

This repository is a configured copy of dwmstatus, version 1.0. The line it sets is:

```
L: <load> B: <battery> T: <local time> U:<UTC>
```

## Features

- Sets `WM_NAME` on the default root window with `XStoreName`, then repeats.
- Load averages for 1, 5, and 15 minutes (`getloadavg`).
- Battery percentage and charge state from Linux sysfs, path `/sys/class/power_supply/BAT1`. It accepts either the `charge_*` or the `energy_*` files.
- Local time in `Europe/Copenhagen`, including the week number, and a separate UTC clock.
- First update as soon as the process starts, then every 60 seconds.
- `make install` / `make uninstall`, and a `make dist` source tarball.

A `gettemperature` helper is in the source and is not called, so temperature is not on the bar. `dwmstatus-temperature.c` and `new-acpi-battery.c` are leftover snippets. The Makefile does not build them.

## Example

On a display with no battery under `/sys/class/power_supply`, `xprop -root WM_NAME` showed:

```
WM_NAME(STRING) = "L: 0.00 0.02 0.00 B:  T: KW 41 Tue 13 Oct 20:19 CEST 2026 U:18:19"
```

| Field | Meaning |
| --- | --- |
| `L` | Load averages, printed as `0.00 0.02 0.00`. |
| `B` | Battery. Empty when `BAT1` is missing. A discharging pack looks like `76%-`, a charging pack like `76%+`, and any other status like `76%?`. `not present` and `invalid` are the other texts. |
| `T` | Local time: literal `KW`, week number, weekday, day, month, `HH:MM`, timezone, year. The example is `Europe/Copenhagen` in summer (`CEST`). |
| `U` | UTC `HH:MM`. The format string has no space after `U:`. |

The percentage is remaining charge divided by **design** capacity (`charge_full_design` or `energy_full_design`), so a worn battery can sit below 100% when it reports full.

## Prerequisites

- `make` and a C99 compiler installed as `cc`.
- X11 headers and `libX11`.
- An X server to run the program.
- Linux sysfs if the battery field should be filled. With no `BAT1` directory the program still runs and leaves that field empty.

Debian and Ubuntu:

```
sudo apt-get install build-essential libx11-dev
```

That set is enough to compile this tree. `config.mk` looks for X11 under `/usr/X11R6`, which current distributions often do not ship. The build still succeeds when headers are in `/usr/include` and `libX11` is on the linker path. If the compile or link fails, set `X11INC` and `X11LIB` in `config.mk`.

## Install

From the repository root:

```
make
sudo make install
```

`PREFIX` defaults to `/usr`, so the executable is `/usr/bin/dwmstatus`. A user-local prefix, or a staged directory, works the same way:

```
make install PREFIX=/usr/local
make install DESTDIR=/tmp/dwmstatus-root PREFIX=/usr
```

Remove it with the same `PREFIX` and `DESTDIR`:

```
sudo make uninstall
```

`make dist` writes `dwmstatus-1.0.tar.gz` (`Makefile`, `config.mk`, `dwmstatus.c`, `LICENSE`). It cleans the tree first.

## Usage

Start it in the background before dwm, for example in `.xinitrc`:

```
dwmstatus &
exec dwm
```

Or run `./dwmstatus` from a built tree. The process keeps running. Stop it with Ctrl-C or by killing it.

If it cannot open the display it exits 1 and prints:

```
dwmstatus: cannot open display.
```

There are no command-line options. The clocks, battery path, text, and interval are compiled in.

## Configuration

Edit `dwmstatus.c` and rebuild.

| What | Where |
| --- | --- |
| Local timezone | `tzlocal`, default `Europe/Copenhagen` |
| UTC timezone name | `tzutc`, default `UTC` |
| Local format | `mktimes("KW %W %a %d %b %H:%M %Z %Y", tzlocal)` in `main` |
| UTC format | `mktimes("%H:%M", tzutc)` in `main` |
| Battery sysfs directory | `getbattery("/sys/class/power_supply/BAT1")` in `main` |
| Field order | the `smprintf` format in `main` |
| Interval | `sleep(60)` on the main loop |

Edit `config.mk` for `PREFIX`, `CC`, `CFLAGS`, `LDFLAGS`, `X11INC`, and `X11LIB`. The active flags are a debug build (`-g -std=c99 -pedantic -Wall -O0`). Optimized flags are present as comments.

## Development and testing

```
make
make options
make clean
```

`make` compiles `dwmstatus.c` to `./dwmstatus`. `make options` prints the flags. `make clean` removes the binary, `dwmstatus.o`, and the dist tarball.

There is no automated test suite, and `make test` is not a target. Check a change by building with `make`, running `./dwmstatus` on an X display, and reading the bar text with:

```
xprop -root WM_NAME
```

Install and uninstall were checked with `DESTDIR=/tmp/dwmstatus-root PREFIX=/usr`. A no-display run was checked with `env -u DISPLAY ./dwmstatus`.

## Project status

Version 1.0, from `config.mk`. The tree is a personal configuration of dwmstatus: the status fields and timezones above are what it ships. The C sources were reformatted after the initial import.

The repository does not publish a roadmap. Temperature stays off the status line unless `gettemperature` is wired into `main`.

## Licence

MIT/X Consortium License. Copyright © 2011-2018 Christoph Lohmann <20h@r-36.net>. See [LICENSE](LICENSE). The program header reads "Copy me if you can. by 20h".
