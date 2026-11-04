> [!CAUTION]
> Because of external reasons, the service is able to do all its functions except the one that it requires most: playing music. This is left as a proof of concept for anyone and everyone.

# AetherStream

A minimal, headless Spotify Connect daemon for Windows, written in modern C++20.

---

## Highlights

- **Lightweight, as intended:** Built to stay locked in on your computer and take no resources (Testing showed 20MB of RAM used by the window + CLI, below 50 if music worked).
- **Instant Zero-Setup Demo:** Pre-built standalone Windows 64-bit binaries in `bin/` ready to run out of the box (no compiler or DLL setup required).
- **Embedded Spotify Connect Layer:** Full protocol implementation featuring session negotiation, ZeroConf / mDNS discovery, and Spirc playback handling powered by `cspot`.
- **Lock-Free SPSC Ring Buffer:** Decoupled Vorbis decoder thread and WASAPI audio pump with zero locks, cache-line alignment (`alignas(64)`), and acquire/release atomic memory semantics.
- **WASAPI Audio Engine:** Shared-mode Windows Audio Session API rendering 44.1 kHz, 16-bit stereo PCM with real-time software volume scaling and `AUDCLNT_STREAMFLAGS_AUTOCONVERTPCM`.
- **Windows Credential Manager:** Secure persistence of authentication credentials via native Windows `CredWriteW` / `CredReadW` APIs alongside `credentials.json`.
- **Auto-Advance Skip Loop Breaker:** State-aware playback guard that halts runaway skipping when tracks fail before audio reaches the soundcard.
- **Headless Daemon Operation:** Non-interactive console service supporting graceful shutdown via `SIGINT`, `SIGTERM`, and Windows Console Control events (`Ctrl+C`).
- **Offline Stub Mode:** Built-in 440 Hz stereo tone generator allowing full pipeline verification (Producer Thread → Ring Buffer → Audio Pump → WASAPI) without requiring Spotify credentials.

---

## Quick Start (Pre-Built Binaries)

The `bin/` folder contains pre-built, statically linked 64-bit Windows executables that run without external dependencies:

| Script | Purpose |
|---|---|
| `bin\run_audio_demo.bat` | Launches AetherStream in **Stub Mode**. Plays a 440 Hz stereo test tone through WASAPI to verify the complete audio pipeline. |
| `bin\run_smoke_test.bat` | Executes the 8-test unit suite for the lock-free SPSC ring buffer (concurrency, saturation, wrap-around). |
| `bin\run_spotify_connect.bat` | Starts the Spotify Connect daemon broadcasting as `AetherStream` on your local network. |

---

## Proof-of-Concept Status & Upstream Spotify Note

### What Works 100%
- **mDNS / ZeroConf Discovery:** Broadcasts `_spotify-connect._tcp` service; appears instantly in Spotify desktop and mobile apps under "Connect to a device".
- **Authentication & Handshake:** Performs Diffie-Hellman key exchange, establishes Shannon encrypted tunnel with Spotify Access Points, and stores credentials securely.
- **Spirc Protocol & Metadata:** Syncs playlist queues, receives remote volume changes, fetches track metadata, CDN URLs, and album art.
- **Lock-Free Audio Engine:** SPSC ring buffer safely bridges asynchronous decoder threads and synchronous WASAPI audio delivery with zero mutex contention.
- **Infinite Loop Protection:** Accurately catches premature track load failures and halts playback instead of spamming Spotify with skip requests.

### Known Upstream Limitation: Server-Side Audio Key Refusal (`0x0001`)
When requesting track decryption keys via Mercury packet `0x0C` (`AUDIO_KEY_REQUEST`), Spotify's Access Point may respond with `0x0E` (`AUDIO_KEY_FAILURE_RESPONSE`) and error code `0x0001` (`KEY_NOT_GRANTED` / `SERVICE_UNAVAILABLE`):

```text
E MercurySession.cpp:171: [KEY_RESP] Received AUDIO_KEY_FAILURE_RESPONSE (0x0E)! Spotify error code: 0x0001 (1)
E TrackQueue.cpp:267: [TRACK_QUEUE] FAILED to get audio key! Spotify refused key. Error code=0x0001 (1)
```

* **The Cause:** Spotify is rolling out a server-side transition from legacy raw AES-128 key delivery to modern DRM (*PlayPlay* / Widevine license v2). This restriction is enforced on newer Spotify accounts (especially auto-generated alphanumeric IDs like `tqx2fmxy...`).
* **Upstream Impact:** This affects all open-source Spotify Connect implementations (documented in [librespot issue #1649](https://github.com/librespot-org/librespot/issues/1649), *cspot*, *ncspot*, *Volumio*).
* **Testing Audio:** Older Spotify Premium accounts (created prior to the PlayPlay rollout) still receive raw AES audio keys on this exact build and will stream audio normally. Alternatively, run `--stub` to test the full local audio hardware and ring buffer path.

---

## Configuration (`config.json`)

AetherStream looks for `config.json` in the current working directory, or at the path specified via `--config <PATH>`. Copy `config.json.example` to get started:

```json
{
  "device_name": "AetherStream",
  "device_type": "speaker",
  "audio_quality": 320,
  "token": "",
  "username": "",
  "password": "",
  "auth_blob": "",
  "log_file": "",
  "stub_mode": false,
  "initial_volume": 1.0
}
```

### Options Reference:
| Field | Type | Default | Description |
|---|---|---|---|
| `device_name` | string | `"AetherStream"` | Device name displayed in the Spotify app |
| `device_type` | string | `"speaker"` | Connect device category (`"speaker"`, `"computer"`, `"smartphone"`) |
| `audio_quality` | integer | `320` | Audio bitrate in kbps (`96`, `160`, or `320`) |
| `token` | string | `""` | Optional Spotify Access Token for direct auth |
| `username` | string | `""` | Spotify username (optional when using ZeroConf) |
| `password` | string | `""` | Spotify password (optional when using ZeroConf) |
| `auth_blob` | string | `""` | Encrypted reusable credential blob (JSON string) |
| `log_file` | string | `""` | Optional file path for logging instead of console |
| `stub_mode` | boolean | `false` | When `true`, plays 440 Hz test tone via WASAPI |
| `initial_volume` | float | `1.0` | Initial software volume (`0.0` to `1.0`) |

---

## Building from Source

### Prerequisites
- **OS:** Windows 10/11 (64-bit)
- **Compiler:** MinGW-w64 (GCC 13+) or MSVC (Visual Studio 2022)
- **CMake:** 3.20 or newer
- **PowerShell:** 5.1+

### 1. Bootstrap Dependencies
Run the included bootstrap script to clone `cspot`, build static `mbedtls`, and download `protoc`:

```powershell
.\setup_deps.ps1
```

### 2. Build with MinGW (GCC / CLion)
```powershell
cmake -B build -G "MinGW Makefiles" -DAETHER_WITH_CSPOT=ON -DCMAKE_BUILD_TYPE=Release
cmake --build build --target aetherstream
```

### 3. Build with MSVC (Visual Studio 2022)
Open **Developer Command Prompt for VS 2022** (x64):
```cmd
cmake -B build -G "Visual Studio 17 2022" -A x64 -DAETHER_WITH_CSPOT=ON
cmake --build build --config Release
```

---

## Command-Line Arguments

```text
AetherStream — Headless Spotify Connect daemon

Options:
  --config, -c PATH   Path to configuration file (default: config.json)
  --device-name NAME  Spotify Connect device name (overrides config)
  --log-file PATH     Log to file instead of stderr
  --stub              Run in stub mode (sine wave test tone, no Spotify)
  --help, -h          Show help and exit
```

---

## Running the Smoke Tests

Run the SPSC ring buffer unit tests:

```powershell
.\bin\test_ring_buffer.exe
```

Or via CTest:
```powershell
ctest --test-dir build --output-on-failure
```

---

## Architecture

For an in-depth explanation of the threading model, lock-free memory barriers, and the WASAPI audio pump, see [ARCHITECTURE.md](ARCHITECTURE.md).
