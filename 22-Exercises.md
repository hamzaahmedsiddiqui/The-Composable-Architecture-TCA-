# Exercises

Sixty exercises — twenty beginner, twenty intermediate, twenty advanced —
with complete solutions at the end of the chapter. Try each exercise
yourself before checking the solution; the solutions are written to be
read after a genuine attempt, not as a substitute for one.

## Beginner Exercises (1-20)

1. Define `State` and `Action` for a "Like button" feature that tracks
   whether a post is liked and how many total likes it has.
2. Write the reducer for Exercise 1: tapping the button toggles `isLiked`
   and increments/decrements `likeCount` accordingly.
3. Add `Equatable` conformance to a `State` struct that currently lacks
   it, and explain why it was missing something important.
4. Write a `CounterFeature` variant that also has a "Reset to 0" action.
5. Convert three independent booleans (`isLoading`, `isLoaded`,
   `hasError`) into a single `LoadingState<Value>` enum for a profile
   screen.
6. Write a computed property `isOverBudget: Bool` on a `BudgetFeature.State`
   that has `spent: Decimal` and `limit: Decimal` stored properties.
7. Given a `TodoFeature` with `todos: IdentifiedArrayOf<Todo>`, write the
   action and reducer case to mark a specific todo (by ID) as done.
8. Write a minimal SwiftUI View for the Like button feature from
   Exercises 1-2, following the "read State, send Action" rule.
9. Explain, in your own words (2-3 sentences), why the View in Exercise 8
   should not contain any logic deciding whether liking is allowed.
10. Define a `DependencyKey` for a `ClipboardClient` with one method,
    `copy: (String) -> Void`, including `liveValue` and `testValue`.
11. Register the `ClipboardClient` from Exercise 10 as a dependency on
    `DependencyValues`.
12. Write a reducer case that uses `@Dependency(\.clipboardClient)` to
    copy a piece of text when a "Copy" button is tapped.
13. Write a `.none`-returning reducer case and a `.run`-returning reducer
    case side by side, and explain when each is appropriate.
14. Given `Action: BindableAction` with a `query: String` field, wire up
    `BindingReducer()` for a simple search text field (no debounce yet).
15. Write a `Store` construction line (`Store(initialState:) { }`) for a
    `CounterFeature`, both for a real app entry point and for a
    `#Preview`.
16. Explain the difference between `state.count += 1` inside a reducer
    and mutating `count` from inside a SwiftUI View directly.
17. Write a `TestStore`-based test asserting that
    `.incrementButtonTapped` changes `count` from `0` to `1`.
18. Add a second assertion to the test in Exercise 17 for a subsequent
    `.decrementButtonTapped` bringing `count` back to `0`.
19. List, in order, the five steps that happen internally between a
    `store.send(action)` call and a `.run` effect calling `send` again.
20. Explain, in 2-3 sentences, why a TCA reducer must not call
    `URLSession.shared` directly, using vocabulary from this book
    (purity, testability, traceability).

## Intermediate Exercises (21-40)

21. Build a complete `WeatherFeature` (State, Action, Reducer) that
    fetches weather for a city name, using `LoadingState<Weather>` and an
    injected `WeatherClient` dependency.
22. Write `liveValue`, `previewValue`, and `testValue` for the
    `WeatherClient` from Exercise 21.
23. Build a `RatingFeature` where tapping one of five stars sets a
    `rating: Int` in State, using an explicit action rather than a
    binding, and explain why an explicit action is a reasonable choice
    here.
24. Add debounced, cancellable search to a `UserSearchFeature` that
    calls a `UserSearchClient`, following Chapter 8/11's search pattern.
25. Extend Exercise 24 so that clearing the search field (empty string)
    cancels any in-flight search and clears the results, without firing
    a new request.
26. Build a `ShoppingCartFeature` with `addItem`, `removeItem`, and a
    computed `total: Decimal`, and write a test verifying the total
    updates correctly after two adds and one removal.
27. Design `State` for an "Onboarding" flow with three sequential steps,
    each with its own small sub-feature, composed with `Scope`.
28. Add a `delegate(.didFinishOnboarding)` action to the composed
    Onboarding feature from Exercise 27, sent after the third step
    completes.
29. Build an `AddItemFeature` presented as a sheet from an
    `InventoryFeature`, using `@Presents`/`PresentationState` and
    `.ifLet`.
30. Add a `delegate(.didSave(item:))` action to `AddItemFeature` from
    Exercise 29, and wire the parent to append the item and dismiss the
    sheet.
31. Build a `NavigationStack`-based flow with `StackState`/`StackAction`
    where tapping a list row pushes a `DetailFeature` screen.
32. Add a second, different feature type (`EditFeature`) to the same
    stack from Exercise 31, using a `@Reducer enum Path`.
33. Write a `TestStore` test that sends an action pushing a detail
    screen and asserts the exact resulting `state.path`.
34. Build a `TimerFeature` that starts a repeating one-second timer
    effect on `startButtonTapped` and can be stopped with
    `stopButtonTapped`, using a stable cancellation ID.
35. Write a `TestStore` test for `TimerFeature` from Exercise 34 using
    `TestClock`, asserting three ticks occur after advancing three
    seconds.
36. Build a `ProfileFeature` that loads a profile on `.onAppear`, lets
    the user edit `name`/`bio` locally, and saves only on an explicit
    `saveButtonTapped`, without saving on every keystroke.
37. Write both a success-path and a failure-path `TestStore` test for
    the `saveButtonTapped` action in Exercise 36.
38. Refactor a reducer with a 40-line single `case` handling both
    validation and a network call into two clearer, smaller pieces
    (an explicit validation step plus a separate effect-producing case),
    explaining your reasoning.
39. Build a `SettingsFeature` composed of two independent sub-features
    (`NotificationSettingsFeature`, `AccountSettingsFeature`) using
    `Scope`, each with its own `BindingReducer`.
40. Write a non-exhaustive `TestStore` test (`store.exhaustivity = .off`)
    for a large, multi-field settings feature, verifying only one
    specific field changes correctly, and explain why you chose
    non-exhaustive testing here specifically.

## Advanced Exercises (41-60)

41. Design and implement `@Shared` state for a "current user" value read
    by three unrelated features (a profile badge, a comments feature,
    and a settings feature), including choosing an appropriate
    persistence strategy.
42. Write a test proving that mutating the shared state from Exercise 41
    in one feature's `TestStore` is visible to another feature reading
    the same shared key.
43. Design a paginated list feature (`fetchPage(page:)`) with proper
    "load more on scroll," a `hasMorePages` flag, and a test proving a
    second page's items are appended, not replacing the first page.
44. Add response caching to Exercise 43 using a `PersistenceClient`
    dependency, and a fallback to cached data when the first page's
    network request fails.
45. Write a `TestStore` test proving the caching fallback from Exercise
    44 works correctly when the dependency is configured to throw.
46. Implement effect cancellation for an image-loading feature so that
    rapidly changing the requested URL never lets a stale image
    overwrite a newer one, and write a test proving out-of-order
    responses cannot occur.
47. Build a `@Reducer enum Path`-based navigation flow with at least
    three different feature types, including one that itself presents a
    `@Presents` sheet (nested navigation).
48. Implement deep linking for the flow in Exercise 47: given a URL,
    construct the exact `State` that represents having already
    navigated to a specific nested screen, and write a test for it.
49. Design a dependency interface for an `AnalyticsClient` used by five
    different features without those features depending on each other,
    and explain your module boundary choices.
50. Build a `TaskGroup`-based effect that fetches details for a dynamic,
    runtime-determined list of IDs in parallel, and write a test that
    verifies all results are collected regardless of completion order.
51. Refactor a feature with four unrelated `@Published`-style booleans
    (imagine it started as MVVM) into a clean TCA `LoadingState`-based
    design, explaining each change.
52. Design a production folder structure (per Chapter 15) for a
    hypothetical "recipe app" with Auth, Browse, Recipe Detail,
    Favorites, and Settings features, listing which folders each feature
    is allowed to depend on.
53. Identify and fix three planted bugs in a given (imaginary) reducer
    snippet that mixes up cancellation IDs, forgets `await` before
    `send`, and stores a derived value instead of computing it. (Write
    your own "buggy" snippet first, then fix it, to practice spotting
    these in review.)
54. Design a `Sendable`-safe `liveValue` for a dependency that needs
    internal, thread-safe caching (e.g. an in-memory response cache),
    using an `actor`.
55. Compare your solution to Exercise 54 against a version using plain,
    immutable value types passed by copy instead of an `actor`. Explain
    which approach you'd choose for a real caching dependency, and why.
56. Write an exhaustive `TestStore` test suite (happy path + failure +
    cancellation) for a debounced search feature that also caches
    previous query results to avoid redundant network calls.
57. Design the `AppFeature.State` "session" state machine for an app with
    three states — logged out, logged in but unverified email, fully
    verified — and the transitions between them, following Chapter 21's
    enum-based session pattern.
58. Explain and diagram (in Mermaid) the full dependency graph you would
    set up for a modularized version of Chapter 21's TaskFlow app as
    separate Swift packages.
59. Design a strategy for testing a feature that depends on three other
    features' delegate actions without making the test suite brittle to
    unrelated internal changes in those features (see Interview Question
    86 for the conceptual answer — implement it as an actual test).
60. Write a short (150-300 word) explanation, as if answering a senior
    interview question, of when you would recommend against adopting
    TCA for a specific hypothetical project of your choosing, and why.

---

## Solutions

### Beginner Solutions

**1-2.**
```swift
@Reducer
struct LikeFeature {
    @ObservableState
    struct State: Equatable { var isLiked = false; var likeCount = 0 }
    enum Action { case likeButtonTapped }
    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .likeButtonTapped:
                state.isLiked.toggle()
                state.likeCount += state.isLiked ? 1 : -1
                return .none
            }
        }
    }
}
```

**3.** Without `Equatable`, `TestStore` cannot compare expected vs. actual
state after each `send`/`receive`, and SwiftUI's diffing is less
effective — both rely on `==`.

**4.**
```swift
case resetButtonTapped
// ...
case .resetButtonTapped: state.count = 0; return .none
```

**5.** `var status: LoadingState<Profile> = .idle` replacing the three
booleans; every place that previously set two of the three booleans now
sets `status` to exactly one case.

**6.** `var isOverBudget: Bool { spent > limit }`.

**7.**
```swift
case toggleDone(id: Todo.ID)
// ...
case let .toggleDone(id): state.todos[id: id]?.isDone.toggle(); return .none
```

**8.**
```swift
struct LikeView: View {
    let store: StoreOf<LikeFeature>
    var body: some View {
        Button {
            store.send(.likeButtonTapped)
        } label: {
            Label("\(store.likeCount)", systemImage: store.isLiked ? "heart.fill" : "heart")
        }
    }
}
```

**9.** Deciding *whether* liking should be allowed (e.g. rate limits,
already-liked state) is business logic; it belongs in the reducer as
computed state or a guard, so it stays testable without rendering the
View, and stays consistent no matter how many Views trigger the action.

**10.**
```swift
struct ClipboardClient { var copy: (String) -> Void }
extension ClipboardClient: DependencyKey {
    static let liveValue = ClipboardClient(copy: { UIPasteboard.general.string = $0 })
    static let testValue = ClipboardClient(copy: unimplemented("ClipboardClient.copy"))
}
```

**11.**
```swift
extension DependencyValues {
    var clipboardClient: ClipboardClient {
        get { self[ClipboardClient.self] }
        set { self[ClipboardClient.self] = newValue }
    }
}
```

**12.**
```swift
@Dependency(\.clipboardClient) var clipboardClient
case .copyButtonTapped: clipboardClient.copy(state.text); return .none
```

**13.** `.none` is appropriate for pure state mutation with no outside
work; `.run` is appropriate whenever async work (network, timers, disk)
must happen, since that work cannot happen synchronously inside the pure
reducer.

**14.**
```swift
enum Action: BindableAction { case binding(BindingAction<State>) }
var body: some ReducerOf<Self> { BindingReducer() }
```

**15.**
```swift
// App entry point
Store(initialState: CounterFeature.State()) { CounterFeature() }
// Preview
Store(initialState: CounterFeature.State(count: 5)) { CounterFeature() }
```

**16.** `state.count += 1` inside a reducer is the one sanctioned,
traceable way to change state, triggered by a specific Action; mutating
`count` directly from a View bypasses the reducer entirely, breaking
traceability and testability (there would be no Action to point to as
the cause).

**17-18.**
```swift
@Test func incrementThenDecrement() async {
    let store = TestStore(initialState: CounterFeature.State()) { CounterFeature() }
    await store.send(.incrementButtonTapped) { $0.count = 1 }
    await store.send(.decrementButtonTapped) { $0.count = 0 }
}
```

**19.** (1) `send` hands the action to the Store; (2) the Store calls the
reducer with `(inout state, action)`; (3) the reducer mutates state and
returns an `Effect`; (4) if not `.none`, the Store runs it as a tracked
`Task`; (5) when that Task calls `send` again, it re-enters step 1.

**20.** Calling `URLSession.shared` directly breaks purity (the reducer's
output would depend on network timing, not just its inputs), breaks
testability (a test would need a real network call to exercise that
code path), and breaks traceability (the response would arrive outside
the Action-driven loop, since nothing dispatched it as an Action).

### Intermediate Solutions

**21-22.** Follow Chapter 8, Section 8.3's `WeatherFeature` exactly —
`State` with `city: String`, `weather: LoadingState<Weather>`; `Action`
with `searchButtonTapped` and `weatherResponse(Result<Weather, EquatableError>)`;
`WeatherClient` with `fetch: (String) async throws -> Weather`, a real
`liveValue` hitting a weather API, a `previewValue` returning fixed data,
and a `testValue` using `unimplemented`.

**23.** An explicit `case starTapped(Int)` documents clear business
intent ("the user rated this content N stars") better than a generic
binding would, and makes the action self-descriptive in tests and logs.

**24-25.** Follow Chapter 11, Section 11.9's debounce pattern; add
`guard !query.isEmpty else { state.results = []; return .cancel(id: CancelID.search) }`
before starting a new debounced effect, exactly as in Chapter 8's search
example.

**26.**
```swift
@Test func totalUpdatesCorrectly() async {
    let store = TestStore(initialState: ShoppingCartFeature.State()) { ShoppingCartFeature() }
    // send two addItem actions, one removeItem action, then assert
    // state.total via a final state check (since total is computed,
    // asserting the underlying items after each send is sufficient).
}
```

**27-28.** Compose three `Scope`s in `OnboardingFeature.body`; after the
third step's own "next" action fires and no more steps remain, return
`.send(.delegate(.didFinishOnboarding))` from the parent's `Reduce`.

**29-30.** Follow Chapter 9, Section 9.2 and Chapter 10, Section 10.7
directly — `@Presents var addItem: AddItemFeature.State?`, `.ifLet`,
`case .addItem(.presented(.delegate(.didSave(let item))))` appending to
`state.items` and setting `state.addItem = nil`.

**31-33.** Follow Chapter 9, Section 9.3; the test:
```swift
await store.send(.itemRowTapped(id: 1)) {
    $0.path.append(.detail(DetailFeature.State(itemID: 1)))
}
```

**34-35.**
```swift
private enum CancelID { case timer }
case .startButtonTapped:
    return .run { send in
        for await _ in clock.timer(interval: .seconds(1)) { await send(.tick) }
    }
    .cancellable(id: CancelID.timer)
case .stopButtonTapped:
    return .cancel(id: CancelID.timer)
```
Test: inject `TestClock`, send `.startButtonTapped`, `await clock.advance(by: .seconds(3))`,
`receive(\.tick)` three times.

**36-37.** Follow Chapter 8, Section 8.8's Profile pattern; edits mutate
`state.profile?.name`/`.bio` directly on field-change actions, and only
`saveButtonTapped` triggers the `.run` effect calling
`profileClient.save`. Failure test overrides `profileClient.save` to
throw and asserts `state.isSaving == false` plus an error message
afterward.

**38.** Split into: a `case fieldsChanged` (or bindings) that only
updates local draft fields and recomputes `isFormValid`, and a separate
`case submitButtonTapped` that, guarded by `isFormValid`, kicks off the
`.run` effect — separating "track input" from "act on input" makes each
piece independently testable and readable.

**39.** Follow Chapter 8, Section 8.9 directly.

**40.** Set `store.exhaustivity = .off` when the feature has many
unrelated fields and the test intentionally only cares about one
specific interaction; document why in a comment so future readers don't
mistake it for laziness.

### Advanced Solutions

**41-42.** `@Shared(.appStorage("currentUser"))` (or `.fileStorage` for a
larger `User` object) declared identically in each feature's `State`;
test by constructing two `TestStore`s sharing the same
`@Dependency(\.defaultAppStorage)` (or shared in-memory container) and
asserting a mutation in one is visible when reading the shared value from
the other.

**43-45.** Follow Chapter 21, Section 21.5's `TaskListFeature` directly —
`fetchPage`, `hasMorePages`, `persistenceClient.saveCachedTasks` after
success, fallback to `loadCachedTasks()` on failure when `tasks.isEmpty`;
the failure test overrides `taskClient.fetchPage` to throw and asserts
`state.tasks` becomes the injected cached fixture.

**46.** Follow Chapter 8, Section 8.6 and Chapter 11, Section 11.8 — use
`.cancellable(id: CancelID.image, cancelInFlight: true)`; test by
overriding the image client so the *first* call's continuation is only
resumed after the second call has already been sent, then asserting only
the second image's data ends up in state (proving the first, stale
response was cancelled and could not overwrite it).

**47-48.** Follow Chapter 9, Sections 9.3 and 9.6; nested navigation
means one of the `Path` cases' own feature has an `@Presents` property of
its own — composition doesn't care how deep it goes. Deep link test:
send `.deepLink(url)` and assert the exact resulting `state.path`
(and, if nested, the nested feature's own presented state).

**49.** A small, focused `AnalyticsClient` (e.g. one `log(Event)` method)
in `Dependencies/`, imported by any feature module that needs it,
depending only on `Core` for the `Event` type — no feature imports
another feature to get analytics.

**50.** Follow Chapter 16, Section 16.8's `TaskGroup` pattern; test by
having each fake per-ID fetch closure return after an artificially
different, injected delay (or simply return immediate but
distinguishable values) and asserting the *set* of results is complete
and correct, not asserting a specific order.

**51.** Identify each boolean's real meaning, map the valid combinations
onto `LoadingState` cases (or a custom enum if the shape doesn't fit
idle/loading/loaded/failed exactly), and remove the now-redundant
booleans — every reducer case that touched two booleans should now touch
one enum assignment.

**52.** `Features/Auth`, `Features/Browse`, `Features/RecipeDetail`,
`Features/Favorites`, `Features/Settings`, each depending on
`Core`, `Dependencies`, and `Shared` (DesignSystem), never on each other;
`Favorites` and `RecipeDetail` might share a `Recipe` model from `Core`.

**53.** (Practice exercise — no single fixed answer; verify your fix
against Chapter 19's Mistakes #1, #24, and #26 respectively.)

**54.**
```swift
actor ResponseCache {
    private var storage: [String: Data] = [:]
    func get(_ key: String) -> Data? { storage[key] }
    func set(_ key: String, _ value: Data) { storage[key] = value }
}
// liveValue closures `await` into the actor for all reads/writes,
// satisfying Sendable without needing @unchecked.
```

**55.** An `actor`-based cache is preferable whenever the cache is
mutated frequently and concurrently (e.g. many parallel fetches racing to
populate it) — the actor serializes access safely. A plain,
immutable-value-type approach (e.g. rebuilding a `Sendable` struct/
dictionary and storing it via `@Shared` or similar) works well when
updates are infrequent or naturally sequential, avoiding actor-hopping
overhead for simple cases.

**56.** Combine Chapter 8's debounce test style (Exercise 24-25) with a
`resultsCache: [String: [String]]` in State (or a dedicated cache
dependency); assert that a repeated query within the same session sends
`.searchResponse` without a matching `taskClient`/`searchClient` call
(e.g. by making the fake client fail on a second call for the same
query, proving the cache was used).

**57.** `enum Session { case loggedOut(AuthFeature.State); case
unverified(UnverifiedFeature.State); case verified(MainTabsFeature.State) }`,
with transitions driven by delegate actions (`.didLogIn` →
`.unverified`, `.didVerifyEmail` → `.verified`, `.didLogOut` →
`.loggedOut` from any state).

**58.** Mirrors Chapter 15, Section 15.4's diagram, with TaskFlow's
specific modules: `CoreKit`, `NetworkingKit`, `DependenciesKit`
(containing `AuthClient`, `TaskClient`, `PersistenceClient`,
`ReachabilityClient`), and feature packages `AuthFeature`,
`TaskListFeature`, `ProfileFeature`, `SettingsFeature`, composed only by
the `TaskFlow` app target.

**59.** Write each dependent feature's tests against only the delegate
action's case/associated values (e.g. assert `AppFeature` correctly
transitions `session` on `.auth(.delegate(.didLogIn(token)))`) without
asserting anything about `AuthFeature`'s internal fields at that point in
the test — those are `AuthFeature`'s own test file's responsibility.

**60.** *(Open-ended — model your answer on Chapter 17, Section 17.5's
"When NOT to use TCA" list: pick a genuinely small, short-lived, or
deadline-constrained hypothetical project, and justify the recommendation
using the specific cost/benefit reasoning from that section, not just a
restatement of "it depends.")*

---

**Previous:** [Chapter 21 — Build a Production App](21-Build-a-Production-App.md)
**Back to:** [Index](00-Index.md)
