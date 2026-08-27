# openssl zig package

This is openssl ported to the Zig Build System.

## Artifacts

Two, matching upstream's own division:

```zig
const openssl = b.dependency("openssl", .{ .target = target, .optimize = optimize });
module.linkLibrary(openssl.artifact("crypto")); // libcrypto.a — primitives
module.linkLibrary(openssl.artifact("ssl"));    // libssl.a — the TLS stack
```

They emit `libcrypto.a` and `libssl.a`, so a consumer whose linker asks
for `-lcrypto` resolves it by name. `ssl` links `crypto` itself; the
dependency runs one way and never back.

Link only what you use. A consumer that implements TLS itself and wants
the primitives links `crypto` alone and never compiles the 95-file TLS
protocol stack — which is the point of two artifacts rather than one
archive the linker is trusted to prune.

Upgrading from the single `artifact("openssl")`: that name is gone.
Pins are by content hash, so nothing breaks until you move your pin, and
when you do it is one line per consumer.

## Status

I was able to use this to build [CPython](https://github.com/thejoshwolfe/cpython) for x86_64-linux.

Adding support for other operating systems and CPU architectures is straightforward and will
require fiddling with the build script to take into account the target.

## Zig version compatibility

- `0.16.x`
- `0.15.x`
- `0.14.x`

## Anti-Endorsement

I do not endorse openssl. I think it is a pile of trash. My motivation for this
project is because it is a dependency of CPython, which is a dependency of the
most active YouTube downloader, [ytdlp](https://github.com/yt-dlp/yt-dlp).
