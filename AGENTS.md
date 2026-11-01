# AGENTS.md

Guidance for AI agents working in this repository. Facts here were verified by
building and running the test suite on macOS (arm64, Darwin 25, Apple clang);
Linux notes are marked as such.

## What this is

`femto` is a small ncurses text editor (public domain, Hugh Barney, derived from
Anthony Howe's "Anthony's Editor") with an embedded Lisp interpreter, **fLisp**.
fLisp also builds as a standalone binary. The editor's higher-level behaviour —
dired, bufmenu, grep, git integration, the string library — is written in Lisp
under `lisp/`, not in C.

Two binaries come out of one source tree:

| Binary  | Objects                                | Purpose                     |
|---------|----------------------------------------|-----------------------------|
| `femto` | `$(OBJ)` incl. `femto_lisp.o`, `main.o`| the editor                  |
| `flisp` | `flisp.o`, `lisp.o`, `double.o`        | standalone Lisp interpreter |

## Build and run

```sh
make                # builds femto (default target)
make flisp          # builds the standalone interpreter
make debug          # both, with -UNDEBUG
make run            # runs ./femto with the in-tree rc and lisp library
make fl             # runs ./flisp likewise
make clean
```

`make run` / `make fl` matter: both binaries need `FEMTORC`/`FLISPRC` and
`FEMTOLIB`/`FLISPLIB` pointing at the in-tree files, or they come up without
their Lisp library. Running `./flisp` bare, or with those variables blanked,
leaves even `+` undefined (it is defined in `lisp/core.lsp`, not in C) — that is
expected, not a bug.

```sh
# correct ad-hoc invocation
echo '(+ 1 2)' | FLISPRC=flisp.rc FLISPLIB=lisp ./flisp
```

### Generated files

`femto.rc` and `flisp.rc` are **generated** from `femto.sht` / `flisp.sht` by the
`sht` shell-template processor (in-tree, tracked), which inlines `lisp/core.lsp`.
They are untracked and `make clean` deletes them. The `.sht.rc` suffix rule
regenerates them, so a plain `make` after `clean` is fine — but if you
hand-delete a `.rc` and then run a binary directly, you get a crippled
interpreter rather than an error that points at the cause.

## Testing

```sh
make test           # builds both binaries, then runs the suite
cd test && ./run    # run all; ./run 6_string.test for one; ./run -? for help
cd test && VERBOSE=1 ./run 1_primitives.test
```

`test/run` is a small TAP harness (`test/tapview` renders it). `SUMMARY=1` prints
per-suite counts only. Many tests are marked `# TODO` and are expected to report
`not ok` — count those as skips, not regressions.

### The suite is broken on macOS for an environmental reason

`test/run` uses `echo -n` in four places. macOS `/bin/sh` is bash 3.2 built with
`xpg_echo` on, so `echo -n` prints a literal `-n`, which gets piped into fLisp as
input and every affected assertion fails. Symptom: ~320 failures, including
things like `error: '-n', has no value` in the output.

Confirm before you debug anything else:

```sh
/bin/sh -c 'echo -n hello; echo "|"'    # macOS: "-n hello|"   Linux: "hello|"
```

Substituting `printf '%s'` for `echo -n` takes the suite from **~320 failures to
11**. This is unfixed in the tree; see `HANDOFF.md`.

### Real baseline, once `echo -n` is accounted for

11 non-TODO failures remain, in three clusters:

- `1_primitives.test` **open-2 … open-6** — the tests `grep` for
  `#<Stream 0x............>` with exactly 12 dots. arm64 macOS prints 9–10 hex
  digits (`0x1f835ba58`), so these are a pointer-width assumption in the test,
  not an editor bug. Note `makefile`'s `test_core` target already normalises this
  with `sed 's/Stream 0x\(.\+\),/Stream /'`.
- `6_string.test` **string-drop_first-1/2, string-shrink-1/2** — adjacent to the
  recent "cleaned up string-shrink-left / right" commit; likely genuine, likely
  in-progress.
- `7_dired.test` **de-dir-up-1/2** — adjacent to the recent "fixed dired,
  de-up-dir was hanging somewhere" commit; likely genuine.

The last two clusters are unverified as to cause. Do not assume they are
platform artefacts.

## Platform gotchas

**This is the big one.** `header.h` opens with `#define _XOPEN_SOURCE 700`. On
Darwin that macro is not a floor but a **ceiling**: `sys/cdefs.h` maps it to a
`_POSIX_C_SOURCE` value which sets `__DARWIN_C_LEVEL`, and the SDK headers hide
everything above that level. Lowering it re-breaks the build in ways that look
nothing like a feature-macro problem:

- at `500` → `__DARWIN_C_LEVEL 199506L`, and `strdup` (gated `>= 200112L`)
  vanishes from `<string.h>`; `undo.c` fails with an implicit declaration, and
  the truncated `int` return would corrupt pointers on arm64 if it linked.
- below `700` → `fmemopen` and `open_memstream` (gated `>= 200809L`) vanish;
  `main.c` and `lisp.c` need both.

glibc is far laxer about these guards, so none of this shows up on Linux. If you
see a libc function reported undeclared on macOS, suspect `__DARWIN_C_LEVEL`
before you suspect a missing include.

**Include order is load-bearing.** Because the `#define` lives *inside*
`header.h`, it only takes effect in translation units where `header.h` is the
first include. `undo.c` is such a file; `main.c` includes `<stdio.h>` first and
so gets `__DARWIN_C_FULL` by accident. Adding or reordering includes can
therefore surface or hide errors in unrelated files. Prefer not to reorder.

**Other Darwin notes:**

- Link `-lncurses`, not `-lncursesw`. The macOS SDK ships no `libncursesw`; its
  curses is a single library with wide-char support built in.
- `errno` is a macro (`(*__error())`), so `extern int errno;` expands to a
  prototype-less declaration and is rejected under `-pedantic-errors`. Just
  include `<errno.h>`.
- `CFLAGS` carry `-pedantic-errors`, so clang warnings that are merely noisy on
  gcc become hard errors here.

## Architecture notes

- **Gap buffer** in `gap.c`; buffers in `buffer.c`, windows in `window.c`, undo
  in `undo.c`, key dispatch in `key.c`, syntax highlighting in `hilite.c`.
- **`lisp.c` is compiled twice**, which is the single most surprising thing in
  the build:
  - as `lisp.o` with `-D FLISP_FILE_EXTENSION -D FLISP_DOUBLE_EXTENSION` → `flisp`
  - as `femto_lisp.o` with `-D FLISP_FEMTO_EXTENSION -D FLISP_DOUBLE_EXTENSION` → `femto`

  `header.h` is included by `lisp.c` **only** under `FLISP_FEMTO_EXTENSION`.
  So a change to `header.h` cannot affect the `flisp` binary — useful when
  bisecting which binary a change could possibly have broken.
- **Some `.c` files are `#include`d, not compiled**: `file.c` (`lisp.c:2069`),
  `femto.primitives.c` (`lisp.c:2199`), `femto.register.c` (`lisp.c:2255`).
  They are deliberately absent from `$(OBJ)`. `double.c` is generated-ish —
  see `double.c.template`.
- **Primitive registration**: C provides only type-specific primitives (`i+`,
  `d+`, …) in `primitives[]` (`lisp.c:2202`) and `double.c`. Generic dispatch
  (`+`, `-`, …) is defined in `lisp/core.lsp`. `funcmap.c` maps Lisp names to
  editor C functions.
- `lisp/core.lsp` is special: it is baked into the `.rc` files at build time
  rather than loaded from `FLISPLIB` at runtime.

## Conventions

- C11, 4-space indent, K&R braces. Indentation is 4 spaces in almost all
  sources; `femto.primitives.c` and `hilite.c` still contain tabs. Match the
  file you are editing rather than reindenting it.
- Keep `-Wall -pedantic -pedantic-errors` clean. The tree is not currently
  warning-free (see `HANDOFF.md`), but do not add new warnings.
- `BUGS` holds known defects and valgrind notes. `CHANGE.LOG.md` is the changelog.
- Docs under `docs/` are **pandoc-generated** from `pdoc/*.html` via `make doc`.
  Do not hand-edit them.

## Git

- `origin` is the fork `github.com/karbak/femto.git`; `upstream` is
  `github.com/hughbarney/femto.git`. Work lands on `master`.
- **Do not commit or push without the user asking.** This is a standing rule from
  the user's global config, and it overrides any inclination to "tidy up" with a
  commit.
