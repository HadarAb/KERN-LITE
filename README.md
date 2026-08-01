# KERN-LITE

KERN-LITE is an embedded "black-box" data logger built on an STM32L476RG (Nucleo-64, LQFP64)
running CMSIS-RTOS2/FreeRTOS. It samples a small array of analog and digital sensors, runs
lightweight DSP (moving average + hysteresis threshold detection) on each channel, and durably
records timestamped, CRC-protected records to a micro-SD card via FatFs in a self-healing
circular log. A binary, CRC32-checked UART protocol streams live telemetry and accepts commands
(start/stop/status/replay/erase) from a companion Python **Ground Station** console, which
decodes, charts, and archives the incoming data in real time.

## Architecture

The firmware is organized as four cooperating FreeRTOS tasks, coordinated by
`kern::system::Orchestrator` (`firmware/system/orchestrator.hpp`):

| Task | Period | Responsibility |
|---|---:|---|
| Sensor | 100 ms | Reads LM35 (analog temp), photodiode, potentiometer, DHT11 (digital temp/humidity), buttons; runs DSP (moving average + threshold/hysteresis) per channel; publishes a `SensorRecord` to `SensorBus`; transmits it live over UART |
| Storage | 100 ms | Drains `SensorBus`, writes records into the SD card's circular log, periodically flushes log metadata |
| Comms | 10 ms | Parses inbound UART frames and dispatches commands via `CommandHandler` |
| System | 50 ms | Runs the recorder state machine, drives status LEDs, kicks the independent watchdog (IWDG) |

Both Sensor and Storage use `vTaskDelayUntil` (fixed schedule, not `vTaskDelay`) specifically to
avoid cumulative drift between the two tasks — see [`docs/design_note.md`](docs/design_note.md)
and [`README.md`](README.md) history for the record-loss investigation that drove this.

### Recorder state machine

`kern::recorder::StateMachine` (`firmware/recorder/state_machine.hpp`) tracks three states:

- **Idle** — not recording
- **Recording** — actively sampling and logging
- **Fault** — SD card unavailable/failed; recovers automatically once the card is remounted

Transitions are driven by events: `ChecksPassed`, `SdFault`, `UartStart`, `UartStop`,
`ShortPress` (onboard button), `FaultCleared`.

### Sensor pipeline (`firmware/dsp`, `firmware/sensors`)

Each analog channel (LM35 temperature, photodiode, potentiometer) runs through a
`kern::dsp::Channel<N>` (moving average over `kDspWindow` samples,
[`firmware/dsp/moving_average.hpp`](firmware/dsp/moving_average.hpp)) and a
`ThresholdDetector` with per-channel hysteresis bounds
([`firmware/dsp/threshold_detector.hpp`](firmware/dsp/threshold_detector.hpp),
configured in [`firmware/system/config.hpp`](firmware/system/config.hpp)). DHT11 humidity/
temperature is polled digitally (bit-banged single-wire protocol) and validated against its own
range/timeout/checksum, deliberately polled *after* the record for that tick has already been
published and transmitted so a slow poll can't stall the sensor bus (see the design note).

### Storage layer (`firmware/storage`)

`kern::storage::CircularLog` maintains a ring of `kLogFileCount` (4) files of
`kRecordsPerFile` (256) fixed-size 32-byte `SensorRecord`s each (`firmware/storage/
sensor_record.hpp`) on a FAT32 SD card via FatFs. A 36-byte `LogMeta` block (magic, version,
file/record indices, wrap count, total record count, CRC32) is flushed every
`kMetaFlushEveryN` (16) records and on STOP. On every mount, `CircularLog` re-derives the true
write head by scanning and CRC-validating every stored record — the flushed metadata is only a
checkpoint and can trail the real position — so recovery never depends on a clean shutdown and
never overwrites or erases a valid record. Full byte layout and the recovery algorithm are
documented in [`docs/design_note.md`](docs/design_note.md).

### Wire protocol (`firmware/protocol`)

Frames are `STX(0xAB) | TYPE | LEN | PAYLOAD | CRC32 | ETX(0xCD)`
(`firmware/protocol/frame.hpp`, `firmware/protocol/codec.cpp`, CRC32 in
`firmware/protocol/crc32.cpp`):

| Direction | Frame type | Purpose |
|---|---|---|
| Host → Device | `CmdStart` (0x01) | Begin recording |
| Host → Device | `CmdStop` (0x02) | Stop recording |
| Host → Device | `CmdStatus` (0x03) | Request a `Status` snapshot |
| Host → Device | `CmdReplay` (0x04) | Stream the last N stored records |
| Host → Device | `CmdErase` (0x06) | Erase the log (requires magic `0xDEADC0DE`) |
| Device → Host | `Status` (0x12) | State, SD mounted, file/record counts, wrap count |
| Device → Host | `Record` (0x10) | One `SensorRecord`, live or replayed |
| Device → Host | `Ack` (0x20) | Command accepted |
| Device → Host | `Nack` (0x21) | Command rejected — `CrcError`, `BadCommand`, `InvalidState`, `StorageError`, `BadMagic` |

`kern::recorder::CommandHandler` dispatches inbound frames against the state machine and the
circular log and drives replies through `CommLink`.

## Repository layout

```
firmware/               Portable C++ application logic (hardware-independent where possible)
  dsp/                  Moving average, hysteresis threshold detector, per-channel wrapper
  hal/                  Thin wrappers over STM32 GPIO/ADC/PWM/EXTI/watchdog
  protocol/             Frame definitions, CRC32, encode/decode codec
  recorder/             State machine, sensor bus (queue), comm link, command handler
  sensors/              LM35, photodiode, potentiometer, DHT11, buttons, radiation latch drivers
  storage/              Circular log (SD/FatFs) and on-disk record layout
  system/               Board pinout, runtime config constants, orchestrator, task entry points

Core/, Drivers/, FATFS/, Middlewares/, STM32*.ld, *.ioc, *.launch, .cproject, .project
                        STM32CubeIDE-generated project: HAL drivers, CMSIS, FreeRTOS/FatFs
                        middleware, linker scripts, and IDE metadata for the STM32L476RG target

groundstation/          Python ground station application (serial link, protocol, UI, analytics)
tests/host/             Host-native (g++) unit tests for firmware protocol/DSP/storage/FSM logic
tests/gs/                pytest suite for the ground station package

docs/design_note.md     Log metadata layout, crash-recovery algorithm, and a record-loss
                         investigation/fix write-up
KERN-LITE_Project_Specification.pdf
KERN-LITE_Complete_Student_Project_Handbook.pdf
                        Original project brief and reference handbook
```

## Ground station (`groundstation/`)

A serial console/telemetry client for talking to the device (`python -m groundstation.main`):

- `link.py` — serial transport (frame sync, CRC checking, error counters)
- `frame.py`, `crc.py` — protocol encode/decode mirroring the firmware's `protocol/`
- `commands.py` — command senders + periodic `CmdStatus` heartbeat poller
- `state.py`, `storage_panel.py` — decoded device/storage state models
- `telemetry.py`, `integrity.py` — record decoding, per-channel scaling, CRC/sequence-gap checks
- `session.py` — persists raw frames and decoded records per connection to `sessions/`
- `chart.py` — live rolling matplotlib chart of all channels with threshold bands
- `stats.py`, `timeline.py`, `alert_log.py`, `link_quality.py` — running statistics, state/alert
  timeline, alert log, and link-quality scoring
- `export.py` — writes `session.csv`, `raw_frames.txt`, `alert_log.txt`, `timeline.txt`
- `cli.py` — minimal one-shot command tool (`start`/`stop`/`status`/`replay`/`erase`/`listen`)
  for quick manual board testing without the full console

### Running the console

```
python -m groundstation.main --port COM4
python groundstation/main.py --port /dev/ttyACM0 --no-chart
```

Interactive commands once connected: `start`, `stop`, `status`, `replay [N]`, `erase <magic>`,
`export [dir]`, `stats`, `alerts`, `quality`, `quit`/`exit`.

### Dependencies

The ground station requires `pyserial`; `matplotlib` and `numpy` are needed for the live rolling
chart (the console falls back to headless mode via `--no-chart` if matplotlib isn't installed).
The `tests/gs` suite additionally uses `pytest`. There is no committed `requirements.txt` —
install these packages directly, e.g.:

```
pip install pyserial matplotlib numpy pytest
```

## Building the firmware

The firmware is an STM32CubeIDE project (`.project`/`.cproject`, `kern-lite.ioc`) targeting an
STM32L476RG (Nucleo-64 board, LQFP64 package). Open the project root in STM32CubeIDE and build/
flash normally, or use the generated `.cproject`/linker scripts (`STM32L476RGTX_FLASH.ld`,
`STM32L476RGTX_RAM.ld`) with `arm-none-eabi-gcc` directly. UART is configured for 115200 baud.

## Testing

**Host-native firmware tests** (`tests/host/`) build the hardware-independent firmware logic
(protocol codec, DSP, circular log, state machine) directly with `g++`, using lightweight
FreeRTOS/FatFs header shims so no target hardware or emulator is needed:

```
cd tests/host
make
./test_codec && ./test_dsp && ./test_storage && ./test_fsm
```

**Ground station tests** (`tests/gs/`) are a standard `pytest` suite covering the protocol codec,
telemetry decoding, integrity checks, session persistence, stats, timeline, link quality, and
storage panel logic:

```
pytest tests/gs
```

## Further reading

See [`docs/design_note.md`](docs/design_note.md) for the on-disk log metadata layout, the
power-loss/crash recovery algorithm, and a detailed account of a record-loss bug (task
scheduling drift compounded by a blocking DHT11 poll) found and fixed through hardware testing,
including the documented deviations from the handbook's reference implementation.
