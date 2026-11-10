# vl53l1x-nv

A driver for ST's VL53L1X, a long-distance time-of-flight ranging sensor
with an I2C interface, specified in the datasheet "VL53L1X: A new generation,
long distance ranging Time-of-Flight sensor based on ST's FlightSense
technology", DocID031281 Rev 3, November 2018 (ST's DS12385), and in the API
user manual "VL53L1X API user manual", UM2356 (DocID031478 Rev 1, March
2018). ST publishes no register map. The register list, the 91-byte default
configuration and the sequences are those of ST's ultra-lite driver
(STSW-IMG009), as the stm32duino/VL53L1X copy of it carries them under ST's
BSD 3-clause licence (see "Sources"). The driver implements the
`Rangefinder`, `RangefinderZones` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the
distance in millimetres with ST's range status.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
sensor, so the driver has run only against a double of the part, in the host
suite. The device program below builds for the nRF52840 DK and the
nRF54LM20 DK and has not been flashed. The offset and cross-talk calibrations
are tested against hand-worked sums only; their results on a part are NOT
VERIFIED.

## Install

```sh
novo pkg add vl53l1x-nv
```

The device program is `examples/range_reader.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/range_reader.nv` brings the sensor up in long distance mode with a
100 ms timing budget, prints a distance about ten times a second over the
board's UART, and prints a 4 by 4 map of the field of view once in ten
readings:

```
range 1234 mm
out of range
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio, BoardTimer }
use dev_range.{ RangeReading, Rangefinder, RangefinderZones }
use dev_bus
use vl53l1x.{ Vl53l1x, Vl53Mode }

// Writes `n`, 0 or more, in decimal.  Formatting into a string would
// allocate, which @tier(embedded) refuses, so each digit goes out as a
// byte.  An Int has at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + n / div % 10)
            div = div / 10

// Writes the line for a reading.
@tier(embedded)
fn report(r: RangeReading) [hw]
    match r.status
        RangeValid      =>
            hal.uart.write("range ")
            digits(r.mm)
            hal.uart.write(" mm\n")
        RangeWeak       =>
            hal.uart.write("weak ")
            digits(r.mm)
            hal.uart.write(" mm\n")
        RangeOutOfRange => hal.uart.write("out of range\n")
        RangeNoTarget   => hal.uart.write("no target\n")
        RangeFault      => hal.uart.write("sensor fault\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    var sensor = Vl53l1x.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, BoardGpio { id: 0 },
                                dev_bus.wiring_i2c(0, 0x29))
    // The sensor needs 1.2 ms after its supply; the program waits 5.
    hal.timer.delay_ms(5)
    if not sensor.begin()
        hal.uart.write("no answer from the sensor at 0x29\n")
    let _ = sensor.set_mode(Vl53Mode.Vl53Long)
    let _ = sensor.set_budget_ms(100)
    while true
        if sensor.range_start() < 0
            hal.uart.write("no answer\n")
        else
            match sensor.range_read()
                Some(r) => report(r)
                None    => hal.uart.write("no answer\n")
        hal.timer.delay_ms(100)
```

The shown program leaves out the zone map. The program in the package is
`examples/range_reader.nv`. A board package that depends on this package
builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs the
image and checks that the program reports that the sensor does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 16,888 B | 4,762 B | 2,644 B |
| `nrf54lm20-dk` | 18,152 B | 5,024 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the runtime,
the compiler's support library and the program's tables; the program
includes the 91-byte configuration table and the code for the whole
initialisation, both distance modes, the zone scan and the two
calibrations. The arena, 65,536 B on both boards, is the runtime's heap and
is reserved whether or not the program uses it; the driver allocates nothing
in it.

Neither Nordic board package names a driver for its two-wire interface
(`@hal(i2c)`), so `BoardI2c` is absent hardware on both: every transfer is
refused, the program prints `no answer from the sensor at 0x29`, and the
build removes the code that follows the first bus read, which makes the rows
above smaller than the image of a board with a bus.

The wiring: SDA and SCL to the board's first I2C bus, the breakout's supply
to 3.3 V, and its XSHUT and GPIO1 pins either open (the breakouts pull XSHUT
up) or on two free pins of the board, which the wiring record names.

### The effects

Every function of the driver that reaches the bus, a pin or the timer
declares `[hw]`, and nothing else. The driver performs no input or output of
its own and allocates nothing of its own. The bus functions of devices-nv
build one short frame per transaction inside a `system` block and release it
when the call returns. The module checks at every tier, `embedded` included.
Bringing the part up makes 15 bus transactions, which take about 4 ms at
400 kHz, and one ranging cycle of the part's own, 100 ms or so; a
measurement spins the core, polling the part, for the length of its timing
budget. A zone scan is 16 measurements.

## What the sensor is

The VL53L1X is a laser rangefinder in a 4.9 x 2.5 x 1.56 mm package. A 940 nm
VCSEL sends pulses, a receiving array of 16 by 16 single-photon avalanche
diodes (SPADs) counts the light that returns, and the part times the return,
so the distance does not depend on the target's reflectance. It ranges to
4 m. It has an I2C interface at up to 400 kbit/s on the address 0x29 (the
datasheet writes 0x52), a shutdown pin XSHUT, active low, and an interrupt pin
GPIO1. It boots in 1.2 ms after XSHUT rises or the supply comes up.

Three registers can be read to validate the bus (datasheet Table 11): 0x010F
reads 0xEA, 0x0110 reads 0xCC and 0x0111 reads 0x10. Register addresses are
16 bits, sent high byte first. Multi-byte values are most significant byte
first at ascending addresses.

### The initialisation

`start` runs the ultra-lite driver's `SensorInit`:

| Step | What it does |
|---|---|
| model check (`model_ok`) | reads 0x010F to 0x0111 and wants 0xEA, 0xCC |
| boot (`boot_wait`) | polls the firmware status 0x00E5 until it is not zero, for 100 ms at most |
| table (`config_write`) | writes the 91 bytes of ST's default configuration to 0x2D to 0x87 in four bursts |
| I/O supply | with `begin`, sets bit 0 of 0x2E, which makes the I2C pads pull up to AVDD, after the table has written 0x2E |
| first ranging (`vhv_cycle`) | starts ranging, waits for the first result, clears the interrupt, stops, and puts the VHV loop bound (0x08) and the temperature register (0x0B) back |

The part comes up in long distance mode with a 100 ms timing budget and the
interrupt active high, which are the table's own values.

### The distance modes and the timing budget

| Mode | Range in the dark (datasheet Table 4, 100 ms, white target) | Under strong ambient light |
|---|---|---|
| `Vl53Short` | 1360 mm | 1350 mm |
| `Vl53Long` | 3600 mm | 730 mm |

The part has a medium mode as well (2900 mm, 760 mm), which the ultra-lite
driver does not set and this driver does not. `range_max_mm` answers the dark
figure of the mode. A budget of 140 ms is the one the datasheet gives for 4 m
in long mode; the budgets the driver sets are the rows of ST's tables:

| Mode | Budgets in milliseconds |
|---|---|
| short | 15, 20, 33, 50, 100, 200, 500 |
| long | 33, 50, 100, 200, 500 |

15 ms is short mode only. The datasheet (section 2.5.2) allows 20 ms in short
mode only and 33 ms as the least that works in every mode, so the long table
has no 20 ms row although the ultra-lite driver writes one. A change of mode
keeps the budget when the new mode has a row for it and steps to 33 ms when it
has none; a budget the tables do not know becomes 100 ms. The timing budget
and the inter-measurement period must not be changed while the part ranges
(UM2356 section 2.5.2).

The inter-measurement period is the delay between two rangings. The register
holds ticks of the part's oscillator: the oscillator value
(`RESULT__OSC_CALIBRATE_VAL`, bits 9:0) times the period in milliseconds times
1.075. A period shorter than the budget makes the part range back to back.

### The range status

The part reports a status in bits 4:0 of the first result byte. UM2356
Table 4 lists eight values; the driver maps the codes the ultra-lite driver
knows onto the `RangeStatus` of devices-nv:

| Raw code | ST name | UM2356 number | Status | Distance |
|---|---|---|---|---|
| 9 | range complete | 0 valid | `RangeValid` | the range |
| 6 | sigma threshold check | 1 sigma fail | `RangeWeak` | the range |
| 4 | MSRC no target (signal) | 2 signal fail | `RangeWeak` | the range |
| 8 | minimum clip | 3 | `RangeOutOfRange` | 0 |
| 5 | range phase check | 4 out of bounds | `RangeOutOfRange` | 0 |
| 7 | phase consistency | 7 wrap target fail | `RangeOutOfRange` | 0 |
| 1, 2, 3, 17 | VCSEL continuity, watchdog, no VHV, multiple-clip | 5 hardware fail | `RangeFault` | 0 |
| 12, 13, 18 and the rest | ignore threshold, user ROI clip, stream-count sync | 9, 13, 10 and others | `RangeNoTarget` | 0 |

`uld_status_of_raw` answers the number of the third column for the codes the
ultra-lite driver lists and 255 for the others. The mapping of 8 (minimum clip)
to `RangeOutOfRange` and of 13 (the region clipped) to `RangeNoTarget` is this
driver's choice: `RangeStatus` has no value for "nearer than the least" or for
a misconfigured region.

The result is read as one block of 17 bytes from 0x0089, so the status, the
distance and the rates belong to one measurement. `Vl53Result` carries the raw
status, the stream count, the effective SPAD count, the ambient and signal
rates in kilo-counts a second and the range.

### The region of interest and the zones

The receiving array is 16 by 16 SPADs. The region of interest (ROI) is a
rectangle of SPADs, 4 by 4 at least, and it narrows the field of view. The
part keeps it as a centre SPAD number (0x007F) and a size byte
(0x0080, `(height - 1) << 4 | (width - 1)`). `spad_of(col, row)` gives the
number of the SPAD at a column and row, in the numbering of the datasheet's
figure as Pololu's header reproduces it: the top half of the array counts down
each column from `128 + 8 col`, the bottom half from `127 - 8 col`, and the
default centre is 199 (column 8, row 7). `set_roi_size` follows the
ultra-lite driver's `SetROI`: it puts the centre back to the part's optical
centre (the register 0x013E, written at manufacture) and to 199 for a region
wider or taller than 10. `set_roi` takes the centre as well.

`RangefinderZones` is a scan: 16 measurements of a 4 by 4 region each, at the
centres of the 16 tiles of a 4 by 4 grid, row by row. `zones_side` answers 4,
`zones_start` answers the time of 16 budgets in microseconds, and `zones_read`
runs the scan, which takes that long, and puts the region the part had back.
The part has one ranging region, so the "grid" is 16 successive ranges, not
one simultaneous picture, and an object that moves during the scan is seen at
more than one place. A scan is refused while the part ranges continuously.
The lens inverts the scene, so the grid is the mirror image of what is in
front of the part. The centre of a region other than 16 by 16 is computed as
column plus half the width and row plus half the height less one, which is
how 199 comes out for the full array; that rule for the other sizes is NOT
VERIFIED, because no document gives it.

### The calibrations

UM2356 chapter 3 asks for three calibrations at the production line: RefSPAD,
offset, cross-talk. The offset corrects for the solder and a cover glass; the
datasheet asks for it in all cases. The cross-talk compensates a cover glass.

`calibrate_offset(target_mm)` clears the three offset registers, takes the
average of 50 results, and sets the offset to the target less the average
(UM2356 section 3.2.2 asks for a 17 % grey chart at 140 mm in the dark).
`calibrate_xtalk(target_mm)` clears the plane offset, averages 50 results and
writes `512 * signal * (1 - distance / target) / SPADs`, with the signal and
SPAD count averaged, as the ultra-lite driver does; the target is the distance
the cover glass's ranging curve ends at (section 3.3.3). Both answer the value
they set, or `None`. Both stop ranging when they finish. **Their results on a
part are NOT VERIFIED**: the tests check the arithmetic against sums worked by
hand and the register writes against ST's list, and no part has been
calibrated.

The offset has two fraction bits in 13 bits of two's complement;
`offset_mm` reads it as C divides, toward zero. The cross-talk is kilo-counts
a second in 7.9 fixed point, shown in counts a second.

## What the package contains

| Module | What is in it |
|---|---|
| `vl53l1x` | The type `Vl53l1x` that holds a board's bus, timer and pins, the trait `Vl53l1xPort`, the bring-up as named functions, the tables of the timing budget, the modes, the region and zones, the calibrations, the thresholds, the measurements and the range status |

## How to choose an entry point

A program on a board uses `Vl53l1x`, built by `attach` with the board's I2C
bus, timer and GPIO and the wiring, as a board's device table builds it. The
wiring's `en` is the XSHUT pin and its `irq` is GPIO1. A program keeps the
sensor in a `var`: starting and stopping continuous ranging, and moving the
address, replace the state it holds.

| Method | What it does |
|---|---|
| `attach(bus, timer, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()`, `begin_io(io_2v8)` | cycles XSHUT if the wiring has the pin, and runs `start`; `begin` sets the pads to AVDD; whether the part came up |
| `mode()`, `set_mode(m)` | the distance mode, short or long |
| `budget_ms()`, `set_budget_ms(ms)` | the timing budget |
| `intermeasure_ms()`, `set_intermeasure_ms(ms)` | the inter-measurement period, 1 to 65535 ms |
| `range_start()` | `Rangefinder`: starts ranging and answers the budget in microseconds; 0 when continuous ranging runs; negative when the part did not take it |
| `range_read()` | `Rangefinder`: waits for the result, at most 1.1 s, and answers the distance with its status; stops a measurement that `range_start` began; `None` when the part did not answer |
| `range_max_mm()` | `Rangefinder`: 3600 in long mode, 1360 in short mode, 0 when the part did not answer |
| `zones_side()`, `zones_start()`, `zones_read(out)` | `RangefinderZones`: 4, the scan time, the scan |
| `measure_now()`, `result_now()` | one blocking measurement with every figure; the result of ranging that runs |
| `continuous_begin()`, `continuous_end()` | continuous ranging at the inter-measurement period; stop |
| `data_ready()`, `gpio1_level()` | the result flag, and the level of GPIO1 |
| `set_interrupt_high(high)`, `interrupt_high()` | the interrupt polarity (active high by default) |
| `roi()`, `set_roi_size(w, h)`, `set_roi(r)` | the region of interest |
| `offset_mm()`, `set_offset_mm(mm)`, `xtalk_cps()`, `set_xtalk_cps(cps)` | the stored calibrations |
| `calibrate_offset(target_mm)`, `calibrate_xtalk(target_mm)` | the calibrations; NOT VERIFIED on a part |
| `temperature_update()` | one full VHV search, after a large change of temperature |
| `signal_limit_kcps()`, `sigma_limit_mm()` and their setters | the limits that turn a range into a warning |
| `set_threshold(low, high, window, no_target)`, `set_interrupt_new_sample()` | the distance threshold interrupt, and back to "new sample ready" |
| `move_to(addr)` | gives the part another I2C address, kept until the next power-up or XSHUT |
| `device_id()` | `Identifiable`: 0xEACC, with the mask revision as `rev` |
| `reg(r)` | one register |

A program that holds its own bus, or a test with a double of the part,
implements `Vl53l1xPort` over its bus and calls the module's functions, which
take the port: `vl53l1x.start(port, true)`, `vl53l1x.measure(port)`. The
framing is `i2c_read_into` and `i2c_write_from`, generic over `I2cBus`.

## The rules a user needs

1. The part boots in 1.2 ms. `begin` waits 2 ms after each edge of XSHUT when
   the wiring has the pin and does not wait after the supply; the program
   waits.
2. After XSHUT the part is back at 0x29. Several sensors on one bus start with
   all XSHUT pins low, then raise one, `begin` it, `move_to` another address,
   and go on to the next.
3. `begin` sets the I2C pads to AVDD, which the common breakouts expect;
   `begin_io(false)` leaves them at 1.8 V. The register 0x002F, which holds
   the same choice for the GPIO pad, is left as the table has it.
4. `range_start` does not wait. It answers the budget; `range_read` waits for
   the result; `range_read_after(waited_us, measure_us)` of the trait refuses a
   read that comes early. In continuous ranging `range_start` answers 0 and
   starts nothing.
5. A single measurement is the part's autonomous ranging started and stopped:
   the ultra-lite driver has no single-shot mode. A measurement started this
   way takes the budget plus the part's own set-up, and a first result after a
   start can carry stream count 0, which Pololu's driver marks as "no wrap
   check".
6. The interrupt is cleared when a result is read; without it the next ranging
   does not update the result.
7. A reading with `RangeWeak` has a distance and a warning: the return was
   below the limit the part checks. `RangeOutOfRange`, `RangeNoTarget` and
   `RangeFault` have no distance.
8. A poll of the data-ready flag that the part does not answer counts as "not
   ready" and is tried again; a part that stays silent ends the wait at the
   timeout, 1.1 s for a result.
9. A scan moves the region of interest and puts it back. A program that uses
   the region of its own sets it again after a scan that failed, because the
   restore is the last transaction of the scan and can fail with it.
10. A cover glass or a window in front of the sensor needs the offset and the
    cross-talk calibration, which are NOT VERIFIED here, and the RefSPAD
    calibration, which is not implemented.

## What is not included

NOT IMPLEMENTED:

- The RefSPAD calibration (`VL53L1_PerformRefSpadManagement`), which UM2356
  asks for first when a cover glass is fitted, and the loading of the
  calibration structure (`VL53L1_SetCalibrationData`). The ultra-lite driver
  has neither; the factory's SPAD values stay in force.
- The medium distance mode, and the full API's single-shot mode with its
  manual calibration steps.
- The range gain of 2011/2048 that Pololu applies (from the full API's tuning
  table); the ultra-lite driver, which this one follows, reports the part's
  distance as it is.
- Fast-mode-plus I2C (bits 2 and 5 of 0x2D), the GPIO pad supply (0x2F), and
  the signal and ambient rates per SPAD that the ultra-lite driver derives.
- The Pololu-style first-result mark ("range valid, no wrap check"): the
  stream count is in `Vl53Result`, and a program that wants it tests it.
- A zone grid of 8 by 8 or 16 by 16 and a ROI scan of another shape than the
  4 by 4 tiles.
- The async forms of `Rangefinder` and `RangefinderZones`: a handle implements
  no trait spelled with `async fn` (SPEC section 14.5).

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `Rangefinder`, `RangefinderZones` and `Identifiable` traits this driver
  implements, the wiring record and the register helpers.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2cBus`, `GpioOut` and `Timer` traits and the board handles.
- [vl53l0x-nv](https://novo-lang.org/packages/vl53l0x-nv) drives the VL53L0X, the
  VL53L1X's 2 m predecessor, as the same `Rangefinder`.
- [hcsr04-nv](https://novo-lang.org/packages/hcsr04-nv) drives the HC-SR04
  ultrasonic module as the same `Rangefinder`.

## Tests

`novo test` runs the module over a double of the part: a flat register file of
16-bit addresses with the model ID of the datasheet, a boot status that
answers 0 for some reads, the data-ready flag that a started ranging raises
after some polls and that a write to 0x86 clears (and re-raises while ranging),
a moveable I2C address, and a log of every transaction. The suite checks the
framing of the bus, the 91 bytes of the table against ST's list and their four
bursts, every step of the initialisation in order, each table row of the
timing budget in both modes, the budget kept across a change of mode, the
inter-measurement arithmetic against hand-worked ticks, the limits, the
thresholds (including the ultra-lite driver's sticky window), the SPAD
numbers of the array's corners and centre, the region encoding, the 16 centres
of a scan, every raw status of the range, the result block against bytes worked
by hand, single and continuous ranging, the offset and cross-talk arithmetic
with the cases that clip, and the type over the host's board handles. The
initialisation, the budget and mode writes, the thresholds, the scan and the
calibrations each run once for every transaction of a good run with that
transaction failing, and must answer that they failed, except for the
transactions the driver is meant to survive (a poll of the flag the part did
not answer, and the stop after a result that was read), which the test names;
so no bus error is read as data. The suite covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

## Sources

The sources read for this driver, for their fixes and not ported, are the
datasheet and the user manual above; ST's ultra-lite driver as the
stm32duino/VL53L1X copy of STSW-IMG009 carries it (BSD 3-clause, ST), for the
register list, the default configuration, the timing-budget tables, the
distance-mode registers, the ROI, the thresholds and the calibrations; and
Pololu's `vl53l1x-arduino` (MIT licence), for the result block, the status
mapping and the SPAD table. Zephyr's `drivers/sensor/st/vl53l1x` wraps ST's
full library and the `vl53l1x-uld` Rust crate is a port of the ultra-lite
driver; neither was opened. The pitfalls taken from them are named in
`src/vl53l1x.nv` at the sites that avoid them:

- The ultra-lite driver overwrites its status with each call, so a part that
  stops answering half-way through the 91 register writes reports success; its
  I2C read loops until the part acknowledges; and `SensorInit` and the
  temperature update spin on the data-ready flag with no limit. Here every
  transfer is checked and every wait ends.
- Its distance-mode getter leaves its answer unset when the phase-calibration
  register holds neither mode's value, and the budget setter branches on it.
  `mode_read` answers `None`.
- Its mode setter re-applies the old budget, which fails silently for a 15 ms
  budget going to long mode and leaves the short timeouts in the long mode's
  registers. Here the budget steps to 33 ms.
- Its inter-measurement setter scales by 1.075, its getter divides by 1.065 and
  truncates the register to 16 bits first. Here both use 1.075 and the whole
  register.
- Its offset calibration adds fifty distances in a 16-bit variable and reads
  the data-ready flag through a variable it never sets. Here the sums are full
  width and each result is waited for.
- Its threshold setter ORs the new window into the old one, so "inside" then
  "outside" leaves "inside", and never clears the no-target bit. Here the
  register is written whole.
- It reads a distance and a status from separate transactions, which can belong
  to two measurements. Here the 17 bytes are one transaction.
- Its 91-byte table overwrites the I/O supply bit, so the bit is set after
  the table.

## Licence

Apache-2.0.
