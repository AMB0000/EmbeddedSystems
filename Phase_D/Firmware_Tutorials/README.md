# Firmware Tutorials

Reference material the instructor provided for the Phase D firmware. Everything here is third-party — my notes on using it live in the parent `Phase_D/README.md`.

## Instructor videos

- **STM32CubeIDE import + USB flashing walkthrough** — how to bring the `3Wheel_Balance_noEncoder` project into CubeIDE and flash the board over USB-C with STM32CubeProgrammer.
  https://youtu.be/xApW40-5goQ

- **Chassis paper validation** — how to validate the laser-cut chassis design on paper before cutting acrylic.
  https://youtu.be/5C3L4LZVVkk

## Reference article

- **engredu — 2-Wheel Balance Robot.** Instructor's writeup of the full project, used as the guideline for this Phase D report.
  http://engredu.com/2026/05/01/2-wheel-balance-robot/

## Flashing procedure (summary)

1. Flip the **5V Selector (S3) to USB** — cuts the motor rail so wheels stay still while programming.
2. Plug USB-C from the board to the laptop.
3. Hold **BOOT0 (S4)**, tap **RESET (SW2)**, release BOOT0 → STM32 enters DFU mode.
4. Open **STM32CubeProgrammer**, connection type **USB**, click Connect.
5. Load the `.hex` from the built CubeIDE project, click Download.
6. Tap RESET to run the new firmware.
