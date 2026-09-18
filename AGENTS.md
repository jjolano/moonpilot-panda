# panda, as moonpilot uses it

This is `jjolano/moonpilot-panda`, a fork of `commaai/panda`. `master` is the fork branch and carries the fork's commits; `upstream` is commaai and stays there. Merge, never rebase, never force-push — the same rule the superproject follows.

The fork's rules live in the superproject, and they apply here too:

@../AGENTS.md

## What the fork changed here

One thing: `HEALTH_FLAG_CONTROLS_ALLOWED_LATERAL` in `board/health.h`, published in `board/main_comms.h` from the forked safety layer's `controls_allowed_lateral`. Nothing else in this tree is fork-owned, and nothing else should become so without a reason that survives the next upstream merge.

## The invariants

- **`board/health.h` is three contracts at once**, which is why it is not a local edit. `health_t` is compiled into the firmware (`board/main.c`, `board/drivers/drivers.h`, `board/body/can.h`) *and* into openpilot's host side, through upstream's own `openpilot/selfdrive/pandad/panda.h` and `openpilot/tools/cabana/panda.h` — so a macro added here has to compile on both. panda's python parses the struct **out of the source text** by name (`_parse_c_struct(… "health_t")` in `python/__init__.py`) and hashes the file into `HEALTH_PACKET_VERSION`, and `SConscript` hashes it into the firmware's own version. Adding a flag is fine; moving a field, renaming the struct, or changing a field's type or width is not.
- **`flags_pkt` is `uint16_t` and bits 0–6 are upstream's.** The fork's bit is `1U << 7`. Do not renumber an existing one, and do not widen the field to make room — openpilot parses this packet, so a wider field silently misaligns everything after it.
- **This tree mirrors the permission, it does not decide it.** `controls_allowed_lateral` is set by the forked safety layer in `moonpilot-opendbc`, which panda compiles into the firmware (`board/main.c` includes `opendbc/safety/safety.h`). Re-deriving an engagement rule here would be a second authority for one safety decision, which is what the fork avoids.
- `python/__init__.py` deliberately does not name the new bit: that list is what `panda.health()` decodes for PC-side tools, and nothing reads bit 7 that way. Add it there only when a host-side tool wants it.

## Gates

- `scons` builds the firmware; so does the superproject's `tools/op.sh build`. A change here changes the signed firmware, which is how openpilot knows to reflash the panda — `pandad.py` flashes when the panda's signature differs from the built one.
- `ruff check .` and `python -m unittest discover -s tests` are panda's own gate (`./test.sh`).
- C changes have to build for the board they ship on, not only for the host.
