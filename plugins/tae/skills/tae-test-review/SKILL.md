---
name: tae-test-review
description: >-
  Review and authoring guidance for new/changed tests in the Fenix TAE
  efficiency framework
  (mobile/android/fenix/app/src/androidTest/java/org/mozilla/fenix/ui/efficiency).
  Use when reviewing a diff, PR, or Phabricator revision that adds or modifies
  files under that path — page objects, selectors, navigation edges, or test
  files. Also use when authoring a new test and self-checking before submission,
  when asked to check a test against TAE conventions, classify a test
  (presence/interaction/behavior), or judge whether a helper belongs in a page
  object vs. BasePage. Also use when auditing a converted test for assertion
  parity against the legacy test it replaces. Applies the framework's principles
  and anti-patterns as a review rubric with severity tiers.
---

# Contributing to the TAE Test Framework

## How to use this document

This one document serves two audiences:

- **Test authors — before you submit.** Read "Principles", "The three test
  types", and "Anti-patterns", then self-check your test against the "Before you
  write" checklist and the severity tiers in "Conducting a review". If any of
  your findings would be *blocking* for a reviewer, fix them first. For temporary
  1:1 smoke conversions, run each finding through the cost-of-fix gate in
  "Migration-era triage": fix the trivial ones yourself now, and flag the rest as
  known gaps so review is one pass, not a loop.
- **Reviewers — during code review.** Work from "Conducting a review". It routes
  you to the relevant principles and anti-patterns and gives severity tiers so
  your feedback tells the author what blocks merge. For a conversion, start with
  the "Parity audit" — it is the one check a green test cannot substitute for.

Both audiences share the same rules; the only difference is who applies them and
when.

## Conducting a review

This section is for the reviewer (and for the author doing a pre-submission
self-check). The rest of the document defines what "good" looks like; this
defines how to apply it to a diff.

### Scope first

Identify what the diff touches, because different files get different scrutiny:

- **Test file** (`tests/*.kt`) — apply test-type classification, three-phase
  structure, and the one-assertion rule (with the Behavior exception below).
- **Page object** (`pageObjects/*.kt`) — check page-boundary discipline,
  whether new methods earn a `[STEP]`, and that navigation edges are registered
  in `init {}`.
- **Selectors** (`selectors/*Selectors.kt`) — check anchor stability, group
  assignment, and that nothing is defined inline in a test.
- **BasePage / helpers** — highest bar. A new primitive on BasePage is a
  blocking discussion, not a rubber stamp. A new `SelectorStrategy` has a
  specific two-path wiring trap — see anti-patterns.
- **`devtools/*.kt`** — anything with an `@Test` in this package runs on
  Firebase, because the TAE flank config targets the whole package. Every
  dev-only method needs `assumeFalse("dev tool, not a CI test", isTestLab())`.

### Severity tiers

Label each finding so the author knows what blocks merge.

**Blocking** — merge only after this is fixed:
- A legacy assertion is missing from a conversion and the gap isn't documented
  (see "Parity audit"). This one outranks everything else here.
- Assertions interleaved with navigation in a Presence or Interaction test
  (see the Behavior exception below before flagging).
- `Thread.sleep()` or a hand-rolled poll/wait loop.
- Inline selector defined in a test file.
- New method added to `BasePage` without a demonstrated missing *category*.
- Page object method crossing page boundaries.
- `@After` used for state that must be clean for the next test to be valid.
- Hard-coded click sequence to reach a page instead of a `navigateToPage()`
  edge.
- A new `SelectorStrategy` added to only one of the two resolution paths
  (`resolveComposeNode()`'s `candidates()` and `mozGetElement()`'s `when`). It
  compiles and fails at runtime for whichever verb you didn't wire.
- An `@Test` under `devtools/` without an `isTestLab()` guard — it will consume a
  Firebase device slot on every TAE run.

**Should-fix** — fix unless there's a stated reason:
- Test can't be classified as one of the three types, or does too much to be one.
- Setup done through the UI when a constructor flag or pre-seeded data exists.
- `mozVerifyElementsByGroup` used for runtime/dynamic data.
- Popup/dialog handling added inside a test instead of in a primitive/config.
- A wrapper method whose name adds no clarity over `primitive + selector`.
- A selector switched from text to a tag, leaving what the text was *asserting*
  unasserted — and often a now-dead parameter still passed by every caller.
- A selector used by a class that sets `shouldUseExpandedToolbar = true` but only
  verified in the default layout. That flag relocates controls and changes the
  handles they expose.
- A nav edge edited without grepping `to = "<TargetPage>"` across `pageObjects/`
  first. Edges are sometimes registered twice, and the loser is invisible.
- Step-by-step narration comments in a converted test (see "Comments in
  converted tests").

**Nit** — note it, don't block:
- Group name could be more meaningful.
- A method that could be reused but is currently test-local.
- Ordering/naming that reads slightly off-spec.

### Migration-era triage (temporary 1:1 conversions)

Most tests under review right now are 1:1 conversions of legacy smoke tests. They
are **temporary** — they exist until the factories generate the suite in CI — so
the goal is to get correct coverage landed with minimal round-trips, not to make
every conversion structurally perfect. The expensive thing is the feedback
*loop*, not the feedback. Recalibrate accordingly.

**Correctness is not a compromise variable.** Everything below relaxes structure,
placement, and primitive design. None of it relaxes whether the test actually
verifies its stated behavior. A green test that doesn't test the thing is
*negative* value during a migration: it reads as coverage while hiding a gap, and
if these hand-conversions seed factory templates, the wrong assertion
propagates. So correctness findings stay land-blocking no matter how the gate
below scores them.

For a conversion, "correctness" means **assertion parity with the legacy test**,
established by the diff in "Parity audit" — not by the test passing. A review of
one 13-conversion stack found **17** legacy assertions silently missing, and none
of them caused a failure.

**Cost-of-fix gate.** For every non-correctness finding, before requesting the
change, ask:

1. Does requesting it require more than ~1 day and another review round-trip?
2. Is the "better" way moderate+ complexity?
3. Is the preferred fix ambiguous because we have no existing example of it?
4. Does the fix require getting into the guts of BasePage primitives?
5. Is the root cause a missing harness/framework feature that should already exist?

If all five are "no", it's a **fix-now** item — trivial, has precedent, no guts —
and the author should just make it. If any is "yes", it's a **defer** item.

**Three buckets** (use these instead of raw blocking/should-fix/nit for
conversion patches):

- **Land-blocking** — correctness only. The test must verify its premise.
- **Fix-now** — trivial structural fixes that pass the gate (all five "no"):
  page-boundary swaps (`on.mainMenu` -> `on.downloads`), single-navigation
  assertion blocks, missing `all`-list registration, missing TestRail link.
- **Defer** — anything that trips the gate: BasePage primitive surgery,
  no-precedent design calls, or work blocked on missing harness support. Let it
  land; file the item into the cleanup backlog (see guardrails).

**Navigation and page-boundary findings are relax-by-default.** Do not block a
conversion solely because it chains across pages or over-navigates — *unless* the
fix is trivial-with-precedent (then it's fix-now). A test written awkwardly
because the harness lacks a clean step is a signal of a **missing feature**, not
an author error; record it as a gap rather than forcing the author to build the
primitive.

**Guardrails so "relaxed" doesn't become invisible debt:**

- **Track every defer.** A deferral that lives only in a review comment
  evaporates when cleanup starts. Emit defer items into the actual cleanup
  backlog (meta-bug / tag) so "temporary and incomplete" stays visible and
  temporary.
- **Watch what templates.** A one-off quirk in a throwaway test is fine; the same
  quirk in a pattern the factories will generalize from is not. That is the one
  structural class worth pushing on even when the fix is mildly non-trivial.
- **Batch, don't iterate.** Deliver one consolidated pass ("N trivial fixes, M
  deferred to cleanup") rather than a multi-round loop.

### Parity audit (conversions only)

**A dropped assertion does not fail — it passes for the wrong reason.** This is
the single highest-yield check in a conversion review, and the only one that a
green CI run actively disguises.

`navigateToPage()` silently verifies every selector in the target page's
`requiredForPage` group. That makes a legacy `verifyPageContent(...)` or
`verifyUrl(...)` look redundant, so it gets dropped. What survives is the
*navigation* assertion; what disappears is the *payload* assertion. The test
still proves it reached a screen. It stops proving the screen is right.

The audit:

1. Open the legacy test body and the port side by side.
2. List every legacy verification — `verify*` calls in the test **and** in the
   robot helper it delegates to.
3. Tick each one off against an explicit assertion in the port.
4. Anything unticked is either added back explicitly, or documented as a gap in
   **both** the test and the commit message. There is no third option.

When the legacy test gets annotated (after the port is green and landed), that
annotation is where a documented gap belongs permanently:

```kotlin
@Converted(
    replacedBy = ["org.mozilla.fenix.ui.efficiency.tests.AutofillTest#verifyAddressAutofillTest"],
    bug = 2057958,
    since = "2026-07",
    notes = "Legacy also asserted X; not carried over — see bug NNNNN.",
)
```

In review, check that `replacedBy` points at a real non-`@Ignore`d `@Test` (the
conversion lint check validates this) and that any parity gap you found in step 4
is recorded in `notes` rather than left in a review comment.

Do not accept "`navigateToPage` already checks it" as a justification. An
implicit assertion is invisible to the next person diffing the port against the
legacy test, which makes the conversion impossible to audit. A faithful port now
with duplicated-looking lines beats a tidy port that quietly lost coverage; house
style gets settled in a later dedicated refactor pass.

Two related traps:

- **Tag swaps delete content assertions.** When a selector moves from text
  matching to tag-only (a legitimate fix for ambiguity — see the toolbar/address
  bar duplication), ask what the text was *asserting*. If it was content, match
  both with `COMPOSE_BY_TAG_AND_TEXT`. Otherwise the assertion degrades to "an
  element with this tag exists", which is nearly always true, and the parameters
  carrying the expected values become dead — passed by every caller, read by
  nothing.
- **Report the inventory and the provenance.** When summarising a stack as
  "green", state which classes the count covers and *whose* runs it covers. An
  agent's verification and a human's are separate sets and neither is visible to
  the other unless recorded. "39 tests green across 6 classes" for a 7-class
  stack reads as full coverage with one class silently missing.

### Triage checklist (per changed test)

0. If it's a conversion, run the **Parity audit** first. Everything below is
   structure; this is whether the test still tests the thing.
1. Classify it: Presence, Interaction, or Behavior. If you can't, that's a
   should-fix — the author doesn't have a clear subject.
2. Strip the navigation mentally. Does the remaining body read as one coherent
   spec? If it reads as several, check the Behavior exception, then decide split
   vs. keep.
3. Find the single assertion (or, for Behavior, the ordered checkpoints) that
   states why the test exists. If it's buried or absent, flag it.
4. Trace setup: could any of it move to a constructor flag, intent extra, or
   pre-seeded data? Every UI setup step is a should-fix candidate.
5. Check every selector used: does it exist in the selectors file with a
   meaningful group? Any inline selector is blocking.
6. Check every new page-object method: does it earn a `[STEP]`, and does it stay
   on the page it operates on?
7. Check navigation: is every hop a registered edge?
8. For a conversion, read the legacy **robot** helper, not just the legacy test
   body. Retry/refresh semantics, per-assertion waits, and fallback behaviour
   live in the robot. Porting the body alone yields a test that looks correct and
   is flaky.

### The Behavior-test exception (read before flagging "interleaved assertions")

The "one test, one assertion" rule applies fully to **Presence** and
**Interaction** tests. **Behavior** tests legitimately span surfaces and may
contain *sequential checkpoints* — a verify after each state transition that a
later step depends on.

Distinguish a legitimate checkpoint from two tests glued together:

- **Legitimate checkpoint** — the verified state is a *precondition for the next
  step*, and removing it would make a later failure impossible to localize
  (e.g. verify a bookmark exists before opening its menu to edit it).
- **Glued-together tests** — the assertion blocks are independent; each could
  stand alone with its own setup, and neither depends on the other's state.
  Split these.

If in doubt, ask: "does the second assertion require the first action to have
happened?" Yes → checkpoint, allowed. No → split, blocking.

### Worked example

**Rejected** — inline selector, UI setup, and two unrelated assertions glued
together across navigation:

    @Test
    fun bookmarksTest() {
        // UI setup that a constructor flag / pre-seed could do
        on.browserPage.navigateToPage(url)
        on.browserPage.openMainMenu().mozClick(MainMenuSelectors.BOOKMARK_THIS)

        on.bookmarks.navigateToPage()
            .mozVerify(Selector(COMPOSE_BY_TEXT, "My Page", groups = listOf())) // inline selector
            .mozClick(BookmarksSelectors.THREE_DOT_MENU)                        // crosses into an action
        on.bookmarks.mozVerify(BookmarksSelectors.EDIT_BUTTON)                  // assertion #1: menu opens

        on.home.navigateToPage()                                               // unrelated surface
            .mozVerifyElementsByGroup("jumpBackIn")                            // assertion #2: independent
    }

Review comments:
- **Blocking**: inline selector — move to `BookmarksSelectors` with a group.
- **Blocking**: two independent assertions (bookmark menu, then Jump Back In)
  glued by navigation — the second doesn't depend on the first. Split into two
  tests.
- **Should-fix**: the bookmark is created through the UI — pre-seed it with
  `createBookmarkItem(...)` so this test starts on the surface it's verifying.

**Accepted** — one Presence test, setup pre-seeded, single assertion. Note the
assertion uses `mozVerify` with a parameterized selector (`BOOKMARK_ITEM(title)`)
rather than a group, because `title` is runtime data — this is correct, not a
missing group:

    @Test
    fun bookmarksListShowsSavedItem() {
        // SETUP
        createBookmarkItem(url, title, null)

        // STEPS
        on.bookmarks.navigateToPage()

        // ASSERT
        on.bookmarks.mozVerify(BookmarksSelectors.BOOKMARK_ITEM(title))
    }

## Principles

### Tests declare what, not how

A test should read as a specification of behavior. Navigation mechanics, element resolution, retries, and waits belong in the framework. If someone reads your test and can't tell what feature it validates within 10 seconds, rewrite it.

### One test, one assertion (with a Behavior-test exception)

For **Presence** and **Interaction** tests: if you have assertion blocks
separated by navigation or interaction steps, you have written multiple tests
glued together. Split them. Use `mozVerifyElementsByGroup` to assert multiple
related elements as one logical check when they belong to the same verification.

For **Behavior** tests: because they intentionally cross surfaces, a verify may
appear after each state transition as a *sequential checkpoint* — but only when
that verified state is a precondition for the next step and removing it would
make a later failure impossible to localize. Independent assertion blocks that
don't depend on each other's state are still two tests; split them.

The distinguishing question: *does the next action require the previous
assertion's state to have happened?* Yes → legitimate checkpoint. No → split.

(See "Conducting a review > The Behavior-test exception" for how this is applied
in review.)

### Three-phase structure

Every test follows this pattern:

```kotlin
// SETUP: what preconditions does this test need?
// Prefer BaseTest constructor flags, pre-seeded data, or pre-runner state.
// Reserve in-test setup for state that genuinely requires UI interaction.
createBookmarkItem(url, title, null)

// STEPS: navigation and page object interactions to reach the point of interest.
on.bookmarks.navigateToPage()
    .openItemMenu(title)
    .mozClick(BookmarksSelectors.SHARE_BUTTON)

// ASSERT: a single assertion block capturing the spirit of the test.
// This is WHY the test exists and WHAT artifact is being verified.
on.shareOverlay.mozVerifyElementsByGroup("shareTabLayout")
```

Push setup as early as possible. If state can be configured via feature flags in the `BaseTest` constructor, do that. If it can be set before the runner initializes (intent extras, shared prefs), do that.

## The three test types

Before writing a test, identify which type it is. Each has a repeatable model:

**Presence** - Navigate to a surface, verify elements render. No state changes. Answers: *"Does this page show what it should?"*
```kotlin
on.home.navigateToPage()
    .mozVerifyElementsByGroup("requiredForPage")
```

**Interaction** - Navigate + perform an action + verify the immediate result. Modifies state on one surface. Answers: *"Does this control do what it should?"*
```kotlin
on.home.navigateToPage()
    .mozClick(HomeSelectors.PRIVATE_BROWSING_BUTTON)
on.home.mozVerifyElementsByGroup("privateBrowsing")
```

**Behavior** - One or more state changes across one or more pages. Answers: *"Does this feature work end-to-end?"* These compose presence and interaction primitives.
```kotlin
on.browserPage.navigateToPage(url)
on.home.navigateToPage()
    .mozVerifyElementsByGroup("jumpBackIn")
```

If you can't classify your test, you don't yet have a clear enough picture of what you're testing.

## Reuse over specificity

### Write for reuse by default

Before adding a function, ask: "Would another test for a different feature need this?" If yes, it belongs in a page object or shared step. If no, reconsider whether you need a new function at all.

### Page objects are shared vocabulary

If you add a method to a page object, it should be useful to any test touching that page. If only your test calls it, it doesn't belong there.

### Test steps belong on the page they operate on

A page object method should only interact with the UI surface it represents. If a method on `BrowserPage` clicks through MainMenu and Collections selectors, it's crossing page boundaries -- put it on the page where the action starts, or split it across the relevant page objects. Similarly, don't chain primitives on one page object while interacting with another page's UI just because the return type allows it.

### Selectors are shared vocabulary too

Define selectors once in the appropriate `selectors/*Selectors.kt` file. Assign meaningful groups. Don't create one-off selectors inline in tests.

## Primitives and test steps

### Use the existing primitives

`mozClick`, `mozSwipeTo`, `mozVerify`, `mozVerifyElementsByGroup`, `mozEnterText`, `mozPressEnter` -- these are your building blocks. Compose tests from them.

> Verified against `helpers/BasePage.kt` on 2026-07-29. This is a point-in-time
> snapshot — new primitives are added over time, so treat the current BasePage
> source as authoritative if it differs. `mozVerifyElement`, `mozGetElement`, and
> `resolve` are **private** to BasePage; don't reach for them from tests or page
> objects.

### Interaction primitives

| Primitive | Purpose |
|-----------|---------|
| `mozClick(selector)` | Click an element |
| `mozLongClick(selector)` | Long-click an element |
| `mozClickIfPresent(selector)` | Click only if the element is present (no failure if absent) |
| `mozClickFirstWithParentText(selector, parentText)` | Click the first match under a parent with given text |
| `mozEnterText(text, selector)` | Enter text into a field |
| `mozClear(selector)` | Clear a field |
| `mozClearAndEnterText(text, selector)` | Clear then enter text |
| `mozPressEnter(selector)` | Press the IME enter key on the field |
| `mozSwipeTo(selector)` | Swipe until an element is visible |
| `mozSwipeElement(selector, direction)` | Swipe a specific element in a direction |
| `mozPressBackUntilGone(selector)` | Press back repeatedly until an element is gone |
| `mozOpenNotificationsTray()` | Open the system notifications tray |
| `navigateToPage()` | Navigate via the navigation graph |

### Verification primitives

| Primitive | Purpose |
|-----------|---------|
| `mozVerify(selector)` | Verify a single element is displayed |
| `mozVerifyElementsByGroup(group)` | Verify all selectors in a group (compile-time selectors only) |
| `mozVerifyElementAbsent(selector)` | Verify an element is not displayed |
| `mozWaitUntilAbsent(selector)` | Wait until an element is no longer present |
| `mozVerifyAnyContainsText(selector, text)` | Verify any match contains the given text |
| `mozVerifyAnyHasChildWithText(selector, text)` | Verify any match has a child with the given text |
| `mozVerifyNoneContainText(selector, text)` | Verify no match contains the given text |
| `mozVerifyElementIsSelected` / `mozVerifyElementNotSelected` | Verify selected state |
| `mozVerifyElementIsChecked` / `mozVerifyElementIsNotChecked` | Verify checked state |
| `mozVerifyElementIsEnabled` / `mozVerifyElementIsNotEnabled` | Verify enabled state |
| `mozVerifyElementHasCheckedSiblingByResName` | Verify a sibling (by res name) is checked |
| `mozVerifyElementHasSiblingWithText` | Verify a sibling has given text |
| `mozVerifyKeyboardVisible()` / `mozIsKeyboardVisible()` | Verify / query soft-keyboard visibility |
| `mozVerifyFileOpensInExternalApp(...)` | Verify a file hands off to an external app |
| `verifySnackbarText(text)` | Verify a snackbar shows the given text |
| `waitForSnackbarToBeDismissed()` | Wait for the snackbar to dismiss |

`dismissKnownOverlaysIfPresent()` is public but you should almost never call it —
`mozClick` and `mozVerify` already fire it automatically on a locate miss and
retry once. Register new blocking overlays in `helpers/OverlayRegistry.kt` instead
of dismissing them from a test.

`BrowserPage.verifyPageContentWithReload(url, text, attempts)` is a page-object
method, not a BasePage primitive. Reach for it when content only appears after
async work (blocked-tracker reports, for example) — a longer wait on the current
document never helps, because the page has to be re-fetched.

The state-verification family (`...IsSelected`, `...IsChecked`, `...IsEnabled`,
sibling checks) is easy to overlook — reach for these before writing a custom
check when asserting toggle/checkbox/selection state. Their exact parameter
lists vary; check the current `helpers/BasePage.kt` signatures before use.

### Do not extend BasePage

Don't add new primitives to `BasePage.kt` unless you've identified a genuinely missing *category* of interaction that multiple pages need. The bar for adding to BasePage is high.

### Don't create framework-level abstractions

No `clickAndWaitForBookmarks()` -- that's `mozClick` + `navigateToPage`. No `interactAndWait()` -- that's a primitive trying to be a framework. These belong in BasePage if anywhere.

### Custom commands over custom waits

If you're tempted to write a new wait/polling mechanism, you probably need a better selector (one that keys off the right element state) rather than new timing logic.

### When to wrap primitives into page object test steps

Page object methods are test steps. The structured log hierarchy is `[STEP]` > `[CMD]` > `[LOC]` > `[SEL]`, and test steps should read like steps in a TestRail test case. The question isn't "how many primitives does it wrap?" but "does wrapping improve logging and root cause analysis?"

**Wrap when:**
- The method name maps to a recognizable user action that would appear as a TestRail step (e.g., `openMainMenu`, `openItemMenu`, `saveEditBookmark`)
- A `[STEP]`-level failure in the log would immediately tell you what user action broke -- without reading the underlying `[CMD]`s -- with close to zero processing effort
- The grouping represents a logical user action, even if it's one primitive today

**Don't wrap when:**
- The wrapper name doesn't add clarity beyond primitive + selector (e.g., `verifyBookmarkTitle(title)` vs `mozVerify(BookmarksSelectors.BOOKMARK_ITEM(title))` -- these read identically)
- You're wrapping just to avoid typing the selector object name

This matters because the structured logging is designed as a bidirectional bridge to TestRail: well-named test steps, commands, locators, and selectors mean TestRail cases can generate tests via factories, and test logging can maintain TestRail cases. AI can also generate test scaffolding from selectors, navigation nodes, and page object test steps. This flow only works when each layer is meaningful and reads like a test case.

## Selectors and element strategy

### Choose stable anchors

Selectors should represent stable, semantic anchors: test tags, resource IDs, accessibility labels. Avoid selectors tied to layout position, localized text, or internal implementation details.

### Use groups to express intent

- `"requiredForPage"` -- elements that prove the page loaded
- `"jumpBackIn"`, `"topSitesCompose"` -- elements that constitute a feature area
- `"homeScreen"` -- elements belonging to a broader surface

Groups turn element lists into meaningful assertions.

### Groups are for static elements; use multiple `mozVerify` calls for dynamic data

`mozVerifyElementsByGroup` works with selectors defined at compile time in the selectors file. When values come from test data at runtime (e.g., verifying that specific page titles appear in a collection), use individual `mozVerify` calls with parameterized selectors. Multiple `mozVerify` calls in the same assertion block on the same page is fine -- it's not the same anti-pattern as splitting assertions across navigation.

### Prefer existing selector strategies

The 25 strategies in `SelectorStrategy` (`helpers/Selector.kt`) cover Compose,
Espresso, and UIAutomator. If none works, the UI itself may need a test hook (a
test tag or content description) rather than a more complex selector.

Ones that are easy to miss because they solve a narrow problem:

| Strategy | Reach for it when |
|---|---|
| `COMPOSE_BY_TAG_AND_TEXT` | A tag disambiguates *and* the text is the thing being asserted. This is the fix for a tag swap that would otherwise drop a content assertion. |
| `UIAUTOMATOR_WITH_COMPOSE_TAG` | Matching a **web/GeckoView DOM id**. It matches the raw resourceId with no package prefix, which is what web content exposes. |
| `UIAUTOMATOR_WITH_WEB_ID_AND_TEXT` | Raw web DOM id plus exact text — e.g. asserting an autofilled value in a form field. |
| `UIAUTOMATOR_WITH_RES_ID_CONTAINING_TEXT` | Package-prefixed app res-id plus a text substring (mirrors legacy `itemWithResIdContainingText`). |
| `UIAUTOMATOR_WITH_DESCRIPTION_CONTAINS` | A control that **moves between surfaces** — see below. |

Note the app/web split: an app View res-id wants `UIAUTOMATOR_WITH_RES_ID`
(which prepends `packageName:id/`), while a web DOM id (`submit`, `username`)
needs a raw-resourceId strategy. Using the app one on web content silently
matches nothing.

### Controls that relocate need a device-level handle, not a tag

`shouldUseExpandedToolbar = true` is not a restyle — it moves controls between
surfaces and changes which handles they expose. Confirmed cases: the tab counter
moves into the bottom navigation bar and exposes **no testTag at all**, only a
content description; "Bookmark page" moves out of the main menu, so a Compose
content-description lookup scoped to the menu finds nothing; the search
placeholder becomes a text node with no content description.

So a testTag cannot be "the stable handle" for a control whose tag doesn't exist
in the other layout. Prefer a device-level
`UIAUTOMATOR_WITH_DESCRIPTION_CONTAINS`, which resolves in both — see
`ToolbarSelectors.TAB_COUNTER_ANY_LAYOUT`. In review: any selector used by a class
that sets the flag must have been verified in *that* layout.

## Navigation

### Use the registry

All navigation goes through `navigateToPage()`. Hard-coded click sequences to reach a page mean the registry is missing an edge -- add the edge, don't work around it.

### Register edges in page object init blocks

This keeps the graph definition co-located with the page that owns the relationship:

```kotlin
class HomePage(...) : BasePage(composeRule) {
    override val pageName = "HomePage"

    init {
        NavigationRegistry.register(
            from = "AppEntry",
            to = pageName,
            steps = listOf(),
        )
        NavigationRegistry.register(
            from = pageName,
            to = "MainMenuPage",
            steps = listOf(NavigationStep.Click(HomeSelectors.MAIN_MENU_BUTTON)),
        )
    }
}
```

### Navigation is not a test step

Getting to the page is infrastructure. Your test starts once you're *on* the page. If navigation dominates your test body, your test is too far from its subject.

## Anti-patterns

These should be flagged in code review:

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Assertions interleaved with navigation | Multiple tests combined into one | Split into separate tests |
| New methods on BasePage | Framework bloat | Use existing primitives |
| Inline selectors in test files | Not reusable | Add to selectors file with groups |
| `Thread.sleep()` or custom polling | Brittle, flaky | Use `requiredForPage` or existing waits |
| Wrappers that don't add log clarity | Name doesn't aid root cause analysis | Use the primitive + selector directly |
| UI setup when config is available | Slower, flakier | Use constructor flags or pre-seeded data |
| Journey tests without foundational coverage | Premature | Write presence/interaction tests first |
| Framework-level abstractions in page objects | Belongs in BasePage if anywhere | Keep page object methods as test steps, not new primitives |
| Page object methods crossing page boundaries | Breaks page object model | Put methods on the page they operate on, or use separate `on.<page>` calls |
| Using `mozVerifyElementsByGroup` for dynamic data | Groups are compile-time; dynamic values won't match | Use individual `mozVerify` calls with parameterized selectors |
| Using `@After` for critical state cleanup | If the runner crashes, `@After` is not called -- leaves dirty state that can break subsequent tests or worse, cause false passes from carried-over state | Push cleanup to pre-test setup, constructor flags, or runner-level mechanisms that run regardless of crash |
| Handling unexpected popups in test assertions | System alerts, permission dialogs, and conditional modals break tests that aren't meant to verify them | Let custom commands handle view-blocking elements via fallback conditional checks -- this keeps the fix in one place (the primitive) rather than scattered across tests |
| Dropping a legacy assertion as "covered by `navigateToPage`" | The navigation check survives, the payload check disappears; the port passes while testing less | Restore it explicitly, or document the gap in the test *and* the commit message (see "Parity audit") |
| Swapping text matching for a tag without asking what the text asserted | Degrades the check to "an element with this tag exists" and leaves dead parameters behind | Use `COMPOSE_BY_TAG_AND_TEXT` when the text was the assertion |
| Step-by-step narration comments in a converted test | Restates the code and buries the parity notes that actually carry information | Keep the conversion header, TestRail link, and parity/why notes; cut the narration |
| A new `SelectorStrategy` wired into one resolution path | `mozClick` and `mozVerify` resolve differently; compiles clean, fails at runtime for the unwired verb | Wire both `resolveComposeNode()`'s `candidates()` and `mozGetElement()`'s `when` |
| An `@Test` under `devtools/` with no `isTestLab()` guard | The flank config targets the whole package, so it burns a Firebase slot every run | `assumeFalse("dev tool, not a CI test", isTestLab())` — not `@Ignore`, which also blocks manual runs |
| A testTag as the handle for a control that relocates | The tag may not exist in the expanded-toolbar layout | Device-level `UIAUTOMATOR_WITH_DESCRIPTION_CONTAINS` |

## Handling unexpected popups and system dialogs

Tests that aren't specifically verifying a popup, alert, or modal should not fail because one appeared unexpectedly. The framework's custom commands are designed to handle this: locators inside primitives can include fallback checks for view-blocking elements (system alerts, client popups, app modals) in priority order.

Don't add popup handling logic to individual tests. If a system dialog or conditional modal is blocking your test:
- If it appears reliably, suppress it via configuration (e.g., `isPageLoadTranslationsPromptEnabled = false` in the BaseTest constructor)
- If it appears sometimes, the custom command that encounters it should handle the dismissal -- one fix in one place
- If it requires state detection (e.g., permission state determines whether a dialog appears), use that state to drive conditional checks within the primitive, not the test

The goal is stability first, speed second. Adding conditional checks for view-blocking elements in custom commands is acceptable overhead -- it's cheaper than flaky tests.

## Comments in converted tests

The repo-wide "almost never comment" rule still applies to narration, but a
converted test has two comment categories that are explicitly welcome, because
they carry context a future reader diffing against the legacy test would otherwise
lose:

1. **Parity mapping** — how the port maps to its legacy counterpart, especially
   any leg intentionally omitted or restructured. One or two lines.
2. **Special-case "why"** — a brief reason for something state-specific or
   non-obvious: why an extra navigation step is needed, why a verification is
   implicit, why a wait or flag exists.

Keep the `// Converted from legacy <Class>.<method>` header and the TestRail link.
Cut comments that restate the code (`// Load a page, open the main menu, tap
Bookmarks` above code doing exactly that), and don't duplicate a long parity
explanation that already lives in the commit message — condense it.

Attribute a harness gap to the gap ("no stateful BookmarksPage -> BrowserPage edge
yet"), not to a selector or locator problem.

## Diagnosing a failure before blaming a selector

Reviewers and authors both waste cycles here, so it's worth stating the order.

**"Not found" can mean "covered", not "absent".** On a locate failure the harness
dumps all layers — Compose, UIAutomator, Espresso — plus a `[windows]` summary
with window titles/types, IME and overlay flags, and the currently focused input.
**Read `[windows]` first.** A non-APPLICATION window on top means an overlay,
popup, or keyboard covered the target; a focused input somewhere unexpected means
focus was stolen. Neither is a selector bug. Known blocking overlays (the Android
stylus-handwriting prompt, for one) are auto-dismissed via `OverlayRegistry`; add
new ones there. Web-form tests should also disable the prompt deterministically
with `settings put secure stylus_handwriting_enabled 0` before focusing a field.

**Group verification names only the first missing element.**
`mozVerifyElementsByGroup` is an `all {}`, so it short-circuits. Expect to iterate
once per missing member, or read the dump and check the whole group in one pass.

**A retry-pass is not a pass.** `clean = false` means it failed once and passed on
retry. During conversions that's most often an overlay rather than a product bug —
check `[windows]` before rewriting anything.

**If a fix changes nothing, suspect a second definition before your diagnosis.**
Two separate cases produced byte-identical failures after a real fix: a duplicate
`NavigationRegistry` edge registered in another file, and a strategy added to one
resolution path but not the other.

**Gradle's JUnit XML is authoritative** for pass/fail
(`androidTest-results/connected/debug/TEST-*.xml`); the logcat trace explains why.
The Test Orchestrator runs each test in its own process, so one
`run finished: 1 tests` per test plus a suite summary is normal, not a stale
buffer.

## Before you write: checklist

1. What type of test is this? (Presence / Interaction / Behavior)
2. What is the single assertion that captures why this test exists?
3. Can the setup be pushed to constructor flags or pre-runner state?
4. Do the selectors I need already exist? Are they in the right groups?
5. Do the page object methods (test steps) I need already exist?
6. For any new test step: does the method name create a meaningful `[STEP]` in the log? Would a failure at that level immediately tell you what broke?
7. If I removed all the navigation, does the test body still make sense as a spec?
8. If this is a conversion: have I listed every `verify*` in the legacy test **and
   its robot helper**, and does each one have an explicit counterpart here or a
   documented gap?

## Adding a new page

1. Create selectors in `selectors/NewPageSelectors.kt` with groups (at minimum `"requiredForPage"`)
2. Create page object in `pageObjects/NewPage.kt` extending `BasePage`
3. Register navigation edges in `init {}`
4. Implement `mozGetSelectorsByGroup()`
5. Add page instance to `helpers/PageContext.kt` for use in tests

Two constraints that only bite later:

- **A new page object can never have an empty navigation path.** The Reachability
  factory auto-registers every page object by reflection over `PageContext` and
  generates a "can I reach this page?" case for each. `steps = listOf()` on the
  only edge produces a case that always fails. If the page only exists under a
  special launch (onboarding, for example), declare a `LaunchConfig` on its
  `AppEntry` edge instead.
- **`requiredForPage` must be state-invariant.** Pick something present in every
  state — a toolbar title, never an empty-list placeholder. And if the *entry*
  control is state-dependent (the trust-panel button's tag varies with page
  security), the edge must `ClickIfPresent` every variant. Static checks can't see
  either of these; verify by hand whenever you build or modify navigation.

## Where the rest of the documentation lives

This skill is the review rubric. The full authoring reference is in-tree and is the
source of truth — read it there rather than trusting a copy:

`mobile/android/fenix/app/src/androidTest/java/org/mozilla/fenix/ui/efficiency/docs/`

| Need | Read |
|---|---|
| Harness bug catalog + authoring checklist (the A*/B* entries cited above) | `docs/gotchas.md` |
| Converting a legacy test end to end | `docs/converting-a-test.md` |
| Architecture and layering | `docs/architecture.md` |
| Tool inventory and what each gate runs | `docs/tooling.md` |
| Selector discovery / authoring, page objects, navigation, BasePage, debugging | `docs/guides/` |

The host-side `eff*` toolchain (`effcheck`, `effnext`, `effscaffold`, `effverify`,
`effloop`, the `effwatch` bridge) lives in the **testops-tools** repo under
`tae-conversion/`; see its README for setup. The device-side dump tools
(`effview`, `effpretty`) ship in-tree under `ui/efficiency/devtools/`.
