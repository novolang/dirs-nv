# Changelog

All notable changes to dirs-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-10-13

The first implementation of the interface published as 0.0.1. Every
signature, effect row and type the interface declared is the one
implemented here; what was a `todo()` is a table lookup, a string join
or a `None`.

### Added

- The table in `dirskind` is the whole mapping and is written as two
  columns per platform: the variable each kind is read from and the
  path it falls back to. `variable_of` and `fallback_of` are matches
  over it, `has_fallback` is `fallback_of(k, p) != ""`, and `same_as`
  compares two kinds' cells, so the roaming and local kinds are the
  same directory on XDG and macOS and two on Windows. The Windows
  fallbacks are the two application-data folders under `USERPROFILE`,
  which is what lets a kind whose variable is unset still name a real
  directory.
- `dirsenv.capture` reads thirteen variables once into the snapshot.
  `capture_as` reads the same environment under a named platform's
  rules, `of` and `with_var` build a snapshot with no environment at
  all, and `var_of` reads one back. `host_platform` asks the runtime
  which machine this is.
- `dirsbase` resolves each kind in three steps: the variable when it is
  set, otherwise the platform's fallback appended to the home
  directory, otherwise `None`. A fallback is never answered without a
  home directory, so a process started without one gets `None` rather
  than a path beginning with a separator. `join` appends a relative
  path with the platform's separator and rewrites `/` to it.
- `dirssite` answers the specification's two system-wide lists, split
  on this platform's separator with the empty entries dropped, and
  `search_path` is the specification's order — the user's own
  directory first, then each list entry. Windows reads its one system
  directory from `ProgramData` and answers an empty list when it is
  unset rather than inventing one.
- `dirsproject` derives each platform's folder name from the project's
  three names, and `dir`, `file` and `candidates` lay it under the
  shared directory, with the kind's suffix as a third level on Windows.
  `candidates` puts the user's own directory before every system entry.
- `tests/dirsedge_tests.nv` asserts all twenty-seven cells of the
  table, both columns, and the branches the four oracle suites leave
  open: an empty relative path, a kind that names no directory at all,
  an unset `ProgramData`, and a machine whose environment names
  nothing.
- `[io]` is still the only effect, and still in one module. Nothing
  opens a directory, creates one or asks whether one exists.

### Known

- **Two compiler shapes are avoided, and each names its filing at the
  site.** A `+` join is kept inside a function rather than written at a
  call site, because a `+` inside a match arm is leaked by the 0.20.2
  compiler whether or not the arm runs
  (`language/a-join-in-a-match-arm-is-leaked-even-when-the-arm-is-not-taken`,
  closed). Every call to a sibling module is spelled qualified, because
  a bare call whose answer is a heap value strands it
  (`memory-perceus/a-bare-call-to-a-sibling-modules-function-leaks-a-heap-result`,
  closed — closed is not a measurement against the 0.20.2 toolchain
  this release is verified on, and the qualified spelling is the
  specification's own in any case).
- Coverage over `src/`, measured by the publish gate across the five
  suites, is 99% of 244 instrumented lines. The two lines no suite runs
  carry a `cov: skip` marker beside their reason: the `"macos"` and
  `"windows"` arms of `host_platform`, which ask the runtime what
  machine this is and cannot be reached from a build running on Linux.
  A supplementary suite would not reach them either, since a test
  cannot change the host.
- **The layer stays `host`.** 0.0.1's entry left the `core` +
  `host_modules = ["dirsenv"]` alternative to the lane that wrote the
  bodies. It was not taken. The alternative would publish the four pure
  modules for a device that builds its own snapshot with `of`, and that
  is a real capability — but it is a release-level decision that moves
  the package's facets and API page, and nothing here measures a device
  build. The manifest's comment above `layer` states the case that was
  kept, and the alternative stays on the record.
- The shipped oracle suite `tests/dirssite_tests.nv` binds two locals
  named `path`, which shadow the stdlib module of that name and draw a
  `W0002` from the compiler on every run. The bindings are assertions'
  fixtures, not code this release owns, and the interface release
  shipped them; they are left as they are.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `dirsenv` — the environment read once into a `DirsEnv`. `capture`,
  `capture_as` and `host_platform` are the only functions in the
  package with an effect row; everything else is a pure function of the
  snapshot. A program that resolves twelve directories reads the
  environment once, a test builds a snapshot with `of` and `with_var`
  and needs no environment at all, and a tool that has to explain where
  it put a file can print what it decided from.
- `dirskind` — the nine kinds and the three platforms as a table, with
  `variable_of` and `fallback_of` making the mapping auditable against
  the XDG Base Directory Specification rather than against a body.
- `dirsbase` — where each kind is, with `None` as a real answer for a
  process with no home directory and for the runtime directory, which
  has no fallback on any platform by design.
- `dirssite` — the system-wide lists, and `search_path` as the
  specification's lookup order: the user's own directory first, then
  each list entry.
- `dirsproject` — a project named once with a qualifier, an
  organisation and an application, and spelled per platform:
  lower-cased on XDG, reverse-DNS on macOS, two nested folders plus a
  kind suffix on Windows.
- `[io]` is the only effect. Nothing opens a directory, creates one or
  asks whether one exists.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  dirs-nv.<module>.<fn>`.
- The package declares `host` because `env.get` is `[io]`. Every
  effectful function is in one module, so the implementation may
  instead declare `layer = "core"` with `host_modules = ["dirsenv"]`
  and let the resolution half build for a device. That is a decision
  for the lane that writes the bodies, and it does not change a
  signature.
