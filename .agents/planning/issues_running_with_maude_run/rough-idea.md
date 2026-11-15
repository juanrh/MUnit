# Issues running Maude through `maude_run`

## Context

While adding `examples/reverse_list.maude` (reverse a list of integers) and
`examples/reverse_list_test.maude` (MUnit tests), I used the Zed Maude tools
(`maude_run`, `maude_program_diagnostics`) instead of the Maude CLI.

- The program itself validated cleanly with `maude_run` (all reductions and the
  sort matches used by the tests behave as expected).
- MUnit itself could **not** be executed through `maude_run`, and an attempt to
  reproduce its parsing non-interactively turned out to be unreliable.
- Environment: Maude 3.5.1 (built Jul 16 2025).

The probe files below are under `probes/`. They are standalone reproducers, not
part of the tool.

## Summary

| # | Issue | Impact |
|---|---|---|
| 1 | MUnit is an interactive `erewrite` session | `maude_run` cannot execute test files end to end |
| 2 | `io.maude` is not loadable as a library | can't reuse `MUNIT-IO`; probes must trim it |
| 3 | `tokenize` keeps `***` comments as tokens | commented MUnit files cannot be parsed by `open` |
| 4 | Reconstructed `metaParse` for `MUNIT-GRAMMAR` is unreliable | a failing probe is not proof that a test file is invalid |
| 5 | `*** (` starts a multiline block comment | line comments starting with `(` silently swallow text |
| 6 | `metaParse` arity / module location surprises | easy to write a terminally stuck probe |
| 7 | Reflection output and string-literal ergonomics | easily floods context or breaks probe files |

## Related knowledge-base entries

- `.agents/summary/index.md` — critical facts; independently confirms comment
  breakage, `tokenize` behavior, and the dead `loop` tail.
- `.agents/summary/review_notes.md` — G2 (comment parse failure), G3 (dead code
  after `eof`), G4 (`load file.maude` placeholder).
- `.agents/summary/dependencies.md` — `STD-STREAM` and `FILE` are expected prelude
  modules, which is why their absence in the tool environment blocks `MUNIT-IO`.

## 1. MUnit is an interactive `erewrite` session; `maude_run` cannot drive it

The session is started by `erew run .` in `io.maude`:

```maude
erew run .
eof
...
loop init-munit .
```

`run` is `<> < munit : MUnit | initAtS > write(stdout, munit, mu-banner)`, and
the `MUNIT-IO` rules then issue `getLine(stdin, munit, "MUnit> ")`, so the
`erewrite` suspends waiting for terminal input. As recorded in
`.agents/summary/index.md` and `review_notes.md` (G3), everything after `eof` is
dead code: `loop init-munit .` never runs and `init-munit` is undefined.

`maude_run` does support `erewrite` as a step, but reproducing the session would
require (a) loading `MUNIT-IO` — impossible here, see issue 2 — and (b) feeding
stdin (`open ...`, `q`), which the tool does not expose.

Consequence: the documented flow

```
$ maude -allow-files examples/reverse_list.maude io.maude
MUnit> open examples/reverse_list_test.maude
```

can only be exercised with the Maude CLI, not with `maude_run`.

## 2. `io.maude` cannot be loaded as a library

Even ignoring the interactive session, loading `io.maude` through the tool is
not possible as-is:

- Line 1 is `load file.maude`. There is no `file.maude` in the repo; per
  `review_notes.md` (G4) a missing file is silently ignored, so the line is an
  undocumented edit point for loading the module under test.
- The tail executes top-level commands on load: `set print ...`,
  `erew run .`, `eof`. `erew run .` starts the session and suspends on the
  first `getLine(stdin, ...)`; the lines after `eof` are dead code (G3).
- `mod MUNIT-IO` fails to compile in the tool's environment:

  ```
  Warning: ... module STD-STREAM does not exist.
  Warning: ... module FILE does not exist.
  Warning: ... this module contains one or more errors ... cannot be used or imported.
  ```

  So `MUNIT-IO` (and the file I/O path used by `open`) is unavailable even
  before the session starts. This may be a deliberately restricted prelude in
  the tool, or a real environment gap; either way, probes must avoid
  `MUNIT-IO`.

Workaround used in `probes/probe_munit_grammar.maude`: a copy of `io.maude`
with the head loads and the executable tail removed. The grammar definitions
(`META-OPS`, `MUNIT-SIGNATURE`, `META-MUNIT-SIGN`) and `procMUnit` from
`mUnit.maude` are enough to probe parsing and test execution.

## 3. `tokenize` does not strip comments

MUnit's `parse-module` rule does:

```maude
if {T, Ty} := metaParse(MUNIT-GRAMMAR, tokenize(D), '@MUnitModule@') .
```

With Maude 3.5.1, `tokenize` keeps comments as ordinary Qids:

```maude
reduce tokenize("*** a comment\nmunit X is assertTrue(true) endu") .
*** result: '*** 'a 'comment 'munit 'X 'is 'assertTrue '`( 'true '`) 'endu
```

So the token list starts with `'***`, which the MUnit grammar cannot parse. A
commented test file leaves the MUnit session stuck in `state: processing`
(an interactive attempt outside the tool with a commented test file did exactly
that). This is the same failure mode as `review_notes.md` G2; the probe here
adds the token-level cause. Consequence: files opened by MUnit must be
comment-free. Both
`examples/reverse_list_test.maude` and the existing `examples/add_even_test.maude`
follow that rule; the `geo2D` tests (comments plus `(munit ...)` parentheses)
look stale for the current MUnit/Maude combination.

Probe: `probes/probe_tokenize_comments.maude`.

## 4. Reconstructing `metaParse` for `MUNIT-GRAMMAR` is not a reliable oracle

`probes/probe_munit_grammar.maude` runs the same `metaParse` call MUnit uses and
then `procMUnit`. Observations:

- `wellFormed(MUNIT-GRAMMAR)` is `true`.
- The same grammar parses fine for `@Token@` and for a single
  `@MUnitPredicate@`:
  `{'assertEqual['bubble[...], 'bubble[...]], '@MUnitPredicate@}`.
- A single-predicate module block parsed in one session:
  `{'munit_is_endu['token[''X.Qid], 'assertTrue['bubble[''true.Qid]]], '@MUnitModule@}`.
- The identical call was left **unreduced** in a later session: the result was
  the input term `metaParse(fmod 'MUNIT-GRAMMAR ... , none, tokens, '@MUnitModule@')`.
- Multi-predicate blocks were left unreduced in every attempt, including a
  control built from the first six assertions of `examples/add_even_test.maude`.

Because the repo's own documented example fails the same reconstruction, the
harness cannot be used to decide whether a test file is valid. Either the
reconstruction differs from the real runtime, or `metaParse` on this grammar is
unstable in Maude 3.5.1. This is the main open issue.

Probe: `probes/probe_munit_grammar.maude` (`singleText`, `twoText`,
`addEvenText`, `testText`, `runTests`).

## 5. `*** (` starts a multiline block comment

Maude Appendix B.3: comments are `***` or `---` to end of line, and
`***( ... )` is a multiline comment with balanced parentheses. Whitespace after
`***` is accepted, so this is **not** a line comment:

```maude
*** (a) w1
```

It is read as the block comment `***(a)`, and then `w1` is tokenized
(`Warning: skipped unexpected token: w1`). A `(` later in the line
(`*** foo (bar) baz`) does not trigger this.

This bit the probe files: any documentation comment whose first non-space
character was `(` silently swallowed the following text and produced confusing
"no parse for statement" diagnostics at later lines. Fixed by rewriting those
comments.

Probe: `probes/probe_block_comments.maude` (intentionally warns about `w1`).

## 6. `metaParse` arity and helper-module surprises

- Documented form: `metaParse(Module, QidList, Type?)`, e.g.
  `metaParse(upModule('INT-LIST, true), tokenize("reverse(1 : 2 : 3)"), 'IntList)`
  works.
- In Maude 3.5.1 the 3-argument form lowers to a 4-argument special op with
  `none` inserted as the second argument
  (`getOps(upModule('META-LEVEL, false))` shows both declarations). Passing
  `nil` or `anyType` in that slot leaves the term stuck; only `none` worked.
- `tokenize` is in `LEXICAL`, not `META-LEVEL`; a probe importing only
  `META-LEVEL` fails with `bad token tokenize` and the module is not defined.
- `ResultPair?`-returning functions are partial: wrong argument sorts produce an
  unreduced kind-level term, not an error, so a "stuck" probe usually means a
  signature mistake.

Probe: `probes/probe_metaparse_basics.maude` (includes a tiny assoc-list
grammar that parses `a b` correctly, as a contrast).

## 7. Reflection output and string-literal ergonomics

- `reduce getOps(upModule('META-LEVEL, false))` produces a very large term and
  floods the context. Filter at the meta-level (e.g. keep only op decls whose
  name is `metaParse`) and reduce the filtered set.
- To match an op declaration in a term, use `(op Q : TyL -> Ty [AtS] .)` and
  declare all metavariables (`Q : Qid`, `TyL : TypeList`, `Ty : Type`,
  `AtS : AttrSet`); otherwise Maude reports `bad token Q`.
- Maude string literals cannot span lines. Embed test text with explicit `\n`
  escapes and concatenate with `+` (`pr STRING`); backslashes need escaping.
- A failed module load yields `Module "X" is not defined by "<file>" (modules
  defined by the load: ...)` from `maude_run`; run
  `maude_program_diagnostics` to see the actual warnings.

## What was validated with the tool

For `examples/reverse_list.maude`, module `INT-LIST`, `maude_run` confirms:

| term | result |
|---|---|
| `reverse(nil)` | `nil` |
| `reverse(7)` | `7` |
| `reverse(9 : nil)` | `9` |
| `reverse(1 : 2)` | `2 : 1` |
| `reverse(1 : 2 : 3)` | `3 : 2 : 1` |
| `reverse(0 : -3 : 8)` | `8 : -3 : 0` |
| `reverse(-1 : 0 : 42 : -7)` | `-7 : 42 : 0 : -1` |
| `reverse(5 : 5 : 5)` | `5 : 5 : 5` |
| `reverse(reverse(4 : 5 : 6))` | `4 : 5 : 6` |

`reverse(1 : 2 : 3)` does not equal `2 : 3 : 1` (for `assertDifferent`). For
the `assertSort` cases, the normal forms were checked directly: `nil` and
`3 : 2 : 1` match `X:IntList` but not `X:Int`.

## Probes

All under `probes/`; run them with `maude_run` using the module named in each
file's header.

| File | What it shows |
|---|---|
| `probe_tokenize_comments.maude` | `tokenize` keeps `***` comment tokens |
| `probe_metaparse_basics.maude` | 3-argument `metaParse` works; tiny assoc lists parse; 4-argument second slot must be `none` |
| `probe_munit_grammar.maude` | trimmed `io.maude` + `procMUnit`; `wellFormed` is true, but `@MUnitModule@` parsing is unreliable, including on `add_even_test.maude` content |
| `probe_block_comments.maude` | `*** (` opens a multiline block comment; `w1` is tokenized |

## Suggested next steps

1. Run MUnit manually outside the tool to settle whether the multi-predicate
   failure is real:
   `maude -allow-files examples/reverse_list.maude io.maude`, then
   `open examples/reverse_list_test.maude`.
2. If it is real, investigate `metaParse`/bubble interactions with the MUnit
   grammar in Maude 3.5.1 (e.g. add an identity to `@MUnitPredList@`, or parse
   with explicit expected sorts), and consider a non-interactive `procMUnit`
   entry point for tool-based testing.
3. Document or remove the `load file.maude` placeholder (`review_notes.md` G4);
   it is silently ignored when absent.
4. Preprocess or strip comments before `tokenize` in `parse-module` (and/or
   support comments in the MUnit grammar) so tests can be documented.
5. Keep MUnit-opened files comment-free until then; use line comments whose
   first non-space character is not `(` in ordinary Maude files.
