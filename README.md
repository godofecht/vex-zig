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
| [azazel](https://github.com/godofecht/azazel) | public | CUE-driven build configuration layer. Generates `build_spec.zig` from `project.cue`. |
| [danzig](https://github.com/godofecht/danzig) | public | VST3 plugin framework in pure Zig. No JUCE, no SDK, no external dependencies. |
| [zaza](https://github.com/godofecht/zaza) | public | Zig-driven build system for C, C++, Zig, CMake interop, and WebAssembly. |
| [zaza-agent](https://github.com/godofecht/zaza-agent) | private | MCP server exposing zaza and azazel as tools, plus its skill docs. TypeScript. |

## Toolchain

Zig **0.14.1**. Not 0.14.0: capy gates on an exact version match and rejects it.

```sh
zig version   # 0.14.1
```

Each submodule builds and tests on its own:

```sh
cd danzig && zig build test        # unit + VST3 ABI integration
cd zaza   && zig build test        # 87 tests
cd azazel && ./gen_build_spec.sh && zig build test
```

azazel needs `cue` on PATH, since `build_spec.zig` is generated and gitignored.

## Not tracked here

Three third-party checkouts live in this directory and are gitignored. They are
nothing to do with the projects above and are kept only for local reference.

| Directory | Upstream | Size |
|---|---|---|
| `capy/` | [capy-ui/capy](https://github.com/capy-ui/capy) | 446 MB |
| `zig/` | [ziglang/zig](https://codeberg.org/ziglang/zig) | 671 MB |
| `zig-libs/zylix/` | [kotsutsumi/zylix](https://github.com/kotsutsumi/zylix) | 79 MB |

Nothing in the four submodules depends on any of them.

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
