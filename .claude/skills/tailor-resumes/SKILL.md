---
name: tailor-resumes
description: Use when the user asks to find jobs on hiring.cafe and build tailored resumes for them, or says things like "tailor resumes", "find me jobs and make resumes", "run the resume sweep", or similar — for AI Engineer, Software Engineer, DevOps Engineer, and Fullstack Engineer roles at the 2-3 year experience level. Searches hiring.cafe, picks genuine YOE-appropriate matches, resolves each job's real employer application URL, builds a LaTeX resume per job tailored to that JD's keywords (zero fabrication), compiles and verifies each PDF, uploads all of them to the connected Google Drive folder, and writes everything to the connected Google Sheet.
user_invocable: true
license: MIT
---

# Tailor Resumes — hiring.cafe job sweep + per-job resume builder

Full pipeline: search → pick matches → resolve official URL → build a tailored
LaTeX resume per job → compile + verify → upload to Drive → log to Sheet.

**Scope is fit-driven, not a fixed count.** Search all four categories (AI
Engineer, Software Engineer, DevOps Engineer, Fullstack Engineer) and take
every job that clears the fit bar in Step 2 — not a target number per
category. Some categories may yield 1 good match, others 5, depending on
what's actually posted and actually fits; don't pad to hit a round number,
and don't cut a genuine match to stay under one. As a sanity rail rather
than a quota: if a single category is returning 8+ fits, tighten the bar
(stack overlap, YOE precision) before shipping all of them — and if the
whole sweep is heading toward 25+ resumes, check in with the user before
continuing rather than silently running a large batch (it's expensive, and
the user has previously asked for quality over volume). If the user gives
an explicit count ("find me 5 DevOps roles"), honor that instead.

## Fixed destinations — reuse these, never create new ones

- **Account:** `deepsantoshgfg@gmail.com` for both Drive and Sheets. The
  `mcp__claude_ai_Google_Sheets__*` / `mcp__claude_ai_Google_Drive__*` tools
  are already authenticated as this account (confirmed via the `owner`
  field on every file created so far) — no action needed there.
- **Google Sheet:** `1HO8CTLuT994fTQIPDU_btvjAFgFVA4meujXKFjym-sY` — tab
  `2-3yr Matches`. Columns: Category, Company, Role Title, Required YOE,
  Official Job URL, Resume PDF (Drive), Notes.
- **Drive folder:** `Career-Ops Resumes`
  (`1hJWEMjBVUJpbYMdxxHUcPkw707ng-755`), already populated — list its
  contents first via `mcp__claude_ai_Google_Drive__search_files` or
  `list_recent_files` before assuming a file doesn't already exist there.
- If either ID stops resolving (deleted/moved), tell the user and ask before
  creating replacements — don't silently spin up a parallel sheet/folder.
- **Permissions:** `career-ops/.claude/settings.json` allow-lists the
  `mcp__claude_ai_Google_Drive__*` / `mcp__claude_ai_Google_Sheets__*` tool
  names this skill uses (create_file, search_files, list_recent_files,
  append_values, update_values, get_values, get_spreadsheet) so they run
  without a per-call approval prompt. If a future call needs a tool not on
  that list, add it there rather than re-asking every session.
- **Use the Sheets connector for every Sheet write; use the browser for
  Drive PDF uploads.** `mcp__claude_ai_Google_Sheets__*` (append_values /
  update_values / get_values) is cheap — the payload is a few short text
  rows — and talks to the right account directly, so always use it over
  the browser for Step 7.
  `mcp__claude_ai_Google_Drive__create_file` was tried for PDF uploads
  and **measured, not assumed, to be impractical**: it has no
  file-path parameter, so the PDF must be base64-inlined as the literal
  `base64Content` argument — a ~30KB resume PDF encodes to ~40KB of
  base64 text, which costs **~39,000 tokens** to produce as a single tool
  call (verified by encoding one real resume and checking the token cost
  directly). Across a 9-resume batch that's 300K+ tokens for file content
  alone, against zero marginal cost for the browser path, which takes
  local file paths directly and uploaded all 9 resumes in one call this
  session. **Keep Drive PDF uploads on `claude-in-chrome` /
  `file_upload`** (Step 6) — don't switch this one to the connector even
  though Sheets did. `create_file` is still the right tool for creating a
  new Sheet/Doc/folder (no binary content) or small text-file uploads.

## Step 1 — Search hiring.cafe

Navigate directly to a `searchState` URL rather than typing into the search
box (the box has flaky focus/race behavior in this UI and silently drops
keystrokes or resubmits a stale query):

```
https://hiringcafe.com/classic?searchState={"searchQuery":"<role>","seniorityLevel":["Entry Level","Mid Level"]}
```

(URL-encode the JSON.) The Entry+Mid seniority filter is what makes results
genuinely 2-3-year-appropriate — without it, search results skew senior and
every pick needs individual YOE verification against the full JD.

Run one search per category: `AI Engineer`, `Software Engineer`,
`DevOps Engineer`, `Full Stack Developer`.

**Date window.** Add `"dateFetchedPastNDays":N` to the same JSON to bound
how recent the postings are (hiring.cafe's own UI presets map to specific
values — confirmed by setting a preset via the UI once and reading the
resulting URL): `2` = past 24 hours, `4` = past 3 days. When the user asks
to widen or narrow the window, change this value directly in the URL
rather than clicking through the UI each time. After navigating, the
result count/age tags can render a beat after the page title updates —
`wait` ~2s before reading results, and re-check if the count still looks
like the previous query's.

A date-bounded search can return **zero genuine matches** — that's a
legitimate outcome, not a failed search. Report it plainly (which postings
were found and why each one failed the fit bar) rather than stretching a
borderline fit to avoid an empty result.

## Step 2 — Pick matches (fit bar, not a quota)

For each category, take every posting that clears the bar — don't target a
specific count. A posting clears the bar when BOTH hold:

- **Genuine stack overlap** with the candidate's `resume.tex` — most of the
  JD's core requirements are things actually in the base resume, not just
  one or two keywords in a long unrelated list.
- **An explicit YOE stated in the posting's own text** lands at or below
  the candidate's actual experience (don't trust the search card's badge
  alone if it's ambiguous — open the posting and check).

All else equal, prefer roles in the candidate's home city (Pune) or
explicitly remote — no commute, no relocation question — but don't let
location override a clearly stronger match elsewhere.

**The search card is not enough — open the full JD before shortlisting.**
A card's keyword list is generated from the whole posting and routinely
surfaces generic overlap (Python, AWS, Docker) on a role whose *actual*
ask is something the candidate doesn't have. Read the full JD text for
every card that clears the first pass, specifically hunting for a single
hard requirement that keyword overlap can't paper over:

- A **named-framework specialization** that is the job, not a skill on
  top of it (seen: "2+ years hands-on **Odoo** development" — candidate's
  genuine Python/Postgres/REST overlap was irrelevant because Odoo itself
  was the ask; same pattern for a Palantir Foundry-specific role).
- A **stated "core of the role" technology** the candidate has zero
  experience with, even when phrased as achievable (seen: a DevOps
  listing that said "Kubernetes... this is the core of the role" — Docker
  experience does not substitute).
- An **explicit institute/degree list** ("IIT, NIT, BITS Pilani... or
  other premier engineering institutes") as a stated requirement, not a
  soft preference — this is a hard gate no resume rewrite fixes. Softer
  phrasing ("strongly preferred") is a flag for the Notes column, not a
  rejection.
- A **people-management or team-leadership requirement** stated as
  required, not optional, when the candidate's whole work history is
  individual-contributor (seen: "lead and manage... teams" as a Required
  Skill on an otherwise strong stack match) — can't be honestly claimed in
  interview-prep terms, so don't build the resume.

A card that fails one of these is not a lesser match to include with
caveats — it's a rejection. Only genuine stack overlap *and* no
disqualifying gate clears the bar.

**Search-result ordering is not stable across reloads.** hiring.cafe's
relevance sort can reorder between the page where a posting was first read
and a later page load of the "same" search (seen: two Software Engineer
postings present on one load, gone on the next). Once a card clears the
fit bar, extract and record its `/job/...` href immediately (e.g. via a
`javascript_exec` scan of `a[href^="/job/"]`) rather than planning to
re-find it by scrolling back to the listing later.

## Step 3 — Resolve the official employer URL

Never use the hiring.cafe link as the "official" URL in the sheet. On the
hiring.cafe job page, find and **click** the "Apply directly on employer's
site" control:

- Use a real mouse click (`computer` tool) at its coordinates — a
  ref-based or JS-dispatched click is frequently silently ignored by this
  button's handler.
- **The first click after navigating to the job page (or after a
  `get_page_text` call on it) is frequently swallowed** — no new tab, no
  visible error. This isn't a bad coordinate (a screenshot right after
  shows the button unchanged in place); it's a page/extension timing
  quirk. Click again — it typically opens on the 2nd or 3rd attempt. Check
  `tabs_context_mcp` after each click rather than assuming the first one
  worked.
- After a new tab opens, check `tabs_context_mcp` for it and confirm its
  domain/title actually matches the company before trusting it as the
  canonical URL.
- **Give each job its own browser tab** via `tabs_create_mcp` if resolving
  multiple jobs in parallel (e.g. via parallel subagents) — sharing one tab
  across parallel workers causes tab cross-talk, where one worker briefly
  sees another's destination page.
- If no outbound employer link exists (hiring.cafe-only apply flow), record
  that explicitly rather than guessing a URL.

## Step 4 — Build the tailored resume

This project tailors a hand-authored LaTeX file, not a generic text resume,
so the usual "paste resume + JD into ChatGPT" prompting technique is run as
a structured multi-pass edit instead of one free-form rewrite. Work through
these passes in order for every job:

**4a. Read the canonical base.** `resume.tex` at the repo root is the
single source of fact. Do not read a per-job `resume.md` scratch file as a
fact source if it predates the current base — those are JD-vocabulary
notes only and routinely go stale.

**4b. Identify top value props.** From the base resume alone (before
touching the JD), name the 3-5 things about this candidate's background
that are genuinely strongest and most verifiable — this anchors the
rewrite so it leads with real strengths rather than chasing every keyword
equally.

**4c. Extract the JD's top keywords.** Pull the 15-25 terms a recruiter/ATS
scan would key on from the job description: tool names, frameworks,
methodologies, domain terms, seniority signals. Keep this list explicit
(even just mentally) — it's what the next passes are optimizing against.

**4d. Rewrite — reorder and reframe, never invent.** Reorder the Skills
lines and the keywords within each line so JD terms land first. Reword
Experience/Project bullets to use the JD's vocabulary **only for things
the candidate genuinely did** (e.g. JD "LLM orchestration" ↔ base "model
routing" is a legitimate reframe of the same true fact — same underlying
work, different words for it).

**4e. The honesty boundary (hard rule, not a suggestion).**
   - A tool/technology may go into a **Skills line** if the candidate has
     confirmed he actually knows it (confirmed so far: AWS Bedrock,
     2026-10-11).
   - A tool may **never** be written into a **work-experience bullet** as
     something used at that employer unless `resume.tex` already says so
     there. The AIApply bullets use OpenRouter — never swap in Bedrock,
     LangChain, or whatever a given JD wants, even if asked to "just put it
     there." If a JD wants something not in the base, either surface it
     truthfully via Skills-line reframing, or omit it. Never fabricate a
     number, employer, or claim not already in `resume.tex`.
   - Every changed line must survive this question: could the candidate
     defend it in an interview, and could a former manager confirm it in a
     reference check? If a rewritten bullet reads like it's overclaiming,
     walk it back to something defensible even if that costs a keyword hit.

**4f. Keyword coverage check.** Against the list from 4c, mark each keyword
`In resume` (exact term or a true synonym already present) or `Missing`.
Aim for roughly 80%+ coverage, but **never close a gap by inventing** —
a keyword that's genuinely missing stays missing and gets written into the
Sheet's Notes column as an honest gap (Step 7), not papered over in the
resume.

**4g. Length-neutral editing** (this is what prevents overflow). Copy the
base's full preamble byte-for-byte — fonts, margins, spacing, macros, the
`\ifdefined\pdfgentounicode` guard. Same sections, same order, same bullet
counts per role. Every rewritten line should render to approximately the
same length as what it replaces.

This is stricter than it sounds, and it's the single most common way a
tailored resume overflows to 2 pages (seen: 6 of 9 resumes in one batch
needed a second pass for exactly this):

- **Never add a 6th Skills line.** The base has 5 (AI & LLM / Backend &
  Desktop / Cloud & DevOps / Data & Observability / Voice AI). Splitting
  one category into two for a JD's sake ("Data" + "Observability" as
  separate lines) adds a full line of vertical height — fold the split
  back into 5 lines, reordering *within* them instead.
- **A 2-3 word addition to a bullet can wrap it onto a new line**, which
  is enough to push the whole page over. "served by FastAPI on Docker"
  becoming "served by a FastAPI REST API on Docker" reads as a harmless
  rephrase but costs a line. If a bullet must gain a word to land a
  keyword, cut a word elsewhere in the same bullet to net zero.
- **Compile and check `pdfinfo` after every job, not at the end of the
  batch.** Fixing one overflowing resume by eye against the original is
  fast; discovering 6 overflowed after building all 9 means re-opening and
  re-diffing each one from scratch.

Write the result to `output/jobs/{company-slug}-{role-slug}/resume.tex`.

## Step 5 — Compile and verify (both checks are mandatory)

```bash
cd "output/jobs/{slug}"
tectonic resume.tex
pdfinfo resume.pdf | grep Pages     # must read "Pages:  1"
```

**Never use `mdls -name kMDItemNumberOfPages`** to check page count — macOS
Spotlight's metadata cache lags or returns garbage (seen returning `11` for
a file mid-rewrite) and will mask a real overflow. If `pdfinfo` isn't
installed: `brew install poppler`.

A page count of 1 is necessary but **not sufficient**. A 1-page PDF can
still have two bullets' text visibly colliding into the next job heading
(seen failure mode: `...production-ready Software Developer April 2025` —
two lines rendered on top of each other). After `pdfinfo` confirms 1 page,
**use the Read tool on the compiled `resume.pdf` and actually look at the
render.** If anything overlaps or looks cramped, shorten the offending line
and recompile — don't ship until the render is visibly clean.

## Step 6 — Upload to Drive (browser, not the connector — see note above)

1. Stage copies with human-readable names in a scratch folder:
   `Deep Santoshwar - {Company} - {Role}.pdf`.
2. In the browser, open the fixed Drive folder, click **New → File upload**
   (a direct click, then re-screenshot/re-find if the menu doesn't appear —
   it sometimes closes on the first click of a sequence), then use `find()`
   for `file input element type=file` to get its ref fresh each time
   (don't cache a ref across navigations).
3. `mcp__claude-in-chrome__file_upload` takes local absolute paths directly
   and accepts multiple files in one call — all 9 resumes in a batch
   uploaded in a single `file_upload` call with zero token cost for file
   content (confirmed this session). If it errors with a permission/
   session-access message, tell the user to run (no surrounding quotes —
   quoted paths break this CLI's absolute-path detection):
   ```
   /add-dir /Users/deepsantoshwar/Projects/Open Source/career-ops
   ```
4. After upload, pull each file's Drive ID via a short `javascript_exec`
   scan of `[data-id]` elements on the folder page, filtering by filename
   substring match (the Drive file list is virtualized and the DOM also
   contains every *other* file already in the folder, so match on the
   target filenames rather than assuming the first N results are the new
   ones). Scroll and re-scan, or use `find()` by exact filename, if a
   just-uploaded file isn't in the first DOM snapshot. Build view links as
   `https://drive.google.com/file/d/{id}/view`.

## Step 7 — Write the Sheet

One row per job in the `2-3yr Matches` tab: Category, Company, Role Title,
Required YOE (from the actual JD, not the search card), Official Job URL
(from Step 3), Resume PDF Drive link (from Step 6), Notes (fit caveats —
state gaps honestly, e.g. "JD wants Django, candidate has FastAPI/Flask
instead"). Write headers once; append rows after.

## Reporting back

Summarize per job: company, role, fit note (including any honest gaps),
official URL, Drive link. Flag anything that needed a judgment call
(ambiguous YOE, no outbound employer link, a JD wanting something outside
the honesty boundary) rather than silently resolving it one way.
