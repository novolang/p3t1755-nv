# p3t1755-nv

A driver for NXP's P3T1755, a digital temperature sensor on an I2C or
I3C bus, specified in NXP's data sheet "P3T1755: I3C, I2C-bus
Interface, 0.5 °C Accuracy, Digital Temperature Sensor"
([nxp.com/docs/en/data-sheet/P3T1755.pdf](https://www.nxp.com/docs/en/data-sheet/P3T1755.pdf)).
The driver speaks I2C and implements the `Thermometer` and
`Identifiable` traits of [devices-nv](https://novo-lang.org/packages/devices-nv).

## What it is

The P3T1755 measures its own temperature from -40 °C to 125 °C,
within 0.5 °C from -20 °C to 85 °C (data sheet Table 29).  It keeps
each result as a 12-bit two's-complement count of 0.0625 °C.  It
converts continuously, one conversion per period, or once on request
from shutdown, which the data sheet calls a one-shot conversion.

Each result is compared with two limits, a high limit (THIGH) and a
low limit (TLOW), to drive the part's ALERT pin.  The fault queue sets
how many results in a row must cross a limit before the pin changes.

The part has four registers, chosen by a pointer byte at the start of
each transaction (data sheet section 7.5):

| Pointer | Register | Bytes | Access | Power-up value |
|---|---|---|---|---|
| 0x00 | Temp, the most recent result | 2 | read | 0x0000 until the first conversion |
| 0x01 | Conf, the configuration | 1 | read and write | 0x28 |
| 0x02 | TLOW, the low limit | 2 | read and write | 0x4B00, 75 °C |
| 0x03 | THIGH, the high limit | 2 | read and write | 0x5000, 80 °C |

The configuration register holds, from bit 7 down: OS, the one-shot
request; R1 and R0, the conversion period; F1 and F0, the fault queue;
POL, the ALERT pin's polarity; TM, the thermostat mode; and SD,
shutdown (data sheet Table 19).

| R1 R0 code | Conversion period |
|---|---|
| 0 | 27.5 ms |
| 1 | 55 ms, the power-up value |
| 2 | 110 ms |
| 3 | 220 ms |

The address is set by the part's strap: the FRDM-MCXN947 board wires
its P3T1755 at 0x48.

## Install

```sh
novo pkg add p3t1755-nv
```

## Example

A board's device table names the part and this driver:

```novo norun:fragment
    devices
        temp  P3t1755  from "p3t1755-nv"  on i2c0  addr 0x48
```

and a program reads it as `bsp.board.temp()`:

```novo norun:needs-pkg
use p3t1755

@tier(embedded)
fn main() [hw]
    let t = bsp.board.temp()
    // Start a conversion and wait the time the part needs: 0 when it
    // converts continuously, 12 ms for a one-shot conversion.
    let us = t.temp_start()
    hal.timer.delay_us(us)
    match t.temp_read_after(us, us)
        // The temperature in thousandths of a degree Celsius.
        Some(c) => hal.uart.write(str.from_int(c.milli_c))
        None    => hal.uart.write("no answer")
```

## What the package contains

| Module | What is in it |
|---|---|
| `p3t1755` | The register map, the conversions between a register word and a temperature, the register access over any `I2cBus`, and the type `P3t1755`, which holds a board's bus |

## How to choose an entry point

A program on a board uses the type `P3t1755`, as the board's device
table builds it.  The functions over `I2cBus`, which the type's
methods call, take any bus, so a program with its own bus or a test
with a double uses them directly.

| Method of `P3t1755` | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` on the board's bus; nothing is sent |
| `temp_start()` | `Thermometer`: in shutdown, a one-shot conversion, answering 12000 µs; converting continuously, 0; -1 when the part does not acknowledge |
| `temp_read()` | `Thermometer`: the temperature register |
| `device_id()` | `Identifiable`: the configuration byte, when the registers have a P3T1755's shape |
| `config()` | the configuration register |
| `shut_down()` and `resume()` | set or clear SD, answering the time to wait, one conversion period |
| `one_shot()` | a one-shot conversion from shutdown, answering 12000 µs, or -1 when the part is converting continuously |
| `conversion_rate()` and `set_conversion_rate(code)` | the R1 R0 code, and setting it, answering the new period in µs |
| `limit_low()`, `limit_high()`, `set_limit_low(t)`, `set_limit_high(t)` | TLOW and THIGH as temperatures |
| `at_reset_values()` | whether the configuration and both limits hold their power-up values |
| `restore_defaults()` | writes the power-up values back |

## The rules a user needs

1. A one-shot conversion runs only from shutdown (data sheet section
   7.5.9).  `one_shot` refuses a part that converts continuously.
2. Shutdown takes effect when the conversion in progress ends (section
   7.5.4), so `shut_down` answers one conversion period to wait.
3. A one-shot conversion takes at most 12 ms (Table 29, typical
   7.8 ms).  `temp_start` answers 12000 µs and `temp_read_after`
   refuses an earlier read.
4. The temperature register reads 0 °C after power-up until the first
   conversion ends (section 7.5.2).
5. A temperature is negative when bit 11 of the count is set (section
   7.5.2): a word of 0xFFC0 is -0.25 °C.
6. A limit is written rounded to the nearest 0.0625 °C and held to
   -128 °C to 127.9375 °C, the range of the register.
7. OS always reads 0 (section 7.5.9).  Every configuration update
   writes it as 0, so an update never starts a conversion.
8. The part has no identification register.  `device_id` answers the
   configuration byte after checking that OS and the four low bits of
   both limits read 0, which every P3T1755 answers.  At power-up the
   byte is 0x28, so `device_probe(p3t1755.CONF_RESET)` holds for a
   part no program has configured.  Neither this nor
   `at_reset_values` tells the P3T1755 from another part with the same
   register layout.

## Running on a microcontroller

Every function builds for a microcontroller.  The driver's own code
builds no list: its bytes move through devices-nv's `DevBytes`, a
32-byte buffer held in the caller's frame.  devices-nv's register
access allocates one frame per transaction and releases it before it
returns, because `I2cBus` takes and answers lists.  The registry shows
the package on the host layer, the layer of the trait packages it
depends on.

## What is not included

- I3C.  The part answers I3C as well (section 7.4); this driver speaks
  I2C only.
- The ALERT pin's use as an interrupt.  The fault queue, POL and TM are
  reachable through `update_conf`; no method names them.
- The general call's reset (section 7.3.3), which resets every part on
  the bus that answers the general call.  `restore_defaults` writes the
  power-up values to this part alone.
- The async form of `Thermometer`.  An `async fn` takes no value type
  such as `P3t1755`.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv): the
  `Thermometer` and `Identifiable` traits, the wiring record a board's
  device table builds, and the register access this driver uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv):
  the `I2cBus` trait and the board's bus handle, `BoardI2c`.

## Tests

`novo test tests` runs the register logic on the host over an I2C
double, with the data sheet's Table 18 as the temperature vectors, and
runs each method of the type `P3t1755` over the host's board handles,
which are absent hardware, so every method answers as for a part that
does not acknowledge.  On hardware, the FRDM-MCXN947's suite in the
novo-lang repository, `orbit/bsp/nxp/mcxn9/frdm-mcxn947/tests/thermometer.sh`,
reads the part through the type and reads back the line coverage of
this package's module on the board.

## Licence

Apache-2.0.
