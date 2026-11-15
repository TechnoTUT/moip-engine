# moip-engine

Rust media engine for [moip-toolkit](https://github.com/TechnoTUT/moip-toolkit): NDI receive/display, send, multiview, signage and preview.

This repository is used by moip-toolkit as a git submodule at `engine/`.

## Layout

```
├── ndi/                # NDI SDK bindings, loaded at run time
└── moip-engine/src/
    ├── rx.rs           # fullscreen viewer
    ├── recv.rs         # capture thread, stall recovery, audio meter
    ├── gfx.rs          # SDL2/OpenGL window, UYVY shader, labels
    ├── tx.rs           # sender: V4L2 or test pattern, plus ALSA
    ├── multiview.rs    # grid viewer
    ├── signage.rs      # image display on GL or the framebuffer
    ├── preview.rs      # H.264 / JPEG preview for the browser
    └── control.rs      # stdin/stdout JSON protocol with the server
```

## Build

Requires stable Rust, a C/C++ compiler, and the SDL2 and ALSA development headers.

```
$ cargo build --release
$ cargo test
```

The NDI SDK is not needed to build: `libndi.so.6` is loaded at run time. See the moip-toolkit README for installing the runtime.

## License

[MIT](LICENSE). The NDI SDK and runtime are subject to the NDI SDK license and are not part of this repository. NDI® is a registered trademark of Vizrt NDI AB.
