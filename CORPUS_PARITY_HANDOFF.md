# Corpus Parity — Full Record

Status as of 2026-08-06. This is the complete record of the azazel + zaza corpus
parity effort: eleven public repos, the azazel features the corpus drove, the
measured build-speed findings, and the honest limits.

Comparison dashboard (all eleven, live):
https://claude.ai/code/artifact/8c37ee83-b358-4351-a1e0-eb02ec0aedd4

## What the corpus is

Real-world Zig projects that azazel and zaza are pressure-tested to reproduce.
Each project is now a standalone **public** repo under `godofecht/azazel-parity-*`
that builds the upstream **two ways** — azazel (a CUE model, build-as-data) and
zaza (the Zig build graph) — with head-to-head CI and a speed/structure
comparison in the README. This supersedes the earlier `.azazel/` fork-overlay
approach (those overlays still exist on the `azazel-zaza-integration` fork
branches and remain valid; the public repos are the presentable layer).

The zaza side aligns with the other agent's in-repo `corpus/` methodology
(zaza issue #46): **never vendor upstream sources** — each repo's `fetch.sh`
stages the pinned upstream into a git-ignored `vendor/`, and a small consumer
exercises the slice.

## The eleven public parity repos

All CI-green. Clean-cache build times (Apple Silicon, fastest of two, deps
pre-fetched). `native` = the upstream's own `zig build` where reproducible.

| Repo (github.com/godofecht/azazel-parity-…) | Slice | Lane | azazel | zaza | native |
|------|-------|------|--------|------|--------|
| **zig** | the Zig compiler's own tokenizer (`lib/std/zig/tokenizer.zig`, ~1770 ln) | 0.16 | 4.5s | 4.3s | — |
| **tigerbeetle** | whole VSR library (`option_values` vs `addOptions`) | 0.14 | 3.2s | 3.0s | 12.0s |
| **libvaxis** | full `vaxis` lib (uucode `fields`) + gwidth consumer | 0.16 | 6.7s | 6.5s | 7.7s |
| **libxev** | `libxev.a` from source / C event-loop consumer | 0.16 | 4.5s | 3.9s | 4.8s |
| **ghostty** | `UTF8Decoder` (terminal lexer) consumer | 0.16 | 4.6s | 4.4s | — |
| **capy** | full GUI library, macOS AppKit backend | 0.14 | 4.5s | 4.1s | — |
| **zig-gamedev** | whole zmath library (package consumer) | 0.16 | 4.8s | 4.6s | 3.8s |
| **mach** | `mpsc` Queue+Pool consumer | 0.17 | 3.5s | 3.2s | — |
| **zls** | `Uri` parse/normalize consumer | 0.17 | 3.5s | 3.3s | — |
| **microzig** | `flags` parser consumer | 0.17 | 3.4s | 3.1s | — |
| **river** | Wayland compositor `util` (allocator + monotonic clock) | 0.16 | 4.4s | 4.3s | — |

Notes:
- **Faster than native** where the upstream ships a full build and azazel/zaza
  build a scoped slice of it: tigerbeetle **3.7×** (3.2s vs 12.0s), libxev,
  libvaxis. The one honest exception is **zig-gamedev**: zmath is consumed as a
  package, so the build runs zmath's own `build.zig` *plus* a consumer — you
  can't build a package faster than it builds itself (~0.8s slower).
- Single-file slices (ghostty, mach, zls, microzig, zig, river) have no
  reproducible native full build here (huge / version-locked / broken deps /
  system-lib dependent), so `native` is left blank rather than faked.

## Azazel features the corpus drove (all merged to `main`)

Each was demanded by a specific repo that couldn't be modeled without it:

| PR | Feature | Driven by |
|----|---------|-----------|
| #36 (#37) | `artifact_name` — decouple artifact name from `@import` name | libxev (lib:xev + module:xev) |
| #39 | the `"0.17"` lane + `ArrayList`→`dependOn` fix | mach · zls · microzig |
| #40 | `pkg_imports.fields` — pass a string-list option to a dep | libvaxis (uucode table selection) |
| #41 | `option_values` — inject typed build-option values (bool/string/u32/`?[40]u8`) | tigerbeetle (`vsr_options`) |
| #42 | `gen_imports` — compile a host tool, run it, import its output as a module | zls (`version_data` codegen) |
| perf | memoize CUE codegen — skip regeneration when the model is unchanged | (build speed) |

## Build-speed findings (measured)

The question was "why can't Zig get Bazel/Ninja speedups?" Measured answer:

| Lever | Zig (measured) | Note |
|-------|----------------|------|
| **Caching** (content-addressed) | **89×** — 14.2s cold → 0.16s rebuild | Zig already has it; azazel/zaza inherit it |
| **Incremental** (edit one file) | **10.8×** — 14.2s → 1.32s | deps stay cached, only the changed target recompiles |
| **CI dependency cache** (shipped) | **2×** — cold 13.3s → warm 6.6s | `actions/cache` on `~/.cache/zig`, rolled out to all repos |
| **Memoized CUE codegen** (shipped) | **20×** — 0.20s → 0.01s | erases azazel's only per-build overhead; azazel now matches zaza |
| **Parallelism** (many cores) | **1.1×** — marginal | see below |
| **GPU** | none | compilation is branchy, sequential, dependency-ordered — anti-GPU |

**Why parallelism barely helps Zig (1.1×):** Bazel/Ninja win on C++ because C++
has thousands of heavy, independent translation units. Zig's model is the
opposite — one mostly-single-threaded compile per artifact, a fast self-hosted
backend, and a shared `std` compiled once and cached. The per-target unique work
is tiny next to `std` + fixed overhead, so there's almost nothing to spread
across cores. Measured on 8 heavy independent targets across 14 cores: parallel
3.8s vs serial 4.1s. **For Zig, caching is the whole game, not parallelism.**

GPU can only legitimately touch build-time **asset generation** (texture
compression, mesh processing, big table generation) in a game engine like mach
or zig-gamedev — the payload, not the build system.

### The real lever: residency, not parallelism

The modern feature for a compiler like Zig is not spreading work across cores —
it's keeping the compiler **resident**. Measured two ways:

- **Batch `-fincremental` gives nothing** (1.16s cold → 1.22s after a 1-line
  edit → 1.19s no-change): each `build-exe` invocation cold-starts and discards
  the in-memory state. Incrementality only pays off against a compiler that stays
  alive.
- **`zig build --watch -fincremental` (resident) does pay off.** On a
  66k-line / 6000-function project: cold **5.66s**, and a resident incremental
  rebuild after editing one declaration **1.94s — 2.9×**. The revealing part:
  editing a single *leaf function* (1.99s) costs the same as editing a *constant*
  (1.94s), and both match a fully-cached *warm rebuild that only relinks* (2.30s).
  So the resident compiler recompiles the changed declaration in milliseconds —
  **the entire residual ~2s is the link step** relinking the 4.9 MB binary,
  independent of edit size.

The measured conclusion: incremental *compilation* is already near-free with a
resident compiler; **the wall is the link**. That is precisely what Zig's
roadmap **in-place binary patching** removes — patch the changed function's bytes
into the existing binary and skip the relink. The 2.9× is the compiler going
resident; the remaining ~2s is the relink that patching would erase.

What would really improve Zig's build philosophy from here:

1. **A resident compile server (daemon).** Keep the compiled program + InternPool
   hot; on edit, recompile only the changed *declarations* and their dependents
   (not files, not TUs) in ms. `zig build --watch` + the compiler server is the
   start of this. It makes "warm" the default instead of something re-earned per
   build.
2. **In-place binary patching** (Zig's roadmap). The self-hosted backend emits
   relocatable, patchable code, so a changed function is recompiled and its bytes
   patched into the running binary — no relink, no restart. This is enabled
   precisely because Zig is in-memory, self-hosted, and declaration-addressed;
   LLVM/C++ can't do it.
3. **Content-addressed *shared* cache** — the transferable half of Bazel. Not
   remote *execution* (parallelism, which doesn't help Zig) but remote *caching*:
   the first machine to compile a pinned dep shares the artifact with the team
   and every CI run.

**azazel's build-as-data is unusually well-placed for all three:** the build is a
query ("what does editing X rebuild?" answerable without running the compiler),
the content-addressed cache key is computable from the pinned model before
invoking anything, and "compile only what's used" (the `fields` trick) is
dead-declaration elimination driven by the model. The build system's job shifts
from "orchestrate a batch of compiles" to "keep one warm compiler and hand it the
minimal delta."

## Config complexity

azazel's declarative CUE model is a **median 1.9× smaller by bytes** than zaza's
`build.zig` for the same result — up to **5×** on libxev (10 lines of CUE vs 51
of build.zig). The exceptions are zls and microzig, tiny std-only consumers
where zaza's dozen lines undercut the CUE. Most of zaza's extra lines are the
Zig build-graph boilerplate azazel generates from data.

## What azazel/zaza do that takes ages in raw Zig

- **Generated-code pipelines (azazel `gen_imports`).** zls hand-writes 22 lines
  to compile a tool, run it with file args, capture the output, and wrap it as a
  module (with `b.graph.host`, `addFileArg` vs `addOutputFileArg`,
  `single_threaded` landmines). azazel: a data block.
- **C/C++ transitive usage requirements (zaza).** `cpp_example.zig` has 235
  references to public/private/usage/cmake — CMake's `PUBLIC`/`PRIVATE`
  propagation of include dirs, defines, and link libs across dep edges, plus
  CMake config import/export. Raw `build.zig` makes you thread every transitive
  include/define by hand.
- **A build graph you can unit-test (azazel).** `build_spec_test.zig` runs 23
  tests over the graph — acyclic, deps resolve, names unique, roots exist. You
  can't unit-test an imperative `build.zig`. Same property powered the dashboard.
- **A four-version toolchain in one config (azazel).** The corpus spans Zig
  0.14–0.17 where `std.Build` changed hard; `compat.zig` (42 lines) absorbs it.

## Method (reproduce a repo)

Each public repo has two self-contained build roots under `azazel/` and `zaza/`,
each with its own `fetch.sh`:

```
azazel-parity-<repo>/
  README.md  .gitignore  .github/workflows/ci.yml
  azazel/  build.zig schema.cue gen_build_spec.sh compat.zig
           project.cue export.cue build.zig.zon  fetch.sh  src/consumer.zig
  zaza/    build.zig build.zig.zon  fetch.sh  src/main.zig  [PROOF.md]
```

1. `fetch.sh` stages the pinned upstream into `vendor/` (git-ignored) — a `curl`
   for single-file slices, a shallow clone for source-tree slices.
2. azazel: `sh gen_build_spec.sh` (CUE → `build_spec.zig`) then `zig build`.
3. zaza: `zig build` (or `zig build run`) via the standard Zig build graph.
4. CI runs both on the repo's lane, with `actions/cache` on `~/.cache/zig`. The
   0.17 repos download the mach-nominated `0.17.0-dev.892` from the hexops
   mirror (it persists, unlike rotating ziglang nightlies). capy runs on
   `macos-14`.

## Notable findings / fixes

- **Bun is no longer a Zig project.** Cloned it: 1495 `.rs`, 612 `.cpp`, a
  `Cargo.toml`, and **zero `.zig` files**. Fully migrated to Rust — cannot be a
  target. That's why "the biggest" landed as the Zig compiler + River rather
  than Bun.
- **capy CI: `macos-14`.** `macos-latest` is now macOS 26 / Xcode 26.6, whose
  SDK Zig 0.14.1's Mach-O linker can't link (undefined libSystem symbols even
  with `SDKROOT`); `macos-13` runners are being retired (jobs queued >24h).
  `macos-14` runs and links fine.
- **zaza PR #57 CLOSED.** A proposed `zig_root` pure-Zig target path on
  `zaza.Target` contradicted zaza's deliberate design (pure Zig uses the standard
  Zig build graph, not the C/C++ DSL — see `corpus/libvaxis/build.zig`). Aligned
  to the existing methodology instead.

## Next steps (open)

- **capy's parity repo depends on a `zig-objc` pin** (`2329503`, the 0.14-era
  commit) staged from the `godofecht/capy` fork; upstream capy's own examples are
  broken at that commit, so its native full build isn't a baseline.
- Blocked-for-real fuller targets (documented, not azazel-fixable here): ghostty
  terminal-core (pulls the font/freetype graph + an enum `terminal_options`),
  zls full library (diffz/lsp deps need a 0.17 toolchain none available compiles),
  mach engine module (generated Vulkan bindings), river full build (Wayland +
  pkg-config).
- A content-addressed / remote dependency-artifact cache is the natural next
  build-speed lever: azazel holds every dep by pinned hash as data, so it can
  reuse compiled artifacts across machines and compile only used fields (as
  libvaxis already does with uucode) — something a hand-written `build.zig`
  can't expose as cleanly.
