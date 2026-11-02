# radio-hil

AI generated command line client for the radio-hil hardware-in-the-loop setup: RIOT
development boards connected to one or more Raspberry Pis. You build firmware
on your own machine, and `radio-hil` flashes it, opens the board's shell and
switches boards on, off or resets them.

## Requirements

- Python 3.8 or newer, `ssh`, `scp` and `make`
- an account on the Pis with your SSH key (ask the admin, send them your
  **public** key, e.g. `~/.ssh/id_ed25519.pub`)
- for `build-flash`: a local RIOT checkout with the toolchains for your boards

## Setup

Add one entry per Pi to `~/.ssh/config`. The trailing number of the host
name is the number the boards of that Pi get:

```
Host radio-hil-1
    HostName <address of the first Pi>
    User <your user>

Host radio-hil-2
    HostName <address of the second Pi>
    User <your user>
```

Check with `ssh radio-hil-1 true`. Other host names can be set with
`RADIO_HIL_HOSTS="host1 host2"`.

## Install

With [pipx](https://pipx.pypa.io) (`sudo pacman -S python-pipx`,
`sudo apt install pipx`):

```sh
pipx install git+https://github.com/Stopkaa/radio-hil
```

Update to the latest version:

```sh
pipx install --force git+https://github.com/Stopkaa/radio-hil
```

### With conda (or another virtual environment)

```sh
conda activate <env>
pip install git+https://github.com/Stopkaa/radio-hil
```

Update with `pip install --force-reinstall --no-deps git+https://github.com/Stopkaa/radio-hil`.
`radio-hil` is then only available while this environment is active.

### From a local checkout

To test changes before pushing them, install your checkout in editable mode.
Changes in the checkout then take effect immediately:

```sh
pip install -e ~/radio-hil            # in a conda/virtual environment
pipx install --force --editable ~/radio-hil
```

If `which radio-hil` still points to `~/.local/bin/radio-hil` from an older
manual install, delete that file.

### First start

Switch the boards on (they may have been switched off to save power or after
a board was added) and get the board list:

```sh
radio-hil poweron -a
radio-hil fetch
```

### Tab completion

Completes commands, options, board names, Pis and firmware files. Add to
`~/.bashrc`:

```sh
eval "$(register-python-argcomplete radio-hil)"
```

or to `~/.zshrc` (after `compinit`, oh-my-zsh already does that):

```sh
eval "$(register-python-argcomplete --shell zsh radio-hil)"
```

With conda, the line must come after the `conda activate` of the environment
that has `radio-hil`, otherwise `register-python-argcomplete` is not found.
In an open terminal, run `source ~/.bashrc` (or `~/.zshrc`) once.

Board names come from the list saved by `radio-hil fetch`, so run it again
after boards were added.

## Usage

```sh
radio-hil fetch                  # get the boards of all Pis
radio-hil list                   # show the boards of each Pi
```

Build and flash (run from your RIOT checkout):

```sh
radio-hil build-flash -b nrf52840dk-1 samr21-xpro-2 -f examples/networking/gnrc/networking
radio-hil flash -b nrf52840dk-1 -f bin/nrf52840dk/app.elf
```

RIOT shell of a board (tmux, close with `Ctrl+B` then `&`):

```sh
radio-hil term nrf52840dk-1
```

Raw serial connection for scripts (no tmux, no lock), e.g. for `riotctrl`.
In a terminal, leave it with `Ctrl+C`:

```sh
radio-hil serial nrf52840dk-1
```

Reset and power:

```sh
radio-hil reset -b nrf52840dk-1  # power off and on, like the reset button
radio-hil reset -u               # all boards currently in use
radio-hil poweroff -a            # all boards off
radio-hil poweron -b frdm-kw41z-1
radio-hil poweroff -p radio-hil-2 # all boards of one Pi (or -p 2)
```

`flash`, `build-flash`, `poweron`, `poweroff` and `reset` work on all given boards at the
same time. With several boards, every output line starts with `[board name]`.

`radio-hil <command> -h` shows the options of each command.

## Interop test

[`tests/interop`](tests/interop/README.md) lets every IEEE 802.15.4 radio in
the HIL ping every other one, in both directions:

```sh
python tests/interop/interop_test.py --riotbase ~/RIOT --dut nrf52840dk
```

## 802.15.4 sniffer in Wireshark

STM32WB55 USB dongles running ST's sniffer firmware hang on the Pis.
`dist/radiohil_sniffer.py` is a Wireshark extcap plugin: every dongle on
every Pi shows up as its own capture interface (e.g. "radio-hil-1 STM
sniffer 206138793036"), the channel is set via the gear icon next to it.
It only needs Python 3 on the laptop and the `~/.ssh/config` entries from
[Setup](#setup).

```sh
mkdir -p ~/.local/lib/wireshark/extcap
cp dist/radiohil_sniffer.py ~/.local/lib/wireshark/extcap/
chmod +x ~/.local/lib/wireshark/extcap/radiohil_sniffer.py
~/.local/lib/wireshark/extcap/radiohil_sniffer.py --extcap-interfaces   # lists the dongles
```

Then restart Wireshark (or Capture → Refresh Interfaces). Several dongles
can be captured at once by selecting several interfaces. Each dongle can
only be used by one person at a time, agree with the others. Stopping the
capture in Wireshark also stops it on the Pi.

## Board names

A board is called `<RIOT board>-<n>`, where `n` is the number of the Pi it is
connected to: the `nrf52840dk` at `radio-hil-1` is `nrf52840dk-1`.

If a Pi has a board type more than once, the boards get a letter in the order
of their USB port: `nrf52840dk-1a`, `nrf52840dk-1b`. A board keeps its name as
long as it stays in the same port. Boards without a serial number (OpenMote-B)
cannot be told apart, so only one of them per Pi.

## Good to know

- **Locking**: `term`, `flash` and `build-flash` lock the board, `serial` does not. If someone
  else is using it, flashing is refused. `reset`, `poweron` and `poweroff` do **not** check
  locks, so make sure nobody else is working on the board.
- **Firmware format**: `flash` needs the file the board's flasher expects,
  e.g. `.elf` for `nrf52840dk`, `.bin` for `samr21-xpro`. `build-flash` picks
  the right one automatically.
- **RIOT versions**: you build with your local RIOT, but flashing and the
  shell use the RIOT checkout on the Pis. Keep both roughly in sync.
- **After a reset** the board's serial port disappears for a moment. If the
  shell does not come back, close `term` and open it again.
