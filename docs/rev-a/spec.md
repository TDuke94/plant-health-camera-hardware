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

- **Shield**: plugs onto the Nucleo's ARDUINO and ST morpho headers. Low speed only, no BGA, hand-solderable.
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
| White LED flash | Shield | PWM + enable | Constant-current driver from 5 V |
| NIR LEDs | Shield | Footprint only | Not fitted on rev A |
| Qwiic header | Shield | I2C1, 3V3 | AS7341, ToF, NFC, buzzer as breakouts |
| Wi-Fi | — | Not on rev A | Footprint in rev B |
| Power | Nucleo CN8 | USB-C | Power bank in the field, the hub at the dock |

## Pin budget

ARDUINO pins come from UM3417 Table 18 and are confirmed in the manual. ST morpho pins are taken from the manual's list of free GPIOs and **must be checked against the MB1940 schematic** for solder bridges and I/O voltage before layout. Some N6 I/O banks run at 1.8 V (the ARDUINO analog pins are 1.8 V); any 3V3 peripheral on a 1.8 V pin needs a level shifter.

| Function | MCU pin | Header | Confirmed |
| --- | --- | --- | --- |
| SPI5_SCK (display, microSD) | PE15 | ARDUINO D13 | Manual |
| SPI5_MISO | PG1 | ARDUINO D12 | Manual |
| SPI5_MOSI | PG2 | ARDUINO D11 | Manual |
| Display CS | PA3 | ARDUINO D10 | Manual |
| Display backlight PWM (TIM1_CH2) | PD7 | ARDUINO D9 | Manual |
| Display DC | PD12 | ARDUINO D8 | Manual |
| Display RST | PE11 | ARDUINO D7 | Manual |
| Flash LED PWM (TIM1_CH4N) | PD5 | ARDUINO D6 | Manual |
| Flash LED enable | PE10 | ARDUINO D5 | Manual |
| microSD CS | PE0 | ARDUINO D4 | Manual |
| GNSS PPS (TIM1_CH1 input capture) | PE9 | ARDUINO D3 | Manual |
| microSD card detect | PD0 | ARDUINO D2 | Manual |
| GNSS TX -> MCU (USART3_RX) | PD9 | ARDUINO D0 | Manual |
| MCU -> GNSS RX (USART3_TX) | PD8 | ARDUINO D1 | Manual |
| I2C1_SDA (SE, IMU, Qwiic) | PC1 | ARDUINO D14 | Manual |
| I2C1_SCL | PH9 | ARDUINO D15 | Manual |
| Shutter half-press | PC0 | morpho | Check schematic |
| Shutter full-press | PC2 | morpho | Check schematic |
| D-pad up / down / left / right / centre | PC3, PC4, PC5, PC8, PC9 | morpho | Check schematic |
| IMU INT | PD6 | morpho | Check schematic |
| GNSS reset | PD10 | morpho | Check schematic |
| SE / spare GPIO | PD11 | morpho | Check schematic |
| CSI-2, I2C2, PWR_EN, NRST_CAM | (fixed) | CN6 FPC | Manual |

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

The flash peak plus the rest is close to the 1.5 A UCPD limit, so the flash driver gets a local bulk capacitor (470 to 1000 uF) and a current limit, and firmware never fires the flash during an SD write.

## Shield BOM (first pass)

| Ref | Part | Alternate | Package | Notes |
| --- | --- | --- | --- | --- |
| U1 | ATECC608B-SSHDA (secure element) | STSAFE-A120 | SOIC-8 | I2C1, 3V3 |
| U2 | LSM6DSO (IMU) | ISM330DHCX | LGA-14 | Needs fab assembly or hot air |
| U3 | AP2112K-3.3 (LDO, 600 mA) | TLV75733P | SOT-23-5 | Shield 3V3 |
| U4 | AL8860 or similar buck LED driver | Low-side MOSFET + sense resistor | SOT-89 / SOT-23 | White flash, ~700 mA |
| J1 | microSD push-push socket with card detect | — | SMD | SPI mode |
| J2 | Display FPC or 2.54 mm header | — | — | Matches the chosen panel |
| J3 | GNSS header: 3V3, GND, TX, RX, PPS, RESET | — | 2.54 mm | For a dual-band breakout |
| J4 | Qwiic (JST-SH 4-pin) | — | SMD | I2C1, 3V3 |
| SW1 | Two-stage tactile switch | Two stacked switches | — | Shutter |
| SW2 | 5-way navigation switch | Five tactile switches | — | D-pad |
| D1 | High-CRI white LED, 1 to 3 W | — | — | Flash |
| D2-D5 | NIR LED footprints (850 nm) | — | — | Not fitted |
| — | ARDUINO + morpho stacking headers | — | 2.54 mm | Match MB1940 positions |

Off-board:

| Item | Pick | Notes |
| --- | --- | --- |
| GNSS breakout | Dual-band L1/L5 with raw observation output (Quectel LC29H family or Unicore UM980) | Confirm the variant outputs raw data for PPK against a local RTK base |
| GNSS antenna | Dual-band L1/L5 active patch | — |
| Display | 2.4 to 2.8 in IPS, ST7789 class, SPI | — |
| Power | USB-C PD power bank, 5 V / 3 A | — |

## CSI-2 mux board

Plugs into CN6 with a 22-pin, 0.5 mm FPC and offers four 22-pin sensor ports in the same pinout.

- **Switching:** three [TI TMUX646](https://www.ti.com/product/TMUX646) 2:1 MIPI switches as a tree (one root, two leaves) carry the clock and both data lanes. TMUX646 replaces the TS5MP646, which TI no longer recommends for new designs. A build option fits only the root switch for a two-port board.
- **Control:** a TCA9548A-class I2C mux gives each sensor its own I2C segment, since identical sensors share an address. A PCA9536/PCA9554-class GPIO expander drives the switch selects and each port's power enable and reset. Everything is controlled over CN6's I2C2, so no extra cable runs to the shield.
- **Power:** one load switch per port from VDD_CAM (3V3), so only the selected sensor is powered.
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

- [ ] Confirm morpho pin choices and I/O voltage domains against the MB1940 schematic
- [ ] Confirm the PD controller's I2C2 address and that the mux-board parts avoid it
- [ ] Pick the GNSS breakout that outputs raw observations
- [ ] Pick the display panel and its connector
- [ ] Pick the band filters and the sensor module source
- [ ] Measure the power budget on the bench before sizing the flash
