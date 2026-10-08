# Changelog

## 0.1.0 (2026-10-08)

The first release: a driver for NXP's P3T1755 digital temperature sensor
on an I2C bus.

- `P3t1755`, the type a board's device table builds from its wiring,
  implementing devices-nv's `Thermometer` and `Identifiable`, with its
  own methods for shutdown and resume, the one-shot conversion, the
  conversion period, the high and low limits, and the power-up values.
- The register access as functions over any `I2cBus`, which the type's
  methods call, and the conversions between a register word and a
  temperature, with the sign taken from bit 11 of the count.
- The host suite runs the functions over a double of the part and the
  type's methods over the host's board handles.
