# dirs-nv

A program that stores something on a user's machine has to put it
somewhere, and each operating system has a different answer. On Linux
and the BSDs the answer is the
[XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/),
version 0.8. On macOS it is the `~/Library` layout, and on Windows it
is the roaming and local application-data folders. This package answers
where each directory is, on all three.

## What the directories are

A **base directory** is a place a program is meant to write into. There
are nine kinds, and they differ in what may happen to what is inside
them.

| Kind | What goes in it |
| --- | --- |
| `home` | The user's home directory. Every other one is derived from it on at least one platform |
| `config` | Configuration a user edits |
| `config-local` | Configuration that should not follow a user between machines |
| `data` | Data a program wrote and must not lose |
| `data-local` | Data that should not follow a user between machines |
| `cache` | Data a program can regenerate. Anything here may be deleted between runs |
| `state` | Logs, history, a window layout, a recently-used list |
| `runtime` | Sockets, named pipes and lock files, cleared when the user logs out |
| `executable` | A user's own programs |

On an XDG system each of the first eight is read from an environment
variable, with a fallback relative to the home directory.

| Kind | Variable | Fallback |
| --- | --- | --- |
| `config` | `XDG_CONFIG_HOME` | `$HOME/.config` |
| `data` | `XDG_DATA_HOME` | `$HOME/.local/share` |
| `cache` | `XDG_CACHE_HOME` | `$HOME/.cache` |
| `state` | `XDG_STATE_HOME` | `$HOME/.local/state` |
| `executable` | `XDG_BIN_HOME` | `$HOME/.local/bin` |
| `runtime` | `XDG_RUNTIME_DIR` | none |

`config-local` and `data-local` are the same directory as `config` and
`data` on every platform except Windows, where there is a real
distinction between a folder that follows a user across a domain and
one that does not. A program can use the local kinds everywhere and be
correct everywhere.

Two of the kinds also have **system-wide** directories: a program looks
for a configuration or a data file in the user's own directory first,
and then in each of these in turn.

| Variable | Default |
| --- | --- |
| `XDG_CONFIG_DIRS` | `/etc/xdg` |
| `XDG_DATA_DIRS` | `/usr/local/share`, then `/usr/share` |

A **project directory** is a program's own folder inside one of those.
Each platform names it differently, so a project is named once with a
qualifier, an organisation and an application, and the spelling is
derived.

| Platform | Folder for `org`, `Novo Lang`, `Novo` |
| --- | --- |
| XDG | `novo` |
| macOS | `org.novo-lang.Novo` |
| Windows | `Novo Lang\Novo\config` |

## Install

```
novo pkg add dirs-nv
```

## Example

```novo
use dirsenv
use dirsproject

fn main() [io]
    // The one call that reads the environment. Everything after it is
    // a pure function of the value it returns.
    let env = dirsenv.capture()

    let novo = dirsproject.project("org", "Novo Lang", "Novo")

    // Where this program's configuration file would be. Nothing is
    // created and nothing is checked: this is a path, not a file.
    match dirsproject.file(env, novo, DirsConfig, "config.toml")
        Some(path) => println(path)
        None       => println("no home directory: cannot place a config file")

    // Every place the same file would be looked for, the user's own
    // directory first.
    let places = dirsproject.candidates(env, novo, DirsConfig, "config.toml")
    for place in places
        println("  looking in ${place}")

    // The runtime directory has no fallback anywhere, so None is a
    // normal answer and a program must have something to do about it.
    match dirsproject.runtime(env, novo)
        Some(path) => println("sockets go in ${path}")
        None       => println("no runtime directory; using a temporary one")
```

Build and test with `novo pkg build` and `novo test`.

## What the package contains

| Module | Contents |
| --- | --- |
| `dirskind` | The nine kinds, the three platforms, and the table of which variable and which fallback each pair uses. |
| `dirsenv` | The environment, read once into a snapshot. The only module with an effect. |
| `dirsbase` | Where each kind of directory is, for this user. |
| `dirssite` | The system-wide lists, and the order a file is looked for in. |
| `dirsproject` | One program's own folder inside the shared directories, named the way each platform wants it named. |

## How to choose an entry point

**`dirsenv.capture` is the first call.** It reads the environment once,
under this machine's rules, and every other function takes its result.

**`dirsenv.capture_as` reads the same environment under a named
platform's rules.** An installer, a container image builder and a
documentation generator that prints where each platform would put a
file all want it.

**`dirsenv.of` and `dirsenv.with_var` build a snapshot with no
environment at all.** A test uses them, and so does a program that has
decided to ignore the environment.

**`dirsproject` is what an application wants.** It puts a folder of the
program's own inside the shared directory, which is what a program
should be doing.

**`dirsbase` is what a tool that manages other programs' files wants.**
It answers the shared directory itself.

**`dirssite.candidates` and `dirsproject.candidates` answer a list.**
Use them when a file may come from the system as well as from the user.

## The rules a user needs

1. **Nothing here touches the filesystem.** Every function answers
   where a path would be. None asks whether it exists, none creates it,
   and none reads it. A caller that wants the directory to exist
   creates it.
2. **Three functions read the environment and no others.**
   `dirsenv.capture`, `dirsenv.capture_as` and `dirsenv.host_platform`.
   Everything else is a pure function of a `DirsEnv`. A program that
   resolves twelve directories reads the environment once, so its
   answers cannot disagree with each other.
3. **`None` is a real answer, and it happens two ways.** Either there
   is no home directory — a daemon, a container, a CI runner — in which
   case almost everything is `None` and the program should say so at
   start-up. Or the kind has no fallback, which is the next rule.
4. **`runtime` has no fallback on any platform.** The XDG specification
   says an application must not create the directory itself, because
   its owner, its permissions and its removal at logout are the login
   session's to provide. A program that needs one falls back to a
   temporary directory and says in its log that it did.
5. **`executable` exists only on XDG.** macOS and Windows have no
   per-user program directory, and answering with an invented path
   would send an installer there.
6. **An empty variable counts as unset.** The XDG specification says a
   variable set to an empty value must be treated as though it were
   undefined, so the snapshot does not distinguish the two.
7. **The user's own directory comes first in every search path.** That
   is what lets a user's file override a package's default.
   `dirssite.search_path` is that order.
8. **`cache` may be emptied at any time**, by the user or by the
   system. A program that cannot survive that is using the wrong
   directory. `state` is the one for a log or a history that should
   survive but does not belong in a backup of the configuration.
9. **A project is named three ways and spelled once per platform.**
   `dirsproject.folder_name` is the whole rule. A program that
   hard-codes one spelling puts a reverse-DNS folder in a Linux user's
   `~/.config`.
10. **On Windows a project directory gets a third level.** Both
    application-data folders hold everything, so without
    `dirsproject.windows_suffix` a program's configuration directory
    and its cache directory would be the same path.
11. **A project with only a name is allowed.** Pass `""` for the
    qualifier and the organisation. The derived names leave out the
    parts that would have been empty rather than producing a stray dot
    or backslash.
12. **Windows separates a directory list with `;`.** A colon there is
    part of a drive letter.
13. **Paths are joined with the platform's separator and are not
    normalised.** A `..` in a relative path stays a `..`.

## What is not included

- **Creating a directory, or checking that one exists.** See rule 1.
  Both need `[fs]`, and this package declares only `[io]`.
- **The user's documents, downloads, pictures and desktop
  directories.** They are read from `~/.config/user-dirs.dirs` on XDG
  and from the shell's known-folder API on Windows. The first needs to
  read a file and the second needs a system call, and neither is an
  environment variable.
- **Expanding `~` in a path a user typed.** `dirsbase.home` gives the
  directory; substituting it into a string is the caller's, and a
  caller that does it has to decide what `~other` means.
- **A temporary directory.** `TMPDIR` is not part of the base
  directory specification, and the answer a program wants there is
  usually its language runtime's, which knows about the per-process
  cleanup as well as the path.
- **Reading or writing a configuration file.** This package answers
  where one goes.

## Related packages

- [config-core-nv](https://novo-lang.org/packages/config-core-nv) reads
  and merges configuration once a file has been found. This package
  says where to look.
- [toml-nv](https://novo-lang.org/packages/toml-nv) parses what is in
  the file.
- [shlex-nv](https://novo-lang.org/packages/shlex-nv) splits a command
  line, and leaves a `~` as an ordinary character for the same reason
  this package exists.

## Tests

```bash
novo test tests/dirskind_tests.nv      # the table, against the specification
novo test tests/dirsbase_tests.nv      # resolution, over a snapshot
novo test tests/dirssite_tests.nv      # the lists and their order
novo test tests/dirsproject_tests.nv   # a project's folder, three ways
novo test tests/dirsedge_tests.nv      # every cell of the table, and the edges
```

The normative source is the XDG Base Directory Specification version
0.8. The reference implementations are the Rust crate `directories`,
whose three-name project identifier this package takes, and the Python
package `platformdirs`.

Every case but one builds its own `DirsEnv` and reads no environment
at all, including the cases that check the macOS and Windows rules; the
exception calls `dirsenv.capture` and `dirsenv.capture_as` and asserts
only the platform each resolves by. The suites assert that a set
variable wins over a fallback and an empty one does not, that no home
directory gives `None` rather than a path beginning with a slash, that
a machine whose environment names nothing gets `None` rather than an
invented path, that the runtime directory is never invented, that the
user's directory comes first in a search path, and that each platform
spells a project folder its own way — asserted cell by cell against the
table rather than sampled.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
