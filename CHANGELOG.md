# Changelog

All notable changes to dirs-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
