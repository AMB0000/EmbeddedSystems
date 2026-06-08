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

The board has:
- An STM32 as the main MCU, programmed over SWD with an ST-Link
- An IMU on I²C for the tilt reading
- A dual H-bridge to drive both motors
- A buck regulator to step the battery down to 3.3V for the logic
- Two tactile buttons for reset and boot
- JST connectors for the motors, the battery, and the IMU

The battery is two 18650 Li-ions in series, about 7.4V nominal. The motors run straight off the battery rail through the driver. There's no on-board charger — I just pull the cells and charge them externally.

> Fill in the exact part numbers (STM32 variant, motor driver, buck IC) from the KiCad BOM in `Phase_B_Final_Board/FINAL_PHASE_B/`.

## The spacer story

This one cost me a few hours. The mechanical plan assumed I'd be using M3 standoffs to mount the PCB to the chassis. When the board came back I realized the mounting holes (H1 through H4) were drilled at 2.2mm, which is M2, not M3. The standoffs I'd bought were way too big.

Re-spinning the board wasn't realistic with the deadline coming up, and even if I dropped in M2 standoffs the board still wouldn't sit flat on the deck — the bottom-side solder joints would touch the acrylic and short something out.

So I designed a tiny custom spacer and 3D-printed four of them: 2.4mm inner diameter so an M2 screw slides through, 5mm outer diameter, and 2.2mm tall. The 2.2mm height was the smallest I could get away with that still cleared the joints. PLA at 100% infill. Total print time was a few minutes. STL is in the repo.

## Firmware

I used STM32CubeIDE and the ST HAL. CubeMX generated the peripheral init. I program the chip over SWD with an ST-Link.

The peripherals I'm actually using are I²C1 for the IMU, two timer channels for the motor PWMs, GPIOs for the H-bridge direction pins and the button/LED, and SysTick to keep the control loop running at a fixed rate.

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

This is the mapping from STM32 pins to what they connect to on the board. **Fill in the actual pins from your KiCad schematic** — I don't know the exact ones you used. Below is the structure to copy.

| STM32 pin | Function | Connects to |
| --- | --- | --- |
| PB6 | I²C1 SCL | IMU SCL |
| PB7 | I²C1 SDA | IMU SDA |
| PA0 | TIM2_CH1 (PWM) | Motor driver, left PWM |
| PA1 | TIM2_CH2 (PWM) | Motor driver, right PWM |
| PA2 | GPIO out | Motor driver, left IN1 |
| PA3 | GPIO out | Motor driver, left IN2 |
| PA4 | GPIO out | Motor driver, right IN1 |
| PA5 | GPIO out | Motor driver, right IN2 |
| PA13 | SWDIO | ST-Link |
| PA14 | SWCLK | ST-Link |
| PC13 | GPIO in (pull-up) | User button SW2 |
| PC14 | GPIO out | Status LED |
| NRST | Reset | Reset button SW1 |

> To find the real pinout: open `FINAL_PHASE_B.kicad_sch` in KiCad, click on the STM32 symbol, and copy each labeled net off its pins.

## Cost breakdown

Rough numbers from what I spent. These are USD and approximate.

| Item | Qty | Unit | Subtotal |
| --- | --- | --- | --- |
| Custom PCB (JLCPCB, 2-layer, 5 pcs) | 1 order | ~$5 | $5 |
| STM32 MCU | 1 | ~$3 | $3 |
| IMU (MPU-6050 or similar) | 1 | ~$3 | $3 |
| Dual H-bridge motor driver | 1 | ~$2 | $2 |
| Buck regulator + passives | 1 set | ~$3 | $3 |
| JST connectors + headers | mix | ~$3 | $3 |
| TT gearmotor + wheel | 2 | ~$4 | $8 |
| 18650 Li-ion cell (2700 mAh) | 2 | ~$7 | $14 |
| 2-cell 18650 holder | 1 | ~$2 | $2 |
| 3 mm clear acrylic sheet (laser cut at makerspace) | 1 | ~$5 | $5 |
| M2 hardware (screws, nuts, metal standoffs) | mix | ~$5 | $5 |
| PLA filament for spacers + motor brackets | small | ~$1 | $1 |
| **Total** | | | **~$54** |

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

- STM32 reference manual and datasheet — *link the exact part you used, e.g.* https://www.st.com/en/microcontrollers-microprocessors/stm32f103c8.html
- MPU-6050 datasheet — https://invensense.tdk.com/wp-content/uploads/2015/02/MPU-6000-Datasheet1.pdf
- TT gearmotor specs — https://www.adafruit.com/product/3777
- ST-Link V2 user manual — https://www.st.com/resource/en/user_manual/cd00262073.pdf
- KiCad 10 docs — https://docs.kicad.org/
- STM32CubeIDE — https://www.st.com/en/development-tools/stm32cubeide.html
- Useful PID-for-balancing background — https://www.youtube.com/@PhilsLab (Phil's Lab, what this class draws on)

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
