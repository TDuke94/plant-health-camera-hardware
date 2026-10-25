# Rev A review checklist

A checklist for reviewing the rev A schematics and layouts. Each item is a yes/no check, so a review doesn't need much recent PCB experience. Tick an item only after you've looked at the design yourself.

## Automated (CI)

Run on the self-hosted build server for every push:

```sh
kicad-cli sch erc --exit-code-violations --format json -o erc.json kicad/<board>/<board>.kicad_sch
kicad-cli pcb drc --exit-code-violations --schematic-parity --format json -o drc.json kicad/<board>/<board>.kicad_pcb
kicad-cli sch export bom -o bom.csv kicad/<board>/<board>.kicad_sch
kicad-cli pcb export gerbers -o gerbers/ kicad/<board>/<board>.kicad_pcb
```

- [ ] ERC passes with no violations, or each waiver is commented in the schematic
- [ ] DRC passes using the fab's rules (trace, space, drill, annular ring)
- [ ] Schematic parity passes (the board matches the schematic)

## Schematic

- [ ] Every IC has decoupling per its datasheet, placed by its supply pins
- [ ] Every I2C bus has exactly one set of pull-ups, sized for its speed and length
- [ ] No two parts on one I2C bus share an address (include the Nucleo's PD controller on I2C2)
- [ ] Every 3V3 peripheral connects to a 3V3 MCU pin, or goes through a level shifter
- [ ] Every pin in the spec's pin budget matches the schematic net
- [ ] Buttons have a pull-up or pull-down and, if wired directly, debounce in firmware
- [ ] The flash LED driver has a current limit and a local bulk capacitor
- [ ] Test points exist for 5 V, 3V3, GND, I2C1, I2C2, PPS and one SPI CS

## Layout: shield

- [ ] Header positions match MB1940 exactly; check by printing 1:1 on paper and laying it on the Nucleo
- [ ] No tall parts collide with Nucleo parts under the shield
- [ ] Ground plane is continuous under the SPI and I2C traces
- [ ] The flash LED's high-current loop is short and away from the GNSS header

## Layout: CSI-2 mux board

- [ ] Stackup and trace widths come from the fab's impedance calculator for 100 ohm differential
- [ ] Each pair is length-matched (P to N) and each lane group is matched to its clock
- [ ] Pairs have no vias where avoidable, and no stubs at switch pins
- [ ] Reference plane under the pairs is solid, with no splits
- [ ] FPC connector pin 1 orientation matches the Nucleo's CN6 and the sensor modules

## Before ordering

- [ ] BOM parts are in stock at the assembly fab, or have listed alternates
- [ ] Footprints for every part are checked against the datasheet land pattern
- [ ] Silkscreen marks pin 1, connector functions and the board revision
