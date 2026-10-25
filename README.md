# plant-health-camera-hardware

Hardware for a handheld plant-health camera. The camera captures raw frames, signs them at capture, and syncs them to a hub (a Raspberry Pi 5) over USB-C. Crop health and nutrient analysis runs on the hub or offline.

Firmware lives in [plant-health-camera-firmware](https://github.com/TDuke94/plant-health-camera-firmware).

## Revisions

| Rev | What it is | Status |
| --- | --- | --- |
| A | A shield for the NUCLEO-N657X0-Q, plus a small CSI-2 mux board on the Nucleo's camera FPC | Spec in progress: [docs/rev-a/spec.md](docs/rev-a/spec.md) |
| B | An integrated STM32N6 board built from the proven rev A design | Not started |

## Layout

```
docs/rev-a/         rev A spec, pin budget, review checklist
kicad/shield/       rev A shield (KiCad), to come
kicad/csi-mux/      rev A CSI-2 mux board (KiCad), to come
```

## Tooling

KiCad 9. ERC and DRC run with `kicad-cli` on the self-hosted build server; see [docs/rev-a/review-checklist.md](docs/rev-a/review-checklist.md).
