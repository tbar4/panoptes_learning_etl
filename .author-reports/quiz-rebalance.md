# Quiz Rebalance — MultipleChoice Length-Tell Fix

## Problem
Across the six concept-check quizzes, the correct MultipleChoice answer was almost
always the longest, most-qualified option — a reader could score 100% by always
picking the longest choice without knowing any Rust.

## Fix
For every MultipleChoice question, only `answer.answer` and `prompt.distractors`
were rewritten so that:
- all four options sit within ~15% length of one another (correct answer is never
  the longest);
- each distractor is a confident, specific statement of a common misconception,
  matched in vocabulary and specificity to the correct answer;
- the correct idea is unchanged — only phrasing/length was rebalanced.
`prompt.prompt`, `context`, `id`, and all `ShortAnswer`/`Tracing` questions were
left untouched.

## Per-file results

| File | MC questions rewritten | Length-tells remaining |
|------|-----------------------|------------------------|
| 00-foundations.toml   | 3 | 0 |
| 01-extraction.toml    | 3 | 0 |
| 02-authenticated.toml | 2 | 0 |
| 03-paginated.toml     | 2 | 0 |
| 04-orchestration.toml | 3 | 0 |
| 05-retrieval.toml     | 2 | 0 |
| **Total**             | **15** | **0** |

A "length-tell" = the correct answer being the single longest option. All 15
questions now have at least one distractor as long as or longer than the correct
answer, and every option set is within 15% length parity.

## Verification
- Automated check (parse each TOML, compare `len(answer)` against `max(len(options))`
  per MC question): **15 MC questions, 0 length-tells**, all parity spreads within 15%.
- `mdbook build` → **exit 0** (validates TOML and that Tracing/ShortAnswer questions
  were not broken).

Note: the referenced `panoptes_etl/crates/*/src/` reference implementation does not
exist on disk at that path; correctness of each rewritten answer was preserved against
the shipped `context` prose in each quiz file (which describes the actual pipeline
behavior), and no correct-idea was changed — only length/phrasing was rebalanced.
