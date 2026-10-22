# pico-usb-debug-jtag

A Raspberry Pi Pico carrier board that turns one Pico into a **level-shifted,
dual-channel JTAG programmer plus dual UART bridge**, intended as the
programming/debug cable for the FPGA boards used by
[`jop-spinalhdl`](../jop-spinalhdl).

Design files in this directory (EasyEDA Pro export, 2026-08-26):

| File | Contents |
|------|----------|
| `SCH_pico-usb-debug-jtag.pdf` | Schematic |
| `Netlist_pico-usb-debug-jtag.enet` | Netlist (JSON) — source of the pin tables below |
| `Gerber_pico-usb-debug-jtag.zip` | Fabrication gerbers |
| `BOM_pico-usb-debug-jtag.csv` / `.ods` | Bill of materials |
| `3D_pico-usb-debug-jtag.step`, `3DShell_…zip` | 3D models / enclosure |
| `ProPrj_pico-usb-debug-jtag.epro2` | EasyEDA Pro project archive |

> **Schematic says Pico 2 (RP2350), breadboard has a Pico 1 (RP2040).**
> The 40-pin header pinout and the 3-pin SWD header are identical between the
> two, so every GPIO mapping below applies unchanged. Only build-time
> `PICO_BOARD` / `PICO_PLATFORM` differ.

---

## Why this board exists

`../pico-usb-blaster` and `../pico-dirtyJtag` both work as bare Picos with
flying leads at 3.3 V, but `jop-spinalhdl` records two hard failures with that
setup:

* **A-E115FB (EP4CE115)** — its JTAG bank is not 3.3 V, so a bare Pico "cannot
  configure it (no level shifter)". It currently needs a genuine/Terasic
  USB-Blaster.
* **pico-usb-blaster** reads IDCODE reliably but has *never* completed a full
  configuration; flying leads were the prime suspect
  (see the CAUTION block in `../pico-usb-blaster/src/blaster_jtag.pio`).

This board addresses both: per-signal `SN74LVC1T45` level shifters referenced
to the target's own VTREF, 33 Ω source termination on every target-side line,
and proper 2.54 mm boxed headers instead of jumper wires.

---

## Block overview

```
                 ┌──────────────────────────── Raspberry Pi Pico ───────────────────────────┐
   USB ──────────┤                                                                          │
                 │  GP17-20 ──▶ U3-U6  (LVC1T45)  ──33R──▶  JTAG0  J5 (2x5) / J6 (1x6)      │
                 │  GP11 ────▶ DIR for JTAG0 TCK/TMS/TDI                                     │
                 │                                                                          │
                 │  GP21,22,26,27 ▶ U7-U10 (LVC1T45) ─33R─▶  JTAG1  J7 (2x5) / J8 (1x6)     │
                 │  GP3 ─────▶ DIR for JTAG1 TCK/TMS/TDI                                     │
                 │                                                                          │
                 │  GP4/5/6   ──▶ UART A  J1 (JST 3-pin, 100R + schmitt on RX)               │
                 │  GP12/13/14──▶ UART B  J2 (JST 3-pin, 100R + schmitt on RX)               │
                 │  GP0/GP1   ──▶ UART C  J4 (JST 3-pin, direct)                             │
                 │  SWCLK/SWDIO ▶ J3  (JST 3-pin — debug the Pico itself)                    │
                 │  GP2,7,8,10,15,16,28 ▶ 7 status LEDs (470R)                               │
                 │  RUN ──────▶ SW4 reset button                                             │
                 └──────────────────────────────────────────────────────────────────────────┘
```

---

## JTAG channel pin map

Both channels are identical in structure. Each of TCK / TMS / TDI goes
Pico → `SN74LVC1T45` A→B → 33 Ω → target. TDO comes back through a shifter
whose `DIR` is **hard-wired to GND**, so it is permanently target→Pico.

### JTAG1 — the channel to bring up first (headers J7 / J8)

| Signal | Pico GPIO | Pico pin | Shifter | Series R | Direction |
|--------|-----------|----------|---------|----------|-----------|
| TDI    | **GP21**  | 27 | U7  | R20 33 Ω | Pico → target |
| TMS    | **GP22**  | 29 | U8  | R19 33 Ω | Pico → target |
| TDO    | **GP26**  | 31 | U9  | R21 33 Ω | target → Pico (DIR tied GND) |
| TCK    | **GP27**  | 32 | U10 | R18 33 Ω | Pico → target |
| **DIR / OE** | **GP3** | 5 | U7/U8/U10 `DIR` | — | drive **high** to enable |

* `GP3` has a 10 kΩ pulldown (R14) → shifters default to *target→Pico*, i.e.
  the target is not driven at power-up.
* `JTAG1_TDO` has a 10 kΩ pulldown to GND (R17) — see *Known hardware notes*.
* Shifter `VCCB` = `JTAG1_VCC_T`, taken from **header pin 4 (VTREF)**.
  Decoupled by C13–C16 (100 nF each).

### JTAG0 — second channel (headers J5 / J6)

| Signal | Pico GPIO | Pico pin | Shifter | Series R | Direction |
|--------|-----------|----------|---------|----------|-----------|
| TDI    | **GP17**  | 22 | U3 | R24 33 Ω | Pico → target |
| TMS    | **GP18**  | 24 | U4 | R23 33 Ω | Pico → target |
| TDO    | **GP19**  | 25 | U5 | R25 33 Ω | target → Pico (DIR tied GND) |
| TCK    | **GP20**  | 26 | U6 | R22 33 Ω | Pico → target |
| **DIR / OE** | **GP11** | 15 | U3/U4/U6 `DIR` | — | drive **high** to enable |

`GP11` pulldown R15 10 kΩ; `JTAG0_TDO` pulldown R16 10 kΩ; `VCCB` =
`JTAG0_VCC_T` from header pin 4; decoupling C5–C8.

### Header pinouts

**J5 / J7 — `BH254V-10P`, 2×5 boxed, standard Altera ByteBlaster / USB-Blaster:**

| Pin | Signal | Pin | Signal |
|-----|--------|-----|--------|
| 1 | TCK    | 2  | GND |
| 3 | TDO    | 4  | **VTREF** (target I/O supply — *must* be present) |
| 5 | TMS    | 6  | n/c |
| 7 | n/c    | 8  | n/c (`nCE` on a real Blaster) |
| 9 | TDI    | 10 | GND |

**J6 / J8 — 1×6 pin header, same nets in parallel:**

| Pin | 1 | 2 | 3 | 4 | 5 | 6 |
|-----|---|---|---|---|---|---|
| Signal | VTREF | GND | TCK | TDO | TDI | TMS |

> The target supplies VTREF on pin 4. Until a target is plugged in the
> B-side of the shifters is unpowered and **nothing is driven** — this is by
> design, not a fault.

---

## Other connectors

| Ref | Type | Pin 1 | Pin 2 | Pin 3 | Notes |
|-----|------|-------|-------|-------|-------|
| **J3** | JST SH 3-pin | SWCLK | GND | SWDIO | Debug **the Pico itself**. Labelled per the Raspberry Pi debug connector spec, but on this revision pin 1 reaches the Pico's SWDIO and pin 3 its SWCLK: use a reversed cable (see Fault 2) |
| **J1** | JST SH 3-pin | TX | GND | RX | UART A — TX ← GP4 via 100 Ω (R1); RX → GP6 via 100 Ω (R2) **and** → 74LVC2G17 schmitt → GP5 (R3/U2A) |
| **J2** | JST SH 3-pin | TX | GND | RX | UART B — TX ← GP12 via 100 Ω (R4); RX → GP14 via 100 Ω (R5) **and** → schmitt → GP13 (R6/U2B). Same wiring as the Debug Probe's SWD port (see *LEDs and controls*) |
| **J4** | JST SH 3-pin | GP0 | GND | GP1 | UART C — direct, no series R, no buffer |

Each buffered UART RX lands on the *natural* SDK UART pin
(GP5 = `uart1` RX, GP13 = `uart0` RX); the unbuffered tap (GP6 / GP14) is a
raw parallel copy — **keep those two GPIOs as inputs** or they will fight the
line.

### LEDs and controls

| Ref | GPIO | Pico pin | Series R | Function |
|-----|------|----------|----------|----------|
| LED7 | GP10 | 14 | R13 470 Ω | JTAG0 |
| LED6 | GP28 | 34 | R12 470 Ω | JTAG1 |
| LED5 | GP2  | 4  | R11 470 Ω | pico-debug: USB connected |
| LED1 | GP7  | 10 | R7 470 Ω | pico-debug: UART RX |
| LED2 | GP8  | 11 | R8 470 Ω | pico-debug: UART TX |
| LED3 | GP15 | 20 | R9 470 Ω | pico-debug: DAP connected |
| LED4 | GP16 | 21 | R10 470 Ω | pico-debug: DAP running |

All are active-high, cathode to GND. **SW4** pulls `RUN` to GND — hardware
reset. `GP9` is the only completely unconnected GPIO on the board.

The five pico-debug LEDs are on the Raspberry Pi Debug Probe's LED pins
(`DEBUG_PROBE_*_LED_PIN` in the pico-sdk's `boards/debug_probe.h`; our local
SDK 2.2.0 checkout does not have that file yet, upstream master does). J1 and J2
follow the same probe's pin map (debugprobe's `board_debug_probe_config.h`):
J1 is its UART port (TX GP4, RX GP5), and **J2 is its SWD port**, not a second
UART: SWCLK GP12, SWDIO GP14, with GP13 reading SWDIO back through U2B.

The Blaster firmware lights its channel's LED while the host has the cable's
outputs enabled, and its JTAG side never touches a pico-debug pin:
`pico-usb-blaster`'s `board_config.h` lists them, and the build fails if a JTAG
profile uses one. Since 2026-10-21 the same image also runs part of the
pico-debug side: J1's UART as a USB serial port, with LED1/LED2 for its
traffic and LED5 for USB connected. Since 2026-10-22 it is also J2's CMSIS-DAP
SWD probe, with LED3/LED4 for DAP connected and running. That code may use only
pico-debug pins, also checked at build time (see *Tested 2026-10-21* and
*Tested 2026-10-22* below).
Until 2026-10-21 the firmware drove LED1/LED2 with the Blaster's nCE/nCS bits
(AS mode, not wired on this board), and JTAG1's profile used LED7.

### Full GPIO allocation

| GPIO | Function | | GPIO | Function |
|------|----------|-|------|----------|
| GP0  | UART C TX (J4.1)        | | GP15 | LED3 |
| GP1  | UART C RX (J4.3)        | | GP16 | LED4 |
| GP2  | LED5                    | | GP17 | JTAG0 TDI |
| GP3  | **JTAG1 shifter DIR**   | | GP18 | JTAG0 TMS |
| GP4  | UART A TX (J1.1)        | | GP19 | JTAG0 TDO (in) |
| GP5  | UART A RX, schmitt      | | GP20 | JTAG0 TCK |
| GP6  | UART A RX, raw tap      | | GP21 | **JTAG1 TDI** |
| GP7  | LED1                    | | GP22 | **JTAG1 TMS** |
| GP8  | LED2                    | | GP26 | **JTAG1 TDO (in)** |
| GP9  | *free*                  | | GP27 | **JTAG1 TCK** |
| GP10 | LED7                    | | GP28 | LED6 |
| GP11 | **JTAG0 shifter DIR**   | | RUN  | SW4 reset |
| GP12 | UART B TX (J2.1)        | | SWCLK| J3.1 |
| GP13 | UART B RX, schmitt      | | SWDIO| J3.3 |
| GP14 | UART B RX, raw tap      | |      | |

---

## Firmware strategy

Three candidate personalities, all previously proven on a bare Pico:

| Personality | Source | USB ID | Host tool |
|-------------|--------|--------|-----------|
| **Altera USB-Blaster** | `../pico-usb-blaster` | `09fb:6001` | Quartus `quartus_pgm`, `jtagconfig`, openFPGALoader `-c usb-blaster` |
| **DirtyJTAG** | `../pico-dirtyJtag` | `1209:c0ca` | `openFPGALoader -c dirtyJtag` (+ CDC UART bridge) |
| **Xilinx / FTDI** | `../ftdi-jtag`, `../pico-fpga` | `0403:6010` | Vivado / openFPGALoader |

**Can they coexist in one image?** Not usefully as a single USB device. The
Blaster personality only works because it clones the FT245 descriptor
(VID/PID `09fb:6001`, bulk-only, vendor class) so Quartus's driver binds to
it; DirtyJTAG is a different vendor protocol on a different VID/PID, and one
USB device has one VID/PID. A second *Blaster* interface does fit in the
same device, but Quartus ignores it (tested below). Realistic options, in
order of preference:

1. **One personality per flashed image**, selected at build time; swap the
   `.uf2` (or reflash over J3 with the debug probe) when you need the other.
   This is the plan for now.
2. **Runtime selection** — read a strap at boot (`GP9` is free, or hold a
   UART pin low) and pick a descriptor set before `tud_init()`. TinyUSB
   supports this; it is a modest amount of work and keeps one binary.
3. **Two boards**, each built for one purpose — the board is cheap and there
   are two independent JTAG channels anyway.

### Tested 2026-10-18: one image, two Blaster interfaces

The two JTAG channels are independent GPIO groups, so one image can drive both.
`../pico-usb-blaster` branch `dual-interface`, `-DBOARD_TYPE=3`, puts JTAG0 on
USB interface 0 and JTAG1 on interface 1 (an FT2232 channel B's endpoints,
`0x83` IN / `0x04` OUT), with Quartus 25.1:

| Question | Result |
|----------|--------|
| Does Quartus list a second cable? | **No.** `jtagconfig` lists the Pico once (`USB Blaster [1-5]`); interface 1 is ignored. |
| Does the extra interface disturb interface 0? | **No.** EP4CGX150 single-try chain scans 22/22, two configurations of `jop_dbfpga.sof` (75 s each, as on the single-channel image), DoAll 68/68 after each. |
| Does channel 1 work? | **Yes.** Driven through interface 1 (`jtag_wiggle e66440a2 tck high 1`), exactly GP27 / GP22 / GP21 move (`GPIO_OUT` read back over SWD) and JTAG0's pins do not. With nothing on J7, TDO reads its pull-up. |

This README used to say a composite device "would break the descriptor match
Quartus relies on". It does not; it just gives Quartus no second cable. The
board went back to the single-channel `BOARD_DEBUG_JTAG0` image. Ways to use
both channels at once:

- **Chain the channels in firmware** — one cable with both targets on its
  chain, which stock Quartus handles (`quartus_pgm ... file.sof@2`). TCK and TMS
  go to both ports, and the Pico relays JTAG0's TDO into JTAG1's TDI on every
  bit. An unpowered target need not break the chain (untested): R16/R17 sit
  on the header side of U5/U9, and an SN74LVC1T45 is high-Z on both sides when
  VCCB (VTREF) is 0, so the firmware can tell an absent target by flipping the
  TDO pad's pull and seeing whether the reading follows. With the FPGA off,
  IDCODE already reads `0xffffffff`, the pull-up.
- **A non-Quartus client on interface 1** — openFPGALoader's USB-Blaster
  driver calls `ftdi_usb_open(0x09fb, 0x6001)`, libftdi's default interface
  of the first match, so it would need a patch selecting `INTERFACE_B` and a
  serial.
- **Two boards** — option 3 above.

### Tested 2026-10-21: the Blaster and the J1 serial port in one device

The pico-debug side's functions are USB *interfaces*, not a VID/PID, so they
can join the Blaster's device. The first one is J1's UART:
`../pico-usb-blaster` branch `cdc-uart`, `-DBOARD_TYPE=2`. It adds a CDC-ACM
serial port ("J1 UART") as interfaces 1-2 on `uart1`, GP4/GP5. The Blaster
stays interface 0, and the device descriptor is unchanged.

| Question | Result |
|----------|--------|
| Does Linux create the port? | **Yes.** `cdc_acm` binds interface 1: `/dev/serial/by-id/usb-Pico_USB_Blaster_e66440a2-if01`. |
| Does the serial port disturb Quartus? | **No.** The standard ten-cycle run: 30/30 single-try chain scans, 10/10 configurations at 75 s (the same as without the port), DoAll 68/68 each, no re-enumeration. |
| Does the bridge carry data? | **Yes, up to 2.5 Mbaud.** In UART1's internal loopback (switched on over SWD), every byte came back at 115200, 921600, 2 M and 2.5 Mbaud. At 3 Mbaud, a continuous stream loses data: the bridge moves about 210 KB/s. |
| Both at once? | **Yes.** A 2 Mbaud loopback stream ran through three scan-configure-DoAll cycles: 66 MB, zero errors, DoAll 68/68 each. Each configuration took 81-82 s instead of 75 s: one Pico loop serves both. |

GP6 matters here. It is wired straight to J1 RX through R2, and an RP2040 pad
comes out of reset with its pull-down on, so J1 RX would sit LOW (a
continuous break) with nothing plugged in. The firmware keeps GP6 an input
with its pull-up instead, so an open J1 idles high, and GP5 reads 1 through
U2A. Not tested yet: the J1 pins themselves (needs a TX-to-RX jumper), J4,
and J2 as a CMSIS-DAP SWD port.

### Tested 2026-10-22: J2 as a CMSIS-DAP probe in the same device

The second pico-debug function is J2's SWD port: `../pico-usb-blaster` branch
`j2-cmsis-dap`, `-DBOARD_TYPE=2`. It adds a CMSIS-DAP v2 interface ("J2
CMSIS-DAP", interface 3, bulk 0x05/0x85) running the Debug Probe's own code:
ARM's CMSIS-DAP command handling and debugprobe's PIO SWD layer, on pio1 beside
the Blaster's pio0. The target was the breadboard Pico (`e6616408`, an RP2040)
on J2.

| Question | Result |
|----------|--------|
| Does OpenOCD find it? | **Yes**, by the carrier's serial: `adapter serial e66440a2` gives "Using CMSIS-DAPv2 interface with VID:PID=0x09fb:0x6001". No kernel driver claims the interface. |
| Does it debug an RP2040? | **Yes.** `test_swd_dap.py`: DPIDR 0x0bc12477 through the multidrop select, CHIP_ID 0x20002927, bootrom block reads that agree, a WATCHDOG_SCRATCH0 round trip, core 0 halt and resume, and 64 KB flash reads at 66 KB/s (1 MHz SWCLK) and 229 KB/s (5 MHz). |
| Does it disturb Quartus? | **No**, with the probe idle. The standard ten-cycle run: 30/30 single-try chain scans, 10/10 configurations at 74-76 s, DoAll 68/68 each, no re-enumeration. |
| Both at once? | **Yes, sharing the Pico.** OpenOCD read the target's flash through J2 nonstop at 5 MHz SWCLK during three scan-configure-DoAll cycles: 5,027 reads of 16 KB (82 MB in 391 s), all identical to the first, no errors. The configurations took 90-91 s instead of 74-76 s, and DoAll ran 68/68 each. |

The first run failed with every SWD acknowledge "JUNK". The cable to the
breadboard Pico had SWCLK and SWDIO reversed: driving J2.3 from the carrier
moved GP13 through U2B, driving J2.1 did not, and a test build with the two
roles swapped connected at once. A cable with the two wires crossed fixed it.
J2 itself follows the Debug Probe's pinout: pin 1 SWCLK, pin 3 SWDIO.

Not tested yet: flashing a target over J2 (the remote-reflash drill), J4, and
the J1 pins themselves.

### Bring-up target: JTAG1 as an Altera USB-Blaster

Port `../pico-usb-blaster` with these changes:

```c
/* src/blaster.c — pin block for pico-usb-debug-jtag, JTAG1 channel */
#define TCK_DCLK_PIN        27
#define TMS_nCONFIG_PIN     22
#define TDI_ASDI_PIN        21
#define TDO_CONF_DONE_PIN   26
#define SHIFTER_DIR_PIN      3   /* NEW: drive high to enable A->B */

/* Signals the board has no wiring for — park them harmlessly */
#define nCE_PIN              7   /* LED1 */
#define nCS_PIN              8   /* LED2 */
#define DATAOUT_nSTATUS_PIN  9   /* free pin, pull-up reads 1 like a real cable */
```

Required firmware behaviour changes versus the flying-lead build:

1. **Drive `GP3` high in `blaster_init()`** *before* setting GP21/22/27 as
   outputs. With `DIR` low the shifter drives its own A pin, so a Pico output
   on the same net would be contention. Simplest safe rule: set `GP3` high
   once at init and leave it high; let the Blaster protocol's OE bit control
   only the activity LED, not the pin directions.
2. **AS / PS modes are not supported by this hardware.** Only TCK/TMS/TDI/TDO
   are wired; `nCE`, `nCS` and `DATAOUT` go nowhere. JTAG mode only.
3. `DATAOUT_nSTATUS` must still read **1** when idle (the pull-up fix
   documented in `blaster.c`) — hence parking it on free `GP9`.

---

## Bring-up status (2026-08-27)

Breadboarded SN74LVC1T45 shifters wired to the PCB's JTAG1 map; Pico 1
(RP2040) in place of the schematic's Pico 2.

| Step | Status |
|------|--------|
| Board profile added to `../pico-usb-blaster/src/board_config.h` | done — `BOARD_DEBUG_JTAG1` is the default |
| `GP3` shifter DIR driven high at init, before any JTAG pad becomes an output | done |
| OE bit no longer tri-states the Pico pads on a shifted board | done |
| Built + flashed over SWD (DAPLink CMSIS-DAP on J3) | done, verified |
| Enumerates as `09fb:6001`, iManufacturer `Pico`, serial `e6616408` | done |
| Firmware liveness — bit-bang reads + 4-byte PIO shift both return | done |
| Return path (TDO/U9/VCCB) proven electrically | done — see below |
| **Detect a real FPGA** | **WORKING — 10CL025 `020F30DD`, EP4CGX150 `028040DD`** |
| **Configure a bitstream end to end** | **WORKING — 4.88 MB .sof to EP4CGX150 in 43 s at 6 MHz** |

`jtag_pintest` with no target attached (this is the healthy result):

```
  all low       -> TDO=1        <- Pico pull-up wins; U9's A output is high-Z
  TCK=1         -> TDO=1           because VCCB (VTREF) is unpowered, which is
  TDI=1         -> TDO=1           the LVC1T45 power-off protection working
  IDCODE = 0xffffffff
```

USB, the bit-bang path and the PIO shift path are all proven.

### Diagnosing the return path: pull-flip test

With a target attached, `jtag_pintest` still reported a stuck TDO. To tell a
*floating* pad from a *driven* one, build with `-DTDO_DIAG_PULLDOWN=1`, which
inverts GP26's internal pull. A floating pad follows the internal pull; a
driven one does not.

| Build | TDO reading | Conclusion |
|-------|-------------|------------|
| pull-up, 6 MHz | all 1 | floating, pulled up |
| pull-down (diagnostic) | all 0 | floating, pulled down |
| pull-up, 750 kHz | all 1 | floating, pulled up |

GP26 follows whichever internal pull is compiled in, in both directions, so the
pad is **floating — U9's A output is not connected to it.**

> **Trap that cost an hour here.** `TDO_DIAG_PULLDOWN` is a CMake *cache*
> variable. Re-running `cmake` without it does **not** clear a previously set
> one, so a "production" rebuild silently kept the inverted pull and two
> supposedly-opposite builds were actually identical. That produced a confident
> but wrong reading of "TDO is actively driven, return path proven".
> `CMakeLists.txt` now prints its config at configure time and warns on a
> diagnostic build. When a result depends on a build flag, verify the flag
> reached the binary.

### `jtag_wiggle` — proving the outbound pins with a meter

`../pico-usb-blaster/jtag_wiggle.c` parks one Blaster output at a known level,
or squares it at ~1 Hz so a DMM reads mid-rail. Every command sets the OE bit.

```bash
gcc -O2 -o jtag_wiggle jtag_wiggle.c -lusb-1.0
./jtag_wiggle 1:50 tck toggle    # meter on target-side TCK -> ~VTREF/2
./jtag_wiggle 1:50 tms high      # -> VTREF
./jtag_wiggle 1:50 tdi low       # -> 0 V
```

Measure on the **target side** of each shifter. A pin that stays at 0 V in
every mode means the Pico's drive is not getting through: check `GP3` actually
reaches the DIR input of all three outbound shifters (U7/U8/U10), and that it
is high.

### Root cause found: DIR below VIH (2026-08-27)

With TCK/TMS/TDI commanded high and confirmed driven at the pad (read back over
SWD: `oe=1 out=1 in=1` on GP21/GP22/GP27, GP3 high), meter readings were:

| Point | TCK | TMS | TDI |
|-------|-----|-----|-----|
| Shifter A side (Pico side) | 0 V | 2.6 V | 0.22 V |
| Shifter B side (target side) | 0 V | 3.29 V | 1.75 V |
| FPGA header pin 1 / 5 / 9 | 0 V | 3.3 V | 3.3 V |
| **GP3 / DIR net** | **1.45 V** | | |

`GP3` measured **1.45 V**. The SN74LVC1T45 needs VIH >= ~0.65 x VCCA = 2.15 V,
so all three outbound shifters read DIR as low and sat in **B->A mode, driving
backwards**. That accounts for every symptom:

* The A-side readings are *contention* — shifter driving A while the Pico drives
  the same net high — not broken wires. The Pico's own registers read `out=1,
  in=1` at the same moment a meter read 0 V on the net.
* The B-side readings are just the Cyclone idling: TMS and TDI show its internal
  weak **pull-ups**, TCK shows its **pull-down** (Altera specifies a pull-down on
  TCK). Nothing from the Pico was reaching the TAP, hence no IDCODE.

**Actual cause: the DIR jumper was on Pico physical pin 4 (= GP2), not pin 5
(= GP3).** GP2 is an LED pin the firmware never drives, so the DIR net was
simply floating, and a floating CMOS net sits at an arbitrary mid-rail voltage —
1.45 V here. (An earlier reading of this as ~50 ohm contention to ground was
wrong: nothing was fighting, the net just wasn't connected to anything driven.)

Worth remembering that Pico *physical pin* numbers and GPn numbers are offset
and easy to transpose: GP2 = pin 4, GP3 = pin 5.

Lesson for the PCB: the netlist is correct here (R14/R15 are 10k pulldowns and
GP3/GP11 are driven push-pull), but a **DIR test point per channel** would have
made this a five-second measurement.

### Working (2026-08-27)

Two wiring faults, both found by measurement, both on the Pico side:

1. **DIR on Pico physical pin 4 (GP2) instead of pin 5 (GP3)** — the DIR net
   floated at 1.45 V, below the shifters' VIH, so all three outbound shifters
   sat in B->A and drove backwards.
2. **TDO not landing on GP26** — the pad was floating, which reads as a clean
   pull-up value and mimics a working cable with a broken chain.

After rewiring, from a cold `jtag_pintest auto`:

```
IDCODE = 0x020f30dd   (10CL025 — matches the predicted value)
```

and Quartus agrees:

```
1) USB Blaster [1-14]
  020F30DD   10CL025(Y|Z)/EP3C25/EP4CE22      <- this Pico
2) USB-Blaster [1-5]
  020F70DD   10CL120(Y|Z)/EP3C120/..          <- genuine cable, other board
```

**Stable at the full 6 MHz** (`JTAG_CLKDIV=5`, the genuine-Blaster rate that
Quartus reports and will not let a host change): 5/5 pin-level IDCODE reads and
3/3 `jtagconfig` scans, on a breadboard. The 750 kHz fallback was not needed.

This is the first time this firmware has had Quartus detect a chain rather than
report "JTAG chain broken", which supports the original theory that the earlier
failures were level-shifting/wiring rather than protocol.

### Configuration proven on the EP4CGX150

The QMTECH EP4CGX150 detected first try at 6 MHz — `0x028040dd` at the pin
level, `028040DD EP4CGX150` in Quartus, 5/5 and 3/3 on repeats. Then a real
configuration:

```
$ quartus_pgm -c "USB Blaster" -m jtag -o "p;.../jop_bram.sof"
Info (213045): Using programming cable "USB Blaster [1-14]"
Info (209017): Device 1 contains JTAG ID code 0x028040DD
Info (209007): Configuration succeeded -- 1 device(s) configured
```

**4.88 MB in 43 s at 6 MHz.** This is the step `../pico-usb-blaster` had never
completed — its `blaster_jtag.pio` carried a CAUTION saying the cable "has never
successfully CONFIGURED an FPGA", with flying leads as the prime suspect. That
was right, and the missing piece was level shifting. The CAUTION has been
updated in place.

Boards proven so far:

| Board | FPGA | JTAG VTREF | IDCODE | Detect | Configure |
|-------|------|-----------|--------|--------|-----------|
| QMTECH 10CL025 core | Cyclone 10 LP 10CL025 | 3.3 V | `020F30DD` | yes | not tried (no bitstream) |
| QMTECH EP4CGX150 | Cyclone IV GX | **2.5 V** | `028040DD` | yes | **yes, 43 s** |

**The EP4CGX150's JTAG header is 2.5 V**, so this run exercised real level
translation rather than a 3.3 V pass-through — the exact case a bare Pico cannot
serve. It also reframes the original failures: a bare Pico drives 3.3 V into
2.5 V-referenced JTAG inputs, which is out of spec for those pins, so this was
never purely a signal-integrity problem. Referencing VCCB to the target's own
VTREF on header pin 4 is what makes the cable safe across a mixed stack, and is
the single most important thing not to shortcut when wiring a breadboard
version.

## Bring-up on the PCB: JTAG0 (2026-10-15)

The PCB, partially populated for JTAG0 (U3–U6, R15, R16, R22–R25, C5–C8, J5),
with a Pico 1 (RP2040, rev B2) in place of the schematic's Pico 2.

| Step | Result |
|------|--------|
| Firmware | `../pico-usb-blaster`, `-DBOARD_TYPE=2 -DJTAG_CLKDIV=5 -DTDO_DIAG_PULLDOWN=0`, fresh build dir `src/build-jtag0` |
| Flash | over USB: the blank Pico boots as `RP2 Boot`, then `picotool load -v -x pico_usb_blaster.uf2`. SWD on J3 works only with a REVERSED cable: the board swaps SWCLK/SWDIO (see Fault 2) |
| Enumerates | `09fb:6001`, iManufacturer `Pico`, iProduct `USB Blaster`, serial **`e66440a2`** |
| No target on J5 | TDO=1, IDCODE `0xffffffff` — healthy: no VTREF, shifters off |
| EP4CGX150 on J5 | IDCODE `0x028040dd`; `jtagconfig` `028040DD EP4CGX150` |
| **Configure** | `jop_dbfpga.sof` (4.88 MB) in 72 s, programmer start to end; DoAll 68/68 on the board afterwards |

### Cycling test (2026-10-15)

Ten cycles, each: pin-level IDCODE, ONE single-try Quartus chain scan
(`JTAG_SCAN_TRIES=1`), a full configuration of `jop_dbfpga.sof`, then DoAll over
the board's console. Ten single-try scans before and after.

| Check | Result |
|-------|--------|
| Single-try chain scans | **30/30** (10 before, 10 in-cycle, 10 after) |
| Configurations | **10/10**, 74-75 s each |
| DoAll | **68/68 every cycle**, no failures, no uncaught exceptions |
| USB | same bus:dev throughout — never re-enumerated |

Run twice: first with R15 off, then again after R15 was refitted (a part that
read 10 kΩ before and after soldering). Both runs gave exactly the table above;
with R15 back the first pin-level read after power-up was `0x028040dd`.

The breadboard cable read 3/10 single-try on the same board in August, and later
died with the cable re-enumerating. Nothing of that kind here so far; worth
re-running after some days of use, since that is when the breadboard one went.

### Fault 1, fixed: a solder bridge across R15 held DIR low

With VTREF up the IDCODE read `0x00000000`: TDO showed only R16's pull-down.
With TCK/TMS/TDI parked high by `jtag_wiggle`:

| Point | Read | Expected |
|-------|------|----------|
| J5.4 VTREF | 2.5 V | 2.5 V |
| GP11 / DIR | **0.72 V** | 3.3 V |
| J5.1 TCK | 0 V | 2.5 V |
| Pico pin 26 (GP20, TCK) | 0.22 V | 3.3 V |
| J5.5 / J5.9 (TMS / TDI) | 2.82 V | 2.5 V — the FPGA board's pull-ups, not our drive |
| Pico pins 24 / 22 (GP18 / GP17) | 3.26 V | 3.3 V |

A DRIVEN pin that sits at 0.72 V is heavily loaded: GP11 to GND measured
**20 Ω**. R15 measures 10 kΩ out of circuit; with it removed GP11 reads 11.2 kΩ
(the unpowered chips' protection diodes) and the chain works — the short was
solder bridging R15's pads. R15 was refitted the same day (10 kΩ before and
after soldering); unpowered, GP11 to GND should then read ~5 kΩ (10k in
parallel with that diode path), not tens of ohms.

**While DIR is low and VTREF is up, the outbound shifters drive the Pico's
TCK/TMS/TDI pads against the Pico's own outputs** — tens of mA per pin, for as
long as the target stays powered. Power the target down until DIR is fixed.

### Fault 2, found: J3's SWCLK and SWDIO are swapped at the Pico (2026-10-17)

OpenOCD through the Raspberry Pi Debug Probe (`adapter serial
E6647C740343412F`, `target/rp2040.cfg`) gave `Too long SWD WAIT, issuing
DAPABORT` then `Failed to connect multidrop rp2040.dap0` at every speed with the
Debug Probe's own straight cable. A cable whose ends are reversed connects.

**The PCB is crossed, and the EasyEDA part is why.**

- Raspberry Pi's pinout drawing, titled "For Raspberry Pi Pico 1 and Raspberry
  Pi Pico 2", puts the debug pins in the order SWCLK, GND, SWDIO, viewed from
  the top with USB up. **SWCLK is on the pin 20 (GP15) side.**
- The footprint (`COMM-SMD_L51.0-W21.0-P2.54_SC0916`) puts pad 41 on the pins
  1-20 side, i.e. at the Pico's physical SWCLK. But the "Raspberry Pi Pico 2"
  symbol names pad 41 **SWDIO** and pad 43 **SWCLK**.
- So net `PICO_SWCLK`, which goes to J3 pin 1, lands on the Pico's SWDIO, and
  `PICO_SWDIO` (J3 pin 3) lands on SWCLK.
- J3 itself follows the Raspberry Pi 3-pin Debug Connector Specification by
  name: pin 1 SC (SWCLK), pin 2 GND, pin 3 SD (SWDIO), joined by a
  **straight-through (pin 1 to pin 1) cable**, which is what the Debug Probe
  ships with. With that cable the probe's SWCLK reaches the Pico's SWDIO, which
  is why it fails. A reversed cable (pin 1 to pin 3) undoes the board's swap.

**For this board:** use a reversed cable, and mark it — it fails on every other
Raspberry Pi target. **For the next revision:** swap the two nets at U1
(`PICO_SWCLK` to pad 41, `PICO_SWDIO` to pad 43), or fix the symbol's pin names
in the library so pad 41 is SWCLK and pad 43 SWDIO.

(An earlier version of this note, written the same morning, blamed the cable
and called the footprint right. That assumed the working cable was straight.)

With SWD working, the firmware's pin state read back from the RP2040 (JTAG0
profile, idle, R15 refitted):

| Register | Value | Meaning |
|----------|-------|---------|
| `GPIO_OE` 0xd0000020 | `0x02160d80` | GP7, 8, 10, 11, 17, 18, 20 driven (+ GP25, the Pico LED); GP19 TDO an input |
| `GPIO_OUT` 0xd0000010 | `0x00060980` | **GP11 DIR high**; TMS, TDI high; TCK low |
| `GPIO_IN` 0xd0000004 | `0x01060b81` | GP11 reads high at the pad; GP19 TDO 0 (R16, target idle); GP9 1 (pull-up) |

Also seen: the Debug Probe runs old firmware (1.0.1), which OpenOCD works around
("Using low-performance workaround"); it was not the cause here.

### Flashing without SWD, from a VM

- The Pico is a different USB device in each mode — `2e8a:0003 RP2 Boot` in
  BOOTSEL, `09fb:6001` under this firmware — so a VM passthrough rule made for
  one does not cover the other. Map both.
- `picotool --ser E0C9125B0D9B` (the RP2 Boot USB serial) finds nothing; with a
  single BOOTSEL device attached, the unfiltered command works.

### Host tools

`jtag_pintest` and `jtag_wiggle` (both in `../pico-usb-blaster`) accept three
forms of probe selection:

```bash
./jtag_pintest e6616408   # iSerial substring — PREFERRED
./jtag_pintest 1:56       # bus:dev — moves on every reflash
./jtag_pintest auto       # first cable with iManufacturer "Pico"
```

**Use the serial.** It comes from the RP2040's unique flash ID, so it is the
only selector that stays unambiguous as probes are added — every Pico running
this firmware reports the same iManufacturer (`Pico`) and iProduct
(`USB Blaster`), so `auto` and any product-string match become ambiguous the
moment there are two. A stale `bus:dev` is worse than wrong: the tool finds
nothing and exits, which looks exactly like a dead pin.

Serial of the probe used here: **`e6616408`**.

Note the descriptor strings are still load-bearing for *Quartus*, which shows
`iProduct` as the cable name — `USB Blaster` (space, this clone) vs
`USB-Blaster` (hyphen, genuine). Do not "fix" the clone's string to match the
genuine cable exactly, or `jtagconfig` loses its only way to tell them apart.

### Known limitation: openFPGALoader picks the wrong cable

`openFPGALoader -c usb-blaster` always opens the **genuine** cable when both are
attached, and `--busdev-num` does not help: `src/usbBlaster.cpp:483` calls

```c
ret = ftdi_usb_open(_ftdi, 0x09fb, 0x6001);
```

which takes the first VID:PID match and never consults the requested bus/device.
Fixing it means switching to `ftdi_usb_open_bus_addr()`, mirroring what
`../openfpgaloader-dirtyjtag-busdev.patch` already does for the DirtyJTAG
backend. Until then, either unplug the genuine cable or drive this one through
Quartus, which selects by cable index and is unaffected.

### Firmware build knobs

| CMake option | Effect |
|--------------|--------|
| `-DBOARD_TYPE=0\|1\|2` | flying leads / JTAG1 / JTAG0 pin profile |
| `-DJTAG_CLKDIV=n` | TCK = sys_clk / (n × 4); `5` = 6 MHz, `40` = 750 kHz |
| `-DTDO_DIAG_PULLDOWN=1` | diagnostic: invert the TDO pull (see above) |

Currently flashed: `BOARD_DEBUG_JTAG1`, `JTAG_CLKDIV=5` (6 MHz), production
pull-up.

### Flashing over SWD

```bash
cd ../pico-usb-blaster/src/build
PICO_SDK_PATH=~/workspaces/pico-sdk cmake .. -DPICO_BOARD=pico
make pico_usb_blaster -j$(nproc)
openocd -f interface/cmsis-dap.cfg -c "adapter serial E6647C740343412F" \
  -f target/rp2040.cfg \
  -c "adapter speed 5000; program pico_usb_blaster.elf verify reset exit"
```

Name the probe: `E6647C740343412F` is the Raspberry Pi Debug Probe on J3.
Since 2026-10-22 the carrier is a CMSIS-DAP probe itself (J2, serial
`e66440a2`). Without `adapter serial`, OpenOCD takes the first CMSIS-DAP it
finds, and if that is the carrier, this command reflashes the Pico on J2
instead of the carrier's own.

Select a different channel/harness with
`cmake .. -DBOARD_TYPE=2` (`0` = flying leads, `1` = JTAG1, `2` = JTAG0),
or edit the default in `src/board_config.h`.

### Two Blasters on one bus

This host has both cables attached, sharing VID:PID `09fb:6001`. They are
distinguishable:

| | iManufacturer | iSerial | `jtagconfig` name |
|-|---------------|---------|-------------------|
| Genuine / Terasic | `Altera` | `91d28408` | `USB-Blaster` (hyphen) |
| This Pico | `Pico` | `e6616408` | `USB Blaster` (space) |

```bash
jtagconfig                       # lists both, numbered
quartus_pgm -c "USB Blaster" …   # the Pico, by name — note: no hyphen
sudo ./jtag_pintest 1:48         # pin-level probe, selected by bus:dev
```

`libusb_open_device_with_vid_pid()` takes the *first* match, so any tool that
does not select by bus:dev or name may grab the wrong cable — this is what
`../openfpgaloader-dirtyjtag-busdev.patch` exists to solve. Stop `jtagd`
(`pkill -x jtagd`) before running a libusb tool, or it will hold the interface.

---

## Known hardware notes / risks

* **TDO pulldowns (R16, R17 — 10 kΩ to GND).** A genuine USB-Blaster
  pulls TDO **up**. `../pico-usb-blaster/src/blaster.c` documents that a
  floating-low TDO makes Quartus report *"JTAG chain broken"* even though
  shifted data is correct, and fixes it with `gpio_pull_up()` on the Pico pin.
  That fix cannot work here: `GP26` is driven by U9's A output at all times,
  so it follows the target-side net, which R17 holds **low** when the target
  tri-states TDO. If chain detection misbehaves, depopulate R17 (and R16) or
  reweork it to `JTAGn_VCC_T` instead of GND. **Check this first if
  `jtagconfig` fails.**
* **VTREF is mandatory.** No target power on header pin 4 ⇒ shifter B-side
  unpowered ⇒ no JTAG at all. Verify 1.8/2.5/3.3 V on J7.4 before debugging
  anything else. Confirmed in practice: the EP4CGX150 supplies **2.5 V** here,
  and feeding VCCB from a local 3.3 V rail instead would put 3.3 V onto 2.5 V
  JTAG inputs — out of spec for the target.
* **J3 (SWD) has SWCLK and SWDIO swapped at the Pico** — the EasyEDA Pico 2
  symbol names pad 41 SWDIO, but pad 41 is the Pico's SWCLK. Needs a reversed
  cable on this revision; fix the symbol or the nets next time (see Fault 2).
* **`GP3`/`GP11` DIR pins are shared** across the three outbound shifters of a
  channel; there is no per-pin output enable.
* **GP6 / GP14** sit directly on the UART RX lines with only 100 Ω — never
  configure them as outputs.
* **GP26/GP27** are ADC-capable pins used here as plain digital I/O; that is
  fine, but note `GP26` = ADC0 if you ever want analog.
* The Pico's own 3V3 rail powers the A-side of every shifter and U2; nothing
  on this board runs off VSYS/VBUS.

## Related work in `~/workspaces`

| Path | What it is |
|------|------------|
| `../pico-usb-blaster` | Altera USB-Blaster clone firmware (PIO shift engine, 6 MHz TCK) |
| `../pico-dirtyJtag` | DirtyJTAG firmware + dual CDC-UART bridge |
| `../pico-fpga` | Logic analyser / UART bridge / JTAG scripting |
| `../ftdi-jtag` | FTDI-based JTAG experiments |
| `../openFPGALoader` + `../openfpgaloader-dirtyjtag-busdev.patch` | Host programmer, with a local patch to select a cable by USB bus:dev |
| `../jop-spinalhdl` | Consumer of this board — see `docs/pico-dirtyjtag-setup.md` |
| `../pico-sdk` | Pico SDK used to build all of the above |
