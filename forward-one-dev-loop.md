# forward-one-dev-loop

<!-- template-version: fa50415 -->
<!-- generated-at: 2026-08-29T02:12:28Z -->

Autonomous iterative development loop for Forward One.

**Identity:** the kind of skill that refuses to let a green suite stand in for a played game — it rejects a scene change nothing has rendered, a regression test that still passes with its own fix reverted, and any claim in `AGENTS.md` or `README.md` the source no longer supports.

**Working directory:** /home/userland/forward-one (resolved at runtime — see Phase 1; do not assume this literal path exists)
**Project repo:** McElyea/forward-one
**Skill repo:** dmccoystephenson/forward-one-dev-loop (issues for self-audit findings go here)
**Project guidance:** read `AGENTS.md` at the start of each cycle if context is cold — it is the repo's own agent-facing constraints file, and `src/docs.test.ts` holds it to the source.

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

**Check for open PRs from previous cycles first.** If any open PR exists, decide before doing anything else:
- **Check for a stuck mid-revert first.** If the PR's CI is red on the final run *and* the HEAD commit message starts with `TEMP:` (the fallback-ladder rung 1 marker in Phase 4), do not treat this as ordinary red CI — a prior session was killed mid-ladder between pushing the temporary revert and restoring the fix. Read the `TEMP:` message for the verified-good SHA it names, `git reset --hard <that-sha>` and force-push to restore the fix, confirm CI is green again, then continue triage as normal.
- If the PR is still valid (CI green, no conflicts), **first confirm a Phase 4 self-review was actually posted** (a carried-over PR from a prior cycle may never have completed it). If none is recorded, perform the Phase 4 self-review now (CI must be green first) before jumping to Phase 5. Otherwise jump to Phase 5 to re-poll for review.
- If the PR is stale or conflicted, close it with a comment explaining why, then proceed with triage.
- **If the open PR was authored by a concurrent session/another author** (not this loop), do not misread "don't open a new PR" as "do nothing": adopt it — bring it current with `main`, re-run full CI, review, and merge if green (or close it with a reason). Under a `git worktree` workflow the main checkout stays on `main` to avoid colliding with the other session's tree — see the concurrent-session entry in Edge cases for where the worktree must live, and why a headless dispatch should not create one at all.

Do not open a new PR while one is already open against the same repo.

**Establish whether a merge path exists.** Before selecting work, check whether CI is *capable* of being green here — a required check that has never passed on `main` is a different condition from one that is red on this branch, and only the first makes every cycle terminate in a hand-off regardless of the change's quality:
```bash
gh run list --branch main --limit 20
# then, for any workflow with no success above:
gh run list --workflow <workflow-file> --branch main --limit 50 --json conclusion
```
If a required workflow shows no successful run in its whole recorded history on `main`, record the repo as **structurally red** for that workflow, together with the cause (absent credentials, an unset secret, an unavailable license) and any issue tracking it. This is a fact about the repository, not about the change — establishing it once at triage is what lets Phase 4 assess per-job signal and Phase 8 name the blocking condition, instead of rediscovering it at the merge gate every cycle.

**Close stale-open issues first.** Check whether any open issues were resolved by
recent PRs but not yet closed. Cross-reference `git log` against open issue titles:
```bash
gh issue close <number> --comment "Resolved in PR #<n>."
```

Scan for improvements not yet tracked:
- Missing tests on new public methods
- Doc drift between sources of truth (`README.md`, `AGENTS.md`, `package.json` scripts, `.github/workflows/ci.yml`, `supabase/migrations/`) — `src/docs.test.ts` mechanically checks a good deal of this; everything else in those files is unguarded prose that only a human read catches
- Unhelpful error messages or missing usage strings on commands
- **Behaviour-bearing logic living inside a scene.** `MenuScene.ts` / `RiverScene.ts` / `LobbyScene.ts` should draw and wire input only; judgment, selection, timing and outcome logic belongs in a plain module beside `rhythm/RhythmEngine.ts`, `run/runOutcome.ts`, `survival/SurvivalEngine.ts`, `multiplayer/lobbyState.ts`
- **Logic stranded behind a DOM call that could be injected.** A `window.*` reach inside an otherwise pure module is a seam, not a dead end — see the injection convention in Phase 3
- **Literal coordinates or sizes in a scene.** Anything positioned with a raw number instead of a region/point from `src/game/ui/layout.ts`, and any interactive element that could fall below `MIN_TOUCH_PX` (currently 44)
- **Inline colour or text-style literals** instead of `COLORS` / `TEXT_COLORS` / `headingStyle()` / `bodyStyle()` from `src/game/ui/theme.ts`
- **Scene fields initialized only by a class-field initializer.** Phaser reuses the scene instance, so any mutable field not reassigned in `init()` leaks state across visits — the failure mode behind issue #1
- **Objects placed without an `onLayout()` closure** in `RiverScene` — anything that will not survive a mid-run rotate
- **`window.innerWidth` / `window.innerHeight` / `document` reads** outside the guarded `localStorage` helpers in `audio/guideAudio.ts` and `multiplayer/playerIdentity.ts` — size comes from `this.scale`
- **A `supabase-js` result whose `error` half is never read.** `.rpc()` resolves rather than rejects on failure; `rpcResult.ts` exists so the check is not rewritten per call site
- **A literal repeated between `src/game/` and `supabase/migrations/`.** The level-id list is gated by `docs.test.ts`; nothing else is
- **Race/progress behaviour that requires editing `RiverScene` for a specific backend** — it belongs behind `RaceAdapter`
- **`noUncheckedIndexedAccess` fallout.** It is deliberately off; `npx tsc --noUncheckedIndexedAccess` reported 120 errors as of 2026-08-29, several real. Re-count rather than trusting that number — it tracks the size of the tree. Fixing a cluster is legitimate scoped work; turning the flag on repo-wide is not a polish-sized PR

**Before filing each issue**, verify every claim against source:
- Method names/call sites — grep to confirm existence and behaviour
- "X doesn't exist" — read the file to confirm the absence
- Example output — trace through code to confirm it is realistic

**After filing a batch of issues**, second-pass each one:
- Title accurately describes what the body says
- Every claim in the body still holds after re-reading the source

**Classify harness-blocked operations up front.** Before selecting work, flag any issue whose fix requires an operation the harness/auto-mode classifier denies — so it is recognized at triage rather than failing mid-implementation. In particular, **editing agent-loaded config (`AGENTS.md`) requires explicit, separate user authorization** — surface such issues to the user instead of attempting them. Never remove a path while `AGENTS.md` or `README.md` still references it (that creates the very doc drift `src/docs.test.ts` and Phase 7 exist to catch). Treat these like the data-volume rule: a known up-front classification, not a mid-cycle surprise. Repo-specific up-front classifications:
- **Regenerating `public/audio/`** downloads an 82M Kokoro model and rewrites 4.6 MB of WAVs — a data-volume operation. Do not attempt it inside a cycle; surface it.
- **Adding `vite.config.ts` / `vitest.config.ts` with `test.environment: 'jsdom'`** is a deliberate infrastructure change plus a new dependency, per `AGENTS.md`. It is its own PR with its own issue — and it is rarely the answer: inject the dependency instead (Phase 3), which has covered every DOM-bound seam reached so far.
- **Retuning `src/game/levels.ts` cue timing** is a gameplay-feel decision reserved to the repo owner; only defect fixes there are in scope.
- **This repo belongs to another account.** The loop runs as a collaborator with `push` and `triage`, not `admin` or `maintain`. Branches and PRs go straight to `McElyea/forward-one`; there is no fork in the path, and `--admin` is never an option.

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

Include `Closes #N` in the PR body for each resolved issue so GitHub auto-closes on merge.

**Alternative cycle work modes.** Implementing issues is the default unit of work, but a cycle may instead be devoted to one of the two stages below. Both are **first-class outcomes** (not filler) and produce a normal PR through Phases 4–8. Prefer them when the open-issue backlog is thin, blocked, or all human-gated — a cycle that tightens the docs or hardens the test suite is real progress. They also rank high on the tiebreaker (documentation review sits with "documentation fixes"; test expansion just below it; see RESEARCH.md §2), so favor them over speculative feature work.

#### Stage A — Documentation accuracy review (sweep)

Sweep the documentation for drift against the *actual source*, independent of any recent change — the proactive, repo-wide complement to the PR-scoped check in Phase 7. Go through every documentation source of truth (the Phase 7 table) and verify each claim against the code, config, or commands it documents. **Verify against source, never memory.**

- Fix drift **in the docs**. If the *code* is what's wrong (the docs describe the intended, correct behavior), do **not** silently change code under a docs cycle — file an issue and leave it for an implementation cycle.
- Respect the Phase 3 scope ceiling. If drift is large, fix the highest-value subset this cycle and file an issue enumerating the rest.
- If a complete sweep finds **no** drift, say so explicitly and fall back to another work mode — an empty docs PR is not an outcome.

#### Stage B — Unit-test expansion (functionality confidence)

Add tests to under-covered, behavior-bearing code to lock in current correct behavior and create regression guards (RESEARCH.md §3). The goal is **confidence**, not a coverage percentage. If the project has no automated test suite, this stage does not apply — pick another work mode.

- Target, in order of preference: core/domain logic with **no** existing test, then entry points (command/handler/controller classes) whose logic is untested, then the remaining behavior-bearing code. Find candidates by comparing the source tree against the existing test files and looking for behavior-bearing units with no mirror.
- Follow the test framework, location, and mocking conventions already established in Phase 3 and in sibling tests (read 2–3 neighbors first). **No real network or database calls** — mock collaborators. Where logic runs inside a scheduled or async callback, capture and invoke that callback so the real logic is exercised, rather than only asserting it was scheduled.
- **Characterization, not change.** These tests must assert the code's *current* behavior. If writing one reveals an apparent bug, do **not** change production code under a test-expansion cycle — document the current behavior (or mark the test skipped with a reason), file a bug issue, and leave the fix to a separate cycle. Never weaken an existing assertion to make a new test pass.
- Scope one cohesive class/area per cycle, within the Phase 3 scope ceiling.

```bash
git checkout -b feature/<short-description>
```

**Plan summary (re-read at the start of Phase 3).** Before exiting Phase 2, write a compressed plan in this shape:
- **Work in scope:** the issues (`#N`, `#M`, …), or `Stage A — documentation accuracy sweep`, or `Stage B — unit-test expansion (<target class/area>)`.
- **Branch:** `feature/<name>`
- **Files I expect to modify:** path, path, ...
- **Invariants to preserve:** `npx tsc` exits 0 and `npm run test` is fully green; every mutable scene field is reset in `init()`; no test imports `phaser` or constructs a `Phaser.Scene`; no new `window.*` reach that could have been an injected argument; `README.md` and `AGENTS.md` still agree with the source `src/docs.test.ts` holds them to.

Phase 3 begins by re-reading this summary. The point is to ground the implementation in a tight statement rather than the full accumulated triage transcript — per RESEARCH.md §4, context rot degrades performance even when the window isn't full.

---

### Phase 3 — Implementation

**Localization verification.** Before writing any code, list the files this PR intends to modify and verify each one:

1. **Confirm the file exists.** `test -f <path>` or `ls <path>`.
2. **Confirm the surface area is present.** For each file, grep for the symbol, heading, config key, or behavior named in the issue. If the issue says "the `validatePermission` method swallows the exception", run `grep -n 'validatePermission' <path>` and confirm the named entity is present. If it isn't, stop and re-triage — the localization is wrong and editing here would produce a misfire.
3. **Confirm the issue's behavioral claims, not just the symbol's existence.** For each behavior the issue asserts ("X bypasses the permission check", "Y defaults to Z"), read the surrounding code path to the point where the behavior would actually be observable — for a permission node, that means reading past the branch that consults it to whatever check runs *next*. If the issue's description and the source disagree, **the source wins**: implement/document what the code does, and file a separate issue for the discrepancy. Never paraphrase an issue body into documentation without this confirmation.

This catches the dominant agent failure mode on uncontaminated benchmarks: finding the right file to edit, not the patch itself (RESEARCH.md §3). Step 3 closes a narrower gap in the same failure mode: an issue can pass its own filing-time verification ("this symbol exists and is checked here") while still misdescribing what happens *after* the check — cheap to verify existence, expensive to verify consequence.

Follow project conventions:

**Extract-then-test is the shape of most good work here.** The scenes cannot be tested and the plain modules can, so the recurring move is to lift the decision out of the scene (or out of the transport) into a module the node suite can reach, then pin it. `lobbyState.ts`, `runOutcome.ts`, `callBanner.ts`, `railLabels.ts` and `rpcResult.ts` all arrived that way. A change that can only be verified by clicking is usually a change that has not been extracted yet.

- **Scenes stay presentation-only.** New logic goes in a plain, framework-independent module (`rhythm/`, `run/`, `survival/`, `ui/levelSelection.ts`, `audio/guideAudio.ts` are the pattern) and is called from the scene.
- **A DOM dependency is injected, not mocked and not jsdom'd.** When logic is stranded in the node suite because it reaches for `window`, take the thing it reaches for as an argument with a browser default. This is the repo's settled answer, used three times: `IntervalScheduler` on `SupabaseRoomConnection` (its only DOM dependency, so removing it made the whole transport node-constructible), storage passed to `readStoredName`/`writeStoredName` in `multiplayer/playerIdentity.ts`, and `failOnError`/`resultData` split into `multiplayer/rpcResult.ts`. Callers keep their signatures, because the default is the browser's own.
- **Multiplayer/race behaviour goes behind `src/game/race/RaceAdapter.ts`.** If a change needs `RiverScene` edited to accommodate a specific backend, it is in the wrong place. `createRaceAdapter.ts` is the only place a mode maps to an implementation.
- **No literal coordinates.** Ask `src/game/ui/layout.ts` for a named region or point; add a rect/point there plus a `layout.test.ts` assertion that it stays on screen and, if interactive, clears `MIN_TOUCH_PX` (currently 44). The canvas is `Phaser.Scale.RESIZE` and sized to the viewport, so one game unit is one CSS pixel.
- **Handle re-layout.** Every object placed in `RiverScene` registers a placement closure via `onLayout()`; that array resets in `init()` like any other scene state. `MenuScene` restarts on resize and carries the chosen level through `init(data)`.
- **Never read `window.innerWidth`/`innerHeight`** — take size from `this.scale`.
- **Reset mutable scene state in `init()`, never with a class-field initializer.** Phaser constructs each scene once and reuses it; ask what a new field's value is on the *second* `create()`. This is the defect class behind issue #1.
- **Colours and text styles come from `src/game/ui/theme.ts`** (`COLORS`, `TEXT_COLORS`, `headingStyle()`, `bodyStyle()`), never inline literals.
- **The compiler is the linter.** There is no ESLint/Prettier/Biome. `verbatimModuleSyntax` means types must be imported with `import type`; `noUnusedLocals`/`noUnusedParameters` fail the build (prefix a deliberately-unused parameter with `_`); `erasableSyntaxOnly` rejects constructor parameter properties (declare the field, assign in the body); `strict` errors are fixed by narrowing, never with `!`, `as any`, or `@ts-expect-error`.
- **Style, matched by hand:** 2-space indent, single quotes, **no semicolons**, trailing commas in multi-line literals, numeric separators for large millisecond values (`38_000`, `2_200`), em dashes rather than `--` in prose comments.
- **Do not touch:** `public/audio/` (generated WAVs — regenerate with `npm run voice:generate`, never hand-edit), the `sharp` override in `package.json` (pinned for `kokoro-js`), `src/game/levels.ts` cue timing/difficulty (gameplay-feel decisions reserved to the repo owner; nearby defect fixes are fine).

Universal rules:
- **Match sibling structure.** Before creating a new file in a directory, read the section headers / structure of every existing file in the same directory and conform to the established pattern. Example: `grep "^##" path/to/dir/*.md` for docs, or read 2–3 neighboring source files for code.
- **Rename siblings together.** When renaming a heading or identifier that is part of a parallel pair or series (e.g. `Required X` / `Optional X`, `loadConfig` / `saveConfig`), scan for the siblings and rename them in the same commit.
- **Scratch-file handling in a sandboxed harness.** When a step needs a scratch file for inspection or transformation (not a project source file — e.g. redirecting `git show` output for byte-level inspection, or a throwaway helper script), prefer the `Write` tool over `> file` shell redirection, and prefer `python3 -c "import os; os.remove(path)"` over `rm` to clean it up afterward. Some harness sandboxes statically block plain `>` redirection and `rm` outright — even for files the same session just created inside the working directory — while `Write` and `os.remove` are not pattern-matched the same way. The same blocking applies to scratch **directory trees** (e.g. an isolated tool-home created to work around a lock-file issue): use `python3 -c "import shutil; shutil.rmtree(path)"` instead of `rm -rf`. Creating such a tree at all may be unavailable — some sandboxes block `mkdir` (and therefore `git clone` into a fresh subdirectory) even for paths *inside* the allowed working directory — so keep scratch work to individual files written into the existing tree rather than a new directory.
- **Name every scratch file uniquely per repo and per cycle.** Use `<repo>-<branch>-pr-body.md`, never a generic `pr-body.md` — the harness scratchpad is shared between concurrently running dev loops, so a colliding name is overwritten silently, with no error to notice and another repo's content left in the file. Write the file immediately before the command that consumes it, and treat it as consumed once used rather than as durable state; where a body has to survive a wait, re-verify what was actually published (`gh pr view <number> --json body`) instead of trusting the file.
- **Avoid command substitution in Bash tool calls.** Some harnesses' command classifiers reject `$(...)` outright, so a prescribed `--body "$(cat <<'EOF' ... EOF)"` or `git commit -m "$(cat <<'EOF' ... EOF)"` can fail before ever reaching the shell. Compose long bodies (PR comments, commit messages, issue bodies) with the `Write` tool to a scratch file and pass them by file instead — `--body-file` (`gh pr comment`, `gh issue create`) or `-F` (`git commit`) — as done in Phase 4 step 5 and Phase 6. Likewise prefer separate `grep` invocations over `\|` alternation, which some classifiers flag as an expansion.

Write or update tests for every change (and see Stage B in Phase 2 when the *whole cycle* is dedicated to expanding coverage of existing functionality):
- **vitest, node environment, no globals.** There is no `vite.config.ts` or `vitest.config.ts`, so vitest's defaults apply. Import everything explicitly: `import { describe, expect, it } from 'vitest'`.
- **Tests are colocated** with their source as `<Source>.test.ts`. There is no top-level `test/` tree.
- **What genuinely cannot be tested here:** anything importing `phaser` or constructing a `Phaser.Scene` — `MenuScene.ts`, `RiverScene.ts`, `LobbyScene.ts`, `startGame.ts`, `main.ts`. Everything else is reachable. Do not "fix" a DOM dependency with a `phaser` mock or a jsdom config; inject it (Phase 3).
- **Read 2–3 neighbours first.** House style: one behaviour per `it`, a message string as `expect`'s second argument when the failure needs to name what drifted, hand-written fakes as small classes (`FakeSession` in `SupabaseRaceAdapter.test.ts`, `FakeChannel` in `SupabaseRoomConnection.test.ts`) rather than a mocking library.
- **Prove a guard bites.** For a regression test, revert the fix and watch the new test fail before keeping it — a test that passes either way is not a guard. `docs.test.ts` gates are proved the same way, by injecting the drift they exist to catch.

Verify the build is clean:
```bash
npx tsc
npm run test
```

Fix all failures before proceeding. Never skip tests or bypass hooks.

**Confirm the anchor actually executed tests.** A zero exit status is not evidence that anything ran: `./gradlew test` prints `BUILD SUCCESSFUL` on `:test NO-SOURCE`, and the pytest/npm equivalents exit 0 on "no tests ran" or an empty match. Read the output for an executed-test count before recording a PASS. Where zero tests ran, the green verifies nothing and must never be reported as "tests pass" — say so plainly and fall back to whatever the `npm run test` substitution names as the real gate for this repo.

**Formatting is scoped to changed files.** Run any formatter against **only the files you changed** (e.g. `black <changed files>` / `autoflake --in-place <changed files>`), not a tree-wide script — a whole-repo reformat pulls unrelated files into the PR and violates the Scope rule below. If you do run a tree-wide formatter, only `git add` files in this PR's scope and `git checkout --` any unrelated files it touched. Pre-existing formatting drift lands as its own formatting-only sweep, never smuggled into a feature/fix PR. (Such scripts may not be executable in the checkout — invoke as `bash format.sh`.)

**Git-staging hygiene.** Stage by name — **never** `git add -A` or `git add .`. The harness writes `.claude/` state (e.g. `scheduled_tasks.lock`) into the tree while the loop runs, and the project `.gitignore` may not cover it; a blanket add leaks harness state into the project repo (and the classifier blocks the `git rm --cached` cleanup as scope-escalation). After staging, run `git status` and confirm no `.claude/` entries are staged before committing.

**Scope ceiling.** Before pushing, check the cumulative net diff for this cycle:
```bash
git diff --stat origin/main
```
Count the soft ceiling against **non-test net LOC**. If non-test changes exceed **~400 net LOC** or the PR modifies more than **~10 files**, stop and rescope: either drop one of the batched issues from the PR, or split the remaining work into a follow-up PR. **Exception:** if you are over the soft ceiling *only* because of (a) test code or (b) a dependency-coupled issue that cannot be split without leaving an unused component (e.g. an endpoint that requires its own hashing service), proceed but state the overage and the reason in the PR body and self-review. Hard stop unchanged at **~800 LOC** or **~20 files** — at that size, agent PRs fail to merge at substantially higher rates (RESEARCH.md §2).

**Implementation summary (re-read at the start of Phase 4).** Before pushing, write a compressed implementation summary:
- **Files actually modified:** path, path, ...
- **Commit summary:** one line per commit
- **Test/validation result:** PASS / FAIL — name the command that produced the verdict and how many tests it executed
- **Open carryovers:** anything in scope that wasn't done and why (becomes input for Phase 9)

---

### Phase 4 — PR

**Before pushing, verify each `Closes #N`.** For every issue number you plan to reference, run `gh issue view <N>` and confirm the title and body match what this PR does. Numbers carried forward from earlier session context or from summarized prior cycles are a common source of wrong-issue auto-closes — if a referenced issue describes unrelated work, omit the `Closes` reference and either file a new issue or note `No tracking issue — gap found during triage.` in the PR body.

```bash
git push -u origin feature/<short-description>
# compose the body with the Write tool to a uniquely-named scratch file first (Phase 3 scratch-file rule)
gh pr create --title "..." --body-file <scratch-file-path>
```

PR body must include:
- Summary bullet points (what changed and why)
- Test plan checklist
- `Closes #N` for each resolved issue

Request a review:
```bash
gh pr edit <number> --add-reviewer McElyea
```

If the command errors (reviewer not configured), proceed directly to the self-review below.

Perform a self-review. This step is anchored on external signals (CI, the rubric below) rather than free-form judgment — empirical findings show LLM self-critique without an external signal is unreliable and can regress quality (see RESEARCH.md §1, §5).

1. **Wait for CI to be green.** This is the external anchor for the rubric below; without it, the rubric is just opinion.

   ```bash
   gh pr checks <number> --watch
   ```

   If it fails, fix the underlying issue, push, and re-confirm. Do not start the rubric until CI passes.

   **A green anchor does not cover what it cannot execute.** `npx tsc` and `vitest` never render a frame: no scene lifecycle, no input handler, no audio playback, no `window` access, and no Supabase round trip is exercised by them. A change to `MenuScene`, `RiverScene`, `LobbyScene`, `startGame.ts`, anything under `public/audio/`, or any multiplayer path therefore needs a **manual browser pass** on top of the green anchor — `npm run dev`, then menu → select a level → start a run → back to the menu → start a second run, which is the path that exercises scene re-`create()`, plus a rotate if layout changed. State in the PR body whether that pass was performed or was not possible; in a sandbox with no browser it will not be, and saying so is the point. A green suite on a scene change is not evidence that the scene works.

   **If the anchor cannot run** because the tool/interpreter is absent or broken in the environment (docker not installed; bare `python` resolving to 2.7; a venv-less interpreter with no pytest; `./gradlew` dying on a sandbox file-lock; the session sandbox restricts filesystem/tool access to only the target repo's own working directory — common in gardener-dispatched sessions), do **not** claim it green and do **not** burn the cycle trying to fix the sandbox. Flag it **UNVERIFIED** and gate on scope: if the PR modifies files the anchor would have validated, the anchor is required — mark UNVERIFIED, do not auto-merge, and hand to CI/a human with the tool (prefer the CI check on the exact PR head SHA as the real anchor where local execution is blocked). If the PR touches none of those files (e.g. docs-only), record UNVERIFIED-not-applicable and continue, stating it in the self-review and PR body. **A path-restricted sandbox can block more than reads outside the allowed root:** creating a *new* directory can be blocked even *inside* the allowed working directory (`mkdir`, and therefore `git clone` into a fresh subdirectory), so "clone a fixture into a scratch subdirectory and run the anchor there" is not a reliable workaround — only writing individual files into the already-existing tree dependably succeeds. Take the UNVERIFIED gate rather than engineering around the sandbox. Capability-check any named interpreter before relying on it (e.g. `<py> -c "import pytest"`) and prefer the project's own venv.

   **Green CI is not verification when CI's scope excludes the changed files.** If this PR changes code the CI structurally cannot execute (platform-specific scripts — `.ps1`/`.bat`/`.command`, installer configs, OS-gated shell paths), a green run does **not** verify those files. State the no-automated-coverage gap in the PR body (and CHANGELOG), lean harder on adversarial hand-review of that code, and recommend a real-platform smoke test before merge — never let a green run on the *other* language imply the script was tested.

   **A structurally-red anchor is not a verdict on this PR.** Where Phase 1 recorded a required job that has never succeeded on `main`, do not read the aggregate red as one blocking signal — assess each failing job separately. A job carries **no signal** for this PR when all three hold: its failure reproduces identically on `main`, its cause is environmental (absent credentials, an unset secret, an unavailable license) rather than anything in the diff, and the diff touches nothing that job would have exercised. Quote the base-branch failure and the failing step to establish each. A no-signal job is not counted as blocking — but it is not verification either: score the CI rubric item from the jobs that *do* carry signal, mark the rest **UNVERIFIED** under the same scope gate as an anchor that cannot run, and state both in the self-review and PR body. A job red for any other reason — including one whose failure the diff could plausibly have caused — is an ordinary failure and blocks as usual.

2. **Read the full diff:**
   ```bash
   gh pr diff <number>
   ```

3. **Run the self-review rubric.** Score each item PASS or FAIL with a one-line justification grounded in the diff or a command output — not in judgment. Frame this adversarially: assume FAIL unless you have direct evidence of PASS. Treat an all-PASS result as suspicious; a reviewer expects you to find at least one issue.

   Universal rubric:
   - **Scope:** every file modified is necessary for one of the issues in `Closes #N` (no unrelated formatting, renames, or comment churn).
   - **Tests-new:** every new public method/function has at least one test that exercises it.
   - **Tests-fix (empirical, not judged):** for each bug fix, temporarily revert the fix, run the new/changed tests and confirm they **FAIL**, then restore the fix and confirm they **PASS**. By the time this item is reached the fix is normally already committed — Phase 3 commits it and Phase 4 has pushed the branch and opened the PR — so the working tree is clean and the **checkout form is the default**: `git checkout origin/main -- <src files>` to revert, `git checkout HEAD -- <src files>` to restore. Revert to the **merge base** (`git merge-base HEAD origin/main`) rather than the branch tip wherever `main` has moved on since the branch was cut, so the experiment isolates this PR's change instead of also pulling in whatever else landed meanwhile. Reach for `git stash push -- <src files>` / `git stash pop` only in the rarer case where the fix is genuinely still uncommitted. **Confirm the revert actually changed the tree** (`git status --porcelain`, or `git diff --stat`) before trusting either half of the result: `git stash push` on a clean tree saves nothing and reports success, the check then runs against the *unmodified* (fixed) tree and passes, and `git stash pop` fails with `No stash entries found` — scored naively that sequence reads as "reverted, still passed → false negative in the test" when in fact nothing was ever reverted. A regression test that still passes with the fix *genuinely* reverted is a real false-negative (common when the "after" state is indistinguishable from the "before") — use distinct/sentinel data so the failure is observable. Do not score this from reasoning alone. When the local anchor cannot run (see UNVERIFIED above), use the fallback ladder below instead of skipping this item.
   - **Sibling structure:** every new file matches the section/structure conventions of its directory siblings (Phase 3 rule).
   - **Sibling renames:** every renamed identifier in a parallel pair/series has its siblings renamed in the same commit (Phase 3 rule).
   - **Docs:** every row in the Phase 7 documentation sources-of-truth table reflects the new behavior.
   - **Issue resolution:** every `Closes #N` issue's named surface area is actually changed; no issue is partially resolved while claiming closure.
   - **CI:** the external anchor is green on the PR head (re-confirms step 1).

   **Tests-fix fallback ladder when the local anchor cannot run.** The revert-and-run experiment above needs a live local anchor. CI on the fixed tree alone cannot substitute for it — CI only ever runs the *fixed* state and can never reproduce the reverted half. Enter this ladder **only** when the tool/interpreter itself is unavailable or broken (the UNVERIFIED case above), so that no local technique can execute. A failure of one git mechanism is not that condition: if `git stash` refuses (a clean tree, a dirty tree, submodule state) but `git checkout` works — or the reverse — that *is* the experiment above via a different mechanism, so score Tests-fix from it directly and do not enter the ladder.
   1. **CI-based temporary revert.** Push a commit that reverts only the production fix while keeping the new/changed tests, with a commit message that makes recovery undoable by a future session without investigation, e.g. `TEMP: revert <fix-commit-sha> to prove regression tests fail — MUST be reverted before merge, see <verified-good-sha>`; confirm CI goes **red** naming exactly the new regression tests; then `git reset --hard` back to the verified fix commit and force-push, confirming CI goes **green** again. Strongest available substitute — costs two CI round-trips and leaves a real broken commit on the branch until the reset completes, so do not leave a session mid-ladder (see Phase 1's orphaned-PR handling). Requires an automated remote signal, so this rung is unavailable where CI names a manual checklist rather than CI.

      **Entry condition — do not use rung 1 under a headless or timeout-bounded dispatch.** Rung 1 is only safe when the session is certain to survive both round-trips. Where no human is present and the run may be killed at any point (`gardener tend` and equivalents — the same condition the autonomous-batch rule in Phase 5 keys on), a killed session leaves a pushed head that deliberately undoes its own fix, and nothing restores it until a later cycle happens to notice: precisely the stuck-mid-revert state Phase 1's orphaned-PR handling exists to repair. Skip rung 1 in that case and fall through to rung 2, or to rung 3's hand-off.
   2. **Pre-existing test changed by the fix.** If a test written before the fix asserted the old behavior and the fix's diff changes that test's assertion to the new behavior, that diff is itself a recorded FAIL→PASS — quote the test name and the changed assertion instead of re-running it.
   3. **Neither is available.** Score Tests-fix **FAIL** and do not auto-merge — hand to a human. A bug-fix PR whose regression evidence can't be established by either rung above does not clear the regression gate.
   
   Repo-specific rubric items:
   - **Scene state resets:** every scene field added or newly mutated in the diff is reassigned in that scene's `init()`, not only by a class-field initializer.
   - **Re-layout survives:** every object newly placed in `RiverScene` registers its placement through `onLayout()`.
   - **Theme, not literals:** no new colour number or text-style literal in the diff; both come from `src/game/ui/theme.ts`.
   - **No viewport globals:** `grep -n 'window\.' <changed files>` returns nothing outside the guarded `localStorage` helpers in `audio/guideAudio.ts` and `multiplayer/playerIdentity.ts`, and the browser defaults of injected dependencies; size reads come from `this.scale`.
   - **DOM dependencies injected:** any new `window.*` reach in a plain module is a constructor or parameter argument with a browser default, not a hard call.
   - **Type-only imports:** every type import uses `import type` — `verbatimModuleSyntax` makes the alternative a build error.
   - **No erased-syntax violations:** no constructor parameter properties and no `enum`/namespace syntax that `erasableSyntaxOnly` rejects.
   - **No strictness escapes:** no `!` non-null assertion, `as any`, or `@ts-expect-error` added; strict errors were fixed by narrowing.
   - **Node-environment safe tests:** no test file in the diff imports `phaser` or constructs a `Phaser.Scene`.
   - **RPC errors read:** every new `.rpc()` call passes its result through `failOnError` or `resultData` from `multiplayer/rpcResult.ts` — `supabase-js` resolves rather than rejects on failure.
   - **Citations re-verified:** every citation this PR adds to or leaves in `AGENTS.md` names a file that still contains a symbol its sentence names. Citations carry no line numbers — do not add any back.
   - **Style match:** no semicolons, single quotes, 2-space indent, trailing commas in multi-line literals, numeric separators on large ms values.
   - **Generated and pinned files untouched:** no change under `public/audio/`, no edit to the `sharp` override, no cue-timing change in `src/game/levels.ts`.
   - **Lockfile honesty:** `package-lock.json` changes only if `package.json` dependencies changed in the same PR.
   
4. **For each FAIL item:**
   - If the fix is mechanical and small, fix it locally, commit, push, and re-score the failed item. Do not re-score items that previously passed.
   - If the item requires judgment (e.g. "is this scope creep?"), leave it as an inline review comment for Phase 6.

5. **Post the self-review as a plain PR comment** — not a formal review API call. The auto-mode classifier blocks `POST /pulls/<n>/reviews` (it misrepresents independent review); a plain comment keeps the audit trail without implying reviewer independence. Compose the body with the `Write` tool to a scratch file and pass it by file — **avoid command substitution** (`--body "$(cat <<'EOF' ... EOF)"`), which some harnesses' command classifiers reject outright:
   ```bash
   # write the body (below) to a scratch file with the Write tool first, named uniquely per the
   # Phase 3 scratch-file rule, e.g. <repo-root>/.self-review-<branch>.md:
   #   Self-review rubric:
   #   - Scope: PASS — <justification>
   #   - Tests-new: PASS — <justification>
   #   - Tests-fix: PASS — checkout-revert-and-run confirmed FAIL→PASS
   #   - ...
   #
   #   <one-line summary; fold any out-of-diff observations into this body>
   gh pr comment <number> --body-file <scratch-file-path>
   ```
   Remove the scratch file afterward per the scratch-file-handling rule above.

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
After 5 wakeups (~22 min) with no review, proceed anyway.

**Autonomous multi-cycle batch mode** (`/forward-one-dev-loop until you run out of issues` / `do N cycles`): there is no human reviewer between back-to-back cycles, so the ~22 min poll is pure latency. Treat the self-review rubric + green CI as the merge gate and **skip (or cap at one short poll)** this Phase-5 wait — except for do-not-auto-merge / charter-gated PRs, which still hand off for human approval. **Stop condition:** end the batch when the only remaining issues are blocked, charter-gated, or too large for a polish-sized PR, or when a cycle yields no appropriately-scoped work.

---

### Phase 6 — Address comments

**External vs. internal signals.** Comments from a real reviewer and CI failures are *external* signals — keep iterating on them until each is resolved. The self-review rubric posted in Phase 4 is *internal* — once its comments are addressed here, do not re-run the rubric. Per RESEARCH.md §1 and §5, repeated intrinsic critique without an external signal is neutral-to-harmful.

For each comment:
1. Read the comment. Read the referenced source before changing anything.
2. If correct, fix the code.
3. If it conflicts with `AGENTS.md`, the issue spec, or a project config file,
   reply with the evidence and do not apply the change.

Compose the commit message with the `Write` tool to a scratch file so the trailer lands on its own line, then commit with `-F` — **avoid command substitution** (`git commit -m "$(cat <<'EOF' ... EOF)"`), which some harnesses' command classifiers reject outright. Stage by name (never `git add -A`). **Do not hardcode the co-author model name** — defer to the harness's standing git rule, which appends the *actual running model* (a hardcoded name misattributes the commit when a different model version is running):
```bash
git add <files>
# write the commit message (fix description + blank line + Co-Authored-By trailer) to a scratch file via the Write tool first
git commit -F <scratch-file-path>
git push
```
Remove the scratch file afterward per the scratch-file-handling rule in Phase 3.

Run `npx tsc && npm run test` after every fix. Do not push a broken build.

---

### Phase 7 — Documentation accuracy check

This phase is **PR-scoped**: it verifies the docs against *this PR's* implementation. The proactive, repo-wide version (drift unrelated to the current change) is a selectable cycle work-mode — Stage A in Phase 2.

**Read the implementation first**, then check each doc against it.

| `README.md` — "Commands" | Every `npm run <script>` listed exists in `package.json` and still does what the line claims (enforced by `docs.test.ts`). |
| `README.md` — controls and survival numbers | The keys named match the `keydown-*` handlers in `RiverScene.ts`; the ejection, recovery and swept-away counts match `MAX_STABILITY` / `RECOVERY_CALLS` / `MAX_DRIFT` in `survival/SurvivalEngine.ts` (enforced by `docs.test.ts`). |
| `README.md` — guide voices | The names listed and the stated default match `GUIDE_VOICES` and `DEFAULT_GUIDE_VOICE_ID` in `audio/guideAudio.ts` (enforced by `docs.test.ts`). |
| `README.md` — "Project structure" | Every listed `.ts` path exists (enforced by `docs.test.ts`), and any module this PR added that a newcomer would look for is listed — completeness is *not* enforced. |
| `README.md` — Node floor and container port | Match `engines.node`, `ci.yml`'s `node-version`, the `Dockerfile` `FROM`/`EXPOSE`/`HEALTHCHECK` and `docker/nginx.conf` (all enforced by `docs.test.ts`). |
| `AGENTS.md` | Every claim and every citation this PR could have invalidated. Citations name a **file, not a line** — `docs.test.ts` checks the file exists and still contains a whole word the citing sentence names, which is far short of the claim being true. The rest is yours to read. |
| `.env.example` | Every `VITE_*` key is read somewhere under `src/`, and every key the code reads is listed. |
| `supabase/migrations/*.sql` | Any level-id or capacity literal still agrees with `src/game/levels.ts` and `multiplayer/roomPolicy.ts`; the level-id agreement is enforced by `docs.test.ts`. |
| `package.json` | `scripts` reflect any command this PR added or renamed; `engines.node` unchanged unless deliberately raised everywhere. |
| `src/docs.test.ts` | Its parsing assumptions still hold against the current `README.md` structure — if this PR restructured a heading or fenced block it parses, its error messages must still name what drifted rather than throwing a format error. |

If any inaccuracy is found: fix it, commit, and restart this phase from the top.
Only proceed when a complete pass finds nothing wrong.

---

### Phase 8 — Merge

**Regression gate.** For each issue in `Closes #N` that is a bug fix or describes incorrect behavior, verify the diff includes a new or modified test (or, for projects whose external anchor is manual validation, a new validation step) that exercises the fix — and that it was confirmed empirically (the Phase 4 revert-and-run, or its fallback ladder when the local anchor cannot run: FAIL/RED with the fix reverted, PASS/GREEN with it restored), not by reasoning alone. If absent, do not merge — either add the regression coverage or reclassify the issue. Per RESEARCH.md §3, regression evidence is the only way to distinguish a real fix from a coincidental patch.

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
- `AGENTS.md` — agent-loaded config; edits need explicit user authorization, so they cannot also be merged autonomously
- `public/audio/**` — generated WAVs; regeneration means an 82M model download, not a change reviewable by diff
- `scripts/generate-guide-voice.mjs` — edits here silently change every future regenerated clip
- `src/game/levels.ts` — cue timing and level data are gameplay-feel decisions belonging to the repo owner
- `supabase/migrations/**` — a migration is replayed by hand against a live project and cannot be rolled back by reverting the commit
- `LICENSE` — MIT terms

If no modified path matches, proceed:

```bash
gh pr merge <number> --squash --delete-branch
git checkout main && git pull
```

**Review-ready hand-off is a valid terminal state — not a failed cycle.** On a repo with no autonomous merge path (branch-protected `main` requiring an approving review, and/or repository auto-merge disabled), a clean, anchor-green PR's true terminal state is *open + self-review posted + awaiting human approval*. Expect `gh pr merge` → `the base branch policy prohibits the merge` and `--auto` → `Auto merge is not allowed for this repository`. **Never use `--admin`** to bypass a deliberately-configured human gate, and a later self-audit must not score "could not auto-merge" as a failure. A do-not-auto-merge path match blocks **autonomous** merge only: if the human codeowner explicitly authorizes the merge after being shown which protected path matched, that authorization satisfies the hold — proceed (still run the Phase 7 docs sweep + the regression gate above first), and state in the merge report which protected path was overridden and by whose authorization.

**Standing merge authorization does not satisfy the anchor gate.** A run-level merge pre-authorization granted before this PR existed (e.g. a headless dispatch launched with a blanket "may merge" flag) conveys *permission*, not *validation* — where Phase 4 has marked the PR UNVERIFIED because the anchor could not run on anchor-relevant files, the hand-off stands regardless of merge authority. The codeowner exception above is retrospective by construction: it applies only to authorization given *after* the specific protected path has been surfaced to the codeowner, so a flag passed before the PR was written does not satisfy it. The terminal state in that case is open + self-review posted + awaiting a human, which the paragraph above already declares valid rather than failed.

**A structurally-red anchor is a named blocking condition, not an ordinary hand-off.** Where Phase 1 recorded a required workflow that has never succeeded on `main`, the merge gate cannot be satisfied by any change this loop could write, so the hand-off must say that rather than reporting "awaiting review" — the two have entirely different remedies. State which jobs were assessed as carrying no signal for this diff and on what grounds, which jobs carried signal and were green, and the issue tracking the structural failure (file one on the project repo if none exists, and link it). The point is that the owner sees the single action that would unblock every future cycle, instead of a hand-off that reads like a queue.

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
   - **External-signal quality:** did `CI` actually catch the kinds of issues it's supposed to catch this cycle?

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
**The external anchor cannot run (tool/interpreter absent or broken in the sandbox, or the sandbox restricts filesystem/tool access to only the target repo's own working directory — common in gardener-dispatched sessions, and possibly extending to blocking creation of a *new* directory inside that allowed root, which rules out cloning a fixture locally as a workaround):** do not claim it green and do not iterate on the sandbox. Flag **UNVERIFIED** and gate on scope (Phase 4): anchor-relevant files changed → mark UNVERIFIED + do-not-auto-merge + hand to CI/human (prefer the CI check on the exact head SHA); no anchor-relevant files (docs-only) → record UNVERIFIED-not-applicable and continue, stating it in the PR body.
**CI has never been green on `main` (structurally red — e.g. a required job fails on absent credentials or an unset secret):** this is not the same as red-on-this-branch, and no change the loop can write will satisfy the gate. Record it at triage (Phase 1), assess per-job signal in Phase 4 so a job carrying no signal for the diff is neither counted as blocking nor mistaken for verification, and hand off in Phase 8 naming the blocking condition and the issue tracking it rather than reporting an ordinary "awaiting review".
**An issue requires a harness-blocked or owner-gated operation (editing `AGENTS.md`, regenerating `public/audio/`, adding a jsdom config and dependency, retuning `levels.ts` difficulty):** recognize it at triage (Phase 1) — surface to the user for explicit authorization rather than attempting it mid-cycle; never remove a path while `AGENTS.md` or `README.md` still references it.
**A change only shows up when the game renders:** `npx tsc` and `vitest` never render a frame. Do the manual pass — `npm run dev`, then menu → start a run → back to the menu → start a second run, plus a rotate if layout changed — and record the result in the PR body. Where the environment has no browser, say so explicitly rather than letting a green suite imply a played game.
**A test fails at import with an error about `phaser`, `window`, or `document`:** vitest runs in the node environment. The fix is never a `phaser` mock and rarely jsdom — extract the logic into a plain module, or inject the DOM dependency with a browser default (Phase 3). What is genuinely out of reach is the `Phaser.Scene` subclasses themselves.
**An `AGENTS.md` citation no longer points at what it claims:** the code is right and the file is a bug (its own words). Citations name a file, not a line — fix the citation in the same PR when the PR caused the drift; file a docs issue when it was already stale. Do not reintroduce line numbers.
**`src/docs.test.ts` throws "the format it is parsed from has changed":** the `README.md` structure it parses was restructured, not merely edited. Restore the parseable shape or update the parser deliberately in the same PR — do not delete the assertion to make the suite green.
**`node_modules` is absent or `package-lock.json` moved:** run `npm ci` before the anchor. A stale `node_modules` can let `npm run test` pass while `npx tsc` fails on a dependency the tree already declares — the suite does not import every module the compiler checks.
**The resolved clone is shallow (`git rev-parse --is-shallow-repository` → `true`):** `git fetch --unshallow` before rebasing or reading history; a shallow tree makes `git rebase origin/main` and `git log` misleading.
**The loaded skill may itself be stale:** this file is read from a local clone that nothing pulls. Before reporting the skill as out of date, `git -C ~/local-skills/forward-one-dev-loop fetch -q origin && git -C ~/local-skills/forward-one-dev-loop status -sb | head -1`. A checkout `behind` origin reads exactly like a stale skill from inside a session.
**Autonomous multi-cycle batch (`/forward-one-dev-loop until …`):** skip/cap the Phase-5 human-review wait (rubric + green anchor is the gate); still hand off do-not-auto-merge/charter PRs. Stop when only blocked, charter-gated, or too-large work remains, or a cycle yields no scoped work.
**A concurrent session holds the tree or an open PR:** adopt its PR (bring current with `main`, re-run the anchor, review, merge if green) rather than doing nothing; work in a `git worktree` to avoid colliding, and treat a harmless local `--delete-branch` failure as success once `gh pr view --json state` confirms the merge. **Put the worktree inside the repository checkout** (e.g. `.worktrees/<branch>`, added to `.git/info/exclude` so an untracked directory doesn't pollute `git status`), never in `/tmp` or anywhere else outside it: a path-restricted sandbox refuses every read and edit outside the checkout, so a worktree placed there locks the session out of its own working tree one file operation at a time. **In a headless dispatch (e.g. `gardener tend`), do not create a worktree at all** — the run already owns a clone dedicated to it, so there is no concurrent session to collide with, and the checkout can be used directly. The harness scratchpad is a third resource sibling loops contend for, alongside the shared tree and the shared `origin/main`, and unlike those two it fails silently: a scratch file named generically is overwritten by the other loop with no error, so name scratch files uniquely per repo and per cycle (Phase 3) and re-verify anything published from one.
**A test fails intermittently (suspected flake):** re-run the test command once. If the same test fails again, treat it as a real failure and investigate. If it passes on the second run, note the flake in the PR body and proceed — do not suppress or `@Ignore` a test without understanding why it is flaky.
**A review comment is a false positive:** reply with evidence, do not apply the change.
**A `gh` command fails with a transient network/transport error (`http2: client conn could not be established`, `unexpected EOF`):** retry once or twice before treating it as blocked — do not treat the first failure as terminal and silently drop a self-review, commit, or comment.
**`gh pr create` fails with "you must first push the current branch to a remote" despite a successful `git push -u`:** the clone's fetch refspec may be restricted to the default branch only (common in gardener-managed or otherwise restricted dedicated checkouts) — confirm with `git config --get-all remote.origin.fetch`. A restricted refspec means `git fetch` never populates a remote-tracking ref for feature branches, so `gh pr create`'s and `git branch --set-upstream-to`'s auto-detection both fail even though the push and upstream config succeeded. Pass `--head <branch>` explicitly to `gh pr create` to bypass the remote-tracking-ref lookup rather than retrying the push.
**Reviewer addition fails:** proceed to the self-review step in Phase 4. Do not skip to Phase 5 without posting self-review comments — Phase 6 addresses those comments like any other review.
**Branch is behind main or has a merge conflict at Phase 8:** rebase onto main, re-run the **same verification set Phase 3 runs** — not just the test command — and force-push before retrying the merge:
```bash
git fetch origin
git rebase origin/main
npx tsc
npm run test
git ls-remote origin refs/heads/<branch>   # full 40-char SHA for the lease below
git push origin <branch>:<branch> --force-with-lease=<branch>:<full-40-char-sha>
```
**Verify with everything the required checks run, not a subset.** A force-push commits to the result, so a repository whose required checks also run a formatter, linter or static analysis will otherwise go red on a task the test command never covered, minutes later, from CI. Where the repository's required checks exceed the Phase 3 verification set, run those too rather than trusting the subset.

**`(stale info)` has three unrelated causes; only one justifies backing off.** A bare `--force-with-lease` compares against a maintained remote-tracking ref, which a restricted clone does not have (fetch refspec covering `main` only — see the `gh pr create` entry above); the push is rejected as `! [rejected] <branch> -> <branch> (stale info)`. The explicit `--force-with-lease=<branch>:<sha>` form fixes that, but the expected value is compared against the remote ref literally and abbreviations are never expanded — so a short SHA copied from `git log --oneline` is rejected with the *same* message. A restricted fetch refspec, an abbreviated SHA, and a genuine concurrent push therefore all read identically. Do not treat `(stale info)` as "another session pushed" until the expected SHA has been confirmed **full 40-character** and re-read from the remote at push time (`git ls-remote`; `git rev-parse origin/<branch>` also yields the full form but only reflects the last fetch, so it is the weaker source where the lease is meant to be meaningful). Never fall back to a plain `--force`, which clobbers unseen work in exactly the one case that warranted stopping.

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
