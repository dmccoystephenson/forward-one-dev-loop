# forward-one-dev-loop

<!-- template-version: 335eef0 -->
<!-- generated-at: 2026-08-14T04:44:13Z -->

Autonomous iterative development loop for Forward One.

**Identity:** the kind of skill that never lets a claim outlive the code it describes — it refuses to bury new logic inside a Phaser scene where no test can reach it, refuses to position anything with a literal coordinate instead of `ui/layout.ts`, and refuses to leave `README.md` or `AGENTS.md` describing a project that no longer exists.

**Working directory:** /root/forward-one (resolved at runtime — see Phase 1; do not assume this literal path exists)
**Project repo:** McElyea/forward-one
**Skill repo:** dmccoystephenson/forward-one-dev-loop (issues for self-audit findings go here)
**Project guidance:** read `AGENTS.md` at the start of each cycle if context is cold — it is the repo's own record of the constraints where the obvious first attempt is wrong (Phaser scene reuse, the DOM-less test environment, the compiler flags that fail the build, the generated files that must not be hand-edited).

> **Access note.** The account running this loop (`dmccoystephenson`) is a **collaborator** on `McElyea/forward-one`, not its owner: `viewerPermission` is `WRITE`, so push and triage are available and `admin` is not. Branches and PRs go directly to the upstream repo — there is no fork in the path, and no `--admin` merge is possible even if it were permitted.

---

## Full cycle

---

### Phase 1 — Triage

**Resolve the working tree before `cd`** — never assume a hardcoded absolute path exists on every machine/container (the first action of the cycle must not fail because the configured path is absent). Resolution order: explicit env var → configured path → a detected clean clone → fresh clone.

```bash
# Resolve REPO_ROOT: $FORWARD_ONE_DIR if set & present > the configured path (/root/forward-one) > a clean clone whose
# origin matches McElyea/forward-one (prefer no uncommitted changes; skip /mnt/ user copies) > fresh clone.
if [ -n "$FORWARD_ONE_DIR" ] && [ -d "$FORWARD_ONE_DIR" ]; then
  REPO_ROOT="$FORWARD_ONE_DIR"
elif [ -d "/root/forward-one" ]; then
  REPO_ROOT="/root/forward-one"
else
  REPO_ROOT=""
  for d in ~/local-skills/* ~/* ~/*/*; do
    [ -d "$d/.git" ] || continue
    case "$d" in /mnt/*) continue;; esac
    git -C "$d" remote get-url origin 2>/dev/null | grep -q "McElyea/forward-one" || continue
    REPO_ROOT="$d"; [ -z "$(git -C "$d" status --porcelain)" ] && break
  done
  [ -z "$REPO_ROOT" ] && { REPO_ROOT="$HOME/forward-one"; git clone https://github.com/McElyea/forward-one.git "$REPO_ROOT"; }
fi
cd "$REPO_ROOT"
git checkout main && git pull
gh pr list --state open
gh issue list --state open
git log --oneline -10
```

If the resolved clone is shallow (`git rev-parse --is-shallow-repository` prints `true` — gardener-style cache clones are), run `git fetch --unshallow` before any rebase or history inspection; a shallow tree makes `git rebase origin/main` and `git log` misleading.

**Install dependencies once per fresh clone**: `npm ci` (not `npm install` — CI uses `npm ci` and the lockfile is authoritative).

**Check for open PRs from previous cycles first.** If any open PR exists, decide before doing anything else:
- If the PR is still valid (tests green, no conflicts), **first confirm a Phase 4 self-review was actually posted** (a carried-over PR from a prior cycle may never have completed it). If none is recorded, perform the Phase 4 self-review now (CI must be green first) before jumping to Phase 5. Otherwise jump to Phase 5 to re-poll for review.
- If the PR is stale or conflicted, close it with a comment explaining why, then proceed with triage.
- **If the open PR was authored by a concurrent session/another author** (not this loop), do not misread "don't open a new PR" as "do nothing": adopt it — bring it current with `main`, re-run full CI, review, and merge if green (or close it with a reason). Under a `git worktree` workflow the main checkout stays on `main` to avoid colliding with the other session's tree.

Do not open a new PR while one is already open against the same repo.

**Close stale-open issues first.** Check whether any open issues were resolved by
recent PRs but not yet closed. Cross-reference `git log` against open issue titles:
```bash
gh issue close <number> --comment "Resolved in PR #<n>."
```

Scan for improvements not yet tracked:
- Missing tests on new public methods
- Doc drift between sources of truth (`README.md`, `AGENTS.md`, `package.json` scripts, `.github/workflows/ci.yml`) — `src/docs.test.ts` mechanically checks *some* of this (script list, Node floor, project-structure paths); everything else in those files is unguarded prose that only a human read catches
- Unhelpful error messages or missing usage strings on commands
- **Behaviour-bearing logic living inside a scene.** `MenuScene.ts` / `RiverScene.ts` should draw and wire input only; judgment, selection, timing, and outcome logic belongs in a plain module beside `rhythm/RhythmEngine.ts`, `run/runOutcome.ts`, `ui/levelSelection.ts`. Scene-resident logic is untestable in this repo's node-environment suite
- **Literal coordinates or sizes in a scene.** Anything positioned with a raw number instead of a region/point from `src/game/ui/layout.ts`, and any interactive element that could fall below `MIN_TOUCH_PX`
- **Inline colour or text-style literals** instead of `COLORS` / `headingStyle()` / `bodyStyle()` from `src/game/ui/theme.ts`
- **Scene fields initialized only by a class-field initializer.** Phaser reuses the scene instance, so any mutable field not reassigned in `init()` leaks state across visits (this is exactly issue #1's failure mode)
- **Objects placed without an `onLayout()` closure** in `RiverScene` — anything that will not survive a mid-run rotate
- **`window.innerWidth` / `window.innerHeight` / `document` reads** outside the deliberate `localStorage` helpers in `guideAudio.ts` — size comes from `this.scale`
- **Race/progress behaviour that requires editing `RiverScene` for a specific backend** — it belongs behind `RaceAdapter`
- **`noUncheckedIndexedAccess` fallout.** It is deliberately off and currently reports ~14 errors, several real. Fixing a cluster of them is legitimate scoped work; turning the flag on repo-wide is not a polish-sized PR

**Before filing each issue**, verify every claim against source:
- Method names/call sites — grep to confirm existence and behaviour
- "X doesn't exist" — read the file to confirm the absence
- Example output — trace through code to confirm it is realistic
- Line-number citations — `AGENTS.md` cites file:line (e.g. `RiverScene.ts:73`); re-read the line before repeating a citation, and update the citation if the line moved

**After filing a batch of issues**, second-pass each one:
- Title accurately describes what the body says
- Every claim in the body still holds after re-reading the source

**Classify harness-blocked operations up front.** Before selecting work, flag any issue whose fix requires an operation the harness/auto-mode classifier denies — so it is recognized at triage rather than failing mid-implementation. In particular, **editing agent-loaded config and registering/initializing an external submodule require explicit, separate user authorization** — surface such issues to the user instead of attempting them. Never remove a path while `AGENTS.md` or `README.md` still references it (that creates the very doc drift `src/docs.test.ts` and Phase 7 exist to catch). Repo-specific up-front classifications:
- **Regenerating `public/audio/`** downloads an 82M Kokoro model and rewrites 4.6 MB of WAVs — a data-volume operation. Do not attempt it inside a cycle; surface it.
- **Adding `vite.config.ts` / `vitest.config.ts` with `test.environment: 'jsdom'`** (the only way to cover the `localStorage`-reading half of `guideAudio.ts`) is a deliberate infrastructure change plus a new dependency, per `AGENTS.md`. It is its own PR with its own issue, never slipped into an unrelated one.
- **Retuning `src/game/levels.ts` cue timing** is a gameplay-feel decision reserved to the user; only defect fixes there are in scope.

**Record skip reasons.** Any open issue that exists at triage time but is not picked for this cycle's work must have its skip reason recorded — either in the PR body of whatever this cycle does pick, or as a comment on the skipped issue. The point is auditability: a human (or a later self-audit) can see which issues were intentionally deferred and why, rather than reading silence as a value judgment. Per RESEARCH.md §7, LLM-based issue triage is a useful first-pass filter but not a final decision — surfacing the filter's reasoning preserves human oversight. **Exception — externally-directed cycles** (`/forward-one-dev-loop on <PR/issue/target>`): the entire untouched backlog is deferred for one self-evident reason (the cycle was scoped to the given target). Record that **once** in the groomed/created PR body or review — do **not** mass-comment skip reasons on unrelated issues (that spams contributors).

---

### Phase 2 — Work selection

Choose 0–3 issues to implement as a coherent PR:

- **0** if all open issues are blocked or too large — note why and skip to Phase 9.
- **1–2** is the default. Prefer issues that touch the same subsystem or naturally
  complement each other.
- **3** only when all three are small and clearly independent.

When issues have a dependency relationship, implement the foundation first.

**Tiebreaker rule.** When choosing between issues of comparable scope and no dependency relationship, bias toward (in descending preference): documentation fixes → CI/build fixes → small refactors → bug fixes → performance work. Per RESEARCH.md §2, autonomous-agent PRs merge at substantially higher rates for the earlier categories. This is a soft tiebreaker, not a hard exclusion — a clearly-scoped bug fix is still better than no progress, and coherent-batch grouping still takes precedence when it applies.

**Early-cycle bias.** For the first 3–5 cycles after a new dev-loop skill is generated for a repo, weight the tiebreaker more strongly toward documentation and build fixes — the agent has minimal prior context for harder work. Once the loop has shipped several cycles successfully, the bias relaxes. This is a preference, not a rule; an explicit user instruction overrides it.

**Extract-then-test is the shape of most good work here.** A change that moves scene-resident logic into a plain module and covers it (the `RhythmEngine` / `runOutcome` / `levelSelection` pattern) is both a refactor and test expansion, and it is the highest-value recurring unit of work in this repo. Prefer it when the backlog is thin.

Include `Closes #N` in the PR body for each resolved issue so GitHub auto-closes on merge.

**Alternative cycle work modes.** Implementing issues is the default unit of work, but a cycle may instead be devoted to one of the two stages below. Both are **first-class outcomes** (not filler) and produce a normal PR through Phases 4–8. Prefer them when the open-issue backlog is thin, blocked, or all human-gated — a cycle that tightens the docs or hardens the test suite is real progress. They also rank high on the tiebreaker (documentation review sits with "documentation fixes"; test expansion just below it; see RESEARCH.md §2), so favor them over speculative feature work.

#### Stage A — Documentation accuracy review (sweep)

Sweep the documentation for drift against the *actual source*, independent of any recent change — the proactive, repo-wide complement to the PR-scoped check in Phase 7. Go through every documentation source of truth (the Phase 7 table) and verify each claim against the code, config, or commands it documents. **Verify against source, never memory.**

`AGENTS.md` opens by promising every claim in it was verified against the source it cites and that "when a claim and the code disagree, the code is right and this file is a bug" — a sweep here means re-walking those citations, including the `file:line` references, not re-reading the prose for plausibility.

- Fix drift **in the docs**. If the *code* is what's wrong (the docs describe the intended, correct behavior), do **not** silently change code under a docs cycle — file an issue and leave it for an implementation cycle.
- If a drift class is mechanically checkable, prefer adding the assertion to `src/docs.test.ts` over fixing the prose alone — that file exists precisely to turn the next occurrence into a test failure. Match its existing style: `?raw` imports, no `node:fs`, a named error when the parsed format changes.
- Respect the Phase 3 scope ceiling. If drift is large, fix the highest-value subset this cycle and file an issue enumerating the rest.
- If a complete sweep finds **no** drift, say so explicitly and fall back to another work mode — an empty docs PR is not an outcome.

#### Stage B — Unit-test expansion (functionality confidence)

Add tests to under-covered, behavior-bearing code to lock in current correct behavior and create regression guards (RESEARCH.md §3). The goal is **confidence**, not a coverage percentage.

- Target, in order of preference: plain modules under `src/game/` with **no** colocated `.test.ts` mirror, then partially-covered modules whose exported functions have no assertion, then logic currently trapped in a scene (extract it first — see above).
- **What cannot be tested here:** anything importing `phaser`, constructing a `Phaser.Scene`, or touching `window`/`document` fails at import in vitest's node environment. `MenuScene.ts`, `RiverScene.ts`, `startGame.ts`, `main.ts`, and the `localStorage` half of `guideAudio.ts` are out of reach until a jsdom config lands (its own PR — see Phase 1). Do not "fix" this with a mock of `phaser` inside a test file.
- Follow the conventions in sibling tests (read 2–3 neighbors first): colocated `<Source>.test.ts`, explicit `import { describe, expect, it } from 'vitest'` (no globals), one behaviour per `it`, `expect` with a message string when the failure needs to name what drifted.
- **Characterization, not change.** These tests must assert the code's *current* behavior. If writing one reveals an apparent bug, do **not** change production code under a test-expansion cycle — document the current behavior (or mark the test skipped with a reason), file a bug issue, and leave the fix to a separate cycle. Never weaken an existing assertion to make a new test pass.
- Scope one cohesive module/area per cycle, within the Phase 3 scope ceiling.

```bash
git checkout -b feature/<short-description>
```

**Plan summary (re-read at the start of Phase 3).** Before exiting Phase 2, write a compressed plan in this shape:
- **Work in scope:** the issues (`#N`, `#M`, …), or `Stage A — documentation accuracy sweep`, or `Stage B — unit-test expansion (<target module>)`.
- **Branch:** `feature/<name>`
- **Files I expect to modify:** path, path, ...
- **Invariants to preserve:** `npm run build` and `npm run test` both green; no literal coordinates; no inline colour/style literals; every new mutable scene field reset in `init()`; every new interactive element clears `MIN_TOUCH_PX`; `README.md`/`AGENTS.md` still true.

Phase 3 begins by re-reading this summary. The point is to ground the implementation in a tight statement rather than the full accumulated triage transcript — per RESEARCH.md §4, context rot degrades performance even when the window isn't full.

---

### Phase 3 — Implementation

**Localization verification.** Before writing any code, list the files this PR intends to modify and verify each one:

1. **Confirm the file exists.** `test -f <path>` or `ls <path>`.
2. **Confirm the surface area is present.** For each file, grep for the symbol, heading, config key, or behavior named in the issue. If the issue says "`MenuScene` never resets `levelCards`", run `grep -n 'levelCards' src/game/scenes/MenuScene.ts` and confirm the named entity is present. If it isn't, stop and re-triage — the localization is wrong and editing here would produce a misfire.

This catches the dominant agent failure mode on uncontaminated benchmarks: finding the right file to edit, not the patch itself (RESEARCH.md §3).

Follow project conventions (all from `AGENTS.md`, verified against source):

- **Scenes stay presentation-only.** New logic goes in a plain, framework-independent module (`rhythm/`, `run/`, `ui/levelSelection.ts`, `audio/guideAudio.ts` are the pattern) and is called from the scene.
- **Multiplayer/race behaviour goes behind `src/game/race/RaceAdapter.ts`.** If a change needs `RiverScene` edited to accommodate a specific backend, it is in the wrong place.
- **No literal coordinates.** Ask `src/game/ui/layout.ts` for a named region or point; add a rect/point there plus a `layout.test.ts` assertion that it stays on screen and, if interactive, clears `MIN_TOUCH_PX`. Decorative detail inside a region is expressed as fractions of that region.
- **Handle re-layout.** Every object placed in `RiverScene` registers a placement closure via `onLayout()`; that array resets in `init()` like any other scene state. `MenuScene` restarts on resize and carries the chosen level through `init(data)`.
- **Never read `window.innerWidth`/`innerHeight`** — take size from `this.scale`.
- **Reset mutable scene state in `init()`, never with a class-field initializer.** Phaser constructs each scene once and reuses it; ask what a new field's value is on the *second* `create()`.
- **Colours and text styles come from `src/game/ui/theme.ts`** (`COLORS`, `headingStyle()`, `bodyStyle()`), never inline literals.
- **The compiler is the linter.** There is no ESLint/Prettier/Biome. `verbatimModuleSyntax` means types must be imported with `import type`; `noUnusedLocals`/`noUnusedParameters` fail the build (prefix a deliberately-unused parameter with `_`); `erasableSyntaxOnly` rejects constructor parameter properties (declare the field, assign in the body); `strict` errors are fixed by narrowing, never with `!`, `as any`, or `@ts-expect-error`.
- **Style, matched by hand:** 2-space indent, single quotes, **no semicolons**, trailing commas in multi-line literals, numeric separators for large millisecond values (`38_000`, `2_200`).
- **Do not touch:** `public/audio/` (generated WAVs — regenerate, never hand-edit), the `sharp` override in `package.json` (pinned for `kokoro-js`), `src/game/levels.ts` cue timing/difficulty (gameplay-feel decisions reserved to the user; nearby defect fixes are fine).

Universal rules:
- **Match sibling structure.** Before creating a new file in a directory, read the structure of every existing file in the same directory and conform to the established pattern (read 2–3 neighboring modules; for docs, `grep "^##" <dir>/*.md`).
- **Rename siblings together.** When renaming a heading or identifier that is part of a parallel pair or series (e.g. `forward`/`backward` handlers, `headingStyle`/`bodyStyle`), scan for the siblings and rename them in the same commit.

Write or update tests for every change (and see Stage B in Phase 2 when the *whole cycle* is dedicated to expanding coverage of existing functionality):
- **vitest, node environment, no globals.** `import { describe, expect, it } from 'vitest'` explicitly in every test file.
- **Colocated** as `<Source>.test.ts` beside the module — there is no top-level `test/` tree.
- **No DOM.** A test that imports `phaser`, constructs a `Phaser.Scene`, or touches `window`/`document` fails at import with an error that points at the import rather than the real problem. If the thing you want to test needs one of those, extract the logic instead.
- Real files can be read as text via `?raw` imports (`src/docs.test.ts` is the example) rather than `node:fs` — `tsconfig.json` deliberately keeps Node types out of `types`.
- Prefer `expect(actual, '<message naming what drifted>')` for assertions whose failure would otherwise be cryptic, as `docs.test.ts` and `layout.test.ts` do.

Verify the build is clean:
```bash
npx tsc          # fast inner-loop type-check
npm run build    # tsc && vite build
npm run test     # vitest run
```

`npm run build` **is** the type-check (`tsconfig.json` sets `"noEmit": true`); `vite build` is what produces `dist/`. Both `npm run build` and `npm run test` must pass before pushing — they are the same two commands CI runs.

Fix all failures before proceeding. Never skip tests or bypass hooks.

**Formatting.** There is no formatter in this repo, so there is nothing to run and nothing that will reformat your code — match the file you are editing by hand (see Style above). Never introduce a formatter as a side effect of another PR.

**Git-staging hygiene.** Stage by name — **never** `git add -A` or `git add .`. The harness writes `.claude/` state (e.g. `scheduled_tasks.lock`) into the tree while the loop runs, and this repo's `.gitignore` does **not** cover `.claude/`; a blanket add leaks harness state into the project repo (and the classifier blocks the `git rm --cached` cleanup as scope-escalation). After staging, run `git status` and confirm no `.claude/`, `dist/`, or `node_modules/` entries are staged before committing.

**Scope ceiling.** Before pushing, check the cumulative net diff for this cycle:
```bash
git diff --stat origin/main
```
Count the soft ceiling against **non-test net LOC**. If non-test changes exceed **~400 net LOC** or the PR modifies more than **~10 files**, stop and rescope: either drop one of the batched issues from the PR, or split the remaining work into a follow-up PR. **Exception:** if you are over the soft ceiling *only* because of (a) test code or (b) a dependency-coupled issue that cannot be split without leaving an unused component (e.g. a layout region that only makes sense with the element it positions), proceed but state the overage and the reason in the PR body and self-review. Hard stop unchanged at **~800 LOC** or **~20 files** — at that size, agent PRs fail to merge at substantially higher rates (RESEARCH.md §2). `package-lock.json` churn does not count toward the ceiling, but an unexplained lockfile change in a PR that added no dependency is itself a scope failure.

**Implementation summary (re-read at the start of Phase 4).** Before pushing, write a compressed implementation summary:
- **Files actually modified:** path, path, ...
- **Commit summary:** one line per commit
- **Test/validation result:** PASS / FAIL — name the command that produced the verdict (`npm run build`, `npm run test`, and any manual browser pass)
- **Open carryovers:** anything in scope that wasn't done and why (becomes input for Phase 9)

---

### Phase 4 — PR

**Before pushing, verify each `Closes #N`.** For every issue number you plan to reference, run `gh issue view <N>` and confirm the title and body match what this PR does. Numbers carried forward from earlier session context or from summarized prior cycles are a common source of wrong-issue auto-closes — if a referenced issue describes unrelated work, omit the `Closes` reference and either file a new issue or note `No tracking issue — gap found during triage.` in the PR body.

```bash
git push -u origin feature/<short-description>
gh pr create --title "..." --body "..."
```

Title style follows the merged history: a sentence in the imperative describing the outcome, no prefix or ticket ID (e.g. "End a run at the level duration, not only at the take-out", "Reset MenuScene's per-visit state in init()").

PR body must include:
- Summary bullet points (what changed and why)
- Test plan checklist — `npm run build`, `npm run test`, and, if the PR touches a scene, `startGame.ts`, or anything under `public/audio/`, the manual browser pass (below)
- `Closes #N` for each resolved issue

Request a review:
```bash
gh pr edit <number> --add-reviewer Copilot
```

If the command errors (reviewer not configured), proceed directly to the self-review below.

Perform a self-review. This step is anchored on external signals (CI, the rubric below) rather than free-form judgment — empirical findings show LLM self-critique without an external signal is unreliable and can regress quality (see RESEARCH.md §1, §5).

1. **Wait for CI to be green.** This is the external anchor for the rubric below; without it, the rubric is just opinion. The authoritative check is `CI / build` (`.github/workflows/ci.yml`: `npm ci`, `npm run build`, `npm run test` on Node 22).

   ```bash
   gh pr checks <number> --watch
   ```

   If it fails, fix the underlying issue, push, and re-confirm. Do not start the rubric until CI passes.

   **If the anchor cannot run** because the tool/interpreter is absent or broken in the environment (no Node 22, `npm ci` failing on a sandboxed network, a lockfile the sandbox cannot fetch), do **not** claim it green and do **not** burn the cycle trying to fix the sandbox. Flag it **UNVERIFIED** and gate on scope: if the PR modifies files the anchor would have validated (anything under `src/`, `package.json`, `tsconfig.json`, the workflow), the anchor is required — mark UNVERIFIED, do not auto-merge, and hand to CI/a human (prefer the hosted `CI / build` check on the exact PR head SHA as the real anchor where local execution is blocked). If the PR touches none of those files, record UNVERIFIED-not-applicable and continue, stating it in the self-review and PR body.

   **Green CI is not verification when CI's scope excludes the changed behaviour.** `tsc` and `vitest` never render a frame: neither exercises the scene lifecycle, input handlers, audio playback, or asset loading. If this PR changes a scene, `startGame.ts`, `main.ts`, or anything under `public/audio/`, a green run does **not** verify it — do the manual browser pass (`npm run dev`, then menu → start a run → return to the menu → start a second run, the path that exercises scene re-`create()`; rotate/resize once if layout changed), state the result in the PR body, and never let a green suite imply the scene was tested.

2. **Read the full diff:**
   ```bash
   gh pr diff <number>
   ```

3. **Run the self-review rubric.** Score each item PASS or FAIL with a one-line justification grounded in the diff or a command output — not in judgment. Frame this adversarially: assume FAIL unless you have direct evidence of PASS. Treat an all-PASS result as suspicious; a reviewer expects you to find at least one issue.

   Universal rubric:
   - **Scope:** every file modified is necessary for one of the issues in `Closes #N` (no unrelated formatting, renames, or comment churn).
   - **Tests-new:** every new exported function/method has at least one test that exercises it — or an explicit note naming why it is unreachable in the node environment.
   - **Tests-fix (empirical, not judged):** for each bug fix, temporarily revert the fix (`git stash push -- <src files>`), run the new/changed tests and confirm they **FAIL**, then `git stash pop` and confirm they **PASS**. A regression test that still passes with the fix stashed is a false-negative (common when the "after" state is indistinguishable from the "before") — use distinct/sentinel data so the failure is observable. Do not score this from reasoning alone.
   - **Sibling structure:** every new file matches the section/structure conventions of its directory siblings (Phase 3 rule).
   - **Sibling renames:** every renamed identifier in a parallel pair/series has its siblings renamed in the same commit (Phase 3 rule).
   - **Docs:** every row in the Phase 7 documentation sources-of-truth table reflects the new behavior.
   - **Issue resolution:** every `Closes #N` issue's named surface area is actually changed; no issue is partially resolved while claiming closure.
   - **CI:** the external anchor is green on the PR head (re-confirms step 1).

   Repo-specific rubric items:
   - **No literal coordinates:** `git diff` shows no raw numeric x/y/width/height passed to a Phaser factory or `setPosition`; every placement comes from `src/game/ui/layout.ts`, and any new interactive element has a `layout.test.ts` assertion that it stays on screen and clears `MIN_TOUCH_PX`.
   - **Theme, not literals:** no new colour number or `Phaser.Types.GameObjects.Text.TextStyle` literal in the diff; colours and styles come from `COLORS` / `headingStyle()` / `bodyStyle()` in `src/game/ui/theme.ts`.
   - **Scene state resets:** every scene field added or newly mutated in the diff is reassigned in that scene's `init()`, not only by a class-field initializer.
   - **Re-layout survives:** every object newly placed in `RiverScene` registers its placement through `onLayout()`.
   - **No viewport globals:** `grep -n 'window.inner\|document\.' <changed files>` returns nothing outside the existing `guideAudio.ts` `localStorage` helpers; size reads come from `this.scale`.
   - **Type-only imports:** every symbol imported from `src/game/types.ts` (or any other type-only import) uses `import type` — `verbatimModuleSyntax` makes the alternative a build error.
   - **No erased-syntax violations:** no constructor parameter properties (`constructor(private readonly …)`) and no `enum`/namespace syntax that `erasableSyntaxOnly` rejects.
   - **No strictness escapes:** no `!` non-null assertion, `as any`, or `@ts-expect-error` added; strict errors were fixed by narrowing.
   - **Node-environment safe tests:** no test file in the diff imports `phaser` or touches `window`/`document`.
   - **Style match:** no semicolons, single quotes, 2-space indent, trailing commas in multi-line literals, numeric separators on large ms values.
   - **Scene logic extracted:** any new judgment/selection/timing/outcome logic lives in a plain module with a colocated test, not inside `MenuScene`/`RiverScene`.
   - **Generated and pinned files untouched:** the diff contains no change under `public/audio/`, no edit to the `sharp` override in `package.json`, and no cue-timing/difficulty change in `src/game/levels.ts`.
   - **Lockfile honesty:** `package-lock.json` changes only if `package.json` dependencies changed in the same PR.

4. **For each FAIL item:**
   - If the fix is mechanical and small, fix it locally, commit, push, and re-score the failed item. Do not re-score items that previously passed.
   - If the item requires judgment (e.g. "is this scope creep?"), leave it as an inline review comment for Phase 6.

5. **Post the self-review as a plain PR comment** — not a formal review API call. The auto-mode classifier blocks `POST /pulls/<n>/reviews` (it misrepresents independent review); a plain comment keeps the audit trail without implying reviewer independence:
   ```bash
   gh pr comment <number> --body "$(cat <<'EOF'
   Self-review rubric:
   - Scope: PASS — <justification>
   - Tests-new: PASS — <justification>
   - Tests-fix: PASS — stash-and-run confirmed FAIL→PASS
   - No literal coordinates: PASS — <justification>
   - ...

   <one-line summary; fold any out-of-diff observations into this body>
   EOF
   )"
   ```
   Reserve inline line-anchored comments for lines this PR actually adds/changes — an inline comment on a line **outside the diff hunk** is rejected (`HTTP 422: Line could not be resolved`); fold those observations into the comment **body** instead. Do not retry a 422 with a different line number. Omit inline comments entirely if every rubric item passed after fixes and there are no judgment calls to flag.

6. **Cap: one intrinsic-critique pass per PR.** After Phase 6 addresses the rubric's inline comments, do not re-run the rubric internally. Only an external signal — a new reviewer comment or a CI failure — should re-open the iteration loop. Empirical findings (RESEARCH.md §5) show repeated intrinsic critique plateaus by iteration 2 and can regress.

7. Proceed to Phase 5 — address your own comments in Phase 6.

---

### Phase 5 — Wait for review

```bash
gh pr view <number> --comments
gh api repos/McElyea/forward-one/pulls/<number>/comments \
  --jq '.[] | "File: \(.path)\nLine: \(.line)\nBody: \(.body)\n"'
```

If no review yet, use `ScheduleWakeup` (delay 270 s) to check again.
After 5 wakeups (~22 min) with no review, proceed anyway. Copilot is the configured reviewer and has reviewed at least one PR here (#13), but most merged PRs carried no review — do not treat its silence as a block.

**Autonomous multi-cycle batch mode** (`/forward-one-dev-loop until you run out of issues` / `do N cycles`): there is no human reviewer between back-to-back cycles, so the ~22 min poll is pure latency. Treat the self-review rubric + green CI as the merge gate and **skip (or cap at one short poll)** this Phase-5 wait — except for do-not-auto-merge / charter-gated PRs, which still hand off for human approval. **Stop condition:** end the batch when the only remaining issues are blocked, charter-gated, or too large for a polish-sized PR, or when a cycle yields no appropriately-scoped work.

---

### Phase 6 — Address comments

**External vs. internal signals.** Comments from a real reviewer and CI failures are *external* signals — keep iterating on them until each is resolved. The self-review rubric posted in Phase 4 is *internal* — once its comments are addressed here, do not re-run the rubric. Per RESEARCH.md §1 and §5, repeated intrinsic critique without an external signal is neutral-to-harmful.

For each comment:
1. Read the comment. Read the referenced source before changing anything.
2. If correct, fix the code.
3. If it conflicts with `AGENTS.md`, the issue spec, or `tsconfig.json`,
   reply with the evidence and do not apply the change. `AGENTS.md` is explicit that when a claim and the code disagree the code wins — so quote the source, not the doc, when the two conflict.

Commit using HEREDOC so the trailer is on its own line. Stage by name (never `git add -A`). **Do not hardcode the co-author model name** — defer to the harness's standing git rule, which appends the *actual running model* (a hardcoded name misattributes the commit when a different model version is running):
```bash
git add <files>
git commit -m "$(cat <<'EOF'
<fix description>

Co-Authored-By: <the actual running model, per the harness git instructions> <noreply@anthropic.com>
EOF
)"
git push
```

Run `npm run build && npm run test` after every fix. Do not push a broken build.

---

### Phase 7 — Documentation accuracy check

This phase is **PR-scoped**: it verifies the docs against *this PR's* implementation. The proactive, repo-wide version (drift unrelated to the current change) is a selectable cycle work-mode — Stage A in Phase 2.

**Read the implementation first**, then check each doc against it.

| Source | What to verify against the implementation |
|--------|-------------------------------------------|
| `README.md` — "Commands" block | Every `package.json` script has a line, and every documented script exists. Enforced by `src/docs.test.ts`, so a mismatch is a test failure, not a review finding. |
| `README.md` — "Run it locally" | The Node floor matches `engines.node` and `ci.yml`'s `node-version` (enforced by `docs.test.ts`); the control keys, guide-voice list/default, and orientation claims still match `RiverScene`/`MenuScene` and `guideAudio.ts`. |
| `README.md` — "Project structure" | Every listed `.ts` path exists (enforced by `docs.test.ts`) and any module this PR added that a newcomer would look for is listed. |
| `README.md` — "Multiplayer path" | Still describes the real `RaceAdapter` boundary and its existing implementations. |
| `AGENTS.md` | Every claim and every `file:line` citation this PR could have invalidated — the gate commands, the architecture invariants, the Phaser `init()` reference (`RiverScene.ts:73`), the testing constraints, the `tsconfig.json` flag line numbers, the "Do not touch" list, and the style rules. This file promises it was verified against source; a stale citation here is a bug in the file. |
| `.github/workflows/ci.yml` | The commands it runs still exist and still constitute the full gate; the Node version still matches `engines.node`. |
| `package.json` | `scripts` reflect any command this PR added or renamed; `engines.node` unchanged unless deliberately raised everywhere. |
| `src/docs.test.ts` | Its parsing assumptions still hold against the current `README.md` structure — if this PR restructured a heading or fenced block it parses, the test's error messages must still name what drifted rather than throwing a format error. |
| `.env.example` | Still matches the variables the code actually reads, if this PR touched configuration. |

If any inaccuracy is found: fix it, commit, and restart this phase from the top.
Only proceed when a complete pass finds nothing wrong.

---

### Phase 8 — Merge

**Regression gate.** For each issue in `Closes #N` that is a bug fix or describes incorrect behavior, verify the diff includes a new or modified test that exercises the fix — and that it was confirmed empirically (the Phase 4 stash-and-run: FAIL with the fix reverted, PASS with it restored), not by reasoning alone. If the fix lives in code the node-environment suite structurally cannot reach (a scene, `startGame.ts`), the regression evidence is the recorded manual browser pass naming the exact steps and the observed before/after — say so explicitly rather than leaving the gate unmet. If neither exists, do not merge — either add the regression coverage or reclassify the issue. Per RESEARCH.md §3, regression evidence is the only way to distinguish a real fix from a coincidental patch.

**Do-not-auto-merge path check.** Before invoking `gh pr merge`, list the files this PR modifies and check them against the do-not-auto-merge list. If any modified path matches, do not merge automatically — leave the PR open and report to the user for manual review.

```bash
git diff --name-only origin/main...HEAD
```

Universal entries (any repo):
- `.github/workflows/*` — CI config changes affect downstream review/automation gates
- Any path under a `security/` directory
- A single file with more than 50 lines deleted (check with `git diff --stat origin/main...HEAD`)

Repo-specific entries:
- `package.json` / `package-lock.json` — carries the `sharp` override pinned for `kokoro-js`, the Node floor CI and the README both assert, and the script list `src/docs.test.ts` parses
- `tsconfig.json` — the compiler *is* the linter here; a flag change silently changes what the build rejects
- `public/audio/**` and `scripts/generate-guide-voice.mjs` — generated 4.6 MB WAV set and the 82M-model generator that produces it; never hand-edited, and a regeneration is a user decision
- `src/game/levels.ts` — cue timing and difficulty are gameplay-feel decisions reserved to the user
- `AGENTS.md` — the agent-facing contract every future cycle reads; a human should see changes to the rules before they bind the next loop

If no modified path matches, proceed:

```bash
gh pr merge <number> --squash --delete-branch
git checkout main && git pull
```

Squash is the repo's established strategy — every merged commit on `main` is a single squashed commit titled with its PR number.

**Review-ready hand-off is a valid terminal state — not a failed cycle.** On a repo with no autonomous merge path (branch-protected `main` requiring an approving review, and/or repository auto-merge disabled), a clean, CI-green PR's true terminal state is *open + self-review posted + awaiting human approval*. Expect `gh pr merge` → `the base branch policy prohibits the merge` and `--auto` → `Auto merge is not allowed for this repository`. **Never use `--admin`** to bypass a deliberately-configured human gate — and this account has `WRITE`, not `admin` (see the access note at the top), so `--admin` is not merely forbidden here, it is unavailable; if a merge is blocked, hand off rather than working around it. A later self-audit must not score "could not auto-merge" as a failure. A do-not-auto-merge path match blocks **autonomous** merge only: if the human codeowner explicitly authorizes the merge after being shown which protected path matched, that authorization satisfies the hold — proceed (still run the Phase 7 docs sweep + the regression gate above first), and state in the merge report which protected path was overridden and by whose authorization.

**Worktree note:** under a `git worktree` workflow (main checkout kept on `main`), `gh pr merge --delete-branch`'s *local* branch deletion can fail with `fatal: 'main' is already checked out` — the merge and remote-branch deletion still succeed. Verify the real state with `gh pr view <number> --json state` and delete the remote branch separately rather than treating the local error as a failed merge.

Verify issues auto-closed. Close any that did not:
```bash
gh issue list --state open
gh issue close <number> --comment "Resolved in PR #<n>."
```

**Proceed to Phase 9.** Do not stop here — the self-audit is mandatory whether or not anything was implemented.

**Cycle summary (re-read at the start of Phase 9).** After merging, write a compressed cycle summary:
- **Issues closed:** `#N`, `#M`
- **PR(s) merged:** `#X`
- **Surprises:** anything that didn't go as expected (anchor for the self-audit's reflection prompts)
- **Followups filed:** any new issues filed mid-cycle and where

---

### Phase 9 — Self-audit

1. **Check for duplicates** before filing:
   ```bash
   gh issue list --repo dmccoystephenson/forward-one-dev-loop --state open
   ```

2. **Run the self-audit rubric.** For each item, mark PASS / FAIL / "no signal this cycle" with a one-line justification grounded in the cycle's actual events:
   - **Identity drift:** did this cycle act in accordance with the identity stated at the top of this skill?
   - **Instruction clarity:** did any step require interpretation or produce a wrong first attempt?
   - **Edge case coverage:** did any failure mode arise that the Edge cases section does not cover?
   - **Phase friction:** did any phase require significantly more or fewer steps than expected?
   - **Drift candidates:** did any decision feel like it should be encoded in `create-dev-loop.md` rather than re-discovered each cycle?
   - **External-signal quality:** did `CI` actually catch the kinds of issues it's supposed to catch this cycle — and did anything reach `main` that only the manual browser pass could have caught?

   Per RESEARCH.md §1 and §5, structured rubrics outperform free-form prompts for LLM critique — the same logic that grounds the Phase 4 self-review applies to this retrospective.

3. **For each FAIL item not already tracked, file a labeled issue** so triage knows where the gap belongs:
   ```bash
   gh issue create --repo dmccoystephenson/forward-one-dev-loop \
     --title "<short description of the gap>" \
     --body "<what was ambiguous or missing, and suggested instruction text>" \
     --label <one-of: template-rule | repo-specific | edge-case | research-gap | process>
   ```

   Label taxonomy:
   - `template-rule` — should be promoted into `create-dev-loop.md`
   - `repo-specific` — belongs only in this skill
   - `edge-case` — should be added to Edge cases
   - `research-gap` — new finding worth a RESEARCH.md entry
   - `process` — meta-issue about how the loop runs

4. **Do not implement or merge changes to the skill itself** — file issues only so a human reviews and approves skill edits.

5. If nothing notable is found, note that explicitly and proceed.

---

### Phase 10 — Next cycle

Return to Phase 1.

---

## Edge cases

**Tests fail during implementation:** diagnose; never skip or use `--no-verify`.
**Tests fail after addressing a comment:** same rule.
**CI fails on the PR (Phase 4 or later):** treat the failing anchor as the highest-priority external signal — fix the underlying cause locally, push, and re-confirm before continuing the rubric or addressing other comments. Do not start or re-run the self-review rubric while CI is failing.
**The external anchor cannot run (Node/npm absent, `npm ci` blocked by the sandbox network):** do not claim it green and do not iterate on the sandbox. Flag **UNVERIFIED** and gate on scope (Phase 4): anchor-relevant files changed → mark UNVERIFIED + do-not-auto-merge + hand to CI/human (prefer the hosted `CI / build` check on the exact head SHA); no anchor-relevant files → record UNVERIFIED-not-applicable and continue, stating it in the PR body.
**A test fails at import with an error about `phaser`, `window`, or `document`:** this is the node-environment constraint, not a bug in the test. Do not add a `phaser` mock or reach for jsdom mid-PR — extract the logic under test into a plain module, or file an issue for the deliberate `vite.config.ts` + jsdom change and pick different work.
**`npx tsc` reports errors in files this PR did not touch:** check whether `noUncheckedIndexedAccess` (or another flag) was enabled by the change. It is deliberately off and would surface ~14 pre-existing errors; enabling it is its own PR, never a side effect.
**A change only shows up when the game renders:** `tsc` and `vitest` never render a frame. Do the manual pass — `npm run dev`, then menu → start a run → Escape back to the menu → start a second run, plus a rotate/resize if layout changed — and record the result in the PR body. A green suite on scene changes is not evidence.
**An issue requires a harness-blocked or user-gated operation (regenerating `public/audio/`, adding a jsdom test config and dependency, retuning `levels.ts` difficulty, editing agent-loaded config):** recognize it at triage (Phase 1) — surface to the user for explicit authorization rather than attempting it mid-cycle; never remove a path while `README.md` or `AGENTS.md` still references it.
**`src/docs.test.ts` throws "the format it is parsed from has changed":** the README structure it parses was restructured, not merely edited. Restore the parseable shape or update the parser deliberately in the same PR — do not delete the assertion to make the suite green.
**The resolved clone is shallow (`git rev-parse --is-shallow-repository` → `true`):** `git fetch --unshallow` before rebasing or reading history; a shallow tree makes `git rebase origin/main` and `git log` misleading.
**Autonomous multi-cycle batch (`/forward-one-dev-loop until …`):** skip/cap the Phase-5 human-review wait (rubric + green CI is the gate); still hand off do-not-auto-merge/charter PRs. Stop when only blocked, charter-gated, or too-large work remains, or a cycle yields no scoped work.
**A concurrent session holds the tree or an open PR:** adopt its PR (bring current with `main`, re-run CI, review, merge if green) rather than doing nothing; work in a `git worktree` to avoid colliding, and treat a harmless local `--delete-branch` failure as success once `gh pr view --json state` confirms the merge.
**A test fails intermittently (suspected flake):** re-run `npm run test` once. If the same test fails again, treat it as a real failure and investigate. If it passes on the second run, note the flake in the PR body and proceed — do not suppress or `.skip` a test without understanding why it is flaky.
**A review comment is a false positive:** reply with evidence, do not apply the change.
**Reviewer addition fails:** proceed to the self-review step in Phase 4. Do not skip to Phase 5 without posting self-review comments — Phase 6 addresses those comments like any other review.
**Branch is behind main or has a merge conflict at Phase 8:** rebase onto main, re-run tests, and force-push before retrying the merge:
```bash
git fetch origin
git rebase origin/main
npm run build && npm run test
git push --force-with-lease
```
If the rebase produces conflicts that cannot be resolved automatically, close the PR and delete the branch, then return to Phase 1:
```bash
gh pr close <number> --comment "Closing: unresolvable merge conflict after rebase."
git checkout main
git push origin --delete feature/<name>
```
**Two issues conflict mid-implementation:** finish the further-along one; file a note on the other.
**Cycle exceeds abort budget:** if the cycle's tool calls exceed ~500 or accumulated context exceeds ~200k tokens without converging, abort rather than push through. The half-life model (RESEARCH.md §4) predicts persistence past a budget is strictly worse than restart with fresh context. Steps:
1. Mark the in-flight PR `Draft` (`gh pr ready --undo <number>`) or, if no PR is open, push the branch with a `WIP:` commit so work isn't lost.
2. File a gap issue on `dmccoystephenson/forward-one-dev-loop` with title `Abort budget exceeded on <branch>` and body containing:
   - **Issues in scope:** the `#N`s the cycle was trying to close
   - **Files modified so far:** path list
   - **Where convergence stalled:** the last phase reached and what was blocking it
   - **Suggested next attempt:** what to try differently on the next run
3. Exit. Do not return to Phase 1 in the same context — restart in a fresh session.
**No issues and no improvements found:** report what was checked; end the loop.
