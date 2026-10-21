# Canonical sources for the transcript formatter, tag generator and timestamp generator

Research for [#5](https://github.com/public-media-work/pbswi-core-skills/issues/5), part of map [#3](https://github.com/public-media-work/pbswi-core-skills/issues/3).
Read on 2026-10-21. All sources are private repos. This note describes and points to them; it does not quote them.

Sources read (refs as of 2026-10-21):

| Repo | Ref | Notes |
|---|---|---|
| `public-media-work/cardigan` | `main` | `prompts/` last touched 2026-10-16; `.claude/skills/tag-scout/` last touched 2026-10-21 |
| `public-media-work/pbswi-ops` | `origin/feat/youtube-chapters` (PR #56, open) | 13 commits, 2026-09-23 |
| `public-media-work/pbswi-ops` | `origin/feat/youtube-keywords-skill` (PR #76, open) | 1 commit, 2026-10-14 |
| `public-media-work/pbswi-ops` | `origin/feat/normalize-srt` (PR #48, open) | 2 commits, 2026-09-18 |
| `public-media-work/pbswi-ops` | `origin/main` | `skills/reference/seo-keywords-primer/`, merged in PR #60 (2026-10-09) |
| `skill-ops` | working tree, `triage/pbswi/.claude/skills/content/` | `youtube-chapters/` last committed 2026-09-11 |

## Summary

| Skill | Canonical starting point | Why |
|---|---|---|
| Transcript formatter | cardigan `prompts/formatter.md` (main), plus cardigan's `chunking`, `completeness` and `seam_coverage` modules as optional bundled scripts | It is the only formatter that is current, tested and used in production. The portable copy (`standalone-agents/transcript-formatter/`) is six months behind and tied to Gemini. |
| Tag / keyword generator | cardigan `.claude/skills/tag-scout/` | It is already a Claude skill that turns a caption file into tags. It never writes anywhere, ships a tested caption decoder, and draws its rules from the SEO prompt. Fold in pbswi-ops `youtube-keywords`' character-budget check and `seo-keywords-primer`'s rules for SEMrush data. |
| Timestamp / chapter generator | pbswi-ops `feat/youtube-chapters` (`skills/youtube-chapters/`), with its drafting and checking steps separated from the YouTube and mmingest plumbing. Add cardigan's `style_engine/timecodes.py` for the Media Manager table. | It is the most complete and best-tested chapter skill (179 tests, live runs, Claude drafts the chapters and scripts check them), and #3 already says to reconcile with it rather than duplicate it. Cardigan's prompt is short and depends on pipeline inputs. The skill-ops triage copy never built generation. |

## 1. Transcript formatter

### Candidates

**A. cardigan `prompts/formatter.md` (main).** This is the system prompt for the production formatter phase: about 400 lines covering speaker attribution, paragraphing, filler removal, review notes, handling of live captions vs plain text, and a quality checklist. Its rule is that every word is kept: it never paraphrases or summarizes. It contains one `{{style:formatter.house_rules}}` token that is filled from `config/house_style.yaml` at runtime. It also expects SST context, which the worker gets from Airtable.
- *Maturity:* production. Edited regularly (last on 2026-10-16), covered by prompt-contract tests (`tests/test_prompt_contracts.py`, `tests/test_validator_formatter_contract.py`).
- *Dependencies:* the house-style YAML (rendered by `scripts/render_prompts.py` or `style_engine.render_prompt_blocks`). SST context is optional. In production, long transcripts are split into chunks (past about 6.5k words) by the worker, not by the prompt.

**B. cardigan deterministic gates (main), formatter-related.** These are pure functions that need only the standard library and PyYAML. Each imports only from the `api.services` package and the `style_engine` subpackage, with no database, worker or HTTP imports:
- `api/services/chunking.py`: `split_transcript`, `merge_formatter_chunks` (removes turns duplicated at chunk seams).
- `api/services/completeness.py`: `check_completeness`, `count_source_words`, `count_content_words`.
- `api/services/seam_coverage.py`: `find_dropped_spans`, which uses trigram coverage to find source spans the output dropped.
- `api/services/speaker_segmentation.py`: `split_interior_speaker_changes`, which splits SRT cues where the speaker changes mid-cue.
- `api/services/utils.py`: `parse_srt`, `SRTCaption`, `clean_srt_captions`, `get_srt_duration`.
- `api/services/style_engine/` `pre_stage` / `post_stage` / `lint`: formatter pre-stage prompt blocks, post-stage substitutions and normalization guard, lint checks (speaker-label consistency, review-note placement, truncation, completeness and seam gates).
- *Maturity:* production, each with its own test file under `tests/`.
- *Dependencies:* bundling them requires **code execution** to be enabled on the Claude Team account (see #2). They import each other through the `api.services.*` package path, so porting them means vendoring that subtree or rewriting the imports.

**C. cardigan `standalone-agents/transcript-formatter/` (main).** A portable formatter made for Gemini Gems, GPTs and Claude Projects. It has `PROMPT.md` (about 230 lines, a simplified copy of the formatter prompt), a `reference/` folder (media ID prefixes, Wisconsin proper nouns, an example output), and `format-transcript.py`, a CLI that splits SRTs into chunks and calls the **Gemini API** in parallel.
- *Maturity:* stale. Last touched 2026-04-07, before the house-style engine and the completeness and seam gates. The prompt has no house-style token, so it has drifted from A.
- *Dependencies:* the CLI path needs `google-genai` and a Gemini API key. The prompt-only path has no dependencies but asks the user to type "continue" on long transcripts.

**D. pbswi-ops `feat/normalize-srt` (`scripts/normalize-srt.py`, `.claude/commands/normalize-srt.md`).** This is **not a formatter**. It repairs caption files at the byte level so YouTube will accept them: it strips the BOM, converts to LF line endings, trims trailing whitespace and adds the final blank line. It also gives advisory warnings on caption quality. It verifies the words are unchanged before writing in place.
- *Maturity:* working command, open PR #48, no tests in the diff.
- *Relevance:* out of scope for the formatter. It is a possible pre-processing step for intake (the open "how a colleague gets a transcript in" question on #3). Its BOM/CRLF handling is a useful input-hygiene step for any skill that parses SRTs.

### Recommendation

Start from **A**. Render its house-style token at build time (the same copy-in-with-checksum pattern #3 sets for pbswi-editor rules) rather than at runtime. Treat **B** as an optional `scripts/` layer: `split_transcript` / `merge_formatter_chunks` for long transcripts, and `check_completeness` / `find_dropped_spans` as a self-check after formatting. When code execution is unavailable, the skill must still work as prompt-only, with a manual completeness check. **C** is useful only as an example of how to fit the prompt into a single conversation (the "continue" pattern). Do not port it.

## 2. Tag / keyword generator

### Candidates

**A. cardigan `.claude/skills/tag-scout/` (main; merged via cardigan#390).** A Claude Code skill: caption file (SRT or SCC) → program resolved from the media ID prefix → tag-rules reference → paste-ready YouTube tags (15–20, or 5–10 for short-form) and a separate Media Manager tag list. Each tag carries an evidence table (slice, direct or implied, relevance, basis) plus a "needs verification" list for proper nouns from live captions. Its rules: never write anywhere, never invent search-volume numbers, and never put paste-ready lines in a code fence (a fence corrupts the Airtable rich-text field, cardigan#380).
- `scripts/captions_to_text.py` decodes SRT and SCC (EIA-608 parity and character table) without third-party dependencies and merges the repeated text in live-caption rolling windows. Tested in `tests/test_tag_scout_captions.py`.
- `references/tag-rules.md` holds the tag-relevant parts of `prompts/seo.md`, `prompts/analyst.md` and `config/house_style.yaml`.
- *Maturity:* spike (it calls itself a v5 per-field spike), but tested and merged. Open follow-ups: cardigan#393 (check the SCC decoder against real files), #395 (prefix-table show names do not match house-style program keys), #392 (mmingest indexes SCC as raw hex).
- *Dependencies:* **tied to the cardigan repo layout at runtime.** It reads `config/house_style.yaml` and `api/services/mmingest/media_id_prefixes.yaml` by relative path. It needs code execution for the decoder. No Airtable, no Cardigan API.

**B. cardigan `prompts/seo.md` (main).** The full SEO phase: title, short and long descriptions, keyword tiers, YouTube tags, hashtags, platform notes, and an optional SEMRush section. It has house-style tokens for copy rules and character budgets. Keywords are one part of a larger report, and the prompt expects analyst and formatter outputs as inputs.
- *Maturity:* production. *Relevance:* the upstream source of A's rules, not a skill to start from. Analyst/SEO phases are out of scope on #3.

**C. pbswi-ops `feat/youtube-keywords-skill` (`.claude/skills/youtube-keywords/`, PR #76 open).** Built for **PBS Wisconsin Education**: it takes a "Title and SEO Analysis Report" (pasted, a Google Doc link, or a media ID whose captions come from mmingest) and produces an ordered YouTube tag list that fits the 500-character field. Tags are ranked by SEMrush volume and keyword difficulty and checked against what the episode actually covers. Scripts: `parse_keywords.py` (the report's keyword table → TSV), `tag_budget.py` (the 500-character limit, including the quote marks YouTube adds to multi-word tags), `fetch_captions.py` (mmingest lookup that picks the newest REV cut).
- *Maturity:* one commit, open PR, no tests. It relies on the layout of the Education report.
- *Dependencies:* rclone with a Google Drive remote for the Doc-link path (the skill itself notes this fails on Cowork and falls back to pasting). mmingest network access (VPN) for the captions path. No Airtable.

**D. pbswi-ops `skills/reference/seo-keywords-primer/` (main, PR #60).** A reference primer, not a generator. It covers the required SEMrush export columns, a CSV loader that handles BOMs, the opportunity score (volume ÷ difficulty, marked **unverified**), and rules for choosing and ordering keywords ("only recommend what is in supplied data"). Its sibling `metadata-copy-primer` holds the station's field limits.
- *Maturity:* DRAFT by its own status table. *Dependencies:* none (the snippet needs code execution).

### Recommendation

Start from **A (tag-scout)**. It is the only candidate that goes from caption file to tags with no outside services, and its "never write, never invent numbers, no fences" rules match #3's read-only Airtable rule. The port needs to:
1. Replace the relative-path reads of `house_style.yaml` and the media ID prefix table with copies made at build time (staleness-checked, per #3).
2. Add C's `tag_budget.py` check, since tag-scout counts tags but not YouTube's 500-character limit.
3. Make D's "use supplied data only" rules an optional branch for when the colleague provides a SEMrush export or an Education SEO report. Keep A's "no invented volume" default otherwise.
4. Keep C's Education report parser out of the core skill. It is tied to one team's document layout.

## 3. Timestamp / chapter generator

### Candidates

**A. cardigan `prompts/timestamp.md` plus the timestamp parts of `style_engine` (main).** The prompt is short (about 50 lines): it identifies chapter breaks, aligns them to speaker transitions, and sets chapter-naming rules. Its output format is a `{{style:timestamp.output_contract}}` token. The deterministic half is the real asset:
- `style_engine/pre_stage.py` (`_run_timestamp_pre_stage`, `_extract_boundary_candidates`): computes candidate boundaries from the SRT for the model to choose from.
- `style_engine/post_stage.py` (`_run_timestamp_post_stage`): snaps the chosen boundaries and checks them.
- `style_engine/timecodes.py`: pure standard library. `snap_chapters`, `emit_youtube_list`, and **`emit_media_manager_table`** (the PBS Media Manager chapter table with exact end-time math).
- *Maturity:* production, tested (`tests/test_style_timecodes_io.py`, `tests/test_style_stages.py`).
- *Dependencies:* the prompt expects the formatter's and analyst's outputs as inputs, and production runs it only for content of 10 minutes or more. Bundling the boundary and emission logic needs code execution.

**B. pbswi-ops `feat/youtube-chapters` (`skills/youtube-chapters/`, PR #56 open).** A self-contained prototype skill. Steps: find a transcript (ingest link → media ID → Airtable lookup → local caption file → YouTube captions → Whisper), build a drafting pack, have the session model write `proposed.json`, then run `check_proposal.py`, which snaps boundaries, enforces rules and flags drift without trimming chapters. It then suggests mid-roll positions (`midrolls.py`) and builds an HTML review page (`build_review.py`) for the required human review. It also audits existing chapters and repairs them (`chapter_audit.py`, `fix_zero_marker.py`, `retitle.py`; repairs are staged for `/youtube-post`, never written directly). Style rules come from `reference/chapter-rules.yaml`, a snapshot standing in for pbswi-editor's rules. `check_rules_sync.py` compares it against `house_style.yaml` at dev time.
- *Maturity:* prototype with 179 tests, live end-to-end runs and an Opus whole-branch review (per the PR body). The pbswi-editor link is recorded at pbswi-editor#18.
- *Dependencies:* PEP 723 scripts run with `uv run` (needs code execution). A YouTube brand OAuth token and Data API key, resolved through `get-secret.sh`. VPN access to mmingest. An Airtable read to find the media ID (the SKILL.md contains raw base/table IDs; do not copy those into this public repo). Only Whisper for the last fallback. Produces YouTube output only: the Media Manager table is listed under "Not yet".

**C. skill-ops `triage/pbswi/.claude/skills/content/youtube-chapters/` (working tree).** An earlier version: detect, repair, review, and write back under guard. Its own build-status table marks **generation (Stage 2, captions → Cardigan → chapter block) as not built**. It planned to call the Cardigan API. It holds per-video run files and rollback scripts, and points to a plan file at an absolute path on one machine.
- *Maturity:* superseded. PR #56 lists "Retire the triage copy in skill-ops" as a follow-up.

### Recommendation

Start from **B**, as #3 asks: reconcile with `/youtube-chapters` rather than duplicate it. B should be split into two parts:
- **Core (goes into pbswi-core-skills):** the drafting pack contract, `check_proposal.py`, `midrolls.py`, `chapter-rules.yaml`, and the caption parsing in `captions.py`, starting from a caption file the colleague supplies. This needs no credentials.
- **Plumbing (stays in pbswi-ops):** the transcript source chain (mmingest, Airtable lookup, YouTube captions, Whisper), the channel audit, and the staged repairs for `/youtube-post`.

Add **A's `timecodes.emit_media_manager_table`** (and its boundary-candidate approach) to fill B's missing Media Manager output. Use A's prompt only as a cross-check for chapter-naming rules, which should come from pbswi-editor in the end. Do not use **C**.

## Dependency matrix

| Candidate | Cardigan API | Airtable | Code execution | Other |
|---|---|---|---|---|
| cardigan `prompts/formatter.md` | no | optional (SST context) | no (prompt) | house-style render |
| cardigan gates | no | no | **yes** | PyYAML, `api.services` import paths |
| cardigan standalone formatter | no | no | CLI only | Gemini API key |
| pbswi-ops normalize-srt | no | no | yes | none |
| cardigan tag-scout | no | no | **yes** (decoder) | cardigan repo layout (relative paths) |
| pbswi-ops youtube-keywords | no | no | yes | rclone/Drive, mmingest VPN |
| pbswi-ops seo-keywords-primer | no | no | for CSV loader | SEMrush export from a person |
| cardigan timestamp prompt + timecodes | no | optional | **yes** for timecodes | analyst/formatter outputs |
| pbswi-ops youtube-chapters | no | read (media ID lookup) | **yes** (`uv run`) | YouTube OAuth + API key, mmingest VPN, Whisper (fallback) |
| skill-ops triage youtube-chapters | planned (unbuilt) | — | yes | YouTube write path |

No recommended canonical source requires the Cardigan API.

## Gaps and open questions

1. **Code execution decides how much is enforced.** All three recommendations work best with bundled scripts (chunking and completeness, the SCC decoder, chapter snapping). If the Claude Team admin does not enable code execution, each skill needs a prompt-only fallback whose checks are left to the model's judgment (#2, "enforced gates").
2. **Three caption parsers.** cardigan `utils.parse_srt`, tag-scout's `captions_to_text.py` (SRT and SCC, live-caption de-duplication) and youtube-chapters' `captions.py` (SRT and VTT, broadcast-offset removal, refuses SCC) overlap. The bundle should share one parser. tag-scout's handles the most formats, and youtube-chapters' handles timing offsets.
3. **The house-style source is moving.** Cardigan's `config/house_style.yaml` and youtube-chapters' `chapter-rules.yaml` are both stand-ins for pbswi-editor's rules (pbswi-editor#1, #18, #22). The build-time copy and staleness check from #3 should target pbswi-editor, with cardigan's YAML as the interim source.
4. **Program-name mismatch** between the media ID prefix table and house-style program keys (cardigan#395) affects both the formatter (program rules) and tag-scout.
5. **Formatter has no skill wrapper yet.** Unlike tags and chapters, there is no current SKILL.md for formatting. The port is new work, not a move.
6. **Media Manager vs YouTube output.** Tags and chapters both have a YouTube form and a Media Manager form. Tag-scout proposes the split, and only cardigan emits the Media Manager chapter table. Decide whether both outputs ship in the first release.
7. **Secret hygiene when porting youtube-chapters.** Its SKILL.md embeds Airtable base/table IDs in a step. Strip them before anything moves into this public repo.

## Embedded directives

The source files are skills and prompts, so they are full of instructions written for the agent that runs them. None of them were addressed to this research task or tried to redirect it. Nothing to report.
