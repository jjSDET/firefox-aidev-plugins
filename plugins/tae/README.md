# Fenix TAE Test Plugin

Skills for authoring, reviewing, and landing tests in the Fenix TAE efficiency test
framework at
`mobile/android/fenix/app/src/androidTest/java/org/mozilla/fenix/ui/efficiency/`.

The three skills cover one workflow end to end — **author → review → land** — and are
useful independently.

## Skills

### `efficiency-test-authoring`

Author or convert a Fenix UI test onto the ui/efficiency framework. Picks the cheapest
viable build mode (factory-generated, hand-composed, or compose-plus-extend) and walks
the per-test feedback loop: pick a candidate, scaffold, resolve navigation, choose
selectors from real handles, static pre-flight, build/run, then audit parity against the
legacy test.

Use it when writing a new efficiency test or porting a legacy robot-based one.
See [`skills/efficiency-test-authoring/SKILL.md`](skills/efficiency-test-authoring/SKILL.md).

### `tae-test-review`

Review rubric and authoring guidance: the three test types, page objects, selectors,
navigation, primitives, severity tiers, and migration-era triage for temporary smoke
conversions. Leads with the **parity audit** — the check a green CI run actively
disguises, because a dropped assertion doesn't fail, it passes for the wrong reason.

Use it when reviewing a diff, PR, or Phabricator revision touching the framework, or as
a pre-submission self-check.
See [`skills/tae-test-review/SKILL.md`](skills/tae-test-review/SKILL.md).

### `efficiency-conversion-loop`

The paperwork around a finished conversion: file a Bugzilla bug, commit with the real bug
number, track the conversion/enablement split in Jira, and open a moz-phab review.

Use it once a conversion is written and needs to become a landed patch.
See [`skills/efficiency-conversion-loop/SKILL.md`](skills/efficiency-conversion-loop/SKILL.md).

## What this plugin does not contain

**Reference documentation.** The authoring guides, harness gotcha catalog, and
architecture docs live in-tree under `<framework-root>/docs/` and are the source of
truth. The skills point at those paths rather than vendoring copies, so there's one place
to update.

**The host-side toolchain.** The `eff*` scripts the skills drive (`effnext`,
`effscaffold`, `effcheck`, `effbuild`, `effverify`, `efftriage`, `effloop`, the paperwork
helpers `effbug`/`effgit`/`effsubmit`, and the `effwatch` bridge)
live in [`testops-tools`](https://github.com/mozilla-mobile/testops-tools) under
`tae-conversion/`. Every tool answers `--version` with a shared CalVer stamp
(`tae-conversion YYYY.MM.DD`), so if a skill cites behaviour your copy does not have, check
that first — these skills are written against **2026.08.19**. They run on the engineer's machine and talk to a device, Bugzilla, and
Phabricator — that's infra tooling, not something to ship inside a plugin. The
device-side dump tools (`effview`, `effpretty`) ship in-tree under
`<framework-root>/devtools/`.

Skills work without the toolchain for authoring and review. The conversion loop needs it.
