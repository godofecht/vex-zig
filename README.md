# vex-zig

Working index for the Zig projects. Each entry below is a submodule pointing at
its own repository; this repo holds only the pins.

## Clone

```sh
git clone --recurse-submodules https://github.com/godofecht/vex-zig.git
```

If you already cloned without submodules:

```sh
git submodule update --init
```

## What is here

| Submodule | Visibility | What it is |
|---|---|---|
| [azazel](https://github.com/godofecht/azazel) | public | CUE-driven build configuration layer. Generates `build_spec.zig` from `project.cue` and carries the 10-repo huge Zig corpus readiness harness. |
| [danzig](https://github.com/godofecht/danzig) | public | VST3 plugin framework in pure Zig. No JUCE, no SDK, no external dependencies. |
| [zaza](https://github.com/godofecht/zaza) | public | Zig-driven build system for C, C++, Zig, CMake interop, and WebAssembly. |

zaza-agent, the MCP server exposing zaza and azazel as agent tools, is a
private repo and deliberately not a submodule here. A public index cannot
carry a private submodule without breaking `--recurse-submodules` for anyone
without access to it.

## Toolchain

Zig **0.14.1, 0.15.2, or 0.16.0**. All three submodules build and test on each,
and CI runs a matrix over all three. 0.14.0 is rejected by capy, which gates on
an exact match.

```sh
zig version   # 0.14.1, 0.15.2, or 0.16.0
```

Each submodule builds and tests on its own:

```sh
cd danzig && zig build test        # unit + VST3 ABI integration
cd zaza   && zig build test        # 87 tests
cd azazel && ./gen_build_spec.sh && zig build test
```

azazel needs `cue` on PATH, since `build_spec.zig` is generated and gitignored.
Its huge-project corpus harness also has a pinned 10-repo batch plan for checking
large Zig integrations across forks:

```sh
cd azazel && tools/huge_corpus.py --plan --expect-count 10
```

## Not tracked here

Three third-party checkouts live in this directory and are gitignored. They are
nothing to do with the projects above and are kept only for local reference.

| Directory | Upstream | Size |
|---|---|---|
| `capy/` | [capy-ui/capy](https://github.com/capy-ui/capy) | 446 MB |
| `zig/` | [ziglang/zig](https://codeberg.org/ziglang/zig) | 671 MB |
| `zig-libs/zylix/` | [kotsutsumi/zylix](https://github.com/kotsutsumi/zylix) | 79 MB |

Nothing in the three submodules depends on any of them.

## Updating pins

A submodule pin is a commit, not a branch, so it does not move on its own.

```sh
git submodule update --remote azazel   # fast-forward one to its tracked branch
git add azazel && git commit -m "Bump azazel"
```

To see what has drifted:

```sh
git submodule foreach 'git log --oneline HEAD..origin/main | head'
```
