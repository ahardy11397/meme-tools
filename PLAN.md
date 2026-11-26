# PLAN — meme-tools: the unified Memelang source-pipeline CLI (self-hosting step 7)

ONE binary for the whole source pipeline: `parse`, `emit`, `check` —
written in Memelang, built with `meme build`, zero dependencies. This is
the tool that ships WITH Memelang v0.3.0-beta.2: the language's own
source-processing toolchain, in the language.

Composed from the self-hosting rungs: vendored lexer/keywords (step 1),
parser+emitter modules (steps 5-6), exact hasher (step 4). Read
/mnt/ssd/projects/meme/docs/roadmap.md and the source module headers
first; the vendored modules are COPIED HERE ALREADY — do not re-copy,
do not refactor them, only fix imports if a header reference is stale.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md. Compiler commands: `uv run --project /mnt/ssd/projects/meme
meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic.
- Every construct must parse with `--provider heuristic`.
- FOREGROUND runs for individual targets; coordinator runs the full matrix.
- NO new language features. Perf constraint (step-5 lesson): clone-light —
  line-list text accumulation joined once; pass only per-line token
  groups to expression parsing.

## CLI (thin argv router in main.meme; unknown command -> usage + exit 1;
every error = one clear line + exit 1)

    parse <file.meme>                # statement/expression tree, indented text
    parse <file.meme> -o out.json    # tree as JSON records
    emit <file.meme> [-o out.meme]   # canonical text (stdout or file)
    check <file.meme> [...]          # freshness verdict per file (below)
    hash <file.meme>                 # print the computed normalized source hash

## check verdicts (one line per file: `<file>: <VERDICT> [detail]`)

Combines the step-2 lexical verdicts with the step-4 exact hash:
1. Read source (missing -> ERROR line + exit 1). Lock = `<file>.lock`.
2. NO LOCK: no lockfile. NO ENTRY: no matching entry (suffix key match,
   longest wins — the rule is documented in the lock-check repo; mirror
   it). Lock unparseable -> ERROR.
3. LOCKED-FRESH — computed hash (hasher.compute_source_hash, which mirrors
   the compiler exactly) == entry.source_sha256.
4. LEXICAL-FRESH — hash differs, but comment-filtered token streams of
   source and entry.canonical_meme are identical (layout/comment-only
   edits; safe to relock). Detail: `[layout-only changes]`.
5. DRIFTED — token streams differ (real edit; recompile required).
6. Exit 0 iff every file LOCKED-FRESH.

`hash` prints the normalized-source hash (useful for debugging verdicts).

## Tests — tests/meme/*.meme, `meme test`

- Reuse the proven fixtures: copy real .meme + .meme.lock pairs from
  /mnt/ssd/projects/meme/examples/advanced/ into tests/tmp/ and assert
  LOCKED-FRESH (the compiler computed those hashes).
- Layout-only edit -> LEXICAL-FRESH (assert detail).
- Substantive edit -> DRIFTED. Comments added -> DRIFTED (hash covers
  comments) but token stream still matches — assert BOTH facts, which
  pins the verdict semantics precisely.
- NO LOCK / NO ENTRY / truncated lock -> ERROR, exit 1, no traceback.
- parse/emit round-trip invariant: parse(emit(parse(x))) == parse(x) on
  this repo's own modules (self-parse) and the copied real examples.
- hash command matches compute_source_hash values from real locks.
- Multi-file aggregate exit codes. Fixtures in tests/tmp/, cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed; rust/cpp individually green; timing:
   self-operations < 5s (coordinator runs the full six-target matrix).
2. `meme build main.meme -o memetools_bin` — demo transcript: parse+emit
   self-source; check all real locks copied to a scratch dir (expect
   mostly LOCKED-FRESH); layout-edit one -> LEXICAL-FRESH; edit one
   substantively -> DRIFTED; aggregate exit codes.
3. README: commands, verdict table, vendored-module provenance (which
   repo/rung each came from), test instructions.
4. Final message: files, test counts, timings, transcript, deviations.
