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

**convert → file bug → commit (with bug #) → track in Jira → submit for review**

## What runs where

- **The agent (sandbox)** does the conversion, static checks, and drives the bridge; it creates Jira
  items via the Atlassian connector. It **cannot** reach the repo, Bugzilla, or Phabricator directly.
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
  "type": "task" }
```
`effbug` files the bug, then **rewords the title to `Bug NNNNN - <summary>`** so it matches the commit
subject exactly, and **self-assigns** to the API-key owner. It returns the number in
`_bug/<id>.bug-result.json`.

### 3. Commit with the real bug number (agent → bridge → `effgit`)
Write the commit message to `conversion-runs/<batch>/msg.txt` with `Bug NNNNN - [efficiency] … r=isabel_rios,aaronmt`
then drop `{ "git":"commit", "message_file":"<batch>/msg.txt", "paths":[...] }`. (If a commit already exists
with a placeholder, backfill by rewording — the loop files the bug *before* committing going forward, so no
reword is needed.)

### 4. Track in Jira (agent → Atlassian connector)
Separate strict conversion from the enablement it sometimes forces, so conversion effort can be measured:
- **Conversion** → a **Sub-task labelled `conversion`** under the Smoke-conversion campaign story **MTE-5731**.
- **Tooling/enablement** discovered during conversion → a **separate Sub-task labelled `enablement`** under
  Harness Hardening **MTE-5715** (or Tech-Debt **MTE-5688**), **linked** ("Relates") to the conversion sub-task.
- Create with `issueTypeName: "Sub-task"`, `additional_fields: {"labels":[...]}`, and self-assign via
  `assignee_account_id`. Put the bug number + Phab revision in the item. cloudId = `mozilla-hub.atlassian.net`.

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
Re-sync the tracker so conversion counts + `@Converted` annotations catch up (see
`tae-conversion/README.md` → "Reconciling the ledger" and `tae-conversion/tools/reconcile_conversion.py`),
and annotate the converted legacy methods with `@Converted(replacedBy = [...], bug = NNNNN, since = "YYYY-MM")`
once green + landed. Record any coverage that didn't carry over in the annotation's `notes` parameter.

## Conventions
- **Faithful-port-first:** don't rewrite behavior during conversion; log quality ideas separately.
- One bug per landable unit; mirror any conversion/enablement split in the Jira items.
- Bugs + Jira items are self-assigned back to the engineer who ran the loop.
