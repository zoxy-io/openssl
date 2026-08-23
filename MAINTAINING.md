# Maintaining this fork

`allyourcodebase/openssl` packages OpenSSL for Zig. This fork exists because
two things it does not do are load-bearing for us:

- **aarch64.** Upstream ships pre-generated perlasm for x86_64 only, and has no
  `.aarch64` case in its arch switch at all. Half our targets are ARM.
  [allyourcodebase/openssl#5](https://github.com/allyourcodebase/openssl/pull/5)
  adds it; this fork carries that PR until it lands.
- **A current OpenSSL.** Upstream pins 3.3.2 (September 2024). We track 3.5.x,
  the LTS line, supported to April 2030.

It is OpenSSL rather than BoringSSL for one reason: the BoringSSL family has no
`CRYPTO_set_mem_functions`, so zoxy's fixed libcrypto heap cannot install and
the proxy refuses to start rather than run without its zero-allocation budget.
That single API decides the whole dependency.

## Consumers

Both pin a commit SHA, so nothing moves under them until someone moves it:

- [zoxy-io/zoxy](https://github.com/zoxy-io/zoxy) — terminates TLS
- [zoxy-io/zrk](https://github.com/zoxy-io/zrk) — originates TLS

## Moving the OpenSSL pin

The generated tree under `gen/` is committed, so a bump is a regeneration, not
a version string. `zig build gen` needs perl and a POSIX sh, and rewrites every
variant in place.

1. Point `build.zig.zon`'s `openssl` dependency at the new tag and hash.
2. `zig build gen`.
3. Build all four consumer targets. **Do not stop at the host** — macOS and
   Linux linkers disagree about demanding definitions for referenced-but-dead
   code, and macOS is the permissive one. A host-only check on an Apple machine
   will pass and ship a Linux build that does not link.
4. Expect the source lists to need edits, and expect the linker to be the only
   thing that can tell you which. An inventory diff finds what upstream
   *deleted*; nothing short of linking finds what it *added* that you now need.

What 3.3.2 → 3.5.7 cost, as a sense of scale: 3 sources removed or moved, 39
added, 4 templates, 1 include path, and 233 regenerated files. Most of the
additions were post-quantum — ML-KEM, ML-DSA, SLH-DSA — which 3.5's default
provider registers unconditionally, so they are not optional. It took five link
passes to converge: 4 → 67 → 2 → 31 → 8 → 0 undefined symbols, each batch of
newly compiled files pulling in the next.

Before choosing a version, measure it. 3.6.3 was rejected for restructuring the
provider tree: 62 sources and 2 templates orphaned, against 3 for 3.5.7.

## Upstream

Send anything portable to `allyourcodebase/openssl`. This fork should shrink,
not grow.
