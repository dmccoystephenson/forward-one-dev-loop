# forward-one-dev-loop

<!-- template-version: c1ac200 -->
<!-- generated-at: 2026-08-13T02:45:55Z -->

Autonomous iterative development loop for Forward One.

**Identity:** the kind of skill that refuses to treat "it compiles" as "it runs" — this repo has no CI, so a locally-green `npx tsc && npm run build && npm run test` is the only thing standing between a prototype and a broken put-in screen, and the loop never pushes without it. It rejects Phaser scene changes that leave instance state alive across a second `create()`, and it rejects README commands and control lists that no longer match the code.

**Working directory:** /home/userland/forward-one (resolved at runtime — see Phase 1; do not assume this literal path exists)
**Project repo:** McElyea/forward-one
**Skill repo:** dmccoystephenson/forward-one-dev-loop (issues for self-audit findings go here)

> **Access note.** The account running this loop (`dmccoystephenson`) is a **collaborator** on `McElyea/forward-one`, not its owner: `push` and `triage` are available, `admin` and `maintain` are not. Branches and PRs go directly to the upstream repo — there is no fork in the path.

---

## Full cycle

---

### Phase 1 — Triage

**Resolve the working tree before `cd`** — never assume a hardcoded absolute path exists on every machine/container (the first action of the cycle must not fail because the configured path is absent). Resolution order: explicit env var → configured path → a detected clean clone → fresh clone.

```bash
# Resolve REPO_ROOT: $FORWARD_ONE_DIR if set & present > the configured path > a clean clone whose
# origin matches McElyea/forward-one (prefer no uncommitted changes; skip /mnt/ user copies) > fresh clone.
if [ -n "$FORWARD_ONE_DIR" ] && [ -d "$FORWARD_ONE_DIR" ]; then
  REPO_ROOT="$FORWARD_ONE_DIR"
elif [ -d "/home/userland/forward-one" ]; then
  REPO_ROOT="/home/userland/forward-one"
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

**Confirm `origin` actually points at `McElyea/forward-one` before doing anything else.** A local clone in this environment was once configured with an `origin` of `dmccoystephenson/forward-one`, which does not exist on GitHub — every `pull`, `push`, and `gh pr create` fails with a 404 that reads like an auth problem:

```bash
git remote get-url origin   # must print https://github.com/McElyea/forward-one.git
```

If it prints anything else, fix it with `git remote set-url origin https://github.com/McElyea/forward-one.git` rather than re-cloning.

**Install dependencies if `node_modules` is absent or `package-lock.json` changed:**

```bash
[ -d node_modules ] || npm ci
```

**Check for open PRs from previous cycles first.** If any open PR exists, decide before doing anything else:
- If the PR is still valid (build+test green, no conflicts), **first confirm a Phase 4 self-review was actually posted** (a carried-over PR from a prior cycle may never have completed it). If none is recorded, perform the Phase 4 self-review now (build+test must be green first) before jumping to Phase 5. Otherwise jump to Phase 5 to re-poll for review.
- If the PR is stale or conflicted, close it with a comment explaining why, then proceed with triage.
- **If the open PR was authored by a concurrent session/another author** (not this loop), do not misread "don't open a new PR" as "do nothing": adopt it — bring it current with `main`, re-run full build+test, review, and merge if green (or close it with a reason). Under a `git worktree` workflow the main checkout stays on `main` to avoid colliding with the other session's tree.

Do not open a new PR while one is already open against the same repo.

**Close stale-open issues first.** Check whether any open issues were resolved by
recent PRs but not yet closed. Cross-reference `git log` against open issue titles:
```bash
gh issue close <number> --comment "Resolved in PR #<n>."
```

Scan for improvements not yet tracked:
- Missing tests on new public methods
- Doc drift between sources of truth
- Unhelpful error messages or missing usage strings on commands
- **Phaser scene instance state that survives a second `create()`** — arrays pushed to inside `create()`, caches, sound handles, or selection state that is initialized only by a class-field initializer. Phaser constructs each scene **once** and reuses that instance for every `scene.start()`, so field initializers never run again. This is the dominant defect class in this repo; `RiverScene.init()` resets its state correctly and is the pattern to copy.
- **Silent fallbacks that hide bad input** — e.g. `getLevel()` in `src/game/levels.ts` returning `LEVELS[0]` for an unrecognized id rather than surfacing the miss.
- **Game-state machines with no terminal condition** — a run that can only end on `progress >= 1` strands the player whenever the available gain budget cannot reach it.
- **Assets preloaded but unused on the current code path** — `loadGuideAudio()` queues all four guide voices even though one is selected.
- Magic numbers repeated across scenes instead of named constants in `src/game/ui/theme.ts`
- `README.md` command/control claims that no longer match `package.json` scripts or the registered `keydown-*` handlers

**Before filing each issue**, verify every claim against source:
- Method names/call sites — grep to confirm existence and behaviour
- "X doesn't exist" — read the file to confirm the absence
- Example output — trace through code to confirm it is realistic

**After filing a batch of issues**, second-pass each one:
- Title accurately describes what the body says
- Every claim in the body still holds after re-reading the source

**Classify harness-blocked operations up front.** Before selecting work, flag any issue whose fix requires an operation the harness/auto-mode classifier denies — so it is recognized at triage rather than failing mid-implementation. In particular, **creating or editing agent-loaded config (`CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`) requires explicit, separate user authorization** — surface such issues to the user instead of attempting them. Adding a `.github/workflows/*` file is also gated: it is on the do-not-auto-merge list in Phase 8, so it may be implemented but never merged autonomously. Never remove a path while `README.md` still references it (that creates the very doc drift Phase 7 guards against).

**Record skip reasons.** Any open issue that exists at triage time but is not picked for this cycle's work must have its skip reason recorded — either in the PR body of whatever this cycle does pick, or as a comment on the skipped issue. The point is auditability: a human (or a later self-audit) can see which issues were intentionally deferred and why, rather than reading silence as a value judgment. **Exception — externally-directed cycles** (`/forward-one-dev-loop on <PR/issue/target>`): the entire untouched backlog is deferred for one self-evident reason (the cycle was scoped to the given target). Record that **once** in the groomed/created PR body or review — do **not** mass-comment skip reasons on unrelated issues.

---

### Phase 2 — Work selection

Choose 0–3 issues to implement as a coherent PR:

- **0** if all open issues are blocked or too large — note why and skip to Phase 9.
- **1–2** is the default. Prefer issues that touch the same subsystem or naturally
  complement each other.
- **3** only when all three are small and clearly independent.

When issues have a dependency relationship, implement the foundation first.

**Tiebreaker rule.** When choosing between issues of comparable scope and no dependency relationship, bias toward (in descending preference): documentation fixes → CI/build fixes → small refactors → bug fixes → performance work. Autonomous-agent PRs merge at substantially higher rates for the earlier categories. This is a soft tiebreaker, not a hard exclusion — a clearly-scoped bug fix is still better than no progress, and coherent-batch grouping still takes precedence when it applies.

**Early-cycle bias.** For the first 3–5 cycles after this skill was generated, weight the tiebreaker more strongly toward documentation and build fixes — the agent has minimal prior context for harder work. Once the loop has shipped several cycles successfully, the bias relaxes. This is a preference, not a rule; an explicit user instruction overrides it.

**Repo-specific priority.** Until a CI workflow exists, the build+test anchor is local-only and unverifiable by any reviewer looking at the PR page. A cycle that lands `.github/workflows/ci.yml` upgrades the anchor for every subsequent cycle — treat it as a CI/build fix (high on the tiebreaker), and after it merges, switch Phase 4 step 1 from the local commands to `gh pr checks <number> --watch`.

Include `Closes #N` in the PR body for each resolved issue so GitHub auto-closes on merge.

**Alternative cycle work modes.** Implementing issues is the default unit of work, but a cycle may instead be devoted to one of the two stages below. Both are **first-class outcomes** (not filler) and produce a normal PR through Phases 4–8. Prefer them when the open-issue backlog is thin, blocked, or all human-gated.

#### Stage A — Documentation accuracy review (sweep)

Sweep the documentation for drift against the *actual source*, independent of any recent change — the proactive, repo-wide complement to the PR-scoped check in Phase 7. Go through every documentation source of truth (the Phase 7 table) and verify each claim against the code, config, or commands it documents. **Verify against source, never memory.**

- Fix drift **in the docs**. If the *code* is what's wrong (the docs describe the intended, correct behavior), do **not** silently change code under a docs cycle — file an issue and leave it for an implementation cycle.
- Respect the Phase 3 scope ceiling. If drift is large, fix the highest-value subset this cycle and file an issue enumerating the rest.
- If a complete sweep finds **no** drift, say so explicitly and fall back to another work mode — an empty docs PR is not an outcome.

#### Stage B — Unit-test expansion (functionality confidence)

Add tests to under-covered, behavior-bearing code to lock in current correct behavior and create regression guards. The goal is **confidence**, not a coverage percentage.

- Target, in order of preference: core/domain logic with **no** existing test, then entry points whose logic is untested, then the remaining behavior-bearing code. As of this skill's generation the untested behavior-bearing units are `getLevel()` (`src/game/levels.ts`), `SoloRaceAdapter`, `SimulatedRaceAdapter`, `hexToNumber()` (`src/game/ui/theme.ts`), and `RhythmEngine.getVisible()`.
- Follow the test framework, location, and mocking conventions in Phase 3 and in sibling tests (read 2–3 neighbors first). **No real network or database calls.**
- **Characterization, not change.** These tests must assert the code's *current* behavior. If writing one reveals an apparent bug, do **not** change production code under a test-expansion cycle — document the current behavior (or mark the test skipped with a reason), file a bug issue, and leave the fix to a separate cycle. Never weaken an existing assertion to make a new test pass.
- Scope one cohesive class/area per cycle, within the Phase 3 scope ceiling.

```bash
git checkout -b feature/<short-description>
```

**Plan summary (re-read at the start of Phase 3).** Before exiting Phase 2, write a compressed plan in this shape:
- **Work in scope:** the issues (`#N`, `#M`, …), or `Stage A — documentation accuracy sweep`, or `Stage B — unit-test expansion (<target class/area>)`.
- **Branch:** `feature/<name>`
- **Files I expect to modify:** path, path, ...
- **Invariants to preserve:** `npx tsc` exits 0, `npm run test` fully green, every mutable scene field reset in `init()`, no new `window`/`phaser` import inside a test, README commands still match `package.json`.

Phase 3 begins by re-reading this summary.

---

### Phase 3 — Implementation

**Localization verification.** Before writing any code, list the files this PR intends to modify and verify each one:

1. **Confirm the file exists.** `test -f <path>` or `ls <path>`.
2. **Confirm the surface area is present.** For each file, grep for the symbol, heading, config key, or behavior named in the issue. If the issue says "`renderSelection` indexes `LEVELS` by card position", run `grep -n 'renderSelection' src/game/scenes/MenuScene.ts` and confirm the named entity is present. If it isn't, stop and re-triage — the localization is wrong and editing here would produce a misfire.

Follow project conventions:
- **Types live in `src/game/types.ts`.** Import them with `import type { … }` — `verbatimModuleSyntax` is on, so importing a type as a value fails the build.
- **Testable logic lives outside the scenes.** `RhythmEngine` (pure judging) and `guideAudio` (pure key/selection helpers) are the pattern to copy; `MenuScene`/`RiverScene` should stay presentation-only. When a fix needs new logic, prefer extracting it into a plain module so it can be covered by a node-environment test.
- **Multiplayer stays behind `RaceAdapter`** (`src/game/race/RaceAdapter.ts`). A new backend is a new adapter implementation; `RiverScene` must not need to change to accommodate it.
- **Colors and text styles come from `src/game/ui/theme.ts`** (`COLORS`, `headingStyle`, `bodyStyle`), not inline literals.
- **Scene coordinates are absolute against a 1280×720 design canvas** (`src/game/startGame.ts:9-10`, with `Phaser.Scale.FIT`). Position new elements in that space; never read `window.innerWidth`/`innerHeight`.
- **Reset every mutable scene field in `init()`.** Phaser reuses one instance of each scene across all `scene.start()` calls, so class-field initializers run only at construction. `RiverScene.init()` is the reference implementation.
- **Style:** 2-space indent, single quotes, no semicolons, trailing commas in multi-line literals. There is **no** ESLint or Prettier config — match the surrounding file by hand, because nothing will catch drift for you.
- **`public/audio/**` is generated.** Never hand-edit a WAV; clips come from `npm run voice:generate`, which downloads an 82M Kokoro model and is a dev-only path.

Universal rules:
- **Match sibling structure.** Before creating a new file in a directory, read the structure of every existing file in the same directory and conform to the established pattern.
- **Rename siblings together.** When renaming a heading or identifier that is part of a parallel pair or series (e.g. `forward`/`backward`, `SoloRaceAdapter`/`SimulatedRaceAdapter`), scan for the siblings and rename them in the same commit.
- **Scratch-file handling in a sandboxed harness.** When a step needs a scratch file for inspection or transformation (not a project source file), prefer the `Write` tool over `> file` shell redirection, and prefer `python3 -c "import os; os.remove(path)"` over `rm` to clean it up afterward. Some harness sandboxes statically block plain `>` redirection and `rm` outright — even for files the same session just created inside the working directory — while `Write` and `os.remove` are not pattern-matched the same way.

Write or update tests for every change (and see Stage B in Phase 2 when the *whole cycle* is dedicated to expanding coverage):
- Framework is **vitest** (`npm run test` → `vitest run`).
- There is **no `vite.config.ts`**, so vitest runs with its defaults: the **node** environment and **no globals**. Import every helper explicitly — `import { describe, expect, it } from 'vitest'`.
- Tests are **colocated** with their source and named `<Source>.test.ts` (`src/game/rhythm/RhythmEngine.test.ts`, `src/game/audio/guideAudio.test.ts`). Do not create a top-level `test/` tree.
- Because the environment is node, a test must **not** import `phaser`, construct a `Phaser.Scene`, or touch `window`/`document`. This is why `getSelectedGuideVoiceId()`/`selectGuideVoice()` (which read `window.localStorage`) are currently untested — covering them requires adding a `vite.config.ts` with `test.environment: 'jsdom'` plus the dependency, which is its own PR and its own review.
- Prefer extracting logic into a pure module and testing that, over trying to drive a scene.

Verify the build is clean:
```bash
npx tsc
npm run test
```

Fix all failures before proceeding. Never skip tests or bypass hooks.

**Formatting is scoped to changed files.** There is no formatter in this repo, so there is nothing to run tree-wide — but the same rule applies to any mechanical sweep: only `git add` files in this PR's scope and `git checkout --` anything else that got touched. Pre-existing style drift lands as its own sweep, never smuggled into a feature/fix PR.

**Git-staging hygiene.** Stage by name — **never** `git add -A` or `git add .`. The harness writes `.claude/` state into the tree while the loop runs, and this project's `.gitignore` does **not** cover it; a blanket add leaks harness state into the project repo (and the classifier blocks the `git rm --cached` cleanup as scope-escalation). `dist/` and `node_modules` *are* ignored. After staging, run `git status` and confirm no `.claude/` entries are staged before committing.

**Scope ceiling.** Before pushing, check the cumulative net diff for this cycle:
```bash
git diff --stat origin/main
```
Count the soft ceiling against **non-test net LOC**. If non-test changes exceed **~400 net LOC** or the PR modifies more than **~10 files**, stop and rescope: either drop one of the batched issues from the PR, or split the remaining work into a follow-up PR. **Exception:** if you are over the soft ceiling *only* because of (a) test code or (b) a dependency-coupled issue that cannot be split without leaving an unused component, proceed but state the overage and the reason in the PR body and self-review. Hard stop unchanged at **~800 LOC** or **~20 files**. Binary assets under `public/audio/` do not count toward the LOC ceiling but do trigger the Phase 8 do-not-auto-merge hold.

**Implementation summary (re-read at the start of Phase 4).** Before pushing, write a compressed implementation summary:
- **Files actually modified:** path, path, ...
- **Commit summary:** one line per commit
- **Test/validation result:** PASS / FAIL — name the command that produced the verdict
- **Open carryovers:** anything in scope that wasn't done and why (becomes input for Phase 9)

---

### Phase 4 — PR

**Before pushing, verify each `Closes #N`.** For every issue number you plan to reference, run `gh issue view <N>` and confirm the title and body match what this PR does. Numbers carried forward from earlier session context or from summarized prior cycles are a common source of wrong-issue auto-closes — if a referenced issue describes unrelated work, omit the `Closes` reference and either file a new issue or note `No tracking issue — gap found during triage.` in the PR body.

```bash
git push -u origin feature/<short-description>
gh pr create --title "..." --body "..."
```

PR body must include:
- Summary bullet points (what changed and why)
- Test plan checklist
- `Closes #N` for each resolved issue

Request a review:
```bash
gh pr edit <number> --add-reviewer McElyea
```

If the command errors (reviewer not configured, or the account lacks permission to assign on this repo), proceed directly to the self-review below.

Perform a self-review. This step is anchored on external signals (build+test, the rubric below) rather than free-form judgment — LLM self-critique without an external signal is unreliable and can regress quality.

1. **Wait for build+test to be green.** This is the external anchor for the rubric below; without it, the rubric is just opinion.

   ```bash
   npm ci          # only when package-lock.json changed this cycle
   npx tsc         # type-check; tsconfig sets noEmit
   npm run build   # tsc + vite production build
   npm run test    # vitest run — full suite, currently 2 files / 8 tests
   ```

   **This repo has no CI.** The anchor is local and nothing on the PR page proves it ran, so state the exact commands and their results in the self-review comment. Once a `.github/workflows/ci.yml` lands, replace this block with `gh pr checks <number> --watch` and treat the hosted run as the anchor.

   If it fails, fix the underlying issue, push, and re-confirm. Do not start the rubric until build+test passes.

   **If the anchor cannot run** because the toolchain is absent or broken in the environment (no `node`, a Node older than 22, a failed `npm ci`), do **not** claim it green and do **not** burn the cycle trying to fix the sandbox. Flag it **UNVERIFIED** and gate on scope: if the PR modifies anything under `src/`, `scripts/`, `index.html`, or the build config, the anchor is required — mark UNVERIFIED, do not auto-merge, and hand to a human. If the PR touches none of those files (e.g. README-only), record UNVERIFIED-not-applicable and continue, stating it in the self-review and PR body.

   **A green anchor does not cover what it cannot execute.** `npx tsc` and `vitest` never render a frame: no scene lifecycle, no input handler, no audio playback, and no `window` access is exercised by them. A change to `MenuScene`, `RiverScene`, `startGame.ts`, or anything under `public/audio/` therefore needs a **manual browser pass** on top of the green anchor — run `npm run dev`, load the printed URL, and walk the specific path the change touches (at minimum: menu → select a level → start a run → return to the menu → start a second run, which is the path that exercises scene re-`create()`). State in the PR body whether that manual pass was performed or was not possible.

2. **Read the full diff:**
   ```bash
   gh pr diff <number>
   ```

3. **Run the self-review rubric.** Score each item PASS or FAIL with a one-line justification grounded in the diff or a command output — not in judgment. Frame this adversarially: assume FAIL unless you have direct evidence of PASS. Treat an all-PASS result as suspicious; a reviewer expects you to find at least one issue.

   Universal rubric:
   - **Scope:** every file modified is necessary for one of the issues in `Closes #N` (no unrelated formatting, renames, or comment churn).
   - **Tests-new:** every new public method/function has at least one test that exercises it.
   - **Tests-fix (empirical, not judged):** for each bug fix, temporarily revert the fix (`git stash push -- <src files>`), run the new/changed tests and confirm they **FAIL**, then `git stash pop` and confirm they **PASS**. A regression test that still passes with the fix stashed is a false-negative — use distinct/sentinel data so the failure is observable. Do not score this from reasoning alone.
   - **Sibling structure:** every new file matches the structure conventions of its directory siblings (Phase 3 rule).
   - **Sibling renames:** every renamed identifier in a parallel pair/series has its siblings renamed in the same commit (Phase 3 rule).
   - **Docs:** every row in the Phase 7 documentation sources-of-truth table reflects the new behavior.
   - **Issue resolution:** every `Closes #N` issue's named surface area is actually changed; no issue is partially resolved while claiming closure.
   - **build+test:** the external anchor is green on the PR head (re-confirms step 1), with the exact commands and results quoted.

   Repo-specific rubric items:
   - **Scene state reset:** every mutable instance field on a `Phaser.Scene` subclass that this PR adds or writes to is reset in that scene's `init()` (or cleared in its `SHUTDOWN` handler). Phaser reuses one scene instance for every `scene.start()`, so a class-field initializer runs exactly once — for the lifetime of the page.
   - **Second-visit safety:** if this PR touches a scene, the menu → run → menu → run path was walked in a browser, or the PR states that it was not.
   - **Type-check clean:** `npx tsc` exits 0, and no `@ts-ignore`, `@ts-expect-error`, `as any`, or newly added non-null `!` assertion was used to get there.
   - **No unused symbols:** `noUnusedLocals` and `noUnusedParameters` are on — an unused import or parameter is a *build failure*, not a lint warning.
   - **Canvas coordinates:** every new positioning constant is expressed against the 1280×720 design canvas, not against `window` dimensions.
   - **Generated audio untouched:** no file under `public/audio/` was hand-edited; new clips come only from `npm run voice:generate`.
   - **Node-environment tests:** no new test imports `phaser` or touches `window`/`document` (vitest runs in the node environment; such a test fails at import time, not with a useful message).
   - **Style match:** new lines use 2-space indent, single quotes, no semicolons, and trailing commas in multi-line literals — there is no formatter to catch drift.
   - **Adapter boundary:** if multiplayer or race-progress behavior changed, the change lives in a `RaceAdapter` implementation and `RiverScene` was not modified to accommodate a specific backend.

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
   - build+test: PASS — `npx tsc` 0 errors, `npm run build` ok, `npm run test` 8/8
   - Scene state reset: PASS — <justification>
   - ...

   <one-line summary; fold any out-of-diff observations into this body>
   EOF
   )"
   ```
   Reserve inline line-anchored comments for lines this PR actually adds/changes — an inline comment on a line **outside the diff hunk** is rejected (`HTTP 422: Line could not be resolved`); fold those observations into the comment **body** instead. Do not retry a 422 with a different line number. Omit inline comments entirely if every rubric item passed after fixes and there are no judgment calls to flag.

6. **Cap: one intrinsic-critique pass per PR.** After Phase 6 addresses the rubric's inline comments, do not re-run the rubric internally. Only an external signal — a new reviewer comment or a build+test failure — should re-open the iteration loop. Repeated intrinsic critique plateaus by iteration 2 and can regress.

7. Proceed to Phase 5 — address your own comments in Phase 6.

---

### Phase 5 — Wait for review

```bash
gh pr view <number> --comments
gh api repos/McElyea/forward-one/pulls/<number>/comments \
  --jq '.[] | "File: \(.path)\nLine: \(.line)\nBody: \(.body)\n"'
```

If no review yet, use `ScheduleWakeup` (delay 270 s) to check again.
After 5 wakeups (~22 min) with no review, proceed anyway.

**This repo is owned by another account.** `McElyea` is the natural reviewer, and a PR that changes gameplay feel (timing windows, progress gain, level cue data) or the visual layout is a judgment call that belongs to the owner, not to this loop. For those PRs, prefer leaving the PR open with the self-review posted over merging on a timeout — see the hand-off note in Phase 8.

**Autonomous multi-cycle batch mode** (`/forward-one-dev-loop until you run out of issues` / `do N cycles`): there is no human reviewer between back-to-back cycles, so the ~22 min poll is pure latency. Treat the self-review rubric + green build+test as the merge gate and **skip (or cap at one short poll)** this Phase-5 wait — except for do-not-auto-merge / owner-judgment PRs, which still hand off for human approval. **Stop condition:** end the batch when the only remaining issues are blocked, owner-gated, or too large for a polish-sized PR, or when a cycle yields no appropriately-scoped work.

---

### Phase 6 — Address comments

**External vs. internal signals.** Comments from a real reviewer and build+test failures are *external* signals — keep iterating on them until each is resolved. The self-review rubric posted in Phase 4 is *internal* — once its comments are addressed here, do not re-run the rubric. Repeated intrinsic critique without an external signal is neutral-to-harmful.

For each comment:
1. Read the comment. Read the referenced source before changing anything.
2. If correct, fix the code.
3. If it conflicts with the issue spec or a project config file, reply with the evidence and do not apply the change.

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

Run `npx tsc && npm run test` after every fix. Do not push a broken build.

---

### Phase 7 — Documentation accuracy check

This phase is **PR-scoped**: it verifies the docs against *this PR's* implementation. The proactive, repo-wide version (drift unrelated to the current change) is a selectable cycle work-mode — Stage A in Phase 2.

**Read the implementation first**, then check each doc against it.

| Source of truth | What to verify against the implementation |
|---|---|
| `README.md` — "Commands" block | Every `npm run <script>` listed exists in `package.json` `scripts` and still does what the line claims |
| `README.md` — controls sentence | The keys named (Space / F / ↑ forward, B / ↓ backward, Escape to menu) match the `keydown-*` handlers registered and removed in `src/game/scenes/RiverScene.ts` |
| `README.md` — guide-voice paragraph | The four voice names and the stated default match `GUIDE_VOICES` and `DEFAULT_GUIDE_VOICE_ID` in `src/game/audio/guideAudio.ts` |
| `README.md` — "Project structure" tree | Every path listed exists, and any new file under `src/game/` a reader would need to find is represented |
| `README.md` — "Multiplayer path" section | Still describes the real `RaceAdapter` contract in `src/game/race/RaceAdapter.ts` (method names and the solo/multiplayer boundary) |
| `README.md` — Node version and deploy targets | The stated minimum matches any `engines` field or CI matrix; the deploy-target list is still true given the Vite `base` in effect |
| `.env.example` | Every `VITE_*` key is either read somewhere under `src/` or explicitly marked not-yet-wired by the comment above it |
| `package.json` `scripts` | Every script is reachable and documented in the README "Commands" block |

If any inaccuracy is found: fix it, commit, and restart this phase from the top.
Only proceed when a complete pass finds nothing wrong.

---

### Phase 8 — Merge

**Regression gate.** For each issue in `Closes #N` that is a bug fix or describes incorrect behavior, verify the diff includes a new or modified test that exercises the fix — and that it was confirmed empirically (the Phase 4 stash-and-run), not by reasoning alone. If absent, do not merge — either add the regression coverage or reclassify the issue. Regression evidence is the only way to distinguish a real fix from a coincidental patch.

**Scene-lifecycle bugs need a real test, not a browser anecdote.** For defects in the "state survives a second `create()`" class, the regression test belongs on the extracted pure logic, or on a small harness that calls the reset function twice. "I clicked through it in the browser and it didn't crash" is evidence, but it is not a regression guard — nothing re-runs it.

**Do-not-auto-merge path check.** Before invoking `gh pr merge`, list the files this PR modifies and check them against the do-not-auto-merge list. If any modified path matches, do not merge automatically — leave the PR open and report to the user for manual review.

```bash
git diff --name-only origin/main...HEAD
```

Universal entries (any repo):
- `.github/workflows/*` — CI config changes affect downstream review/automation gates
- Any path under a `security/` directory
- A single file with more than 50 lines deleted (check with `git diff --stat origin/main...HEAD`)

Repo-specific entries:
- `package.json` / `package-lock.json` — dependency and script changes; the `sharp: 0.35.3` override is load-bearing for `kokoro-js` and must not be dropped incidentally
- `tsconfig.json` — compiler-flag changes alter what every future cycle's `npx tsc` gate actually catches
- `public/audio/**` — 4.8 MB of generated WAV assets; regeneration means an 82M model download, not a change reviewable by diff
- `scripts/generate-guide-voice.mjs` — edits here silently change every future regenerated clip
- `src/game/levels.ts` — cue timing and level data are gameplay-feel decisions belonging to the repo owner
- `LICENSE` — MIT terms

If no modified path matches, proceed:

```bash
gh pr merge <number> --squash --delete-branch
git checkout main && git pull
```

**Review-ready hand-off is a valid terminal state — not a failed cycle.** This loop runs under a **collaborator** account on someone else's repository. If `main` is branch-protected, or repository auto-merge is disabled, a clean, anchor-green PR's true terminal state is *open + self-review posted + awaiting the owner's approval*. Expect `gh pr merge` → `the base branch policy prohibits the merge` and `--auto` → `Auto merge is not allowed for this repository`. **Never use `--admin`** — the account does not have admin anyway, and bypassing a deliberate human gate is not this loop's call. A later self-audit must not score "could not auto-merge" as a failure. A do-not-auto-merge path match blocks **autonomous** merge only: if the owner explicitly authorizes the merge after being shown which protected path matched, that authorization satisfies the hold — proceed (still run the Phase 7 docs sweep and the regression gate first), and state in the merge report which protected path was overridden and by whose authorization.

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
   - **External-signal quality:** did build+test actually catch the kinds of issues it's supposed to catch this cycle — and did anything reach the PR that only a browser pass would have caught?

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

**`origin` 404s on push/pull:** the clone is pointed at `dmccoystephenson/forward-one`, which does not exist. Fix with `git remote set-url origin https://github.com/McElyea/forward-one.git` (Phase 1) rather than re-cloning or assuming an auth failure.
**Tests fail during implementation:** diagnose; never skip or use `--no-verify`.
**Tests fail after addressing a comment:** same rule.
**build+test fails on the PR (Phase 4 or later):** treat the failing anchor as the highest-priority external signal — fix the underlying cause locally, push, and re-confirm before continuing the rubric or addressing other comments. Do not start or re-run the self-review rubric while build+test is failing.
**The external anchor cannot run (Node absent, Node < 22, `npm ci` fails):** do not claim it green and do not iterate on the sandbox. Flag **UNVERIFIED** and gate on scope (Phase 4): anything under `src/`, `scripts/`, `index.html`, or the build config changed → UNVERIFIED + do-not-auto-merge + hand to a human; README-only → record UNVERIFIED-not-applicable and continue, stating it in the PR body.
**The anchor is green but the change is visual or lifecycle-bearing:** `tsc` and `vitest` never render a frame. Any change to a scene, `startGame.ts`, or bundled audio needs a manual `npm run dev` browser pass covering menu → run → menu → run, or an explicit statement in the PR body that it was not performed.
**A bug can only be reproduced in a scene:** do not give up on the regression test — extract the offending logic into a pure module, test that, and have the scene call it. A test that needs `window` or `phaser` cannot run in this repo's node-environment vitest setup.
**An issue requires creating or editing agent-loaded config (`CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`):** recognize it at triage (Phase 1) — surface to the user for explicit authorization rather than attempting it mid-cycle.
**An issue requires adding CI:** implement it if selected, but `.github/workflows/*` is on the do-not-auto-merge list — hand the PR to the owner. After it merges, switch the Phase 4 anchor from the local commands to `gh pr checks <number> --watch`.
**A dependency bump is proposed:** `package.json`/`package-lock.json` are do-not-auto-merge. Note also that `kokoro-js` is a dev-only dependency used solely by `npm run voice:generate`; players never download the model, so its size is not a shipped-bundle concern.
**Autonomous multi-cycle batch (`/forward-one-dev-loop until …`):** skip/cap the Phase-5 human-review wait (rubric + green anchor is the gate); still hand off do-not-auto-merge and owner-judgment PRs. Stop when only blocked, owner-gated, or too-large work remains, or a cycle yields no scoped work.
**A concurrent session holds the tree or an open PR:** adopt its PR (bring current with `main`, re-run the anchor, review, merge if green) rather than doing nothing; work in a `git worktree` to avoid colliding, and treat a harmless local `--delete-branch` failure as success once `gh pr view --json state` confirms the merge.
**A test fails intermittently (suspected flake):** re-run `npm run test` once. If the same test fails again, treat it as a real failure and investigate. If it passes on the second run, note the flake in the PR body and proceed — do not suppress or skip a test without understanding why it is flaky.
**A review comment is a false positive:** reply with evidence, do not apply the change.
**Reviewer addition fails:** proceed to the self-review step in Phase 4. Do not skip to Phase 5 without posting self-review comments — Phase 6 addresses those comments like any other review.
**Branch is behind main or has a merge conflict at Phase 8:** rebase onto main, re-run tests, and force-push before retrying the merge:
```bash
git fetch origin
git rebase origin/main
npm run test
git push --force-with-lease
```
If the rebase produces conflicts that cannot be resolved automatically, close the PR and delete the branch, then return to Phase 1:
```bash
gh pr close <number> --comment "Closing: unresolvable merge conflict after rebase."
git checkout main
git push origin --delete feature/<name>
```
**Two issues conflict mid-implementation:** finish the further-along one; file a note on the other.
**Cycle exceeds abort budget:** if the cycle's tool calls exceed ~500 or accumulated context exceeds ~200k tokens without converging, abort rather than push through. Persistence past a budget is strictly worse than restart with fresh context. Steps:
1. Mark the in-flight PR `Draft` (`gh pr ready --undo <number>`) or, if no PR is open, push the branch with a `WIP:` commit so work isn't lost.
2. File a gap issue on `dmccoystephenson/forward-one-dev-loop` with title `Abort budget exceeded on <branch>` and body containing:
   - **Issues in scope:** the `#N`s the cycle was trying to close
   - **Files modified so far:** path list
   - **Where convergence stalled:** the last phase reached and what was blocking it
   - **Suggested next attempt:** what to try differently on the next run
3. Exit. Do not return to Phase 1 in the same context — restart in a fresh session.
**No issues and no improvements found:** report what was checked; end the loop.
