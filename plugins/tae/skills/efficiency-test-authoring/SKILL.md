---
name: efficiency-test-authoring
description: >
  Author or convert Fenix Android UI tests onto the ui/efficiency framework. Use this whenever the
  task involves writing a new efficiency UI test, converting/porting a legacy ui/ (robot-based) test
  onto the efficiency framework, or building what a test needs — page objects, selectors,
  navigation-graph nodes/edges, or BasePage primitives. Trigger even when the user doesn't name the
  framework: "convert this smoke test", "add a page object for the bookmarks screen", "wire up
  navigation to Settings", "write an efficiency test for X" all qualify. Do NOT use for edits to
  legacy robot-based tests that are staying on the old framework, or for non-Android test work.
---

# Efficiency test authoring

This skill turns "I need this UI test on the efficiency framework" into a reliable, repeatable
build. Its job is to (1) figure out which of three build modes a test needs and (2) route you to
the right rule-set for each missing piece. The detailed rules live **in-tree** — read only the
one(s) you need, when you need them.

Framework root in the repo (called `<eff>` below):
`mobile/android/fenix/app/src/androidTest/java/org/mozilla/fenix/ui/efficiency/`

Every `docs/…` path in this skill is relative to `<eff>`. Those docs are the source
of truth — read them from the tree rather than relying on a summary here, since the
monorepo moves under you.

The host-side `eff*` scripts this skill drives (`effnext`, `effscaffold`, `effcheck`,
`effbuild`, `effverify`, `effloop`, the `effwatch` bridge) live in the
**testops-tools** repo under `tae-conversion/tools/`. The device-side dump tools
(`effview`, `effpretty`) ship in-tree at `<eff>/devtools/`.

## Status: this is a v0 guide we are dogfooding (read this)

This skill is refined as we go — it is **not a hard rule set yet**. Treat it as the **live** guide
and follow it, but when we agree on something that isn't aligned with what's written here, **stop and
flag the mismatch, discuss it, then update this skill (and the reference/logs) at that time.** New
lessons land first in the running logs and get folded up into this skill:
- `<eff>/docs/gotchas.md` — harness bug catalog + authoring/review checklist (A* bugs, B* checks),
  distilled and landed in-tree.
- `testops-tools/tae-conversion/docs/HARNESS-GOTCHAS.md` — the running catalog, ahead of the in-tree
  copy while entries are still being confirmed.
- `testops-tools/tae-conversion/docs/CONVERSION-LESSONS.md` — assumption→reality→rule, tagged by
  whether a tool enforces it.
Check those as if they were part of this skill; they are the source the skill distills.

## Three build modes (cheapest first)

Every test is one of these. Picking the cheapest workable mode is the point.

1. **Factory-generated** — a factory emits the test from metadata; you write nothing. Today only the
   Reachability factory is production-ready (covers "does this page open" for pages already in the
   graph). Prefer this for pure page-open checks.
2. **Hand-composed** — write the test by composing existing building blocks (`BasePage` `moz*`
   verbs, page objects, selectors, nav nodes). Default for real smoke tests; usually ~5–20 lines.
   **Reuse first:** a lot of capability already exists (custom-tab launch, trust-panel state verify,
   recently-closed screen, web-form submit + save-login prompt, settings→Home back-edge, screen-dump
   dev tools). Check before building.
3. **Hand-composed + extend the harness** — a needed block is missing, so add it first (page object /
   selectors / nav node / primitive), then compose. Extending is the exception — add the smallest
   general thing; every extension is reusable by later tests.

## The feedback loop (run this per test)

Work the gates in order. At each gate you either compose with what exists or add the missing block,
then continue. Navigation is the spine — resolve reachability first. Lean on the tools (below); they
make each gate faster and safer than doing it by hand.

0. **Pick the next test (local, no network).** Run `effnext --json` — next candidate(s) from the local
   prioritized pool minus what's done, minus skips, minus anything whose method already exists in the
   efficiency tests package. **Never call the Google Sheet to choose** — it's slow and the local pool is the
   working queue. (The Sheet is systems-of-record for status, not the per-test picker.) If the pick isn't one
   to take now — too complex for whoever is picking it up, blocked on a harness gap, deliberately deferred —
   record that rather than stepping over it: `effnext --skip Class.method --reason "…"` parks it (reversible
   with `--unskip`; it never marks the test converted) and prints the new next pick.
1. **Scaffold + extract intent.** Run `effscaffold <Class.method> --json` first — it pulls the legacy
   body, TestRail id, whether an efficiency test of that name already exists (don't re-convert!), the
   robots + their selector lines, and which screens are already modeled. From that, write the
   template: entry state, target page(s), ordered steps, assertions. Keep the *what*; drop legacy DSL.
2. **Navigation gate.** Can you route to each target page? Before choosing selectors, **discover the
   real handles** an element exposes — never trust a stubbed locator. → `docs/guides/discovering-selectors.md`
   (uses `effdump`). If a page object, its selectors, or a nav edge is missing, build it. →
   `docs/guides/adding-navigation.md`, `docs/guides/creating-a-page-object.md`, `docs/guides/authoring-selectors.md`.
3. **Interaction gate.** Expressible with existing `moz*` verbs? Reuse first. If not, add a primitive
   or page-object helper — new verbs go through `resolve()` and keep its guarantees (exception-safe
   presence, preserve per-strategy Compose tree). → `docs/guides/extending-basepage.md`.
4. **Assertion gate.** Verifications expressible (`mozVerify*` family)? If not, add a verify primitive.
   → `docs/guides/extending-basepage.md`. Two traps worth knowing before you assert on anything you also
   click: a disabled Compose button still *accepts* the click gesture and silently skips `onClick`, so
   "clicked" in the report does not mean the app acted; and an enabled-check against a `COMPOSE_BY_TEXT`
   selector is a no-op, because it resolves the text node inside the button (which reports enabled while the
   button is disabled) — use `COMPOSE_BY_TEXT_MERGED` for anything you act on. Prefer a positive assertion
   over waiting for something to disappear: absence cannot tell "it worked" from "the click was dropped".
   See HARNESS-GOTCHAS A16/A17.
5. **Static pre-flight.** Run `effcheck … --json` before spending a device build — it catches string/id
   resolution, empty nav paths (gotcha B1), inline selectors (B2), missing BasePage verbs, and
   test-class boilerplate (MWS/IMP). Fix everything it flags first.
6. **Write + run + verify — JSON verdict only.** Compose the test → `docs/guides/writing-a-test.md`. Build+run
   it in isolation via the bridge, then read **only** `effbuild --json` (compile verdict + error lines) and
   `effverify … --json` (the named test ran, was NOT skipped, `failed_total`=0, and `clean`=true i.e. not a
   retry-pass). **Do NOT `cat` `run-report.txt` / `raw-run.log`, and do NOT read `effpretty` output** — on a
   failure, `effverify --json` now carries a capped `failure_excerpt` (exception + top frames) which is all you
   need. `effpretty` is for a human eyeballing a run, not for the agent. "green + 0 failed" alone is NOT proof;
   a retry-pass (`clean`=false) is flaky, not done. → `docs/guides/debugging-tests.md`.
   On a failed step the auto-dump now covers all three layers plus a **window/focus summary** — read the
   `[windows]` block first to tell "covered by an overlay / focus stolen" from "element genuinely absent"
   before touching a selector. Blocking overlays are auto-dismissed via `OverlayRegistry`; add new ones
   there rather than handling them in a test.
7. **Parity + close-out.** Diff the legacy body against the port and list every legacy `verify*` — in the
   test **and in the robot helper it delegates to**, where retry/refresh semantics hide. Each one needs an
   explicit counterpart or a documented gap. Green is not evidence of parity: a dropped assertion doesn't
   fail, it passes for the wrong reason, and what gets dropped is usually the *payload* check
   (`verifyPageContent`, `verifyUrl`, a tab count) rather than the navigation. Never justify an omission
   with "`navigateToPage` already checks it" — an implicit assertion can't be audited. If you omit a leg
   (e.g. no stateful return edge), **log it as a harness gap** in the test and the commit message — don't
   silently drop it. THEN annotate the legacy test with `@Converted`. The gate is **green locally** (gate 6's
   `effverify` verdict); annotate it **in the same commit as the conversion** — never defer this to a
   post-landing pass, which would need a second bug and a second review:
   ```kotlin
   @Converted(
       replacedBy = ["org.mozilla.fenix.ui.efficiency.tests.AutofillTest#verifyAddressAutofillTest"],
       bug = 2057958,
       since = "2026-07",
       notes = "Legacy also asserted X; not carried over — see bug NNNNN.",
   )
   ```
   `replacedBy` is required and every entry must resolve to a real, non-`@Ignore`d `@Test` (validated by
   the conversion lint check). Use `notes` for coverage that intentionally did not carry over — that's the
   mechanism the parity rule above asks for. Annotate the legacy method **in place** — do not delete or
   `@Ignore` it; it keeps running alongside the replacement, and the annotation is what the burndown counts.
   Check the annotation is actually in your staged diff before committing: a conversion that lands without
   it looks unconverted to the ledger, and this is the single most-missed step in the loop.
8. **Land it.** Hand off to the **efficiency-conversion-loop** skill for bug → commit → Jira → submit.
9. **Feedback.** Recurring shape (nav→click→verify) → flag as a factory candidate. Every new
   assumption-correction → add to `CONVERSION-LESSONS.md`; if it changes the workflow, update this skill.

## Tools (fast + safe; prefer `--json` when driving them programmatically)

| Tool | Use at | Does |
|---|---|---|
| `effnext` | gate 0 | Next candidate(s): pool minus done, minus skips, minus what's already in-tree. `--skip`/`--unskip`/`--skips`. Local-only, no network. `--json`. |
| `effscaffold` | gate 1 | Legacy body, TestRail, already-converted check, robots+selectors, existing coverage. |
| `effdump` / `ScreenDump` | gate 2 | Dumps a screen's real handles in all 3 layers (Compose / Espresso / UIAutomator). Author from ground truth, not stubs. |
| `effcheck` | gate 5 | Static pre-flight (no device) — resolution, nav, inline selectors, verbs, boilerplate. |
| `effbuild` | gate 6 | Compile verdict + only the error lines. `--json`. Read this, not the raw build log. |
| `effverify` | gate 6 | Done-gate (aggregates **every** run block, not just the last): `ok`/`clean`, `failed_total`, `runs`, `retried`, per-test `status` incl. `retry-pass`, and a capped `failure_excerpt` on failure. `--json` — the agent reads THIS, never the raw report. |
| `effpretty` | (human) | Renders the `Eff` run log for a **person** inspecting a run. Not part of the agent's read path. |
| effwatch bridge | gate 6 | Runs the build/run on the engineer's device and returns reports. |

Gates **3 (interaction)** and **4 (assertion)** have no tool — they're code edits (add a `moz*` verb or a
`mozVerify*` primitive) via `docs/guides/extending-basepage.md`. `effwatch` is a persistent bridge you start once,
not a per-step tool; `effscaffold`/`effcheck`/`effbuild`/`effverify` may each run several times per test.

## Reference rule-sets

| When you need to… | Read |
|---|---|
| Find an element's real handles before choosing a strategy (`effdump`, stubs, web ids, state-invariance) | `docs/guides/discovering-selectors.md` |
| Reach/route to a screen; add graph nodes/edges; onboarding/launch-flag entries | `docs/guides/adding-navigation.md` |
| Model a new screen | `docs/guides/creating-a-page-object.md` |
| Add element locators + their groups | `docs/guides/authoring-selectors.md` |
| Compose the actual test method | `docs/guides/writing-a-test.md` |
| Add a `moz*` primitive or page-object helper | `docs/guides/extending-basepage.md` |
| Run and debug a test (effpretty, ScreenDump, retry-masking, SKIPPED trap) | `docs/guides/debugging-tests.md` |

## Guardrails

- Tests describe only the *what*; the harness owns the *how* (navigation, retries, selectors).
- Reuse an existing capability before adding one; add the smallest general block, not a test-specific hack.
- Selector priority: Compose `testTag` → resource id → content-description → text (last resort). Verify
  handles against the live app UI (via `effdump`), not from how a legacy robot matched.
- Verify every claim against the live repo; it's a syncing monorepo and state shifts between runs.
- This skill is a soft v0 — when reality and the skill disagree, flag it, discuss, then update the skill.
- You can draft skill content but can't install it from a Cowork session — install via Settings → Capabilities.
