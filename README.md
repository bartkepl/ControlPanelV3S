# ControlPanelV3S — Hardware

KiCad project for the CAN-bus control panel board built around the Allwinner V3S (LicheePi Zero),
part of the [CANbusSensors](../) family of MPCC/MPSWP sensor-node projects. Two assembly variants
on one PCB: a handheld CAN-bus terminal and a VoIP endpoint. The next hardware iteration (Allwinner
T113-S3/4) will be a separate repo.

## Layout

- **`pcb/`** - KiCad project: CPU, power, CAN transceiver front-end, display/touch, GPS, Ethernet,
  audio.
- **`doc/`** - hardware design decisions and reference material. Start with
  [doc/README.md](doc/README.md), which says which document is the source of truth for what.
- **`media/`** - board photos.
- **`prod/`** - fabrication outputs (BOM, interactive BOM, schematic/PCB PDFs).
- **`analysis/`** - raw output of automated KiCad design-review runs (ERC/DRC/SPICE/EMC), not
  version-controlled - regenerable, kept locally as a reference for past findings.
- **`TODO.md`** - open items from the latest design review (see `doc/DESIGN_REVIEW.md`).

## Firmware

The firmware that runs on this board (V3S-side kiosk app, CAN-bus device monitor, offline
navigation/map stack) lives in a separate, currently private companion repo.

## License

MIT for original work - see [LICENSE](LICENSE). Files under `pcb/local_lib/FunKey.pretty/`,
`pcb/local_lib/BAT_MS621FE-FL11E.kicad_mod`, and `doc/FunKey Schematics.pdf` originate from the
FunKey project and are licensed CC BY-NC-SA, not MIT - not compatible with the MIT license this
repo otherwise uses. These are planned to be replaced with an original library before this repo is
made public.
