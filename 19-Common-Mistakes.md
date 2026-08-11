# Chapter 19 — Common Mistakes

Fifty-two real mistakes engineers make with TCA, grouped by topic, each
with wrong code, correct code, and why it matters. Treat this chapter as
a checklist to review against your own code.

## State Mistakes

### 1. Storing derived values instead of computing them

```swift
// Wrong
struct State: Equatable {
    var items: [Item] = []
    var total: Decimal = 0 // must be manually kept in sync
}
// Correct
struct State: Equatable {
    var items: [Item] = []
    var total: Decimal { items.reduce(0) { $0 + $1.price } }
}
```
A stored, duplicated value can drift out of sync with what it's derived
from; a computed property cannot.

### 2. Using loose booleans instead of a state machine enum

```swift
// Wrong
var isLoading = false
var isLoaded = false
var hasError = false
// Correct
var status: LoadingState<Data> = .idle
```
Three independent booleans can reach invalid combinations the compiler
cannot prevent; an enum can only ever be one case at a time.

### 3. Forgetting `Equatable` on `State`

```swift
// Wrong
struct State { var count = 0 }
// Correct
struct State: Equatable { var count = 0 }
```
Without `Equatable`, `TestStore` assertions and efficient diffing don't
work as intended.

### 4. Putting UIKit/SwiftUI-only types directly in State

```swift
// Wrong
struct State: Equatable { var color: Color = .red }
// Correct
struct State: Equatable { var colorName: String = "red" }
// (map colorName -> Color in the View layer)
```
Keeps `State` plain, testable data instead of coupling it to a specific
rendering framework.

### 5. One giant flat State struct for a whole app

```swift
// Wrong
struct State: Equatable {
    var email = ""; var cartItems: [Item] = []; var settingsToggle = false
    // ...200 more properties
}
// Correct
struct State: Equatable {
    var login = LoginFeature.State()
    var cart = CartFeature.State()
    var settings = SettingsFeature.State()
}
```
Nested, per-feature State scales; one flat struct becomes unreadable and
expensive to copy on every change.

### 6. Mutable reference types stored in `State`

```swift
// Wrong
struct State: Equatable { var cache = NSCache<NSString, UIImage>() }
// Correct
struct State: Equatable { var cachedImageIDs: Set<String> = [] }
```
Reference types can be mutated from outside the reducer, breaking the
single-source-of-truth guarantee and `Equatable` correctness.

### 7. Giving `State` no sensible default

```swift
// Wrong
struct State: Equatable { var name: String; var age: Int }
// Correct
struct State: Equatable { var name = ""; var age = 0 }
```
`State()` should always be constructible with sensible starting values
for previews, tests, and simple composition.

### 8. Mixing "editable draft" and "saved" values in one property

```swift
// Wrong
var profile: Profile // mutated directly while editing, hard to "cancel"
// Correct
var savedProfile: Profile
var draftProfile: Profile
```
Separating draft from saved state makes "Cancel" trivial (just discard
the draft) and keeps the last-known-good value available.

## Action Mistakes

### 9. One giant "replace everything" action

```swift
// Wrong
case setState(State)
// Correct
case incrementButtonTapped
case factResponse(Result<String, EquatableError>)
```
A catch-all action throws away traceability — you lose the ability to
know *what happened*, only that *something* did.

### 10. Vague, reused action names

```swift
// Wrong
case update // used for five unrelated things across the file
// Correct
case emailChanged(String)
case passwordChanged(String)
```
Specific names make `switch` statements, tests, and search results
meaningful.

### 11. Non-`Sendable` payloads in actions

```swift
// Wrong
case imageLoaded(UIImage) // UIImage is not reliably Sendable across contexts in strict mode
// Correct
case imageDataLoaded(Data) // convert to UIImage in the View layer
```
Swift 6 strict concurrency requires action payloads crossing actor
boundaries to be `Sendable`; keep them as simple value types.

### 12. Forgetting to model the failure case

```swift
// Wrong
case dataResponse(Data)
// Correct
case dataResponse(Result<Data, EquatableError>)
```
Every effect that can fail should have its failure explicitly
represented, not silently dropped.

### 13. Encoding UI-only concerns as feature Actions

```swift
// Wrong
case textFieldDidBeginEditing // pure UI focus event, no logic needed
// Correct: use SwiftUI's @FocusState locally in the View for this
```
Not everything needs to flow through the reducer; purely cosmetic,
non-business-logic UI state can stay local to the View.

### 14. Missing `delegate` case for reusable child features

```swift
// Wrong
// (parent pattern-matches AddItemFeature's internal .saveButtonTapped directly)
// Correct
case delegate(Delegate)
enum Delegate: Equatable { case didSave(name: String) }
```
Without a delegate case, the child is tightly coupled to a specific
parent's internal expectations, hurting reusability (Chapter 10).

## Reducer Mistakes

### 15. Calling async APIs directly inside `Reduce`

```swift
// Wrong
case .fetchButtonTapped:
    Task { let data = try await client.fetch(); ... } // fire-and-forget, untracked
    return .none
// Correct
case .fetchButtonTapped:
    return .run { send in
        await send(.response(await Result { try await client.fetch() }))
    }
```
Untracked `Task`s can't be cancelled, tested, or traced back to a
specific effect lifecycle.

### 16. Forgetting to return an Effect on every path

```swift
// Wrong (won't compile, but a near-miss version):
case .a: state.x = 1  // missing return
// Correct
case .a: state.x = 1; return .none
```
Every case must return an `Effect<Action>`.

### 17. Doing too much work in a single action handler

```swift
// Wrong: 150-line switch case doing validation, formatting, and three effects
// Correct: split into multiple, more specific actions and smaller effects
```
Long handlers are hard to test precisely and hard to reason about;
smaller, focused actions are easier to verify with `TestStore`.

### 18. Mutating unrelated state "while you're in there"

```swift
// Wrong
case .incrementButtonTapped:
    state.count += 1
    state.lastActivity = "increment" // unrelated concern, easy to forget consistency
    return .none
// Correct: keep unrelated concerns in their own action/handler, or make
// the relationship intentional and well-tested.
```
Bundling unrelated mutations into one case makes tests brittle and
intent unclear.

### 19. Not snapshotting values needed inside a later-running effect

```swift
// Wrong
case .fetchButtonTapped:
    return .run { send in
        await send(.response(try await client.fetch(state.id))) // 'state' captured, may have changed
    }
// Correct
case .fetchButtonTapped:
    let id = state.id
    return .run { send in await send(.response(try await client.fetch(id))) }
```
`state` may mutate again before the effect runs; capture the exact value
you need at the moment you need it.

### 20. Ignoring the ordering of composed reducers

```swift
// Wrong: parent's Reduce needs updated child state but runs BEFORE the Scope
Reduce { state, action in /* reads state.child, but child hasn't updated yet */ }
Scope(state: \.child, action: \.child) { ChildFeature() }
// Correct
Scope(state: \.child, action: \.child) { ChildFeature() }
Reduce { state, action in /* now sees updated state.child */ }
```
`body`'s reducers run in the order listed; order matters when one
depends on another's result for the same action (Chapter 10, Section
10.4).

### 21. Duplicating logic instead of composing

```swift
// Wrong: copy-pasting AddItemFeature's validation into EditItemFeature
// Correct: extract shared validation into a plain function or a shared
// small reducer both features use.
```
Defeats the entire purpose of a composable architecture.

### 22. Silently swallowing errors

```swift
// Wrong
case .response(.failure):
    return .none // user sees nothing
// Correct
case let .response(.failure(error)):
    state.errorMessage = error.localizedDescription
    return .none
```
Always surface failures in state so the UI can inform the user.

## Effect Mistakes

### 23. Not cancelling superseded effects

```swift
// Wrong: every keystroke starts a new, uncancelled search request
// Correct
return .run { ... }.cancellable(id: CancelID.search, cancelInFlight: true)
```
Leads to out-of-order response bugs (Chapter 11, Section 11.8).

### 24. Using a non-stable cancellation ID

```swift
// Wrong
.cancellable(id: UUID()) // different every time; cancel(id:) can never match it
// Correct
.cancellable(id: CancelID.search) // a stable, private enum case
```
A fresh UUID per call means no future `.cancel(id:)` can ever target it.

### 25. Debounce used where throttle is needed (or vice versa)

```swift
// Wrong: debouncing a "log scroll position" effect means it may never
// fire while the user keeps scrolling continuously.
// Correct: throttle it, so it fires periodically during continuous activity.
```
Pick the operator that matches the actual desired behavior (Chapter 11,
Section 11.9).

### 26. Forgetting `await` before `send`

```swift
// Wrong (compile error, but a common confusion point):
send(.response(result)) // missing await
// Correct
await send(.response(result))
```
`send` is `async`; it must be awaited to correctly interleave with the
Store's processing.

### 27. Not handling cancellation-aware errors distinctly

```swift
// Wrong: treats CancellationError the same as a real network failure,
// showing "Something went wrong" after the user simply navigated away.
// Correct
} catch is CancellationError {
    return // don't send a failure action for an intentional cancellation
} catch {
    await send(.response(.failure(EquatableError(error))))
}
```
Cancellation is expected and normal; don't surface it as a user-facing
error.

### 28. Using `.merge` when strict order is required

```swift
// Wrong: .merge(saveLocallyEffect, syncToServerEffect) when sync must
// only happen after a successful local save.
// Correct: .concatenate(saveLocallyEffect, syncToServerEffect)
```
`.merge` runs concurrently; `.concatenate` guarantees order (Chapter 11,
Section 11.7).

### 29. Long-running subscriptions never cancelled on teardown

```swift
// Wrong: a location-updates effect keeps running after the screen
// showing it has been dismissed.
// Correct: cancel it explicitly in response to the relevant dismissal
// action, using the same cancellation ID it was started with.
```
Wastes battery/CPU and can deliver actions to a feature no longer in use
(though the Store's own deallocation eventually cleans this up — don't
rely on that alone for effects that should stop earlier).

## Dependency Mistakes

### 30. Calling `Date()`/`UUID()` directly in a reducer

```swift
// Wrong
state.createdAt = Date()
// Correct
@Dependency(\.date.now) var now
state.createdAt = now
```
Makes tests non-deterministic; injected versions can be frozen in tests.

### 31. Calling `URLSession.shared` directly instead of an injected client

```swift
// Wrong
let (data, _) = try await URLSession.shared.data(from: url)
// Correct
@Dependency(\.apiClient) var apiClient
let data = try await apiClient.send(request)
```
Couples the reducer to a concrete networking implementation and makes it
untestable without hitting the real network.

### 32. Missing `testValue`, relying purely on crashes to notice

```swift
// Wrong: no testValue defined at all, so tests get a runtime trap with
// a confusing message.
// Correct: define testValue using unimplemented(...) for clear failures.
```
Clear, named failures save debugging time.

### 33. Dependency interfaces that are too large

```swift
// Wrong
struct AppClient { var login, logout, fetchUser, fetchPosts, uploadImage, ... }
// Correct: split into AuthClient, UserClient, PostsClient, ImageClient
```
Large interfaces are painful to fake fully in every test; small,
focused interfaces are easy to stub.

### 34. Sharing unprotected mutable reference state in `liveValue`

```swift
// Wrong
final class Cache { var storage: [String: Data] = [:] } // not thread-safe
// Correct: use an actor, or a Sendable, immutable data structure with
// explicit synchronization.
```
Can cause data races under Swift 6 strict concurrency, or silent bugs if
suppressed with `@unchecked Sendable` carelessly.

### 35. Overriding dependencies globally instead of per test/preview

```swift
// Wrong: mutating DependencyValues.liveValue directly at app startup
// for "testing convenience," affecting the real app.
// Correct: use withDependencies at Store construction, scoped to the
// specific test, preview, or build configuration that needs it.
```
Keeps overrides local and intentional instead of leaking into
production behavior.

## Testing Mistakes

### 36. Turning off exhaustivity by default

```swift
// Wrong
store.exhaustivity = .off // set in every single test "just in case"
// Correct: keep exhaustive testing as the default; opt out deliberately
// per test, with a clear reason.
```
Throws away most of TCA's bug-catching value.

### 37. Forgetting to account for a `receive`

```swift
// Wrong: sends an action that triggers an effect, never calls
// store.receive for the resulting action — test fails, and the failure
// is misread as "flaky" rather than investigated.
// Correct: always account for every action an effect produces.
```
This failure is usually correct and meaningful — investigate before
suppressing it.

### 38. Using real time instead of `TestClock`

```swift
// Wrong
try await Task.sleep(for: .milliseconds(350)) // in the test itself
// Correct
await clock.advance(by: .milliseconds(350)) // using an injected TestClock
```
Real sleeps make the test suite slow and can still be flaky under CI
load.

### 39. Hardcoding non-deterministic expected values

```swift
// Wrong
$0.id = UUID() // a fresh UUID, will never equal the one the reducer generated
// Correct
$0.id = UUID(0) // matches an overridden, deterministic uuid dependency
```
Pair deterministic dependency overrides (Chapter 12) with matching
deterministic expectations in assertions.

### 40. Testing implementation details instead of behavior

```swift
// Wrong: asserting a private helper function was called a specific
// number of times via ad hoc instrumentation.
// Correct: assert the resulting State and Actions, which are the
// actual public contract of the feature.
```
Tests should verify behavior, so they only break when behavior actually
changes.

### 41. Not testing the failure path

```swift
// Wrong: only ever testing the happy path for every network call.
// Correct: write a paired failure test for every effect that can throw.
```
Failure handling is exactly the kind of logic that regresses silently
without a test.

## Navigation Mistakes

### 42. Forgetting `.ifLet`/`.forEach`

```swift
// Wrong: declares @Presents var child but never calls .ifLet(\.$child, ...)
// Correct
.ifLet(\.$child, action: \.child) { ChildFeature() }
```
Without it, the child's actions are never routed to its reducer, even
though the screen may still appear.

### 43. Manually managing dismissal instead of using PresentationAction

```swift
// Wrong: a hand-rolled isPresented: Bool alongside separate child state,
// manually kept in sync.
// Correct: use @Presents var child: ChildFeature.State?, nil = dismissed.
```
`@Presents` already encodes "presented vs. not" as the optionality
itself — don't duplicate that with a second boolean.

### 44. A `Path` enum that mixes unrelated flows

```swift
// Wrong: one giant Path enum spanning onboarding, main app, and settings
// Correct: separate root states/paths per major, unrelated flow
```
Keeps each navigation flow's cases focused and easier to reason about.

### 45. Deep linking logic scattered across the View layer

```swift
// Wrong: parsing a URL directly inside a SwiftUI `.onOpenURL` closure and
// imperatively mutating navigation.
// Correct: send a .deepLink(url) action and let the reducer build the
// resulting State, so it's testable (Chapter 9, Section 9.6).
```
Keeps deep linking traceable and testable like everything else.

## Dependency Injection & Composition Mistakes

### 46. Feature modules depending on each other directly

```swift
// Wrong: import HomeFeature inside LoginFeature's module
// Correct: LoginFeature reports delegate(.didLogIn); the App module wires
// the transition to HomeFeature.
```
Breaks independent buildability/testability of features (Chapter 15,
Section 15.4).

### 47. Overusing `@Shared` for state that belongs to one feature

```swift
// Wrong: @Shared for a value only ever read/written by one screen
// Correct: keep it a plain, local State property
```
Sharing state that doesn't need to be shared adds unnecessary indirection
and makes ownership less clear.

### 48. Business logic (like validation) written inside the View

```swift
// Wrong
Button("Submit") {
    if email.contains("@") { store.send(.submitButtonTapped) }
}
// Correct: compute isFormValid in State, disable/enable based on it
```
Puts logic back in the untestable View layer — the exact Chapter 1
problem TCA is meant to solve.

### 49. Creating a new `Store` on every `body` evaluation

```swift
// Wrong
var body: some View {
    ChildView(store: Store(initialState: .init()) { ChildFeature() })
}
// Correct: hold the Store in the parent's State/derive via scope; never
// construct a fresh root Store inside a View's body.
```
Destroys and recreates the feature's entire state (and cancels its
effects) on every render — usually not the intended behavior at all.

### 50. Treating `Environment` (old TCA) patterns as still required

```swift
// Wrong: hand-writing a full custom Environment struct and threading it
// through every layer manually, in a brand-new, modern TCA project.
// Correct: use @Dependency/DependencyValues (Chapter 12).
```
Old tutorials and blog posts sometimes show the pre-2023 style; modern
projects should default to the macro-based, `@Dependency` approach.

### 51. Ignoring compiler warnings about `Sendable`

```swift
// Wrong: silencing every Sendable warning with @unchecked Sendable
// without understanding why it appeared.
// Correct: understand the actual data race risk, and fix the underlying
// design (usually: prefer value types, or isolate mutable state in an
// actor) before reaching for @unchecked.
```
`@unchecked Sendable` is an escape hatch, not a fix — overusing it
reintroduces the exact bugs Swift 6's concurrency checking exists to
prevent.

### 52. Not writing any tests at all, "because the reducer is simple"

```swift
// Wrong: shipping a reducer with zero TestStore coverage because "it's
// obviously correct."
// Correct: write at least the core happy-path and one failure-path test
// for every non-trivial reducer.
```
"Simple" reducers are exactly the ones most likely to be casually
modified later without anyone re-verifying behavior by hand — a test
suite is what catches that regression automatically.

## 19.1 Chapter Summary

Most of these fifty-two mistakes trace back to a small number of root
causes: breaking the pure-reducer/effect boundary (calling async APIs
directly, using `Date()`/`UUID()` directly), losing traceability (giant
actions, swallowed errors, missing delegate actions), skipping
cancellation discipline, or reintroducing coupling that composition is
meant to prevent (feature-to-feature dependencies, logic in the View).
Reviewing new TCA code against this list is a fast, concrete way to catch
the majority of real-world issues before they reach production.

---

**Previous:** [Chapter 18 — Performance](18-Performance.md)
**Next:** [Chapter 20 — Interview Questions](20-Interview-Questions.md)
