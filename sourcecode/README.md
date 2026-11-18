# Pass-based pipeline (sample)

All pipeline code lives in `sourcecode/`. Paths are derived from
`commonFunctions.py`'s own location, so the folder can live anywhere.

**Start here:** `../README.md` (a five-minute test with the mock server)
and `../docs/SETUP.md` (a real run against Ollama).

This is a sanitized sample: the characters, the storyboard and the
setting (a coffee shop) are invented. The code, the pass structure and
the 90 unit tests are the real ones.

**No .docx output.** `pipeline.py` and `supportPipeline.py` write only
Markdown. Use `toDocx.py` on the corrected lessons when a .docx is
needed.

## Files

- `rule_utils.py` — pulls named `#BEGIN RULES:` / `#BEGIN DICTIONARY:`
  blocks (and named sub-sections of `SECTION_CONTENT`) out of
  `MainLessonTemplate.txt`, so each pass sees only its own rules.
- `storyboard_utils.py` — parses the raw storyboard file: weekly verb
  focus, each day's narrative bullets, each day's `=== VOCAB DAY ===`
  block, plus the optional `WEEK NOTES` block, optional per-day
  `STUDY NOTES` blocks, and each day's focus line (the "Introduce ...",
  "Light review of ..." lines above its Examples).
- `qa_utils.py` — `lint_hints()`: no-model check, run at the end of both
  pipelines. Logs `WARNING: HINT [...]` for any Warmup / Grammar / Student
  Questions hint longer than 3 words or made only of common English
  words. Warns only; never edits the lesson.
  Since Oct 20, 2026 also `lint_student_life()`, `lint_translation_direction()`
  and `lint_story_length()` .
- `state.py` — `WeekState`: holds each day's finished section text as
  passes fill it in, and assembles the final Markdown in canonical
  section order.
- `passes.py` — the MAIN passes (in dependency order below), plus
  `fiveminute_self_review_pass()` used by `supportPipeline.py`.
- `commonFunctions.py` — shared library: paths, `send_to_ollama()`
  (thin wrapper over `ollama_client.py`), Markdown validation, the
  self-review guard, corrections, file writers, `debug_path()`, and the
  `extract_day_vocab()` / `extract_day_story()` helpers used by
  `supportPipeline.py`.
- `pipeline.py` — MAIN orchestrator / CLI. Also writes `mainSupport.md`
  (fixed filename) alongside the timestamped output, so
  `supportPipeline.py` has a stable name to point at.
- `supportPipeline.py` — Five-Minute orchestrator, with a self-review
  pass before corrections/validation/write.
- `toDocx.py` — converts the corrected `NN_MAIN_corrected.md` /
  `NN_FIVEMIN_corrected.md` to .docx.
- Also in this folder:
  `config.py` (settings, Macbook/Studio detection), `ollama_client.py`
  (streaming calls, timeout, retries), `errors.py` (exit codes),
  `runs.py` (run folders, checkpoints, `--resume`), `runlog.py`
  (run.log, summary), `cli.py` (shared flags), `tools/mock_ollama.py`.
- `tests/` — unit tests (stdlib `unittest`, ~2 s, no Ollama needed):
  `python3 -m unittest discover -s tests -t .` from `sourcecode/`.

## Optional storyboard blocks

Both are optional. A storyboard without them runs exactly as before.
Write them as plain facts, not as instructions to the model: the model
treats everything in them as source material.

**WEEK NOTES** - one block, placed above `# MONDAY STORYBOARD`. Use it
for facts and cast limits that hold for the whole week (café
hours, who appears on which days, how Nora gets to work).
Passed to the Story, Warmup/Grammar, Examples and Student Questions
passes, which are told to obey it.

```
# === WEEK NOTES ===
Setting: ... Hours: ...
Cast for this week: Lila, Hana and Pia appear only on Monday. ...
# === END WEEK NOTES ===
```

**STUDY NOTES** - one block per day that needs exact source material
(for example practice-test questions with the correct
answers and explanations). Place it after that day's bullets and before
the next `# DAY STORYBOARD` heading.

```
# === STUDY NOTES SATURDAY ===
Source: ... (book, page)
Question 1 ...
Correct answer: ...
Why the others are wrong: ...
# === END STUDY NOTES SATURDAY ===
```

The Story and Vocabulary passes never see it (it is stripped from the
story requirements). Only the Examples and Student Questions passes
receive it, and only for the day it belongs to: Examples become one
numbered item per question (question, correct answer, why, why each
wrong choice is wrong), the dialogue requirement is waived for that day,
and Student Questions are written about the notes. The day's bullets
should still say what the character is doing (for example "studies
questions 1, 2 and 3"), because the Story pass cannot see the notes.

Check the notes against the source before running; the model is told to
use only what is in them.

## Run it

```bash
cd sourcecode
time python3 pipeline.py 1 Week01StoryBoard.md MainLessonTemplate.txt Debug ; \
time python3 supportPipeline.py 1 mainSupport.md Week01StoryBoard.md FiveMinuteTemplate.txt promptFive.md Debug
```

After the main lesson is corrected, point the
Five-Minute run at the corrected file instead of `mainSupport.md`, so the
five-minute lesson inherits the fixes:

```bash
time python3 supportPipeline.py 1 01_MAIN_corrected.md Week01StoryBoard.md FiveMinuteTemplate.txt promptFive.md Debug
```

(`mainSupport.md` is written automatically by `pipeline.py` and holds the
uncorrected main lesson.)

Storyboards and templates are read from `source_docs/WeeklyTemplates`,
lessons are written to `source_docs/WeeklyLessons`, and each run's
log, checkpoints and debug files go to `runs/<id>_<KIND>_<ts>/`.
Output uses `MAIN_PASSPIPELINE` / `FIVEMIN_PASSPIPELINE` suffixes.
Typical run time on the reference setup (a 27B model on an M4 Max):
about 30 minutes for MAIN and about 10 for Five-Minute.

## MAIN pass order (each depends only on layers already written)

1. **Vocabulary** — from storyboard `VOCAB` blocks only.
2. **Story** — from Story Requirements + Vocabulary. Sees the whole
   week at once so register can vary day to day.
3. **Warmup + Grammar** — from each day's Story Requirements (so the
   correct verb/tense is taught, not an invented one) + Vocabulary.
   Grammar is written in English here.
4. **Grammar → Spanish translation** — narrow, single-purpose: takes
   the English Grammar explanation and translates only the prose (not
   the Pattern Practice list, not quoted English target phrases) into
   natural, everyday Spanish. Split out from step 3 deliberately, so "write it" and
   "translate it, but leave X/Y untouched" aren't one overloaded call.
5. **Examples + Translation Practice** — from finished Story +
   Vocabulary + each day's focus line. Enforces language direction
   explicitly (Spanish→English items must be in Spanish, including the
   required professional-register item; English→Spanish items in
   English). Requires at least two Examples items that use the day's
   verb(s), forbids grammar commentary, and uses STUDY NOTES and WEEK
   NOTES when present.
6. **Student Questions** — whole week visible, so it can avoid
   repeating a question across days. Hint rule is repeated in the
   instructions (one Spanish word or a short Spanish phrase, never
   English, never the answer). Uses STUDY NOTES and WEEK NOTES when
   present.
7. **Self-review** — judgment-only pass over the fully assembled week:
   register variety across days, Examples dialogue naturalness, no B1
   vocabulary in Story prose. Explicitly told NOT to flag trailing time
   expressions (deliberately cut from the template - natural in everyday
   Spanish) and NOT to touch item counts/hints/vocabulary (checked by
   code in `common.validate_markdown()`, not by model judgment).

## Weekly workflow

1. Write `WeekNNStoryBoard.md` in `source_docs/WeeklyTemplates`
   (use WEEK NOTES / STUDY NOTES when the week needs them).
2. `pipeline.py` (MAIN, about 30 minutes on the reference setup).
3. Read the output, correct it by hand or with Claude, and save it as
   `NN_MAIN_corrected.md` in `source_docs/WeeklyLessons`. Watch the
   end-of-run summary for `HINT [...]` warnings.
4. `supportPipeline.py` pointed at the corrected file (see "Run it"),
   then correct that output as `NN_FIVEMIN_corrected.md`.
5. Keep the lessons you want in `source_docs/WeeklyLessons/`. `runs/`
   holds each run's log and checkpoints and can be deleted any time.

## Five-Minute flow

Extraction/generation carried over from the retired `generateSupport.py`, plus:
- **`fiveminute_self_review_pass`** — narrow, single check: every
  Grammar Focus must have a *character* as the grammatical subject
  ("Nora tells...", not "'have' shows..."). Deliberately does not
  touch Mini Story word counts, comma limits, or connector choice -
  see "Known gaps" below.

## Known gaps / next steps

- **No deterministic checker yet** for: Mini Story word-count-per-
  sentence (8-12), comma limit (<=1), the and/but/because connector
  restriction, or Translation Practice language direction. All of
  these are currently enforced only by prompt instruction, which is
  why they slipped at least once each during testing. Candidate: add
  checks to `commonFunctions.validate_markdown()` so both pipelines
  benefit, rather than duplicating logic here.
- **Quality still depends on the storyboard.** Errors that recur across
  runs are usually fixed by a storyboard fact, a WEEK NOTES / STUDY
  NOTES entry, or a `REGIONAL_REGISTER` entry in `MainLessonTemplate.txt`,
  not by editing the model output each time.
- **Model-written Spanish needs a read.** Typical misses: English titles
  ("Mr.") inside Spanish sentences, a wrong gender on acronyms (la
  ATM), and the answer repeated inside a hint.
- **Per-pass retry is coverage-only** (since Oct 20, 2026): a pass
  missing a day or section is run once more, then kept with a warning.
  Nothing repairs content quality automatically.
