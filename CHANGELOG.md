# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Vl53l1xPort` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Vl53l1xPort` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Vl53l1xPort` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Vl53l1xPort for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `irq_level_of`, `rd`, `rd16`, `wr`, `wr16`, `upd`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Vl53l1xPort` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- `i2c_setup` stays a function of the module and is not deprecated: it makes the shutdown pin an output as well, which `wiring_i2c_setup` does not.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 22,404 bytes before and 22,308 after on the read-out (nrf52-qemu: 11,836 and 11,868).

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed.  `examples/range_reader.nv` adds the `RangeBlinded` and `RangeSaturated` arms that devices-nv 0.4.0 gives `RangeStatus`.

The device program builds for the boards its README names, flash nrf52840-dk 22,432 B before and 22,384 B after; nrf54lm20-dk 23,928 B before and 23,880 B after; nrf52-qemu 11,848 B before and 11,848 B after.

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
  map. Built for the nRF52840-DK it is 26,920 B of flash and for the
  nRF54LM20-DK 28,360 B; before the Nordic buses landed, with the reading path
  left out, it was 16,888 B and 18,152 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  sensor. The nRF52840 DK drives I2C with TWIM0 and the nRF54LM20 DK with
  TWIM22, so the device program's images carry the reading path; a reply from
  the sensor over either bus is NOT VERIFIED.
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
