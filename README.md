# meme-tools

The unified Memelang source-pipeline CLI — self-hosting step 7.

`meme-tools` provides ONE binary for the entire source pipeline: `parse`, `emit`, `check`, and `hash` — written 100% in Memelang, built with `meme build`, with zero dependencies. It ships with Memelang v0.3.0-beta.2 as the language's own native source-processing toolchain.

## Commands

```bash
# Parse statement/expression tree into indented text:
./memetools_bin parse <file.meme>

# Parse statement/expression tree into JSON records:
./memetools_bin parse <file.meme> -o out.json

# Emit canonical Memelang text to stdout:
./memetools_bin emit <file.meme>

# Emit canonical Memelang text to a file:
./memetools_bin emit <file.meme> -o out.meme

# Check exact and lexical lockfile freshness for one or more files:
./memetools_bin check <file.meme> [file2.meme ...]

# Print the computed normalized source hash:
./memetools_bin hash <file.meme>
```

### Exit Codes

- `0`: Success (for `check`, exits 0 iff every specified file is `LOCKED-FRESH`).
- `1`: Any failure, unknown command, syntax error, or unparseable/drifted file in `check`.

Every error produces one clear line to stderr/stdout and exits with code `1` (no stack traces).

## Check Verdict Table

Output format per file: `<file>: <VERDICT> [detail]`

| Verdict | Detail | Meaning | Exit Code |
| :--- | :--- | :--- | :--- |
| `LOCKED-FRESH` | (empty or `[ambiguous key, matched longest: <key>]`) | Computed source hash matches `entry.source_sha256` exactly. File is byte/semantically identical to locked build. | `0` (if all fresh) |
| `LEXICAL-FRESH` | `[layout-only changes]` | Exact hash differs, but comment-filtered token stream matches `entry.canonical_meme`. File has layout, whitespace, or comment-only changes; safe to relock without logic change. | `1` |
| `DRIFTED` | (empty or ambiguous key note) | Comment-filtered token stream differs from `entry.canonical_meme`. Substantive code edits occurred; recompilation required. | `1` |
| `NO LOCK` | (empty) | Lockfile `<file>.lock` does not exist. | `1` |
| `NO ENTRY` | (empty) | Lockfile exists, but contains no matching entry for the file path or basename. | `1` |
| `ERROR` | `[<reason>]` | Source file cannot be read, or lockfile JSON is malformed/unparseable. | `1` |

## Vendored-Module Provenance

`meme-tools` is composed from components developed across the self-hosting ladder:

| Module | Source Rung | Provenance Repo | Description |
| :--- | :--- | :--- | :--- |
| `keywords.meme` | Step 1 | `projects/meme-tokenizer` | Core language keyword definitions and classifier. |
| `lexer.meme` | Step 1 | `projects/meme-tokenizer` | Full Memelang tokenizer (`Token`, `tokenize`). |
| `hasher.meme` | Step 4 | `projects/meme-lock-check` | Exact source normalization (test block sorting, CRLF/LF) and SHA-256 digest computation. |
| `ast.meme` | Step 5 | `projects/meme-parser` | AST node model (`Node`), tree equality, tree diff, and tree text printing. |
| `expr.meme` | Step 5 | `projects/meme-parser` | Clone-light precedence climbing expression parser. |
| `stmt.meme` | Step 5 | `projects/meme-parser` | Statement parser and block parser (`parse_source`). |
| `emit.meme` | Step 6 | `projects/meme-parser` | Canonical AST-to-text emitter (`emit_program`). |
| `checker.meme` | Steps 2 & 4 | `projects/meme-canonical-checker` & `projects/meme-lock-check` | Exact hash matching combined with lexical token comparison. |
| `main.meme` | Step 7 | `projects/meme-tools` | Unified CLI router composing parser, emitter, hasher, and checker into a single tool. |

## Build Instructions

Compile to a zero-dependency standalone native binary via the C backend:

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o memetools_bin
```

## Test Instructions

Unit and integration tests are located in `tests/meme/` and can be run against individual targets using `meme test`:

```bash
# Python target:
uv run --project /mnt/ssd/projects/meme meme test tests/meme/ --target python

# Rust target:
uv run --project /mnt/ssd/projects/meme meme test tests/meme/ --target rust

# C++ target:
uv run --project /mnt/ssd/projects/meme meme test tests/meme/ --target cpp
```

### Test Suite Structure

- `01_hash_oracle_test.meme`: NIST SHA-256 test vectors, newline normalization, test block sorting, and validation against real lockfile hashes.
- `02_check_locked_fresh_test.meme`: Real `.meme` and `.meme.lock` fixtures verifying `LOCKED-FRESH` verdicts and suffix key-matching rules.
- `03_check_lexical_fresh_test.meme`: Layout modifications (spaces vs tabs, blank lines) and comment edits verifying `LEXICAL-FRESH` and `[layout-only changes]`.
- `04_check_drifted_test.meme`: Substantive edits (identifiers, literals, added/deleted statements) verifying `DRIFTED`.
- `05_check_errors_test.meme`: Missing source files, missing lockfiles (`NO LOCK`), missing entries (`NO ENTRY`), and corrupt JSON (`ERROR`).
- `06_multi_file_check_test.meme`: Multi-file aggregate exit code semantics (exits 0 iff all files are fresh).
- `07_parse_emit_roundtrip_test.meme`: Self-parse and emitter invariant `parse(emit(parse(x))) == parse(x)` across all repo modules and real examples.
