# Interop test

Checks that the IEEE 802.15.4 radios in the HIL work with each other. Every
pair of radios pings in both directions, like
[Release-Spec 04](https://github.com/RIOT-OS/Release-Specs/blob/master/04-single-hop-6lowpan-icmp/test_spec04.py).
Which pings run is set in `radios.yaml` (see [Ping runs](#ping-runs)), by
default:

| Ping run | Shell command | Duration |
|---|---|---|
| `short` (unfragmented) | `ping <addr> -s 10 -c 10000 -i 30` | ~5 min |
| `long` (fragmented) | `ping <addr> -s 1024 -c 10000 -i 300` | ~50 min |

`long` only runs if `short` passed in that direction. A ping run passes if
the packet loss is below 10 % and the packet buffer (`pktbuf`) of both boards
is empty 10 s after the ping. If a shell does not answer anymore, it is a
`hang`.

The script runs on your laptop. It builds and powers the boards with
`radio-hil` and talks to the RIOT shells with `riotctrl` over
`radio-hil serial`.

## Setup

1. `radio-hil` installed and working (`radio-hil fetch` shows the boards).
2. `socat` on all Pis: `sudo pacman -S socat`.
3. In the same Python environment as `radio-hil`:

   ```sh
   pip install -r tests/interop/requirements.txt
   ```

   `riotctrl_shell` is not installed, the script takes it from
   `<riotbase>/dist/pythonlibs`.
4. A RIOT checkout and its toolchains for all boards in `radios.yaml`.

## Run

From your RIOT checkout (or give it with `--riotbase`):

```sh
cd ~/RIOT
python ~/radio-hil/tests/interop/interop_test.py --dry-run           # only show the plan
python ~/radio-hil/tests/interop/interop_test.py --count 100 --radios nrf52840dk samr21-xpro
python ~/radio-hil/tests/interop/interop_test.py --pings short      # only the short ping
python ~/radio-hil/tests/interop/interop_test.py --dut nrf52840dk    # only pairs with the nrf52840dk
python ~/radio-hil/tests/interop/interop_test.py                     # everyone against everyone
```

A full run takes many hours, so start it in `tmux` (or with `nohup`) and keep
the laptop from going to sleep.

| Option | |
|---|---|
| `--riotbase DIR` | RIOT checkout to build from (default: current folder) |
| `--dut RADIO` | only the pairs with this radio |
| `--radios R1 R2 ...` | only these radios |
| `--pings NAME ...` | only these ping runs from `radios.yaml`, e.g. `--pings short` |
| `--count N` | pings per ping run (instead of `count`), for a quick check |
| `--no-parallel` | one run at a time |
| `--boards-per-pi N` | at most N boards on at once per Pi (default: 4) |
| `--results DIR` | where the result folders go (default: `interop-results`) |
| `--resume DIR` | continue an aborted test, see below |
| `--skip-build` | use the firmware of an earlier build of the same RIOT commit |
| `--skip-flash` | no initial flash, the boards already have the firmware |
| `--radios-file FILE` | another radio file (default: `radios.yaml` next to the script) |

## What happens

1. **Build** `examples/networking/gnrc/networking` for every radio, with
   `USEMODULE=shell_cmd_gnrc_pktbuf` and the radio's `make` variables. Every
   radio gets its own folder: `<riotbase>/build/interop/<commit>/<radio>/`.
   So a run on another RIOT version (e.g. `--riotbase ~/RIOT-master`) does not
   touch these builds.
2. **Initial flash**: on every Pi, up to `--boards-per-pi` boards at a time
   are switched on, flashed and switched off again (the Pis at the same
   time). With too many boards on, the USB controller of the Pi runs out of
   bandwidth ("Not enough bandwidth" in `dmesg`) and boards disappear. A board that fails is power cycled and flashed once more; if that
   fails too, it is tried again when a run needs it. A board that is more
   than one radio (OpenMote-B) gets the first one in `radios.yaml`; the other
   one is flashed when a run needs it.
3. **Runs**: one run is one direction of a pair. Only the two boards of a run
   are on. The run sets the channel on both boards, pings `short`, then
   `long`, checks `pktbuf` after each, and switches the boards off again. That
   also resets them, there are no retries.

Runs that share no board run at the same time, one per channel and with
at most `--boards-per-pi` boards on per Pi: 2.4 GHz on
channels 15, 20 and 26 (between the WLAN channels 1/6/11), sub-GHz on channel
0 (868.3 MHz, the only O-QPSK channel in the EU). If a radio has two
exemplars, the two directions of a pair can run at the same time on swapped
boards. A radio with two exemplars is also tested against itself, in one
direction only (both ends are the same radio).

## Results

Every test gets a folder `interop-results/<date>-<time>-<commit>/`:

| File | |
|---|---|
| `summary.md` | table of all runs: sender, receiver, band, channel, ping run, result, loss, RTT, pktbuf |
| `results.jsonl` | one line per run and ping run with all values |
| `plan.json` | commit, options, boards and the planned runs |
| `flashed.json` | which radio firmware is on which board |
| `logs/build-<radio>.log` | build output |
| `logs/<n>-<sender>-to-<receiver>-<board>.log` | full shell output of both boards of a run |
| `logs/flash-<board>.log` | output of the initial flash |

Results: `passed`, `failed` (loss >= 10 % or pktbuf not empty), `hang`
(a shell did not answer), `error` (e.g. no 802.15.4 interface found),
`skipped` (the ping run before did not pass). The script exits with 0 only if
everything passed.

**Resume:** results are saved after every ping run. After `Ctrl+C` (all boards
are switched off) or a crash, run the same command again with
`--resume <result folder>`; finished runs are skipped. With `--skip-build
--skip-flash` it starts right away.

## Ping runs

The `test` section of `radios.yaml` has a list `pings`: one entry per ping
run, done one after another in each direction. Add, remove or change entries
as you like:

```yaml
test:
  pings:
    - name: short        # name in the report and for --pings
      size: 10           # ping -s, payload in bytes
      count: 10000       # ping -c
      interval: 30       # ping -i, ms between pings
    - name: long
      size: 1024
      count: 10000
      interval: 300
      max_loss: 5        # optional, only for this ping run
  ping_timeout: 1000     # ms, ping -W (per ping run: timeout)
  max_loss: 10           # percent, a ping run passes below this
  pktbuf_wait: 10        # s after a ping before pktbuf is checked
  stop_on_fail: true     # next ping run only if this one passed
```

`--pings short` runs only some of them (always in the order of the file),
`--count 100` changes `count` of all of them.

## Adding a radio

Add it to `radios.yaml`, the pairs follow from its properties:

```yaml
radios:
  my-radio:
    board: nucleo-f767zi           # RIOT board, as in 'radio-hil list'
    phys: [oqpsk-2.4]              # oqpsk-2.4 and/or oqpsk-subghz
    make: {USEMODULE: at86rf233}   # optional build variables
```

`make` variables are passed as environment variables, so they add to what
the board sets (`USEMODULE`, `CFLAGS`, `DISABLE_MODULE`, ...). Pairs that
should not run go into `exclude`, e.g. `exclude: [[frdm-kw41z, mrf24j40]]`.
The ping parameters are in the `test` section, see [Ping runs](#ping-runs).
