# libbytevc2 — ByteDance VVC accelerator integration

This fork patches FFmpeg with an extra video decoder, `libbytevc2`, which
wraps **`libbyteVC2dec.so`** — ByteDance's proprietary software VVC
(H.266) decoder. According to ByteDance documentation it is **2–4x faster
than FFmpeg's built-in `vvc` decoder**, which matters because (as of
2025–2026) almost no mobile SoC implements hardware VVC decoding yet.

The wrapper is **arm64-only** (the SO only ships for `arm64-v8a`). For other
ABIs the build automatically falls back to FFmpeg's built-in `vvc` decoder.

## What this fork does

1. Adds `libavcodec/libbytevc2.c` — a dlopen-based AVCodec wrapper.
   No link-time dependency on the SO; loads at runtime.
2. Inserts the wrapper into `allcodecs.c` **before** `ff_vvc_decoder`, so
   `avcodec_find_decoder(AV_CODEC_ID_VVC)` picks the accelerated wrapper
   when enabled.
3. Adds `--enable-libbytevc2` to the FFmpeg configure script.
4. Patches `buildscripts/flavors/default.sh` to enable the wrapper only
   when building for `aarch64`.

The patch lives at `buildscripts/patches/ffmpeg/libbytevc2.patch`.

## How `libbytevc2` finds the SO at runtime

The wrapper tries these paths in order:

1. `libbyteVC2dec.so`     (whatever `dlopen` resolves it to — usually the
                            APK's own `lib/arm64-v8a/` directory)
2. `/system/lib64/libbyteVC2dec.so`
3. `/vendor/lib64/libbyteVC2dec.so`

If none is found, the wrapper's `init()` returns `AVERROR(ENOSYS)`, and
FFmpeg falls back to the next available decoder for `AV_CODEC_ID_VVC`
(typically the built-in `vvc`).

**You must ship `libbyteVC2dec.so` separately** — it is not in this
repository and cannot be redistributed here for license reasons. Place it
alongside `libmpv.so` in your APK's `lib/arm64-v8a/` directory.

## How to make `avcodec_find_decoder` actually pick this

There are two paths:

1. **Default selection (recommended)**: as long as the wrapper's `init`
   succeeds (i.e., the SO loads), it is the first match for
   `AV_CODEC_ID_VVC` in the codec list and will be selected automatically.

2. **Explicit selection (debug / A-B)**:
   - mpv: `--vd=libbytevc2,vvc` (try wrapper first, fall back to native)
   - ffmpeg CLI: `-c:v libbytevc2`
   - ffmpeg API: `avcodec_find_decoder_by_name("libbytevc2")`

## What input the wrapper expects

The wrapper converts each `AVPacket` to Annex-B start-code framing on
the fly (it accepts both length-prefix mp4-style packets and already-
Annex-B packets transparently). It then calls `byteVC2dec_decode()` —
the synchronous entry point — which is the only one we found to work
reliably from non-zygote processes during reverse engineering.

Quirks observed in PoC:

* Input MUST start with an IDR/CRA NAL or contain a complete AU (SPS +
  PPS + IDR) — otherwise the SO logs `The first frame is not I frame.`
* `byteVC2dec_send_packet` + `byteVC2dec_get_frame` (async path) tends
  to deadlock outside a real Android app process; the wrapper avoids
  this by using `byteVC2dec_decode` exclusively.

## Verifying the build

After CI produces `default-arm64-v8a.jar`, unzip it and check:

```bash
unzip -p default-arm64-v8a.jar lib/arm64-v8a/libmpv.so > libmpv.so
llvm-nm -D libmpv.so | grep -E 'ff_(vvc|libbytevc2)_decoder'
# Expected:
#   <addr> D ff_libbytevc2_decoder
#   <addr> D ff_vvc_decoder
```

Both decoders should be present; `ff_libbytevc2_decoder` is the one
selected at runtime when `libbyteVC2dec.so` is available.

## License

* This patch (everything under `buildscripts/patches/ffmpeg/`): LGPL-3.0+
  (matches FFmpeg LGPL build).
* `libbyteVC2dec.so` itself: ByteDance proprietary. Ship it separately.
