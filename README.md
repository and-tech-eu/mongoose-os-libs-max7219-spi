# MAX7219/MAX7221 SPI Driver

A Mongoose library for MAX7219, an 8-digit LED Display Driver.

## Implementation details

The chip is straight forwardly implemented, and allows for daisy chaining
multiple devices (by tying `DOUT` of the first chip to `DIN` of the second
chip, while connecting `CS`, `CLK`, `VCC` and `GND` to all chips).

After initialization, bytes can be written to so called `digits`, which are
the individual lines of an 8x8 LED display, or the actual digits of a 7-segment
display.

***NOTE***: The MAX7219 is a 5V device, and as such level shifters are
necessary between the `MOSI`, `SCLK` and `CS` pins of the MCU and the pins
on the chip(s). It may work without, but it's not recommended!

## API Description

Create the object with `mgos_max7219_create()`, and if there are multiple
devices connected, set the correct number with `mgos_max7219_set_num_devices()`.
When you're done using the display(s), call `mgos_max7219_destroy()` to turn
them off and clean them up.

***`mgos_max7219_set_intensity()`*** Intensity of all displays will be set,
where the argument is between 0 (very dim) to 15 (very bright). 

***`mgos_max7219_set_mode()`*** Setting the `mode` is done with a boolean
mode argument. If `true` is specified, all connected devices are set into what
is called `Code B decoding`, in which the values map internally to segments of
a 7-segment display, for example writing 0 will light up the segments needed to
draw a 0 on a 7-segment display. There are only 15 values in this system, see
the header file for details. The mode is initialized as `false`, in other words
do not use `Code B`, and now each bit on the 8-bit values correspond to one LED,
typically on an 8x8 LED display.

Writes can be performed in three ways:

***`mgos_max7219_write_raw()`*** writes an 8-bit value to a certain line (`digit`) of
a certain device (`deviceno`). This will only work if the `mode` was set to `false`.

***`mgos_max7219_write_digit()`*** writes a number to a certain position (`digit`) of
a certain device (`deviceno`). This will only work if the `mode` was set to `true`.

***`mgos_max7219_write_line()`*** writes an array of bytes, which must be of exactly
`num_devices` length, to the given line (`digit`) of all connected displays.


## Example application

```
#include "mgos.h"
#include "mgos_config.h"
#include "mgos_max7219.h"

const uint8_t program[] = {
  0,   153, 90,  60,  24,  24,  36,  66,  5,
  0,   24,  24,  255, 24,  24,  36,  66,  5,
  0,   24,  24,  60,  90,  153, 36,  66,  5,
  0,   24,  24,  255, 24,  24,  36,  66,  5,
};

void timer_cb(void *data) {
  struct mgos_max7219 *d        = (struct mgos_max7219 *)data;
  static uint8_t       waiting  = 0;
  static uint16_t      position = 0;

  if (!d) return;

  if (waiting > 0) {
    waiting--;
    return;
  }

  waiting = program[position * 9 + 8];
  LOG(LL_INFO, ("Writing program position %u, then waiting %.1f seconds", position, 0.1 * waiting));

  for (int i = 0; i < 8; i++) {
    mgos_max7219_write_raw(d, 0, i, program[position * 9 + i]);
    mgos_max7219_write_raw(d, 1, i, ~(program[position * 9 + i]));
  }

  position++;
  if (position == sizeof(program) / 9) position = 0;
}

enum mgos_app_init_result mgos_app_init(void) {
  struct mgos_max7219 *d = NULL;

  if (!(d = mgos_max7219_create(mgos_spi_get_global(), mgos_sys_config_get_max7219_cs_index())) {
    LOG(LL_ERROR, ("Could not create MAX7219 display"));
    return MGOS_APP_INIT_ERROR;
  }
  mgos_max7219_set_num_devices(d, 2);
  mgos_set_timer(100, true, timer_cb, d);

  return MGOS_APP_INIT_SUCCESS;
}
```

# Disclaimer

This project is not an official Google project. It is not supported by Google
and Google specifically disclaims all warranties as to its quality,
merchantability, or fitness for a particular purpose.
