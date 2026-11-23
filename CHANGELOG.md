# Changelog

## 0.1.3 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.0) and devices-nv ^0.4.0 (was ^0.3.0), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The FRDM-MCXN947's `tests/thermometer_probe.nv`, which reads the part through the board's devices block, builds against the package: flash 26,680 B before and 26,692 B after.

## 0.1.2 — 2026-10-25

A call that passes a variable to a `var` parameter now writes the `var` marker, which the 0.21.1 compiler asks for (E2050).  The package needs Novo 0.21.1 or later.  No signature and no behaviour changed.

## 0.1.1 (2026-10-08)

The toolchain floor is 0.19.4.  A build that resolves embedded-hal-nv
afresh takes 0.3.2, whose floor is 0.19.4, so 0.1.0's floor of 0.19.3
named a toolchain the dependency does not build on.  Nothing else
changed.

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
