# panda, as moonpilot uses it

This is `jjolano/moonpilot-panda`, a fork of `commaai/panda`. `master` is the fork branch and carries the fork's commits; `upstream` is commaai and stays there. Merge, never rebase, never force-push — the same rule the superproject follows.

The fork's rules live in the superproject, and they apply here too:

@../AGENTS.md

## What the fork changed here

Three things. `HEALTH_FLAG_CONTROLS_ALLOWED_LATERAL` in `board/health.h`, published in `board/main_comms.h` from the forked safety layer's `controls_allowed_lateral`. The MISRA mutation harness in `tests/misra/test_mutation.py`, which hands the child process both its own interpreter and the opendbc this run imported: `test_misra.sh` calls bare `python3`, and `SConscript` reads its include path from an `import opendbc` of its own, so without that the check dies on an import or compiles against the wrong safety headers, depending on whether the venv was activated. And one line of `board/drivers/gpio.h` — the alternate-function mask cast to `uint32_t`, because MISRA 12.2 reads `0xFU`'s essential type as 8 bits wide while the shift reaches 28, and whether cppcheck reports it depends on what else is in that translation unit. Nothing else in this tree is fork-owned, and nothing else should become so without a reason that survives the next upstream merge.

## The invariants

- **`board/health.h` is three contracts at once**, which is why it is not a local edit. `health_t` is compiled into the firmware (`board/main.c`, `board/drivers/drivers.h`, `board/body/can.h`) *and* into openpilot's host side, through upstream's own `openpilot/selfdrive/pandad/panda.h` and `openpilot/tools/cabana/panda.h` — so a macro added here has to compile on both. panda's python parses the struct **out of the source text** by name (`_parse_c_struct(… "health_t")` in `python/__init__.py`) and hashes the file into `HEALTH_PACKET_VERSION`, and `SConscript` hashes it into the firmware's own version. Adding a flag is fine; moving a field, renaming the struct, or changing a field's type or width is not.
- **`flags_pkt` is `uint16_t` and bits 0–6 are upstream's.** The fork's bit is `1U << 7`. Do not renumber an existing one, and do not widen the field to make room — openpilot parses this packet, so a wider field silently misaligns everything after it.
- **This tree mirrors the permission, it does not decide it.** `controls_allowed_lateral` is set by the forked safety layer in `moonpilot-opendbc`, which panda compiles into the firmware (`board/main.c` includes `opendbc/safety/safety.h`). Re-deriving an engagement rule here would be a second authority for one safety decision, which is what the fork avoids.
- `python/__init__.py` deliberately does not name the new bit: that list is what `panda.health()` decodes for PC-side tools, and nothing reads bit 7 that way. Add it there only when a host-side tool wants it.

## Gates

- `scons` builds the firmware; so does the superproject's `tools/op.sh build`. A change here changes the signed firmware, which is how openpilot knows to reflash the panda — `pandad.py` flashes when the panda's signature differs from the built one.
- **`./test.sh` cannot build this fork's firmware on its own.** Its `setup.sh` installs `opendbc @ git+https://github.com/commaai/opendbc.git@master` (`pyproject.toml`), which is *upstream* opendbc and has no `opendbc/safety/moonpilot/`, so panda's standalone `scons` fails with `'controls_allowed_lateral' undeclared` in `board/main_comms.h` before any test runs. That is the fork's dependency, not a defect in the docs: build and test from the superproject, or point the standalone run at the fork — `PYTHONPATH=../opendbc_repo .venv/bin/python -m unittest discover -s tests`. Do not "fix" it by repointing panda's `pyproject.toml` at the fork: that line is upstream's, and the superproject already puts `opendbc_repo` on the include path where it belongs. The MISRA mutation test needs that path *absolute* — it copies the tree to a temp directory before it runs the checker — which is what the harness edit above is for.
- `ruff check .` and `python -m unittest discover -s tests` are the rest of panda's own gate (`./test.sh`).
- C changes have to build for the board they ship on, not only for the host.
