# Changelog

## 0.1.0

The first release: a driver for ST's VL53L1X long-distance time-of-flight
ranging sensor on an I2C bus.

- `Vl53l1x`, the type a board's device table builds from the board's I2C bus,
  timer and GPIO handles and the wiring, implementing devices-nv's
  `Rangefinder`, `RangefinderZones` and `Identifiable`. `Vl53l1xPort` is the
  trait a test or a program implements to run the module's functions over its
  own bus.
- The initialisation as named steps in the order of ST's ultra-lite driver:
  the model check (0xEACC), the wait for the boot, the 91-byte default
  configuration in four bursts, the I/O supply bit, and one VHV cycle.
- The short and long distance modes and the timing budgets of ST's tables
  (short 15 to 500 ms, long 33 to 500 ms), the budget kept or stepped to 33 ms
  across a change of mode, and the inter-measurement period.
- Ranging, single measurement or continuous, with the 17-byte result block read
  in one transaction: the range, the status mapped from ST's raw codes onto
  `RangeStatus`, the stream count, the effective SPADs, and the signal and
  ambient rates.
- The region of interest from 4 by 4 up to 16 by 16 SPADs, the SPAD numbering
  of the array, and a 4 by 4 zone scan as `RangefinderZones`.
- The offset and cross-talk values and their calibration functions (results NOT
  VERIFIED on a part), the signal and sigma limits, the distance threshold
  interrupt, the interrupt polarity, the temperature update, XSHUT and the
  change of the I2C address.
- The device program `examples/range_reader.nv`, a distance reader with a zone
  map. Built for the nRF52840-DK it is 16,888 B of flash and for the
  nRF54LM20-DK 18,152 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  sensor. Neither Nordic board has a driver for its two-wire interface (TWIM)
  at this release, so the device program builds for them and finds an absent
  bus.
- ST publishes no register map. The register list, the default configuration
  and the sequences are those of ST's ultra-lite driver as the
  stm32duino/VL53L1X copy of it carries them; the range status is mapped from
  its raw codes with this driver's own choice for the minimum clip and the
  clipped region.
- Not implemented: the RefSPAD calibration and the calibration structure, the
  medium distance mode, the full API's single-shot mode, the range gain of the
  full API, fast-mode-plus I2C, the GPIO pad supply, a zone grid other than
  4 by 4, and the async forms of the traits.
- `devices-nv` is the registry range `^0.3.3`, the first release with the
  `Rangefinder` and `RangefinderZones` traits. `embedded-hal-nv` is the
  registry range `^0.3.2`.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module also
  checks at `rt` and `system`.
