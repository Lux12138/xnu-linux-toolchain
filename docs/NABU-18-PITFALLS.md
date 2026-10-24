# Building xnu-10063.141.1 for nabu on Linux (WSL Ubuntu 24.04)

Field record from the Xiaomi Pad 5 (SM8150) port. The upstream LINUX-BUILD.md covers the toolchain
itself; this file adds the board-specific results and the pitfall list gathered while driving the
build every cycle.

## Result

| item | value |
|---|---|
| build time | 72 s per cycle (GitHub macOS runner: 15-17 min) |
| artifact | 49 MB Mach-O arm64, `NOUNDEFS\|PIE` |
| `__TEXT` vmaddr | `0xfffffff007040000` -- matches the original kernelcache |
| symbols | 22,708 |
| markers | `nabu_mark` / `nabu_mark_map` / `nabu_mark_num` / `nabu_mark_putc` |
| UFS probe | `nabu_ufs_register_probe.cpp` |
| sha256 | `c756ecd8b89add50155dc27d2cdf4a0a0ddf93fcf204040fa57804f814fc3c15` |

## The 18 pitfalls

| class | count | examples |
|---|---|---|
| toolchain | 8 | `libdispatch` has no Ubuntu package; `//` hostcompat.h; full `libc.h` had to be written; `copyfile.h` setmode |
| path hardcoding | 5 | `/usr/bin/xcrun`, `/usr/sbin/sysctl`, `compression_tool`, `dSYMutil`, `lzfse` |
| BSD vs GNU | 3 | `sed -i ''`, `awk \`(`, unknown `-Wno-` flags |
| logic conflicts | 2 | mockfs double link (23 duplicate symbols), `TargetConditionals.h` re-include |

### The most valuable one

`makesyscalls.sh` prints `printf("\t{ \($sys_call_t *)...")`. BSD awk eats the backslash, gawk keeps it,
so the generated `init_sysent.c` is full of `\(sy_call_t *)` and the compiler rejects it with
`expected expression`. `patches/xnu/zzzz-nabu-awk-parens.patch` removes the backslashes. The `zzzz-`
prefix keeps it last in the apply order, because `unixnu tree` rebuilds `.cache` on every cycle and any
hand edit to the tree is wiped.
