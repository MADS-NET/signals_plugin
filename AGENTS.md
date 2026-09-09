# siggen — working notes for AI coding agents

This is a **MADS source plugin** written in C++20 (SigGen requires it).

Framework context — the plugin lifecycle, the order in which the host agent
calls each method, what every return value does, how settings reach the plugin,
topics, blobs, testing, deployment and protocol migration — is in the
`mads-plugin` skill under [`.claude/skills/mads-plugin/`](.claude/skills/mads-plugin/SKILL.md).
**Read `SKILL.md` before changing plugin code**, and follow its `reference/`
files rather than inferring host behaviour from the template comments.

Regenerate that skill for a newer MADS with `mads plugin --update`.

## This plugin

Generates synthetic signals with the header-only
[SigGen](https://github.com/pbosetti/SigGen) library, so that a MADS network can
be fed realistic test data with no hardware attached. Each entry of the
`signals` setting becomes one SigGen generator; every loop iteration draws
**one sample from each** and publishes them as a *name → value* map.

One sample per call means the sampling frequency is the reciprocal of the loop
period, which is where `sample_rate` comes from: explicit setting, else
`1000 / period`, else the measured call period (`track_period`, on by default,
covers the case of a period given with `-p`, which the plugin cannot read).

- Behavior: **source** — host agent `mads source`
- Class: `SignalsPlugin` in [`src/signals.cpp`](src/signals.cpp)
- Driver name: `signals` — must stay equal to the CMake target name, so that `kind()` and the file stem agree
- INI section: `[signals]` (or whatever `-n` selects at run time)

## Parameters

Keep this table and the README in sync — it is the plugin's public interface.

| Key | Type | Default | Meaning |
|---|---|---|---|
| `signals` | object | *(required)* | Map of signal name to SigGen signal description. Nested TOML tables: `[<agent>.signals.<name>]`. |
| `sample_rate` | number (Hz) | `1000 / period` | Explicit sampling rate; also switches `track_period` off. |
| `seed` | integer ≥ 0 | *(random)* | Base seed; the *n*-th signal gets `seed + n`. A `seed` inside a signal description wins. |
| `nest` | string | `""` | Key holding the value map; empty publishes the values flat. |
| `track_period` | bool | `true` | Follow the measured call period. |
| *(other scalars)* | — | — | Passed to SigGen as document-level constants, referable as `"$f0"`. |

`period` is an *agent* key, read here only to derive the sampling rate; the
other reserved agent keys are listed in `ReservedKeys` and are kept out of the
document handed to SigGen.

## Output frames

One frame per loop iteration, on `pub_topic` (the agent name by default):
`t` (seconds since the first sample), `n` (sample index), `sample_rate` (Hz),
and one field per signal — or a single object under `nest`. Signal names are
written last, so they win over the three fields above.

## Conventions for this project

- C++20, built with CMake; LLVM formatting, two-space indent.
- `CamelCase` for classes and namespaces, `snake_case` for methods and
  variables, `_leading_underscore` for private members, declared last.
- Do not add third-party dependencies without asking. `nlohmann/json` is
  already available, and the plugin base classes also provide a `SerialPort`
  helper in `serialport.hpp`.
- All plugin logic must be reachable from the test `main()` at the bottom of
  the source file, and that test must be deterministic and self-checking:
  assert on both the returned status and the payload, and exit non-zero on
  failure.
- An exception must never escape a plugin method: catch it, set `_error` and
  return `return_type::error`.
- Never block, sleep or busy-wait inside a plugin method.

## Build and test

```sh
cmake -Bbuild -DCMAKE_INSTALL_PREFIX="$(mads -p)"
cmake --build build -j4
./build/signals.plugin               # the standalone test driver (./build/signals off macOS)
mads inspect_plugin build/signals.plugin
mads source build/signals.plugin -s mads.ini
```

The test driver is self-checking: it asserts every sample against its analytic
value and exits non-zero on failure. Keep it that way when adding features.
