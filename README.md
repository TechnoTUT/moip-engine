# moip-engine

A Rust media engine for NDI on Linux. It receives NDI sources and shows them fullscreen with minimal latency, sends capture devices as NDI sources, shows several sources in a grid, displays still images as signage, and encodes low-rate browser previews.

It was written for [moip-toolkit](https://github.com/TechnoTUT/moip-toolkit) and lives in its own repository so that other projects can use it too. There are two ways to use it:

- **Run the `moip-engine` binary** as a child process and drive it over a JSON protocol on stdin/stdout. Any language that can spawn a process can control it, and this is how moip-toolkit's Go server uses it.
- **Depend on the `ndi` crate** from Rust, for safe NDI bindings that load `libndi.so.6` at run time.

## Contents

```
├── ndi/                # NDI SDK bindings, loaded at run time (library crate)
└── src/                # moip-engine binary
    ├── rx.rs           # fullscreen viewer
    ├── recv.rs         # capture thread, stall recovery, audio meter
    ├── gfx.rs          # SDL2/OpenGL window, UYVY shader, labels
    ├── tx.rs           # sender: V4L2 or test pattern, plus ALSA
    ├── multiview.rs    # grid viewer
    ├── signage.rs      # image display on GL or the framebuffer
    ├── preview.rs      # H.264 / JPEG preview for the browser
    └── control.rs      # stdin/stdout JSON protocol
```

## Requirements

- Linux on x86_64 or arm64
- The NDI SDK runtime `libndi.so.6`, needed to run but not to build. It is looked up on the normal library path, or in `NDI_RUNTIME_DIR_V6`. moip-toolkit's [`scripts/install-ndi-sdk.sh`](https://github.com/TechnoTUT/moip-toolkit/blob/main/scripts/install-ndi-sdk.sh) installs it.
- To build: stable Rust, a C/C++ compiler, pkg-config, and the SDL2 and ALSA development headers

```bash
# Debian / Ubuntu
$ sudo apt install build-essential pkg-config libsdl2-dev libasound2-dev
# Fedora / RHEL
$ sudo dnf install gcc gcc-c++ pkgconf SDL2-devel alsa-lib-devel
```

The `ndi` crate on its own needs none of these.

## Build

```bash
$ cargo build --release      # target/release/moip-engine
$ cargo test
```

## Command line

```bash
$ moip-engine version                       # NDI runtime version
$ moip-engine discover --once               # list NDI sources
$ moip-engine rx -s "PC (OBS)" -f           # receive and show fullscreen
$ moip-engine tx -n CAM1 -d 0               # send /dev/video0 and the default ALSA device
$ moip-engine tx -n BARS -d -1 --no-audio   # send colour bars
$ moip-engine multiview -s CAM1 -s CAM2 -f  # grid; omit -s to show every source
$ moip-engine signage poster.png -f         # still image
$ moip-engine devices                       # capture devices as JSON
```

Run `moip-engine <command> --help` for every option. A source can be given as the full name `HOST (STREAM)` or as the stream name alone. On networks without mDNS, pass `--extra-ips 10.0.0.5,10.0.0.6` to query senders directly.

Video goes straight to KMS/DRM by default, so the viewers run on a console without a desktop session. When KMS/DRM is not available they fall back to the desktop; `--video-driver auto|wayland|x11` chooses a driver explicitly.

## Controlling the engine from another program

Start a subcommand with `--control`. The engine then exchanges newline-delimited JSON with its parent:

- **stdin → engine:** one command per line.
- **engine → stdout:** one event per line, each with a `"type"` field.
- **stderr:** human-readable log lines.

Closing stdin is treated as `quit`, so the engine exits when its parent goes away.

### Commands

| Command | Used by | Effect |
|---|---|---|
| `{"cmd":"switch","source":"PC (OBS)"}` | `rx` | Connect to another source |
| `{"cmd":"set_sources","sources":["CAM1","CAM2"]}` | `multiview` | Replace the grid; `[]` shows every source found |
| `{"cmd":"image","path":"/path/to/image.png"}` | `signage` | Show another image |
| `{"cmd":"preview","h264":true,"jpeg":false,"jpeg_width":360,"jpeg_fps":5}` | `preview` | Turn outputs on or off; the width and fps fields are optional |
| `{"cmd":"keyframe"}` | `preview` | Ask the H.264 encoder for an IDR frame |
| `{"cmd":"quit"}` | all | Exit |

### Events

| `type` | Sent by | Contents |
|---|---|---|
| `status` | `rx`, `tx`, `multiview`, `signage`, `preview` | Current state, sent periodically: connection, resolution, frame rate, latency and so on, depending on the command |
| `sources` | `discover` | `{"type":"sources","sources":[{"name":"PC (OBS)", ...}]}`, sent when the list changes and at least every 5 seconds |
| `error` | all | `{"type":"error","error":"..."}`, sent before the engine exits with status 1 |

`devices` writes a single JSON object (`{"type":"devices","video":[...],"audio":[...]}`) and exits.

### Example

```bash
$ moip-engine rx --control -s "PC (OBS)"
{"type":"status","is_connected":true,"current_source":"PC (OBS)","width":1920,"height":1080, ...}
{"cmd":"switch","source":"CAM1"}          # typed on stdin
{"type":"status","is_connected":true,"current_source":"CAM1", ...}
```

### Preview output

`preview` always reads commands from stdin and writes binary frames on stdout, so a server can forward them without re-encoding:

```
[kind: u8][flags: u8][len: u32 little-endian][payload: len bytes]

kind 0 = JSON status, 1 = H.264 access unit (Annex B), 2 = JPEG
flags bit 0 = keyframe (H.264)
```

It receives the NDI proxy stream unless `--full-bandwidth` is given.

## Using the `ndi` crate

```toml
[dependencies]
ndi = { git = "https://github.com/TechnoTUT/moip-engine.git" }
```

Cargo finds the `ndi` package inside this workspace. Add `rev = "<commit>"` or `tag = "..."` to pin a version.

```rust
use std::time::{Duration, Instant};

fn main() -> ndi::Result<()> {
    ndi::init()?; // fails with a clear message if libndi.so.6 is missing
    println!("NDI {}", ndi::version()?);

    let finder = ndi::Finder::new(false, None)?;
    finder.wait(3000);
    let Some(source) = finder.sources().into_iter().next() else { return Ok(()) };

    let recv = ndi::Receiver::new(Some(&source), &ndi::RecvOptions::default())?;
    let start = Instant::now();
    while start.elapsed() < Duration::from_secs(5) {
        if let ndi::Capture::Video(frame) = recv.capture(1000, true, false) {
            println!("{}x{} @ {:.2} fps", frame.width(), frame.height(), frame.fps());
        }
    }
    recv.destroy_in_background(); // tearing down a dead connection can block
    Ok(())
}
```

The crate also has `Sender` for publishing video and audio. No SDK headers are needed at build time, and no NDI code is linked into your binary. [`ndi/examples/churn.rs`](ndi/examples/churn.rs) is a fuller example.

## Environment variables

| Variable | Effect |
|---|---|
| `NDI_RUNTIME_DIR_V6` | Directory that contains `libndi.so.6` |
| `MOIP_MEMSTATS=1` | Log RSS and glibc heap statistics periodically |
| `MOIP_NO_MALLOPT=1` | Keep glibc's default mmap and trim thresholds. By default the engine fixes them so that frame buffers go back to the OS |

## License

[MIT](LICENSE). The NDI SDK and runtime are subject to the NDI SDK license and are not part of this repository. NDI® is a registered trademark of Vizrt NDI AB.
