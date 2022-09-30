# raspi-os

Read [Operating System development tutorials in Rust on the Raspberry Pi](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials).

## Building in contents

- `Makefile` targets:
  - `doc`: Generate documentation
  - `qemu`: Run the `kernel` in QEMU
  - `clippy`
  - `clean`
  - `readelf`: Inspect the `ELF` output
  - `objdump`: Inspect the assembly
  - `nm`: Inspect the symbols
  - `test`
  - `miniterm`: Read the signals from UART
  - `chainboot`: Be ready to transmute binary

use `BSP=rpi4 make {command}` for Raspi4

for Mac, use UART with `DEV_SERIAL=/dev/tty.usbserial-0001`

(for  **my** Mac, use UART with `DEV_SERIAL=/dev/tty.usbserial-AU01PR2F`)

## My notes

[here](notes.md)
