# signals plugin for MADS

This is a Source plugin for [MADS](https://github.com/MADS-NET/MADS).

It generates synthetic signals with the
[SigGen](https://github.com/pbosetti/SigGen) library. Any number of signals can
be described in the agent's INI section; on every loop iteration the plugin
draws **one sample from each** of them and publishes a frame holding the
resulting *name → value* map. It is meant for feeding a MADS network with
realistic test data — sine waves, drifting offsets, coloured noise, ARIMA
processes — without any hardware attached.

Since exactly one sample is produced per call, the sampling frequency of the
signals is the reciprocal of the loop period: at the default period of 100 ms a
signal declared at 0.5 Hz completes one cycle every two seconds of wall-clock
time.

*Required MADS version: 2.4.0.*


## Supported platforms

Currently, the supported platforms are:

* **Linux**
* **MacOS**
* **Windows**


## Installation

Linux and MacOS:

```bash
cmake -Bbuild -DCMAKE_INSTALL_PREFIX="$(mads -p)"
cmake --build build -j4
sudo cmake --install build
```

Windows:

```powershell
cmake -Bbuild -DCMAKE_INSTALL_PREFIX="$(mads -p)"
cmake --build build --config Release
cmake --install build --config Release
```


## INI settings

The plugin supports the following settings in the INI file:

```ini
[signals]
period = 200            # ms, loop period: the sampling rate is its reciprocal
seed = 20260909         # optional, for reproducible noise
nest = ""               # "" publishes the values flat, a name nests them
track_period = true     # follow the call period actually observed
epoch = 0               # non-zero unix-s epoch: enables synchro generation
f0 = 0.25               # any extra key is usable in expressions, as "$f0"

# one table per signal, named after the field it feeds
[signals.signals.temperature]
type = "sine"
frequency = "$f0"       # Hz
amplitude = 10.0
offset = 25.0
snr_db = 30             # intrinsic noise, in dB; omit for the 40 dB default

[signals.signals.pressure]
type = "composite"
op = "sum"
components = [
  { type = "square", frequency = "$f0 / 5", amplitude = 0.5, offset = 3.0, noiseless = true },
  { type = "white_noise", sigma = 0.02 }
]

# an array of tables packs its signals into one JSON array, in order:
# {"position": [x, y, z]}
[[signals.signals.position]]
type = "sine"
frequency = "$f0"
amplitude = 1.0

[[signals.signals.position]]
type = "sine"
frequency = "$f0"
phase = 1.5707963

[[signals.signals.position]]
type = "sawtooth"
frequency = "$f0 / 10"
amplitude = 0.2
```

| Key | Type | Default | Meaning |
|---|---|---|---|
| `signals` | object | *(required)* | Map of signal name to [SigGen signal description](https://github.com/pbosetti/SigGen). The name is the key the value is published under. An entry may instead be an *array* of signal descriptions (a TOML `[[signals.signals.<name>]]` array of tables): each element is built as its own generator, and their values are packed into a JSON array, in declaration order, under that one key — e.g. a 3-element `position` publishes `{"position": [x, y, z]}`. Every element gets its own derived seed, exactly like a top-level entry. |
| `sample_rate` | number (Hz) | `1000 / period` | Explicit sampling rate. Setting it also switches `track_period` off. |
| `seed` | integer ≥ 0 | *(random)* | Base seed. The *n*-th signal gets `seed + n`, so each has its own random stream; a `seed` inside a signal description overrides it. |
| `nest` | string | `""` | Key under which the value map is published. Empty means the values go at the top level of the frame. |
| `track_period` | bool | `true` | Keep the sampling rate equal to the reciprocal of the call period *measured* at run time. Needed when the period is set with `-p`, which the plugin cannot read from the settings. Forced off when `epoch` is set — see below. |
| `epoch` | number (Unix s) | `0` | Shared reference instant. `0` disables it: signals are drawn sequentially, one `next()` per call, as above. Any other value switches to **synchro generation**: every sample is addressed by the index its wall-clock instant belongs to (`SigGen`'s `at()`/`index_at()`), so any number of plugin instances — on the same machine or several — that share the epoch, the sampling rate and the seed produce the exact same value at the exact same moment, however far apart each of them started. |
| *(any other scalar key)* | — | — | Copied into the document handed to SigGen, so a signal can refer to it as an algebraic expression: `"frequency": "$3 * f0"`. |

All settings are optional except `signals`; if omitted, the default values are
used. An empty or invalid `signals` map stops the agent with a `critical`
return and the reason in the startup banner.

Not every signal type can be addressed by index — an ARIMA process with a
nonzero order of integration (`d > 0`) is a cumulative sum with unbounded
history and cannot. If `epoch` is non-zero, every configured signal must
support it (`SigGen`'s `is_addressable()`); otherwise the plugin refuses to
start, the same way an empty or invalid `signals` map does.

Signal types, their parameters and the expression syntax are documented by
[SigGen](https://github.com/pbosetti/SigGen): `sine`, `square`, `triangle`,
`sawtooth`, `white_noise`, `pink_noise`, `brown_noise`, `custom`, `arima` and
`composite`.

The agent keys `period` and `pub_topic` behave as usual; `period` is *also*
read by the plugin, as the source of the sampling rate.


## Published frames

One frame per loop iteration, on the agent's `pub_topic` (the agent name by
default):

```json
{
  "t": 0.4,
  "n": 2,
  "sample_rate": 5.0,
  "temperature": 30.543,
  "pressure": 3.497,
  "position": [0.844, 0.535, 0.04]
}
```

| Field | Meaning |
|---|---|
| `t` | Time of the sample, in seconds since the first one — or, under synchro generation, the wall-clock instant (Unix s) the addressed sample belongs to. |
| `n` | Index of the sample — under synchro generation, the absolute index its wall-clock instant maps to, shared by every synchronized instance. |
| `sample_rate` | The sampling frequency, in Hz, the sample was drawn at. It varies slightly while `track_period` follows the measured loop period; it is fixed when `sample_rate` is set explicitly or `epoch` enables synchro generation. |
| *signal names* | One field per configured signal, or a single object when `nest` is set. An entry configured as an array of signals publishes a JSON array here, in declaration order, instead of a scalar. A signal named `t`, `n` or `sample_rate` overrides the field above. |

The agent adds `agent_id`, `hostname`, `timestamp` and `timecode` on its own.


## Executable demo

`build/signals` (`build/signals.plugin` on macOS) runs the plugin without a
broker as a self-checking test: it generates a noiseless sine and sawtooth at a
fixed 8 Hz and compares every sample with its analytic value, then checks the
nested output mode, the rate derived from `period`, the reproducibility of
seeded noise, that two independently started instances sharing an `epoch`
agree on the same synchro-generated frame, that a non-addressable signal
refuses to start under synchro generation, that an array-valued signal entry
publishes an ordered array with each element on its own random stream, and
that a broken configuration is reported as `critical`. It prints the last
frame and exits non-zero if any check fails.
