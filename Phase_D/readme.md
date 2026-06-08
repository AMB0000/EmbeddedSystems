# Phase D — Final Documentation

**Project:** Two-wheel self-balancing robot
**Class:** Embedded Systems, University of Denver
**Author:** Ali Behbehani

## What I built

A two-wheel robot that's supposed to balance itself like a tiny Segway. There's a custom PCB I designed in KiCad, a laser-cut acrylic chassis I cut at the makerspace, two TT gearmotors with rubber wheels, and a 2-cell 18650 pack on top. The STM32 on the board reads tilt from an IMU and runs a PID loop to keep itself upright.

It works in the sense that everything powers on, the firmware runs, and the motors spin in the right direction when it starts to tip. It balances for a second or two before falling over, so I'd call it a partial success. More on why in the issues section.

## The chassis

I designed the chassis in two tiers from 3mm clear acrylic so I could fit everything compactly. The bottom layer holds the PCB and the two motors. The top layer holds the battery pack. There's a small ball-caster looking piece at the back that I'm using more as a counterweight than as an actual third wheel.

The motors are TT gearmotors with the standard yellow 65mm wheels. They sit in 3D-printed brackets that screw to the bottom deck.

Everything is held together with M2 hardware.

## The PCB

Two-layer board, designed in KiCad 10. I assembled it by hand with hot air and a soldering iron, which meant a few joints needed rework after the first power-on (you can see the touched-up spots in the side photo).

The main components:

- **MCU:** STM32F401RBTx — 84 MHz Cortex-M4, LQFP-64. Programmed over SWD with an ST-Link.
- **Motor driver:** TB6612FNG — dual H-bridge, up to 1.2 A per channel, separate PWM and direction pins.
- **IMU:** MPU-6050 on a breakout board, plugged into the on-board connector (CONN_MPU6050). I²C1 at the default 0x68 address. There's also an interrupt line wired back to PC13.
- **Regulators:** LD1117S33TR (3.3 V) and LD1117S50TR (5 V) LDOs in SOT-223.
- **Clock:** 16 MHz crystal (Y3) on OSC_IN/OSC_OUT.
- **USB-C:** USB 2.0 receptacle with a USBLC6-2SC6 for ESD protection. USB DM/DP are wired to the STM32's USB FS pins so the board can be powered or talked to over USB-C.
- **Status LEDs:** USB, 3V3, 5V, user LED, plus a WS2812B addressable LED on a TIM5 PWM pin.
- **Protection:** PTC fuse (F1) on the input rail.
- **Buttons:** reset (SW2), boot0, regulator switch.

The board also has connectors I designed in but didn't end up using for this build:

- **ENCODER_CONN** — wheel encoder header (quadrature on PB6/PB7). My TT gearmotors don't have encoders, so this never got hooked up.
- **MT6701_CONN** — connector for an MT6701 magnetic angle sensor on I²C3.
- **ESP_Conn** — UART link to an ESP32 (USART1 TX/RX) for future wireless telemetry.

Power: two 18650 Li-ions in series, about 7.4 V nominal, into the battery JST. From there the 5 V LDO feeds the motor driver's logic side and the 3.3 V LDO feeds the MCU and IMU. The motors run directly off the battery rail through the H-bridge. No on-board charger — cells are charged externally.

## The spacer story

This one cost me a few hours. The mechanical plan assumed I'd be using M3 standoffs to mount the PCB to the chassis. When the board came back I realized the mounting holes (H1 through H4) were drilled at 2.2mm, which is M2, not M3. The standoffs I'd bought were way too big.

Re-spinning the board wasn't realistic with the deadline coming up, and even if I dropped in M2 standoffs the board still wouldn't sit flat on the deck — the bottom-side solder joints would touch the acrylic and short something out.

So I designed a tiny custom spacer and 3D-printed four of them: 2.4mm inner diameter so an M2 screw slides through, 5mm outer diameter, and 2.2mm tall. The 2.2mm height was the smallest I could get away with that still cleared the joints. PLA at 100% infill. Total print time was a few minutes. STL is in the repo.

## Firmware

I used STM32CubeIDE and the ST HAL. CubeMX generated the peripheral init. I program the chip over SWD with an ST-Link.

The peripherals I'm actually using are I²C1 for the IMU (pins PB8/PB9 on the connector, internally PB6/PB7 on the chip), TIM2 channels 1 and 2 for the two motor PWMs, GPIOs for the H-bridge direction pins and standby line, GPIO for the user LED, and SysTick to keep the control loop running at a fixed rate. There's also a WS2812B on TIM5_CH1 that I haven't done much with yet.

The control loop runs at 200 Hz. Every 5ms it does this:

1. Reads accelerometer and gyro from the IMU
2. Runs a complementary filter to estimate the tilt angle (`angle = 0.98 * (angle + gyro*dt) + 0.02 * accel_angle`)
3. Computes a PID output from the angle error (setpoint is 0° — upright)
4. Clamps the output and sends it as signed PWM to both motors. The sign picks the H-bridge direction, the magnitude sets the duty cycle.

I'm sending the same command to both motors right now since I only care about balancing, not driving.

PID gains I ended up using: *fill in your final Kp / Ki / Kd here.*

## What worked and what didn't

The board powers on cleanly, the 3.3V rail is steady under load, ST-Link talks to the chip, I can read the IMU's WHO_AM_I register, and both motors spin in both directions when I tell them to. So the electronics are good and the firmware is wired up correctly.

Where it falls apart is the actual balancing. It holds upright for maybe one to three seconds before the angle runs away and it tips over. It doesn't recover from any kind of push. The reasons are in the problems section below.

## Problems and fixes

This is the part where I'm being honest about what went wrong. Every one of these cost me time and most of them taught me something I wouldn't have learned if the build had just worked first try.

### 1. PCB mounting holes were the wrong size

The mechanical plan I drew up assumed M3 standoffs. When the board came back from fab I tried to drop it onto the chassis and the screws wouldn't go through. I measured the holes in KiCad — H1 through H4 were drilled at 2.2mm, which is the clearance for an M2, not M3. I'd missed it in the design review.

Re-spinning the board wasn't an option with the deadline coming up. Switching to M2 hardware fixed the screw problem, but the board still wouldn't sit flat on the deck because the bottom-side SMD joints would touch the acrylic and short out.

**Fix:** designed a custom 3D-printed spacer in CadQuery — 2.4mm ID for the M2 to slide through, 5mm OD for surface area, 2.2mm tall (just enough to clear the joints). Printed four in PLA at 100% infill in maybe five minutes. STL is in the repo. Honestly the spacers turned out cleaner than the off-the-shelf ones would have been.

**Lesson:** double-check the drill size on every mounting hole before sending the board off. KiCad shows it in the status bar when you click the pad — would have caught this in two seconds.

### 2. Regulator output was unstable on first power-on

First time I powered the board the 3.3V rail was bouncing around. The MCU wouldn't boot reliably.

**Fix:** reflowed the buck IC with hot air. One pin wasn't fully wetted — you couldn't see it without a magnifier. Rail came up clean after that.

**Lesson:** when something doesn't work after assembly, look at the joints under magnification before you start swapping components or rewriting code.

### 3. KiCad version mismatch

When I tried to reopen my own board file on a different machine, KiCad refused — the file had been saved in version 10 and the other machine had an older release.

**Fix:** installed KiCad 10.0.3 on every machine I touch the project from. Pinned the version in my notes so this doesn't surprise me again.

**Lesson:** if you're going to use a brand-new KiCad version, write it down somewhere in the repo so future-you knows.

### 4. Robot oscillates and falls instead of holding

This is the big unresolved one. With the gains I have, the robot tips, overcorrects, and the angle runs away.

There are three things working against me here:

- **Motor backlash.** TT gearmotors have a noticeable amount of slop in the gear train and a deadband at low PWM. Small PID corrections just don't move the wheels — by the time the output is big enough to actually do something the angle has already gone too far.
- **High center of mass.** The batteries are on the top deck. Higher CoM is actually easier to balance in theory (longer time constant), but it asks more torque from the wheels, which my motors don't really have.
- **Undertuned PID.** Without live tuning I have to re-flash for every gain change. I didn't get nearly enough iterations.

**Partial fix:** raised Kp and Kd until the response was at least visible. It buys me a second or two of upright before it tips.

**Real fix (didn't get to):** add wheel encoders for a velocity feedback term, move the batteries to the bottom deck to drop the CoM, and add a UART/Bluetooth channel to change gains live.

### 5. PID tuning was painfully slow

Related to the above but worth calling out separately. Every gain change meant: edit the code, build, flash, set the robot upright, watch it tip, repeat. Two minutes a try, and you need dozens of tries.

**Fix (partial):** put a `#define` block at the top of `main.c` with the three gains so I only had to touch one spot.

**Real fix (for next time):** expose the gains over UART so I can tweak them from a serial terminal while the robot is running.

### 6. *Add any other problems you ran into here*

Stuff I'd ask yourself: did you have any issues with wire polarity on the motors? IMU axis orientation? Any ground loops or noise on the analog rails? Anything weird with the buttons bouncing? Add those if they happened.

## If I were doing it again

Add wheel encoders (or swap the TT motors for ones that come with encoders) so the controller has velocity feedback, not just tilt. Move the battery to the bottom deck and put the PCB on top to drop the CoM. Add a UART or Bluetooth channel to tune PID gains live. Maybe split it into a cascade controller — outer loop on position, inner loop on angle — instead of a single PID. And upgrade the IMU to one with onboard sensor fusion so I don't have to trust my complementary filter.

## Phase-by-phase timeline

The project was broken into four phases over the quarter:

**Phase A — Concept and requirements.** Picked the project (a balancing robot), wrote up what it had to do (stay upright on two wheels, no tether, run off batteries), and put together a block diagram and a parts shortlist.

**Phase B — Schematic and PCB.** Designed the schematic in KiCad, picked footprints, laid out the board, ran DRC, generated gerbers, and sent the board out for fab. This is the folder `Phase_B_Final_Board/FINAL_PHASE_B/` in the repo.

**Phase C — Assembly and bring-up.** Hand-assembled the board (hot air + iron), powered it on, fixed the reg-rail problem, got the ST-Link to flash the chip, and wrote the first cut of firmware to read the IMU and spin the motors.

**Phase D — Integration, tuning, documentation.** Mounted everything to the chassis (this is where the spacer issue showed up), got the PID loop running, attempted to balance, and wrote this document.

## Pinout

This is the real pinout pulled from the KiCad project (`FINAL_PHASE_B__1_.kicad_pcb`). The STM32F401RBTx is reference U1.

**Programming / debug**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 46 | SWDIO | SWDIO |
| 49 | SWCLK | SWCLK |
| 7 | NRST | RESET (tied to reset button) |
| 60 | BOOT0 | BOOT0 |

**Clock**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 5 | OSC_IN | OSCIN (16 MHz crystal) |
| 6 | OSC_OUT | OSCOUT (16 MHz crystal) |

**IMU — I²C1 + interrupt** (MPU-6050 on breakout)

| Pin | STM32 function | Net |
| --- | --- | --- |
| 61 | I2C1_SCL | IMU_SCL |
| 62 | I2C1_SDA | IMU_SDA |
| 2 | PC13 | IMU_INT |

**Motor driver — TB6612FNG**

| Pin | STM32 function | Net | Role |
| --- | --- | --- | --- |
| 15 | TIM2_CH2 | MD_PWMA | Motor A speed (PWM) |
| 17 | PA3 | MD_AIN1 | Motor A direction 1 |
| 16 | PA2 | MD_AIN2 | Motor A direction 2 |
| 21 | TIM2_CH1 | MD_PWMB | Motor B speed (PWM) |
| 22 | PA6 | MD_BIN1 | Motor B direction 1 |
| 23 | PA7 | MD_BIN2 | Motor B direction 2 |
| 20 | PA4 | MD_STBY | Driver enable (hold high to enable) |

**Wheel encoders (quadrature) — not used in this build**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 58 | PB6 | ENC_B |
| 59 | PB7 | ENC_A |

**MT6701 magnetic angle sensor — I²C3 (not used in this build)**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 40 | I2C3_SDA | MT_SDA |
| 41 | I2C3_SCL | MT_SCL |
| 37 | PC6 | MT_PWM |

**ESP32 link — USART1 (not used in this build)**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 42 | USART1_TX | ESP_RX |
| 43 | USART1_RX | ESP_TX |

**USB-C — USB FS**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 44 | USB_OTG_FS_DM | ESD_D− |
| 45 | USB_OTG_FS_DP | ESD_D+ |

**LEDs**

| Pin | STM32 function | Net |
| --- | --- | --- |
| 14 | TIM5_CH1 | WS_DIN (WS2812B addressable LED) |
| 27 | PB1 | USER_LED |

**Power**

| Pin | Function | Net |
| --- | --- | --- |
| 13 | VREF+ | +3.3V |
| 19, 32, 48, 64 | VDD | +3.3V |
| 12 | VSSA | GND |
| 18, 31, 47, 63 | VSS | GND |
| 30 | VCAP1 | Internal cap |
| 1 | VBAT | Tied off (no battery backup used) |

## Cost breakdown

Rough numbers from what I spent. These are USD and approximate.

| Item | Qty | Unit | Subtotal |
| --- | --- | --- | --- |
| Custom PCB (JLCPCB, 2-layer, 5 pcs) | 1 order | ~$5 | $5 |
| STM32F401RBTx (LQFP-64) | 1 | ~$5 | $5 |
| MPU-6050 breakout | 1 | ~$3 | $3 |
| TB6612FNG motor driver | 1 | ~$2 | $2 |
| LD1117S33TR + LD1117S50TR LDOs | 2 | ~$1 | $2 |
| USBLC6-2SC6 ESD protection | 1 | ~$1 | $1 |
| USB-C receptacle (14-pin) | 1 | ~$1 | $1 |
| WS2812B addressable LED | 1 | ~$0.50 | $0.50 |
| 16 MHz crystal + load caps | 1 set | ~$1 | $1 |
| Passives (R, C), LEDs, buttons, PTC fuse | mix | ~$5 | $5 |
| JST connectors + headers | mix | ~$3 | $3 |
| TT gearmotor + wheel | 2 | ~$4 | $8 |
| 18650 Li-ion cell (2700 mAh) | 2 | ~$7 | $14 |
| 2-cell 18650 holder | 1 | ~$2 | $2 |
| 3 mm clear acrylic sheet (laser cut at makerspace) | 1 | ~$5 | $5 |
| M2 hardware (screws, nuts, metal standoffs) | mix | ~$5 | $5 |
| PLA filament for spacers + motor brackets | small | ~$1 | $1 |
| **Total** | | | **~$63** |

> Update these with what you actually paid if you saved receipts (eFatoorah, hint hint).

## KiCad screenshots

Drop these in `Final-Project/images/` and they'll render below. The image names are just suggestions.

**Schematic:**
```
![Schematic](images/schematic.png)
```

**PCB layout (top copper):**
```
![PCB top](images/pcb_top.png)
```

**PCB layout (bottom copper):**
```
![PCB bottom](images/pcb_bottom.png)
```

**3D render (front):**
```
![3D front](images/pcb_3d_front.png)
```

**3D render (back):**
```
![3D back](images/pcb_3d_back.png)
```

> In KiCad: schematic → File → Export → SVG/PNG. PCB → File → Export → SVG, or View → 3D Viewer → File → Export current view for renders.

## Photos and video

**Assembled robot:**
```
![Top view](images/robot_top.jpg)
![Side view](images/robot_side.jpg)
```

**Demo video:** record a 10–20 second phone clip of it trying to balance, upload to YouTube as "Unlisted," and drop the link here:

```
[Watch the demo](https://youtu.be/YOUR_VIDEO_ID)
```

## References and datasheets

- STM32F401xB/xC datasheet — https://www.st.com/resource/en/datasheet/stm32f401rb.pdf
- STM32F401 reference manual (RM0368) — https://www.st.com/resource/en/reference_manual/rm0368-stm32f401xbc-and-stm32f401xde-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
- TB6612FNG dual motor driver — https://www.sparkfun.com/datasheets/Robotics/TB6612FNG.pdf
- MPU-6050 datasheet — https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Datasheet1.pdf
- LD1117 LDO datasheet — https://www.st.com/resource/en/datasheet/ld1117.pdf
- USBLC6-2SC6 ESD protection — https://www.st.com/resource/en/datasheet/usblc6-2.pdf
- WS2812B addressable LED — https://cdn-shop.adafruit.com/datasheets/WS2812B.pdf
- TT gearmotor specs (Adafruit) — https://www.adafruit.com/product/3777
- ST-Link V2 user manual — https://www.st.com/resource/en/user_manual/cd00262073.pdf
- KiCad 10 docs — https://docs.kicad.org/
- STM32CubeIDE — https://www.st.com/en/development-tools/stm32cubeide.html
- Phil's Lab (background on STM32 + PID for balancing) — https://www.youtube.com/@PhilsLab

## Repo layout

```
EmbeddedSystems/
├── Phase_B_Final_Board/FINAL_PHASE_B/   KiCad project (schematic, PCB, BOM)
├── Final-Project/                       firmware, CAD, this doc, images
│   └── images/                          screenshots and photos
├── DMA_STM32/                           DMA experiments
├── Lab_01/                              early labs
├── All_Labs_CHALLENGE_COMPLETED/        challenge labs
└── Phils_Lab_Tutorial/                  reference tutorials
```
