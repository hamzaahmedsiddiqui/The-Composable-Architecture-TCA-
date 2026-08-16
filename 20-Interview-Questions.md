# Chapter 20 — Interview Questions

One hundred questions, organized by level, each with a full answer. Use
this chapter for interview prep, but also to self-check your
understanding after reading Chapters 1-19 — every answer here should feel
familiar, not new.

## Beginner (1-25)

**1. What is The Composable Architecture?**
A Swift library for structuring apps around a single, unidirectional flow
of State, Action, Reducer, Effect, and Store, inspired by Redux and the
Elm Architecture, built to work naturally with SwiftUI.

**2. What are the five core building blocks of TCA?**
State (what the feature knows), Action (what happened), Reducer (computes
new State and Effects from State + Action), Effect (describes outside
work), and Store (the runtime object tying them together).

**3. Why is State a struct rather than a class?**
Structs are value types: copies are independent, which prevents hidden,
shared mutation and keeps state predictable and easy to compare with
`==`.

**4. Why is Action an enum?**
An enum can represent exactly one of several possibilities, each with its
own associated data, and `switch` over it is exhaustive — the compiler
forces every case to be handled.

**5. What does a reducer return?**
An `Effect<Action>` — often `.none` if no side effect is needed, or a
`.run` effect describing async work.

**6. What is `.none`?**
An `Effect` value meaning "no side effect is required for this action" —
just a plain state mutation.

**7. Why can't a reducer call an API directly?**
Doing so would make it impossible to test in isolation and would break
the guarantee that every state change traces back to a dispatched Action.

**8. What are the only two things a TCA View is allowed to do?**
Read State to know what to render, and send Action when something
happens.

**9. What is `store.send(...)` for?**
It dispatches an Action into the Store, which runs the reducer and
updates State.

**10. What does `@ObservableState` do?**
It makes a `State` struct participate in Swift's Observation framework,
so SwiftUI Views only re-render when the specific properties they read
actually change.

**11. What does `@Reducer` do?**
A macro that generates the boilerplate needed for a type to conform to
the `Reducer` protocol and work with TCA's composition and observation
tools.

**12. What is `StoreOf<Feature>`?**
Shorthand for `Store<Feature.State, Feature.Action>`.

**13. What is dependency injection, in plain terms?**
Supplying a service (like a networking client) to code from the outside,
rather than the code creating it itself, so it can be swapped for a fake
in tests/previews.

**14. What is `@Dependency` used for?**
Reading a currently-active dependency value (live, preview, or test)
inside a reducer.

**15. What is the difference between `liveValue` and `testValue`?**
`liveValue` is the real implementation used when the app runs normally;
`testValue` is used automatically inside tests, and by default fails
loudly if called without being explicitly stubbed.

**16. What is `IdentifiedArrayOf`?**
An array-like collection where elements conform to `Identifiable` and can
be looked up, updated, or removed efficiently by ID.

**17. What is a "single source of truth"?**
The idea that exactly one place holds the authoritative value for a given
piece of data — in TCA, that place is `State`.

**18. Why is unidirectional data flow useful?**
Because state can only change through one path (dispatch Action → reducer
computes new State), every change is traceable and reproducible.

**19. What happens when you tap a button in a TCA View?**
It sends an Action to the Store, which runs the reducer, which computes
new State (and maybe an Effect); SwiftUI then re-renders based on the new
State.

**20. What is `TestStore`?**
A TCA tool for testing a feature's full behavior — state changes, and
effect-produced actions — deterministically and without touching real UI
or networking.

**21. What does `Equatable` conformance on `State` enable?**
Comparing state values with `==`, which `TestStore` assertions and
SwiftUI's diffing rely on.

**22. What is a "delegate action"?**
A dedicated action case a child feature uses to report events upward to
its parent, without the parent needing to know the child's internal
implementation details.

**23. What is `Scope` used for?**
Wiring a slice of a parent's State and a case of a parent's Action to a
whole child reducer.

**24. What is the difference between a sheet and a full screen cover in
TCA navigation terms?**
Both are typically driven by the same `@Presents`/`PresentationState`
mechanism; the difference is purely which SwiftUI modifier
(`.sheet`/`.fullScreenCover`) is used to present the resulting Store.

**25. Who created TCA?**
Point-Free — Brandon Williams and Stephen Celis — starting around 2019,
released as an open-source package in 2020.

## Intermediate (26-55)

**26. Explain the full loop from a button tap to a network response
updating the UI.**
View sends an Action → Store passes State + Action to the reducer →
reducer mutates State and returns a `.run` Effect → Store executes the
Effect as a Task, which awaits the network call → on completion, the
Effect calls `send` with a response Action → that re-enters the same
pipeline → reducer updates State again → SwiftUI re-renders.

**27. Why does the reducer mutate `state` with `inout` instead of
returning a new State value?**
Because rebuilding a whole new (potentially large, nested) State value on
every action would be wasteful; `inout` lets Swift mutate the existing
value's properties directly and efficiently via copy-on-write.

**28. How does modern TCA's observation differ from the old
Combine-based `ObservableObject` approach?**
`ObservableObject` published one signal for any state change, causing
every View reading anything from the Store to re-render; `@ObservableState`
tracks reads per-property, so only Views reading the specific changed
property re-render.

**29. What is a case path, and why does TCA need it?**
The enum equivalent of a key path — it lets you extract or embed a
specific enum case's associated value, which structs' key paths can't
express since enums may or may not currently be in a given case.

**30. What is `@Presents` and what does it mark?**
A macro marking an optional State property as driving presentation
(sheet, full screen cover, alert): `nil` means "not presented," non-nil
means "present this screen with this state."

**31. What is the difference between `PresentationAction.presented(_:)`
and `.dismiss`?**
`.presented(childAction)` wraps an action from inside the currently
presented child; `.dismiss` represents the presented screen being
dismissed by any means.

**32. What does `.ifLet(\.$child, action: \.child) { ChildFeature() }`
do?**
Runs `ChildFeature`'s reducer whenever `state.child` is non-nil, routing
matching actions to it, and manages the presentation lifecycle.

**33. How does `StackState` differ from `PresentationState`?**
`StackState` models a whole push-based navigation stack (any number of
screens, of possibly different types); `PresentationState`/`@Presents`
models a single, optional, presented screen (sheet/cover/alert).

**34. What is a `@Reducer enum Path` used for?**
Allowing a single `NavigationStack`'s `StackState` to hold different
kinds of screens (different feature types), with the macro generating
the combined routing logic.

**35. How would you implement deep linking in TCA?**
Parse the incoming URL into the exact `State` value representing "already
navigated to the right place" (e.g. building up `state.path`), typically
via a dedicated `.deepLink(url)` action so the logic is testable.

**36. Explain how a child feature communicates upward to a parent
without knowing the parent exists.**
Via a `delegate` action case containing named events (e.g. `.didSave`);
the parent listens for `.delegate(...)` cases and decides what they mean,
while the child's own logic never references the parent.

**37. What is `@Shared` and when should you use it?**
A property wrapper letting unrelated features observe/mutate the same
piece of state (optionally persisted), used when two distant parts of
the app need the same live data without a direct parent/child
relationship — not as a substitute for normal composition.

**38. What is `BindingReducer` and when is it appropriate?**
A reducer that generically applies `BindingAction`s (from two-way
`$store.field` bindings) to State; appropriate for simple, direct field
edits without extra business logic attached.

**39. Why should `Date()`/`UUID()` not be called directly in a
reducer?**
It makes tests non-deterministic; TCA provides injectable, controllable
`date` and `uuid` dependencies specifically to solve this.

**40. What does `.cancellable(id:cancelInFlight:)` do?**
Attaches a stable ID to an effect so it can be cancelled later;
`cancelInFlight: true` cancels any existing effect with the same ID
before starting the new one.

**41. Why does effect cancellation matter for correctness, not just
performance?**
Without it, a slower, now-obsolete effect (e.g. a stale search request)
can complete after a newer one and overwrite correct, newer state with
stale data — an "out-of-order response" bug.

**42. What is the difference between debounce and throttle?**
Debounce waits for a pause in activity before running once; throttle
limits execution to at most once per time window regardless of how many
calls occur.

**43. How do you test a debounced effect without waiting in real
time?**
Inject `TestClock` as the `continuousClock` dependency and call
`await clock.advance(by:)` to move simulated time forward instantly.

**44. What does exhaustive testing mean in `TestStore`?**
Every state change from a `send`, and every action produced by an
effect, must be explicitly accounted for in the test, or the test fails
— catching bugs that only-check-final-state testing would miss.

**45. When would you turn off exhaustivity, and what's the tradeoff?**
For large or loosely-scoped tests that intentionally only care about one
outcome; the tradeoff is losing some of TCA's automatic bug-catching for
unaccounted-for changes.

**46. Why must most `Action` payloads be `Sendable` under Swift 6?**
Because actions are often created inside a background `Task` (a `.run`
effect) and sent into the `@MainActor`-isolated `Store`, crossing a
concurrency boundary that Swift 6 requires to be safe at compile time.

**47. Why is `Store` marked `@MainActor`?**
To guarantee state mutation and SwiftUI observation always happen on the
main thread, matching SwiftUI's requirements and turning a whole class of
background-thread UI bugs into compile-time errors.

**48. What is the purpose of the `Dependencies/` folder in a production
TCA project structure?**
It holds every injectable service interface and its live/preview/test
implementations, separate from both low-level networking plumbing and
feature-specific code.

**49. Why shouldn't feature modules depend directly on each other?**
It breaks independent buildability/testability; communication should
happen via delegate actions or `@Shared`, composed together only at the
app level.

**50. What is the Dependency Rule from Clean Architecture, and how does
it apply to TCA project structure?**
Source code dependencies should only point inward/toward more
fundamental layers; in a TCA project, `Core`/`Networking` should never
depend on `Features`, only the reverse.

**51. What's the difference between `Scope` and `.ifLet`/`.forEach`?**
`Scope` composes a child feature that is always present (a fixed
property); `.ifLet`/`.forEach` compose a child feature that is
optionally present (`@Presents`) or present in a collection (`StackState`
or an array of child states).

**52. How would you model a shopping cart's total price in State?**
As a computed property derived from the cart's items, not as a separately
stored, manually-synced value.

**53. Why is `LoadingState<Value>` (idle/loading/loaded/failed) often
better than three separate booleans?**
It makes invalid combinations (like loading and loaded at once)
impossible to represent, whereas independent booleans allow any
combination, valid or not.

**54. What does `withDependencies` do at `Store` construction?**
Overrides specific dependency values for that Store and every child
feature composed into it, unless a child overrides them again further
down.

**55. Why is testing considered one of TCA's biggest advantages?**
Because pure reducers, effects-as-values, and injectable dependencies
together make it possible to deterministically test complete feature
behavior — state changes and side effects — without touching UI or the
network.

## Senior (56-80)

**56. Walk through, internally, what happens when `store.send` is
called.**
The action is handed to the Store's processing, which calls the
reducer's `reduce(into:action:)` synchronously on the main actor with
`state` as `inout`; the reducer mutates state directly and returns an
Effect; if not `.none`, the Store wraps it in a tracked, cancellable
`Task`; any `send` calls from that Task re-enter the same pipeline.

**57. Why does TCA process actions synchronously and in order?**
To guarantee full traceability — every state change can be attributed to
exactly one action, processed in a known order, which is essential for
debugging, testing, and reasoning about the app.

**58. Explain copy-on-write and why it matters for TCA's performance.**
Swift value types share underlying storage until one copy is mutated, at
which point it's copied; this lets `inout` mutation of large State
structs stay efficient, since unrelated copies of unchanged data are
never actually duplicated in memory.

**59. What specific problem does `@ObservableState`'s property-level
tracking solve compared to Combine's `objectWillChange`?**
It prevents unrelated Views from re-rendering when a State property they
don't read changes, since Combine's single "will change" signal fired
for any mutation regardless of which View cared about it.

**60. Describe how a Swift macro like `@Reducer` actually works.**
It runs at compile time, transforming the annotated declaration into
additional, ordinary Swift code (protocol conformances, generated types,
routing logic) that you could have written by hand — Xcode's "Expand
Macro" shows exactly what's generated.

**61. How would you architect a feature where five sub-features must all
independently know about the user's login state?**
Use `@Shared` for the login/user state so every feature can observe and
react to it without needing a direct parent/child relationship or manual
threading through every intermediate layer.

**62. What's the risk of overusing `@Shared`, and how would you decide
when it's appropriate?**
Overuse erodes traceability, since more than one reducer can write to the
same state with no single clear owner; use it only when two genuinely
unrelated parts of the app need the same live data, and prefer normal
composition otherwise.

**63. How does TCA's Effect cancellation interact with Swift's
structured concurrency?**
Each `.run` effect is backed by a real, structured `Task`; cancelling it
via `.cancel(id:)` triggers Swift's standard cancellation propagation,
so any `await` inside the effect throws `CancellationError` at its next
suspension point.

**64. Design a state shape for a paginated list with pull-to-refresh,
"load more," and error handling for each.**
Something like: `items: IdentifiedArrayOf<Item>`, `nextPageStatus:
LoadingState<Void>`, `refreshStatus: LoadingState<Void>`, `currentPage:
Int` — separating the two independent async operations (refresh vs.
load-more) into their own status fields prevents them from interfering.

**65. How would you test a feature with a debounced search, a loading
spinner, and an error state, fully exhaustively?**
Use `TestStore` with `TestClock` injected as `continuousClock`, override
`searchClient` per scenario, `send` the query-changed bindings, advance
the clock past the debounce window, and `receive` the resulting response
action for both success and failure cases, asserting exact state after
each step.

**66. What's the tradeoff between splitting a large `Path` enum into
several smaller root-level flows versus one big `Path`?**
Smaller, focused flows are easier to reason about and less likely to mix
unrelated concerns, but too much splitting can duplicate common
navigation infrastructure; the right split usually follows genuinely
distinct user-facing flows (e.g. onboarding vs. main app).

**67. How would you migrate a VIPER-based screen to TCA incrementally,
in a large legacy app?**
Rewrite one screen at a time as a `@Reducer` feature with its own
State/Action, keep VIPER's Router-equivalent responsibilities as
TCA-native navigation state, and bridge communication between
still-VIPER and already-TCA screens through a thin adapter layer until
the whole flow is migrated.

**68. What does it mean for a reducer to be "pure," and where exactly
does purity break down if you're not careful?**
Given the same State and Action, it always produces the same new State
and Effect description, with no side effects performed directly; purity
breaks if you call non-deterministic APIs (`Date()`, `UUID()`) or perform
real I/O directly inside `Reduce` instead of describing it as an Effect.

**69. How do you decide whether something belongs in `Core` versus
`Shared` in a production folder structure?**
`Core` holds framework-agnostic data/logic with no UI dependency; `Shared`
holds reusable *UI* building blocks (design tokens, small dumb Views) and
does import SwiftUI — the split is data/logic vs. presentation.

**70. Explain a realistic scenario where `.concatenate` is required
instead of `.merge`.**
Saving a record locally, then syncing it to a server only if the local
save succeeded — `.concatenate` guarantees the sync effect doesn't start
until the local save effect has fully completed.

**71. How would you handle an effect that needs to both debounce user
input and be cancellable by a separate "cancel search" button?**
Give the search effect a stable cancellation ID used both by
`.debounce(id:...)` and by an explicit `.cancel(id:)` returned from the
cancel button's action — both operate on the same underlying
cancellation registry.

**72. What Swift 6 concurrency concept most directly explains why
`@Dependency` clients are usually written as structs of closures rather
than classes?**
`Sendable` — a struct of plain, stateless closures is trivially safe to
pass across concurrency boundaries, whereas a class with mutable internal
state would need additional synchronization (e.g., an actor) to satisfy
strict concurrency checking.

**73. How would you structure tests for a feature composed of three
child features, without re-testing each child's internal logic in the
parent's test?**
Test each child feature's reducer independently and exhaustively in its
own test file; in the parent's tests, focus on composition-specific
behavior — routing, delegate action handling, and how the parent reacts
to child events — rather than re-verifying each child's internal logic.

**74. What is the practical effect of forgetting `.ifLet` for a
`@Presents` property?**
The child feature's actions are never routed to its reducer — the screen
may still visually appear (if the View layer is wired to render it), but
none of its logic will run, since routing depends entirely on this
composition call.

**75. Why does TCA favor closure-based dependency clients (`struct` of
function properties) over protocol-based dependency injection?**
Struct-of-closures avoids subclassing/protocol-conformance machinery,
lets you construct alternate implementations as plain values (helpful for
per-test overrides of just one method), and composes naturally with
`Sendable`/value-type requirements.

**76. Describe how you'd add offline support to a TCA feature that
currently only fetches from a live API.**
Introduce a `PersistenceClient` dependency for local storage, adjust the
reducer to read cached data first (optimistic UI), fire a `.run` effect
to fetch fresh data, and on success both update state and write through
to the persistence client; on failure, fall back to showing cached data
with an "offline" indicator.

**77. What's a realistic reason a senior engineer might choose MVVM over
TCA for a specific new project despite knowing TCA well?**
A small, short-lived project, a tight deadline with a team unfamiliar
with TCA, or a genuinely simple app unlikely to grow — the learning
curve and ceremony cost may outweigh TCA's benefits for that specific
context (Chapter 17).

**78. How does scoping (`store.scope`) affect memory management for
child features?**
A scoped Store shares the same underlying data/effect machinery as its
parent rather than duplicating it; when the parent's relevant state
(e.g. an `@Presents` property) becomes nil or the owning feature is torn
down, normal reference counting deallocates the scoped Store and cancels
its remaining effects.

**79. Explain why `IdentifiedArrayOf` is preferred over a plain array
plus a separate dictionary for ID lookups.**
`IdentifiedArrayOf` keeps ordering and by-ID lookup consistent
automatically in one collection, avoiding the bug class where a
hand-maintained array and dictionary drift out of sync with each other.

**80. What's the architectural argument for keeping `Networking/` and
`Dependencies/` as separate layers?**
`Networking` provides low-level, reusable HTTP plumbing; `Dependencies`
provides feature-facing, injectable, swappable interfaces built on top of
it — separating them avoids duplicating request/error-handling logic
across every dependency file and keeps the injectable surface small and
focused.

## Architect (81-100)

**81. How would you design a TCA-based app to support both a phone
target and a widget extension sharing business logic?**
Put framework-agnostic State/Action/Reducer logic and dependency
interfaces in a shared Swift package with no UIKit/SwiftUI dependency;
the phone app and the widget extension each link that package and build
their own thin View layers on top of the shared Stores/reducers.

**82. What would you consider before adopting TCA for a 100-person
engineering organization migrating from a mixed MVC/MVVM codebase?**
Learning curve and ramp-up time across the org, a realistic
feature-by-feature migration plan (Chapter 17's decision framework),
tooling/CI implications (build times, modularization strategy from
Chapter 15), and whether the specific pain points TCA solves (testing,
navigation consistency, state predictability) match the org's actual
problems.

**83. How do you prevent "God feature" reducers from forming as an app
scales to hundreds of screens?**
Enforce feature-based modularization (Chapter 15) with narrow,
compiler-checked module boundaries; use delegate actions and `@Shared`
deliberately rather than reaching for tight coupling; and review new
features against boundaries like "can this module be deleted without
breaking unrelated ones."

**84. Design the dependency graph for a modularized TCA app with
CoreKit, NetworkingKit, DependenciesKit, and five feature modules. What
rules would you enforce, and how?**
Features depend on DependenciesKit/CoreKit/DesignSystem but never on each
other; DependenciesKit depends on NetworkingKit and CoreKit;
NetworkingKit depends only on CoreKit. Enforce via Swift Package Manager
target dependency declarations (a genuine compiler error results from a
disallowed `import`), and optionally CI checks/lint rules for
architecture conformance.

**85. How would you evaluate whether a specific team's TCA codebase has
become "too composable" (too many tiny features)?**
Look for excessive routing/delegate boilerplate relative to actual logic
per feature, features that only ever appear composed together and are
never independently reusable/testable, and rising cognitive overhead
tracing a single user flow across many tiny files — sometimes a slightly
larger, cohesive feature is the right call over maximal decomposition.

**86. What is your strategy for testing a feature that depends on
several other features' delegate actions, without the test suite
becoming brittle to internal changes in those other features?**
Assert on the delegate action's public contract (its case and associated
values) rather than internal state of the other feature, keep each
feature's own exhaustive tests responsible for verifying its internals,
and treat delegate actions as the stable "API surface" between features
in tests just as in production code.

**87. How would you approach introducing TCA gradually into an existing
MVVM app, screen by screen, without a risky big-bang rewrite?**
Start with a new or actively-changing screen as a pilot, wrap
communication with the existing MVVM navigation/DI (a Coordinator can
still push a TCA-backed View, and a TCA feature can emit a delegate
action a legacy Coordinator listens to), and expand feature by feature as
confidence and team familiarity grow.

**88. What's your position on using `@Shared` for cross-cutting concerns
like feature flags or the current user, at the architecture level?**
It's a reasonable, intentional use of `@Shared` since these are
genuinely global, read-mostly values needed across many unrelated
features; the key architectural discipline is treating writes to shared
state as deliberate, well-defined operations (often owned by one specific
feature, like an AuthFeature owning writes to the current user) rather
than allowing arbitrary features to mutate it freely.

**89. How would you design for testability and traceability in an
effect that must coordinate three different backend calls with partial
failure handling (e.g., two succeed, one fails)?**
Model each sub-call's outcome explicitly (e.g. an array or struct of
per-call `Result`s) rather than collapsing to one overall success/failure,
dispatch a single response action carrying that structured outcome, and
let the reducer decide how to render partial failures — keeping the
whole coordination logic inside one testable `.run` effect using
`TaskGroup` or `async let`.

**90. What tradeoffs would you weigh when deciding whether a piece of
state should live in `@Shared(.fileStorage(...))` versus a dedicated
backend-synced feature?**
File-backed `@Shared` state is simple and fully local, appropriate for
device-only preferences; backend-synced state needs conflict resolution,
network awareness, and offline handling, and is usually better modeled
as its own feature with explicit sync effects rather than transparent
shared persistence.

**91. How would you architect deep linking for an app with both
authenticated and unauthenticated deep link targets?**
Parse the URL early (at the App-level reducer) into an intent value, and
route it through an "await authentication, then apply" mechanism — e.g.
store the pending navigation intent in `AppFeature.State`, and apply it
(building the appropriate `state.path`) either immediately if already
authenticated or after a successful `.delegate(.didLogIn)` from the auth
flow.

**92. Explain how you'd measure and address a real performance
regression reported in a large, deeply-composed TCA app.**
Reproduce and measure with Instruments (SwiftUI instrument, Time
Profiler) first — never guess; check for overly broad Store scoping,
un-cancelled long-running effects, redundant state writes in
high-frequency reducers, and oversized flat State structs, applying only
the fix that matches what was actually measured (Chapter 18).

**93. What governance/process would you put in place to keep a large,
multi-team TCA codebase consistent (e.g. consistent action naming,
dependency usage, testing standards)?**
A shared style guide (naming conventions from Chapter 5, testing
expectations from Chapter 13), lint rules or code review checklists
referencing the common-mistakes list (Chapter 19), a small
platform/architecture team owning `Core`/`Dependencies`/`DesignSystem`
modules, and periodic architecture reviews for new feature modules.

**94. How would you decide between building a custom, in-house
Redux-style architecture versus adopting TCA for a new, large iOS
platform team?**
Weigh the cost of building and maintaining equivalent tooling (testing
infrastructure, navigation, dependency injection, macros) in-house
against TCA's existing, actively maintained implementation and community;
generally, adopting a mature, well-tested open-source solution like TCA
is favored unless there's a very specific, unmet requirement that
justifies the ongoing maintenance cost of a custom system.

**95. Describe a scenario where TCA's strict, unidirectional design
would genuinely be a poor fit, even for a large app.**
An app dominated by highly specialized, performance-critical, imperative
rendering (e.g. a real-time drawing/animation tool, a game) where the
overhead of routing every micro-interaction through Action/Reducer would
add friction without proportional benefit — such apps often use a
lighter, more direct state model for the performance-critical core, with
TCA (if used at all) managing only the surrounding, more traditional app
chrome.

**96. How would you handle a situation where two features both need to
own writes to overlapping shared state, causing confusing update order
issues?**
Redesign ownership so exactly one feature is the authoritative writer
(others become read-only observers via `@Shared` or delegate actions
reporting requests to change it), which restores the single-writer
discipline that made the state's behavior easy to reason about in the
first place.

**97. What's your approach to onboarding a team of MVVM-experienced iOS
engineers onto TCA within a fixed timeframe (e.g. one sprint)?**
Start with the Chapter 1-4 mental model (why architecture, why
unidirectional flow) before any syntax, build one small real feature
together end-to-end (mirroring Chapter 7), pair on writing its
`TestStore` tests immediately (to build the "testing changes everything"
intuition early), and only then let engineers work independently on
larger features with code review focused on the common mistakes in
Chapter 19.

**98. How would you design a feature module boundary for a "Notifications"
capability used by both a Home feature and a Settings feature,
avoiding direct coupling between Home and Settings?**
Extract a `NotificationsClient` dependency (and possibly a small shared
`NotificationsCore` module for models) that both `HomeFeature` and
`SettingsFeature` depend on independently; neither feature imports the
other, and any cross-feature effect (e.g. Settings changing a preference
that Home's badge count should reflect) flows through `@Shared` state or
the App-level composition, not a direct import.

**99. What would you look for when reviewing a pull request that
introduces a new TCA feature, at an architect level?**
State shape (single source of truth, no duplicated/derivable stored
values), action design (specific, traceable, proper delegate cases if
reusable), effect discipline (cancellation IDs, injected dependencies, no
direct system API calls), test coverage (exhaustive happy-path and
failure-path tests), and module/folder placement consistent with the
project's established boundaries (Chapter 15).

**100. In one paragraph, how would you explain to a skeptical engineer
why TCA's upfront complexity is worth it for a large, long-lived app?**
Every piece of TCA's ceremony — a dedicated State type, an exhaustive
Action enum, effects as inspectable values, injected dependencies — exists
to make a specific, common class of large-app bug impossible or
immediately visible: state drifting out of sync, untestable business
logic, inconsistent side-effect handling, and ad hoc feature composition.
The cost is paid once, upfront, per feature; the benefit compounds every
time that feature is touched again by someone else, months or years
later, who can trust the tests, trace any state change to its cause, and
safely compose it into something bigger — which is exactly the kind of
codebase a large, long-lived app needs to keep moving fast without
accumulating unmanageable risk.

---

**Previous:** [Chapter 19 — Common Mistakes](19-Common-Mistakes.md)
**Next:** [Chapter 21 — Build a Production App](21-Build-a-Production-App.md)
