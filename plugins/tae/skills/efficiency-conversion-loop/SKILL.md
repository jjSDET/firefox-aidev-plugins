---
name: efficiency-conversion-loop
description: >
  Run the end-to-end loop for landing a legacy Fenix UI test converted onto the ui/efficiency
  framework: convert → file a Bugzilla bug → commit with the real bug number → track in Jira
  (conversion vs. enablement) → open a moz-phab review. Use this when a conversion is written and
  needs to become a filed bug, a properly-numbered commit, Jira tracking, and a submitted revision —
  i.e. the "paperwork + submit" workflow around a ui/efficiency conversion, not the test authoring
  itself (that's the efficiency-test-authoring skill). Also covers keeping the tracker/dashboards in
  sync after landing.
---

# ui/efficiency conversion loop

This skill drives the workflow *around* a conversion once the test itself is written. Test authoring
is a separate skill (`efficiency-test-authoring`). The five steps:

**convert → file bug (blocking meta 2030727) → annotate `@Converted` + commit (with bug #) → track in Jira
→ submit for review**

## What runs where

- **The agent (sandbox)** does the conversion, static checks, and drives the bridge; it creates Jira
  items with `tools/jiratool.py` (preferred — it works headless), falling back to the Atlassian
  connector only if that tool is unavailable. It **cannot** reach Bugzilla or Phabricator directly.
- **The host bridge (`effwatch`)** runs the git + Bugzilla actions on the engineer's machine and returns
  results. It never pushes and never submits.
- **The engineer** runs `effwatch`, keeps a device attached, and runs the final `moz-phab submit`.

The eff\* tools live in the **testops-tools** repo under `tae-conversion/tools/` (see
`tae-conversion/README.md` for setup). Clone that repo and start `effwatch` before using this loop.

## Prerequisites (one-time)
1. `effwatch` running on your machine (from `testops-tools/tae-conversion/tools/`), device/emulator attached.
2. A Bugzilla API key available to effbug — set it once in `~/.zshenv` (`export BUGZILLA_API_KEY=…`) or a
   `tae-conversion/tools/.eff.env` file (gitignored). Never paste it into chat.
3. `moz-phab` installed and authenticated to phabricator.services.mozilla.com.

## Project-specific IDs — edit these for your team

This skill is written for the Fenix smoke-conversion campaign. If you're running it elsewhere, these are
the only values you need to change:

| Value | Current | What it's for |
|---|---|---|
| Template bug | `2057054` | Cloned for product/component/version → Firefox for Android :: UI Tests |
| Tracking meta bug | `2030727` | Every *conversion* bug must block it (`"blocks": [2030727]`) |
| Reviewers | `isabel_rios`, `aaronmt` | Default Phabricator reviewers (`EFF_REVIEWERS`) |
| Conversion parent | `MTE-5731` | Story that conversion sub-tasks hang off |
| Enablement parent | `MTE-5715` (or `MTE-5688`) | Harness-hardening / tech-debt story |
| Jira cloud | `mozilla-hub.atlassian.net` | Atlassian connector target |

## The loop

### 1. Convert + pre-flight
Convert the legacy test onto ui/efficiency (see efficiency-test-authoring). Run `effcheck.py` (static
pre-flight) then build/run via the bridge (`effverify` / `effwatch`) until green, or "good enough + notes."

Before you file anything: run the **parity audit** from the `tae-test-review` skill. A conversion that
dropped a legacy assertion still goes green, so a passing run is not the gate — the legacy-vs-port diff is.
Land-blocking findings are cheaper to fix now than after the bug and commit exist.

### 2. File the Bugzilla bug (agent → bridge → `effbug`)
Drop `conversion-runs/_queue/<id>.request.json`:
```json
{ "bug": "create",
  "summary": "[efficiency] Convert <Test>.<method> to ui/efficiency",
  "comment": "<what was ported; parity notes>",
  "why": "<one-line rationale>",
  "kind": "conversion",            // conversion | enablement | tooling — picks the description footer
  "testrail": "<id(s)>",
  "template_bug": "2057054",        // clones product/component/version → Firefox for Android :: UI Tests
  "blocks": [2030727],              // REQUIRED for conversions — the tracking meta bug (see below)
  "type": "task" }
```
`effbug` files the bug, then **rewords the title to `Bug NNNNN - <summary>`** so it matches the commit
subject exactly, and **self-assigns** to the API-key owner. It returns the number in
`_bug/<id>.bug-result.json`.

**Hang the bug off the tracking meta bug — `"blocks": [2030727]`.**
[Bug 2030727](https://bugzilla.mozilla.org/show_bug.cgi?id=2030727) is `[meta] TAE - Migrate and remove
legacy tests`; it tracks the campaign via its `depends_on` list, so each conversion bug must *block* it.
Pass `blocks` at create time — that is one field, versus a second round-trip afterwards, and a conversion
that never gets linked is invisible to whoever reads the meta bug for campaign status.

Scope: **test-conversion bugs go on the meta; tooling/docs/harness bugs do not.** The meta is specifically
about migrating and removing legacy tests, which is why e.g. the effview-tool and harness-docs bugs are
deliberately absent from it. If a conversion also needed harness work, the conversion bug still blocks the
meta — the enablement is tracked in Jira (step 4), not by a second meta entry.

To backfill one you already filed:
```json
{ "bug": "update", "ids": [NNNNN], "blocks": [2030727] }
```
`update` wraps relation lists as `{"add": [...]}` so this appends. Never PUT a bare list to a meta bug's
`depends_on` — Bugzilla treats that as *replace* and it would drop every other bug the meta tracks.

### 3. Commit with the real bug number (agent → bridge → `effgit`)

**First annotate the legacy method(s) you just replaced — this goes in the SAME commit as the conversion,
and it is the step most often forgotten:**
```kotlin
@Converted(
    replacedBy = ["org.mozilla.fenix.ui.efficiency.tests.<Class>#<method>"],
    bug = NNNNN,          // the bug you filed in step 2
    since = "YYYY-MM",
    notes = "Legacy also asserted X; not carried over because …",   // only if coverage was dropped
)
```
The gate is **green locally** (step 1's `effverify` verdict), *not* landed — you cannot annotate after
landing without a second bug and a second review, and every conversion in this campaign has landed the
annotation alongside its replacement. `replacedBy` is required, one entry per replacement, and each must
resolve to a real non-`@Ignore`d `@Test`. Put the parity gaps from step 1 in `notes` — that is the
auditable record of what did not carry over. Annotate the legacy method in place; do **not** delete it, it
keeps running alongside the replacement.

Then write the commit message to `conversion-runs/<batch>/msg.txt` with
`Bug NNNNN - [efficiency] … r=isabel_rios,aaronmt` and drop
`{ "git":"commit", "message_file":"<batch>/msg.txt", "paths":[...] }` — the `paths` list must include the
legacy test file you just annotated as well as the new/changed efficiency files. (If a commit already
exists with a placeholder, backfill by rewording — the loop files the bug *before* committing going
forward, so no reword is needed. `effgit`'s `amend` does **not** stage: send `stage` first, then `amend`.)

### 4. Track in Jira (agent → `jiratool.py`)
Separate strict conversion from the enablement it sometimes forces, so conversion effort can be measured:
- **Conversion** → a **Sub-task labelled `conversion`** under the Smoke-conversion campaign story **MTE-5731**.
- **Tooling/enablement** discovered during conversion → a **separate Sub-task labelled `enablement`** under
  Harness Hardening **MTE-5715** (or Tech-Debt **MTE-5688**), **linked** ("Relates") to the conversion sub-task.
- Put the bug number, branch/commit and Phab revision in the item.

Use `tools/jiratool.py` (works headless; no Atlassian connector needed). Bodies come from a file, so write
the description to a temp file first:
```
python3 jiratool.py create '<summary>' --file body.txt --parent MTE-5731 --issuetype Sub-task --label conversion
python3 jiratool.py create '<summary>' --file body.txt --parent MTE-5715 --issuetype Sub-task --label enablement
python3 jiratool.py link <enablement-key> Relates <conversion-key>
python3 jiratool.py assign <key> --me        # create does NOT self-assign; do this explicitly
```
`create` defaults to `--issuetype Story` and no labels, so pass both every time or the item lands as an
unlabelled Story in the wrong shape. `--label` is repeatable. If the Atlassian MCP connector happens to be
connected it also works, but do not count on it — it is absent in headless/cron runs.

An enablement sub-task is warranted whenever the conversion needed a *new* page object, selector twin, nav
edge or `moz*`/`mozVerify*` primitive — i.e. build mode 3. Write up what the gap was and what the proper
fix would be, not just the workaround you shipped.

### 5. Submit the finished stack (engineer)
Submitting/landing stays with the engineer. **Mozilla's moz-phab has no `--dry-run`** — it's interactive: it
prints the commit list and prompts Y/n before creating anything (that's your preview). Bound the range so it
can't touch already-landed base commits:
```
moz-phab submit --reviewer isabel_rios --reviewer aaronmt <first-new-commit>
```
(or `python3 tae-conversion/tools/effsubmit.py --start <first-new-commit> --execute`). Then add the
`testing-exception-unchanged` tag in the Phabricator web UI (no moz-phab CLI flag exists for it). moz-phab
keys off `Differential Revision:` trailers, so base commits that carry them are excluded automatically — even
if they landed on autoland and aren't in your local central yet.

## After landing
Re-sync the tracker so conversion counts catch up with the `@Converted` markers that landed in step 3 (see
`tae-conversion/README.md` → "Reconciling the ledger" and `tae-conversion/tools/reconcile_conversion.py`).

If reconcile reports a converted test with no `@Converted` marker, the annotation was missed in step 3 —
that is a gap to backfill under a follow-up bug, not the normal path. Annotating is step 3's job.

## Conventions
- **Faithful-port-first:** don't rewrite behavior during conversion; log quality ideas separately.
- One bug per landable unit; mirror any conversion/enablement split in the Jira items.
- Bugs + Jira items are self-assigned back to the engineer who ran the loop.
