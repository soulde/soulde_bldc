# Repository Guidelines

## Project Structure & Module Organization

This is VESC embedded firmware written mainly in C. Root files contain entry
points and shared configuration, including `main.c`, `conf_general.*`,
`datatypes.h`, and `Makefile`. Feature code is grouped by directory: `motor/`,
`comm/`, `encoder/`, `driver/`, `imu/`, `applications/`, `hwconf/`, and `util/`.
Build fragments live in `make/` and per-module `*.mk` files. Tests are under
`tests/`; project notes are in `documentation/`.

## Build, Test, and Development Commands

- `make`: show supported boards and targets.
- `make arm_sdk_install`: install the expected GNU ARM toolchain into `tools/`.
- `make fw_<board>` or `make <board>`: build firmware, for example
  `make fw_100_250`; outputs go under `build/`.
- `make PROJECT=<board> fw`: alternate board build form.
- `make fw_<board>_flash`: flash a built image with OpenOCD/SWD.
- `make all_fw`: build all boards; use before broad hardware changes when
  practical.
- `make all_ut`, `make all_ut_run`, `make all_ut_xml`: build/run unit tests and
  optionally write XML output.

## Coding Style & Naming Conventions

Follow the existing C style. Use tabs for indentation with a visual width of 4.
Put opening braces on the same line, always brace blocks, and format `else` as
`} else {`. Prefer C99 `//` comments; use Doxygen only for function comments.
Keep lines under about 90 characters when readable. Use `float` and `sinf`,
`cosf`, `fabsf`, etc. instead of double-precision math on STM32 targets. Avoid
dynamic allocation unless there is a strong reason.

## Testing Guidelines

Unit tests use per-directory Makefiles and Google Test support from
`make/unittest.mk`. Name new test directories by behavior or module, such as
`tests/packet_recovery/`. Keep tests runnable with `make all_ut_run`; for
isolated C tests, support `make run` inside the test directory. New firmware
logic should compile warning-free and include focused host tests when practical.

## Commit & Pull Request Guidelines

Keep commits logical and working. History uses short imperative or descriptive
subjects such as `Limit angle interpolation` and `Added phase shunt maxim`.
Avoid vague messages like `bug fix`; explain the technical change. For pull
requests, discuss large changes on the VESC forum when appropriate, describe
affected hardware/configurations, and list tests or builds run.

## Agent-Specific Instructions

Do not commit `build/` artifacts, downloaded toolchains, or local IDE state.
Keep changes scoped and preserve board compatibility unless one hardware variant
is explicitly targeted.
