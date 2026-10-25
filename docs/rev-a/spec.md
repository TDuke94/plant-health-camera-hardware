# Rev A hardware spec

Status: draft, 2026-10-25.

## Goal

Rev A proves the camera's core loop before anyone lays out an STM32N6 BGA:

1. capture a raw frame from one of several sensors,
2. hash and sign it with a key that never leaves a secure element,
3. tag it with GNSS time and position,
4. store it on microSD,
5. sync it to the hub over USB-C.

Rev A is two small boards around an unmodified NUCLEO-N657X0-Q:

- **Shield**: plugs onto the Nucleo's two ST morpho headers (CN3, CN15), which carry every ARDUINO signal too. Low speed only, no BGA. Schematic: [kicad/shield](../../kicad/shield).
- **CSI-2 mux board**: connects to the Nucleo's 22-pin camera FPC (CN6) and fans out to four sensor ports. It is the only board with high-speed signals.

Rev B folds both into one N6 board once rev A works.

## What the Nucleo already provides

From [UM3417](https://www.st.com/resource/en/user_manual/um3417-stm32n6-nucleo144-board-mb1940-stmicroelectronics.pdf) (STM32N6 Nucleo-144, MB1940):

| Feature | Nucleo detail | Rev A use |
| --- | --- | --- |
| MCU | STM32N657X0H3Q | As is |
| External flash | 512 Mbit octo-SPI NOR, 1.8 V | Firmware and settings |
| External RAM | **None** | See "Frame memory" below |
| Camera | CN6, 22-pin ZIF (Pi Zero style), MIPI CSI-2 clock + 2 data lanes, I2C2 (PB10/PB11), PWR_EN (PA0), NRST_CAM, VDD_CAM 3V3 | Feeds the mux board |
| User USB | CN8, USB-C DRP, USB 2.0 HS (OTG1), UCPD1 | Power in, and sync to the hub as CDC-NCM |
| ST-LINK | CN10, USB-C | Debug and flashing |
| Ethernet | RJ45 + RMII PHY | Unused |
| Buttons, LEDs | B1 on PC13; LEDs on PG0, PG8, PG10 (active low) | Bring-up status |
| Size | 133.35 x 70.00 mm | Sets the enclosure |

### Frame memory

The Nucleo has no PSRAM, and wiring RAM through the headers is not viable at XSPI speeds ([ST community thread](https://community.st.com/t5/stm32-mcus-boards-and-hardware/adding-external-ram-to-nucleo-n657x0-q/td-p/849454)). Rev A works inside the N6's ~4.2 MB of internal SRAM:

| Frame | Size | Fits? |
| --- | --- | --- |
| 1 MP mono RAW10 (OV9281 class, 1280 x 800) | ~1.3 MB | Yes, one band at a time |
| 2 MP RAW10 (IMX219 2x2 binned, 1640 x 1232) | ~2.5 MB | Yes |
| 8 MP RAW10 (IMX219 full) | ~10 MB | No, rev B (PSRAM) |

Each band is captured, hashed, written to microSD, then the next band is captured. Hashing runs over the buffer before it is written, so the signature covers exactly what the sensor delivered.

## Block allocation

| Block | Where | Interface | Notes |
| --- | --- | --- | --- |
| Sensors (up to 4) | Mux board | MIPI CSI-2, 2 lanes, via CN6 | One powered at a time |
| CSI-2 switch | Mux board | Select lines from an I2C GPIO expander | TMUX646 tree |
| I2C mux + GPIO expander | Mux board | I2C2 via CN6 | Sensors share an address |
| Secure element | Shield | I2C1 | ATECC608B (SOIC-8); STSAFE-A120 alternate |
| GNSS | Shield header + breakout | USART3 + PPS | Breakout keeps RF off our board |
| microSD | Shield | SPI5 (shared), dedicated CS | SDMMC in rev B |
| Display | Shield connector + off-the-shelf panel | SPI5 (shared) + DC/RST/backlight PWM | 2.4 to 2.8 in, ST7789 class, 320 x 240 |
| IMU | Shield | I2C1 + INT | LSM6DSO class |
| Shutter + D-pad | Shield | 7 GPIOs | Two-stage shutter, 4-way + centre |
| White LED flash | Shield | PWM | Low-side MOSFET, resistor current limit, from 5 V |
| NIR LEDs | Shield | Enable (PE10) | Footprints only, not fitted on rev A |
| Qwiic header | Shield | I2C1, 3V3 | AS7341, ToF, NFC, buzzer as breakouts |
| Wi-Fi | — | Not on rev A | Footprint in rev B |
| Power | Nucleo CN8 | USB-C | Power bank in the field, the hub at the dock |

## Pin budget

Header pins come from UM3417 Table 20 (ST morpho CN3 and CN15). The manual states that the MCU's digital I/O on these headers is 3.3 V; only the ARDUINO A0-A5 inputs are 1.8 V analog, and the shield doesn't use them. No solder bridge is documented on any pin below. PC3 and PC4 are taken on the Nucleo (ST-LINK USB PD CC lines), so the D-pad avoids them; PC9 isn't on the morpho headers.

| Function | MCU pin | Header pin | Net |
| --- | --- | --- | --- |
| 5 V in | — | CN3-6, CN3-18 | +5V |
| Ground | — | CN3-8, 19, 20, 22; CN15-9, 20 | GND |
| Shutter half-press | PC0 | CN3-17 | SHUTTER_HALF |
| Shutter full-press | PC2 | CN3-3 | SHUTTER_FULL |
| D-pad up | PC5 | CN3-15 | NAV_UP |
| D-pad down | PC8 | CN3-37 | NAV_DOWN |
| D-pad left | PE4 | CN3-21 | NAV_LEFT |
| D-pad right | PE7 | CN15-22 | NAV_RIGHT |
| D-pad centre | PE1 | CN15-24 | NAV_CENTER |
| I2C1_SCL (SE, IMU, Qwiic) | PH9 | CN15-3 | I2C1_SCL |
| I2C1_SDA | PC1 | CN15-5 | I2C1_SDA |
| IMU INT1 | PD6 | CN15-6 | IMU_INT |
| SPI5_SCK (display, microSD) | PE15 | CN15-11 | SPI_SCK |
| SPI5_MISO | PG1 | CN15-13 | SPI_MISO |
| SPI5_MOSI | PG2 | CN15-15 | SPI_MOSI |
| Display CS | PA3 | CN15-17 | LCD_CS |
| Display backlight PWM (TIM1_CH2) | PD7 | CN15-19 | LCD_BL |
| Display DC | PD12 | CN15-21 | LCD_DC |
| Display RST | PE11 | CN15-23 | LCD_RST |
| Flash LED PWM (TIM1_CH4N) | PD5 | CN15-25 | FLASH_PWM |
| NIR LED enable (not fitted) | PE10 | CN15-27 | NIR_EN |
| microSD CS | PE0 | CN15-29 | SD_CS |
| GNSS PPS (TIM1_CH1 input capture) | PE9 | CN15-31 | GNSS_PPS |
| microSD card detect | PD0 | CN15-33 | SD_CD |
| GNSS reset | PD10 | CN15-34 | GNSS_RST |
| MCU -> GNSS RX (USART3_TX) | PD8 | CN15-35 | GNSS_RXD |
| Spare (test point) | PD11 | CN15-36 | SPARE_PD11 |
| GNSS TX -> MCU (USART3_RX) | PD9 | CN15-37 | GNSS_TXD |
| CSI-2, I2C2, PWR_EN, NRST_CAM | (fixed) | CN6 FPC | Mux board |

I2C1 addresses: ATECC608B 0x60, LSM6DSO 0x6A (SA0 to GND), AS7341 0x39, VL53L4 0x29. The manual doesn't say whether the Nucleo pulls up I2C1, so the shield has 4.7 k pull-ups (R1, R2) to fit or leave off.

USART1 (PE5/PE6) stays the ST-LINK virtual COM port for the Zephyr console. The CN6 I2C2 bus is shared with the Nucleo's USB-C PD controller, so mux-board I2C addresses must avoid the PD controller's address (check the schematic).

## Power

Rev A has no battery circuitry. The Nucleo's user USB-C port (CN8, up to 1.5 A at 5 V from UCPD) powers everything: a USB-C power bank in the field and the hub when docked. This drops the charger, fuel gauge and battery from the first board, which removes the riskiest analogue parts. Rev B adds them.

The shield takes 5 V from the morpho 5 V pin and makes its own 3V3 with an LDO, instead of drawing on the Nucleo's 3V3 rail.

Estimated budget (rough, to be measured on the bench):

| Load | Rail | Typical | Peak |
| --- | --- | --- | --- |
| Nucleo (N6 active, NPU idle) | 5 V in | 250 mA | 400 mA |
| Sensor (one powered) | 3V3 via CN6 | 60 mA | 100 mA |
| GNSS breakout (dual-band) | 3V3 shield | 50 mA | 80 mA |
| Display + backlight | 3V3 shield | 60 mA | 90 mA |
| IMU, SE, Qwiic sensors | 3V3 shield | 10 mA | 30 mA |
| White LED flash | 5 V | 0 | 700 mA for < 50 ms |

The flash peak plus the rest is close to the 1.5 A UCPD limit, so the flash has a local 1000 uF bulk capacitor and a fixed current limit, and firmware never fires the flash during an SD write. The driver is a low-side AO3400A with two 5.6 ohm 1 W resistors in parallel: (5 V - 3.0 V) / 2.8 ohm gives about 0.7 A. A buck LED driver is a rev B option if the flash needs to be brighter or more efficient.

## Shield BOM (first pass)

References match the schematic in `kicad/shield`.

| Ref | Part | Alternate | Footprint | Notes |
| --- | --- | --- | --- | --- |
| J1, J2 | 2x19 pin sockets, 2.54 mm | Stacking headers | PinSocket_2x19 | Mate with Nucleo CN3, CN15 |
| U1 | AP2112K-3.3 (LDO, 600 mA) | TLV75733P | SOT-23-5 | Shield 3V3 |
| U2 | ATECC608B-SSHDA (secure element) | STSAFE-A120 | SOIC-8 | I2C1, 0x60 |
| U3 | LSM6DSO (IMU) | ISM330DHCX | LGA-14 3 x 2.5 mm | Drawn with the pin-compatible LSM6DSL symbol; needs fab assembly or hot air |
| J3 | Qwiic, JST-SH 4-pin | — | SM04B-SRSS-TB | GND, 3V3, SDA, SCL |
| J4 | Hirose DM3AT-SF-PEJM5 microSD, push-push, card detect | — | DM3AT | SPI mode |
| J5 | 1x8 socket for the display module | Panel FPC on rev B | PinSocket_1x08 | 3V3, GND, SCK, MOSI, CS, DC, RST, BL |
| J6 | 1x7 socket for the GNSS breakout | — | PinSocket_1x07 | 5V, 3V3, GND, TXD, RXD, PPS, RST |
| SW1, SW2 | Tactile switches, half and full press | One two-stage switch | TL3342 | Shutter |
| SW3-SW7 | Tactile switches | One 5-way nav switch | TL3342 | D-pad |
| D1 | High-CRI white LED, Cree XP-G class | — | LED_Cree-XP | Flash, ~0.7 A pulses; pad 3 is thermal only |
| Q1 | AO3400A | Any logic-level N-FET, SOT-23 | SOT-23 | Flash switch |
| R8, R9 | 5.6 ohm, 1 W | — | 2512 | Flash current limit, in parallel |
| C9 | 1000 uF, 10 V electrolytic | — | Radial D10 | Flash bulk |
| D2-D5, Q2, R12-R15 | 850 nm LEDs and driver | — | 1206, SOT-23 | Not fitted on rev A |
| R1, R2 | 4.7 k | — | 0603 | I2C1 pull-ups, optional |
| R3-R7 | 47 k / 10 k | — | 0603 | microSD pull-ups and card detect |
| TP1-TP8 | Test pads | — | 1.5 mm pad | 5V, 3V3, GND, PPS, SDA, SCL, SD_CS, PD11 |

The Nucleo has its own ST-LINK, so the shield needs no SWD header.

Off-board:

| Item | Pick | Notes |
| --- | --- | --- |
| GNSS breakout | Dual-band L1/L5 with raw observation output (Quectel LC29H family or Unicore UM980) | Confirm the variant outputs raw data for PPK against a local RTK base |
| GNSS antenna | Dual-band L1/L5 active patch | — |
| Display | 2.4 to 2.8 in IPS, ST7789 class, SPI | — |
| Power | USB-C PD power bank, 5 V / 3 A | — |

## CSI-2 mux board

Plugs into the Nucleo's CN6 with a 22-pin, 0.5 mm FPC and offers four 22-pin sensor ports. Schematic: [kicad/csi-mux/](../../kicad/csi-mux/).

- **Upstream (J1):** CN6 uses ST's 22-pin pinout, not the Pi's. It carries 2 data lanes, the clock, I2C2 (PB10/PB11), a reset line, a power-enable line and 3V3. I2C2 is shared with the Nucleo's USB-C PD controller, so the mux board's addresses must avoid it.
- **Sensor ports (J2-J5):** Raspberry Pi 22-pin pinout, 2 lanes used. Pin 17 (IO0) is the module's power enable and pin 18 (IO1) is left for its LED or a spare. Pi-style modules regulate their own rails from 3V3, so no per-port load switches are needed; the expander turns each module on through IO0, with a 100 k pull-down so ports stay off at reset.
- **Switching:** three [TI TMUX646](https://www.ti.com/product/TMUX646) 2:1 MIPI switches as a tree carry the clock and both data lanes. The root (U1) picks a leaf; leaf U2 serves ports 0/1 and leaf U3 serves ports 2/3, with one shared leaf select. ~OE is active low and is shared by all three. TMUX646 replaces the TS5MP646, which TI no longer recommends for new designs.
- **I2C:** a TCA9548A (0x70) gives each port its own I2C segment, since identical sensors share an address. Each segment has 2.2 k pull-ups. The upstream 4.7 k pull-ups are DNP until we know whether the Nucleo already pulls up I2C2. The mux's reset comes from the CN6 reset line.
- **GPIO expander:** a TCA9554 (0x20) on the upstream bus:

  | Pin | Net | Use |
  | --- | --- | --- |
  | P0 | SEL_ROOT | Root switch: low = ports 0/1, high = ports 2/3 |
  | P1 | SEL_LEAF | Leaf switches: low = even port, high = odd port |
  | P2 | SW_OE_N | All switches off when high |
  | P3-P6 | CAM0_EN-CAM3_EN | Port power enable (module IO0) |
  | P7 | spare | |

- **Package:** the TMUX646 comes only as a 36-ball, 0.4 mm pitch nFBGA (ZEC, 2.45 mm). That needs via-in-pad or a fab's fine-pitch BGA rules, and assembly by the fab. The schematic uses a stock 6x6 0.4 mm BGA footprint as a placeholder; a footprint drawn from the ZEC land pattern replaces it before layout.
- **Layout:** 4 layers, 100 ohm differential pairs with the fab's impedance-controlled stackup, length-matched within each lane group, no stubs at the switch pins, solid reference plane under the pairs.
- **Sensors:** identical global-shutter mono modules (OV9281 class) in the Pi 22-pin pinout, each behind a narrowband filter: green, red, red edge, NIR. Rev A fits one or two.

## Firmware dependencies

These affect the hardware choices and should be checked in the firmware repo before ordering boards:

- Zephyr has the `nucleo_n657x0_q` board and the STM32 DCMIPP video driver. Sensor drivers in tree include IMX335; check IMX219, and expect to write an OV9281 driver.
- The TMUX646 tree and the I2C mux need a small Zephyr driver that switches port, I2C segment and power together.
- microSD in SPI mode uses Zephyr's `zephyr,sdhc-spi-slot`.
- USB-C sync uses Zephyr's CDC-NCM device class on OTG1.

## Mechanical

The enclosure is 3D printed (PETG or ASA) around the Nucleo + shield stack, with the mux board and sensors in a front module and the display and controls on the back. The sensors and the flash share one rigid front plate, so the lenses' relative positions stay fixed for band registration.

## Open items

- [x] Confirm morpho pin choices and I/O voltage against UM3417 (Table 20; digital I/O is 3.3 V)
- [ ] Cross-check the pin table against the MB1940 schematic before ordering
- [ ] Find out whether the Nucleo pulls up I2C1, and fit or remove R1/R2
- [ ] Confirm the PD controller's I2C2 address (the manual doesn't give it) and that the mux-board parts avoid it
- [ ] Run ERC on the shield and mux schematics in KiCad 9 and tidy their layout
- [ ] Draw the TMUX646 ZEC footprint and confirm the fab's via-in-pad rules
- [ ] Confirm the CN6 pinout and FPC contact side against the MB1940 schematic
- [ ] Pick the GNSS breakout that outputs raw observations
- [ ] Pick the display panel and its connector
- [ ] Pick the band filters and the sensor module source
- [ ] Measure the power budget on the bench before sizing the flash
