# Monday

A vision-guided assistive grasping prototype for upper motor neuron injury research.

## What this is

People with an upper motor neuron injury can reach for an object but cannot finish closing the hand around it.
Monday is a research prototype that does the closing for them while the pathway rebuilds.
It combines a voice trigger, a camera, a grip planner built on Claude, and electrical muscle stimulation (EMS) of three fingers through electrodes on the back of the hand.
The patient says what they want and the camera captures the scene; Claude picks a grip, and the system speaks an acknowledgement and waits through a short abort window before it fires the EMS channels.

This is not a medical device.
It is for research use only, with explicit consent and a physical kill switch inline with the EMS positive lead.

## Repo layout

```
firmware/     Arduino Micro sketch. Relay control with mutex, watchdog, per-finger caps.
hardware/     Flask HTTP bridge on 127.0.0.1:5001. Owns the USB serial port to Arduino.
app/          Python application layer.
              state.py         shared enums and dataclasses, wire-contract truth
              vision.py        webcam grabber thread
              voice.py         wake-phrase + VAD intent capture, ManualTrigger fallback
              brain.py         Claude vision call and JSON validator
              orchestrator.py  state machine that glues the layers together
              main.py          entry point, cv2 overlay window, hotkeys, shutdown
calibration/  PyQt5 tool for electrode placement and intensity tuning
prompts/      system_prompt.txt for the brain layer
tests/        pytest suite, mock receiver, fixtures, integration scripts
docs/         known_issues.md and other operator notes
```

## Quick start (software only, no hardware)

```bash
pip install -r requirements.txt
python tests/mock_receiver.py &
python -m app.main
```

In the overlay window, press SPACE, type `grab the cup`, and press Enter.
The mock receiver's terminal then logs the `/stimulate` POSTs.
Press Q or ESC to quit.

The brain layer needs `ANTHROPIC_API_KEY`, either in the environment or in a `.env` file (copy `.env_empty` to `.env` and fill in the key).

## Full setup (with hardware)

The full cold-start procedure is in [docs/integration_bringup.md](docs/integration_bringup.md).
In short:

1. Flash the firmware.
   [firmware/README.md](firmware/README.md) has the board settings, the pinout, and the bench checklist to go through before any session with a subject.
2. Wire the electrodes: three channels on the back of the hand (over the index, middle, and pinky metacarpals), a shared ground on the wrist, and the physical kill switch inline with the EMS positive lead.
   The detailed wiring notes are in [hardware/README.md](hardware/README.md).
3. Calibrate with [calibration/stimGUI.py](calibration/stimGUI.py), which helps find an electrode placement that gives a clean twitch and the right Belifu dial setting for each channel.
   Record the session in [hardware/calibration_log.md](hardware/calibration_log.md), and run Verify Safety before every session.
4. Start the Pi services by running `./pi/start_services.sh` on the Pi over SSH (the integration guide has the details).
5. On the laptop, join the phone hotspot, point the three `MONDAY_*_URL` env vars at the Pi's IP, and run `python -m app.main`.

## Configuration

Settings that rarely change (camera resolution, wake phrase, confidence scaling, overlay geometry) are in [config.yaml](config.yaml).
These environment variables override individual components:

| variable | default | effect |
|----------|---------|--------|
| `ANTHROPIC_API_KEY` | _required_ | brain calls Claude |
| `MONDAY_CLAUDE_MODEL` | `claude-haiku-4-5-20251001` | brain model choice |
| `MONDAY_SERIAL_PORT` | auto-detected by VID | bridge serial port override |
| `MONDAY_CAMERA_INDEX` | 0 | which `/dev/videoN` vision opens |
| `MONDAY_CAMERA_WIDTH` / `MONDAY_CAMERA_HEIGHT` | 1280 x 720 | capture size |
| `MONDAY_AUDIO_INPUT` | system default | mic pin by index or name substring |
| `MONDAY_WAKE_PHRASE` | `hey monday` | voice trigger |
| `MONDAY_WHISPER_MODEL` | `base` | faster-whisper model |
| `MONDAY_WHISPER_DEVICE` | `cpu` | `cpu` or `cuda` |
| `MONDAY_VAD_THRESHOLD` | 500 | RMS floor for silence detection |

## Hardware constraint: three fingers only

The current EMS harness drives three relays, for the index, middle, and pinky fingers, and the thumb is not actuated.
The grip types are defined around that: cylindrical turns on all three, pinch uses index and middle, and lateral uses middle and pinky.
See [docs/known_issues.md](docs/known_issues.md).

## Troubleshooting

### Camera

If the camera won't open, check that `/dev/video0` exists and that the user is in the `video` group.
WSL2 does not forward the host webcam by default.
You can change the index with `camera_index` in `config.yaml` or with `MONDAY_CAMERA_INDEX`.

To check that frames are fresh, run `python -m app.vision` and compare `/tmp/monday_test_frame_{1,2,3}.jpg`.
If the three are identical, the grabber thread is not draining new frames, which is usually a driver or buffer-depth quirk.

### Voice

"Voice disabled: typing fallback active" at startup means `faster-whisper`, `sounddevice`, or PortAudio is missing; install them as described in the voice dependencies section below.
The demo still works without them, because `ManualTrigger` fires on SPACE plus a typed intent.

If nothing triggers when you speak, run `python -m app.voice --list-devices` to check that the mic is visible, then pin it with `MONDAY_AUDIO_INPUT=<index or name>` or `voice.input_device` in `config.yaml`.

If the wake phrase is often missed, try a bigger model (`MONDAY_WHISPER_MODEL=small`), or lower `MONDAY_VAD_THRESHOLD` when the mic has a noisy floor.

A phone can work as the mic.
Install WO Mic on the phone and its driver on the laptop, and connect over USB, which was more reliable than WiFi for live demos.
Check it shows up with `--list-devices`, then either make the phone the system default input or pin it with `MONDAY_AUDIO_INPUT`.
Hold the phone 15 to 30 cm from the mouth.

### TTS

If there are no spoken acknowledgements, either `pyttsx3` is missing or there is no audio output device.
The orchestrator prints `[TTS] ...` to stdout instead, which is fine for a dry run.

### Receiver

A 503 on every request means the bridge cannot reach the Arduino.
Check the cable, then look for the board in `dmesg` or `lsusb`.
The bridge retries one reconnect by itself; after that it needs a restart.

The bridge finds the Arduino by VID `0x2341`.
Non-genuine boards may use a different VID, so if the board is not detected, set `MONDAY_SERIAL_PORT=/dev/ttyACM0` to skip auto-detection.

If the bridge does not respond at all, make sure `hardware/receiver.py` or `tests/mock_receiver.py` is running on the URL in `config.yaml` (default `http://127.0.0.1:5001`).

### Overlay

A `cv2.error`, or a window that never opens, means you are in a headless environment.
`app.main` needs a real display or X forwarding; for smoke-testing the state machine without a window, use `tests/test_orchestrator_integration.py` instead.

### Voice and vision dependencies

`requirements.txt` installs these, but you can skip them on a machine that is headless or has no mic.

```bash
pip install faster-whisper sounddevice
sudo apt install libportaudio2   # Linux system package needed by sounddevice
```

## Install map

| file | contents | when |
|------|----------|------|
| `requirements.txt` | demo runtime: bridge, brain, vision, voice, TTS | always |
| `requirements-dev.txt` | pytest and fixture validator (Pillow) | running the test suite |
| `requirements-calibration.txt` | PyQt5 for the calibration GUI | operator doing electrode placement |

`requirements-calibration.txt` is kept separate because PyQt5 has no wheel on PyPI for aarch64.
On Linux ARM it builds from source and needs `qtbase5-dev` first, while x86_64 Linux, macOS, and Windows get wheels.

## Testing

```bash
pip install -r requirements-dev.txt
python -m pytest tests/ -q
```

This takes about 10 seconds on a laptop and makes no API calls.

The scripts below are not part of pytest.
The first two make real Claude calls, so they cost money:

- `python tests/test_brain.py` runs an eight-case prompt eval, one Claude call per case, and needs `ANTHROPIC_API_KEY`.
- `python tests/test_orchestrator_integration.py` traces a full run against the mock receiver, with three Claude calls per run.
- `python tests/measure_stop_latency.py` benchmarks the HTTP latency of a priority stop, using local timing only.

## Safety

Read the full safety checklist in [docs/safety_checklist.md](docs/safety_checklist.md) before powering the Belifu.
The runtime invariants are summarized below, and also at the top of [firmware/monday_ems/monday_ems.ino](firmware/monday_ems/monday_ems.ino) and [hardware/receiver.py](hardware/receiver.py).

1. The physical kill switch on the EMS positive lead is the primary safety.
2. The firmware enforces a channel mutex, a 3 s serial watchdog, a 2 s per-finger on-time cap, and boot-OFF.
3. The bridge caps `duration_ms` at 1000, validates enum values, and serializes writes.
   `/stop` bypasses the command lock and returns in microseconds.
4. The orchestrator adds a 500 ms abort window after every acknowledgement and scales durations down by confidence.
5. Every layer assumes the layers above and below may fail.
