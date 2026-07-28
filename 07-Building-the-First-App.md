# Chapter 7 — Building the First App

Time to build something real. We will build the Counter + Fact app from
Chapters 5-6 as a complete, runnable Xcode project, file by file. Every
file's purpose is explained before its code, and every line is explained
after.

## 7.1 Setting up the project

1. Create a new Xcode project: **App**, name it `CounterDemo`, interface
   **SwiftUI**, language **Swift**.
2. Add the TCA package: **File → Add Package Dependencies…**, enter
   `https://github.com/pointfreeco/swift-composable-architecture`, add the
   `ComposableArchitecture` library to your app target.
3. Target Swift 6 language mode (Project Settings → Build Settings →
   Swift Language Version → Swift 6), and set the deployment target to
   iOS 17 or later (required for `@ObservableState`/Observation).

## 7.2 Why we split the project into these specific files

A one-screen demo *could* live in one file, but we will split it the way
a real, small TCA feature is organized in production, because the goal of
this chapter is to build the habit early:

```
CounterDemo/
├── CounterDemoApp.swift      <- app entry point, creates the root Store
├── Features/
│   └── Counter/
│       ├── CounterFeature.swift   <- State, Action, Reducer (the logic)
│       └── CounterView.swift      <- the SwiftUI View (the UI)
└── Dependencies/
    └── NumberFactClient.swift     <- the injectable networking dependency
```

This mirrors the "feature-based organization" idea from Chapter 2,
Section 2.10: everything about the Counter feature lives together, and
the dependency it needs lives in its own clearly-named place.

## 7.3 `NumberFactClient.swift` — the dependency

**Why this file exists:** it defines the *interface* for fetching a
number fact, plus three implementations (live, preview, test), completely
independent of the Counter feature itself. This means the Counter feature
never has to know or care whether it's talking to a real API or a fake
one — Chapter 5, Section 5.7 explained why this matters.

```swift
import ComposableArchitecture
import Foundation

struct NumberFactClient {
    var fetch: (Int) async throws -> String
}

extension NumberFactClient: DependencyKey {
    static let liveValue = NumberFactClient(
        fetch: { number in
            let url = URL(string: "http://numbersapi.com/\(number)")!
            let (data, response) = try await URLSession.shared.data(from: url)
            guard let httpResponse = response as? HTTPURLResponse,
                  httpResponse.statusCode == 200
            else {
                throw URLError(.badServerResponse)
            }
            return String(decoding: data, as: UTF8.self)
        }
    )

    static let previewValue = NumberFactClient(
        fetch: { number in
            try? await Task.sleep(for: .seconds(1))
            return "\(number) is a number with its own personality (preview data)."
        }
    )

    static let testValue = NumberFactClient(
        fetch: unimplemented("NumberFactClient.fetch")
    )
}

extension DependencyValues {
    var numberFactClient: NumberFactClient {
        get { self[NumberFactClient.self] }
        set { self[NumberFactClient.self] = newValue }
    }
}
```

**Line by line:**

- `struct NumberFactClient { var fetch: (Int) async throws -> String }` —
  the whole "interface" is a single closure property. `async throws`
  means calling it may suspend (wait) and may fail with an error.
- `extension NumberFactClient: DependencyKey` — registers this type as a
  dependency TCA knows how to look up.
- `liveValue` — builds a real `URLRequest`... actually a `URL`, fetches
  it with `URLSession.shared.data(from:)`, checks the HTTP status code is
  200 (OK), and decodes the raw bytes as a UTF-8 string.
- `previewValue` — sleeps for a second (to simulate a loading state you
  can actually see in a preview) and returns fixed, fake text. Used
  automatically inside `#Preview`.
- `testValue: unimplemented("NumberFactClient.fetch")` — `unimplemented`
  is a Point-Free helper that creates a closure which fails the current
  test loudly if it's ever called without first being overridden. This
  guarantees tests never silently hit the network.
- `extension DependencyValues { var numberFactClient: ... }` — the
  familiar getter/setter pattern that registers the `\.numberFactClient`
  key path (Chapter 5, Section 5.7).

## 7.4 `CounterFeature.swift` — State, Action, Reducer

**Why this file exists:** this is the feature's entire brain. Nothing
about SwiftUI or `View` appears here at all — this file could be unit
tested with zero UI involved.

```swift
import ComposableArchitecture

@Reducer
struct CounterFeature {
    @ObservableState
    struct State: Equatable {
        var count = 0
        var fact: String?
        var isLoadingFact = false
    }

    enum Action {
        case incrementButtonTapped
        case decrementButtonTapped
        case fetchFactButtonTapped
        case factResponse(Result<String, Error>)
    }

    @Dependency(\.numberFactClient) var numberFactClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .incrementButtonTapped:
                state.count += 1
                return .none

            case .decrementButtonTapped:
                state.count -= 1
                return .none

            case .fetchFactButtonTapped:
                state.isLoadingFact = true
                state.fact = nil
                let count = state.count
                return .run { send in
                    let result = await Result { try await numberFactClient.fetch(count) }
                    await send(.factResponse(result))
                }

            case let .factResponse(.success(fact)):
                state.isLoadingFact = false
                state.fact = fact
                return .none

            case .factResponse(.failure):
                state.isLoadingFact = false
                state.fact = "Couldn't load a fact. Please try again."
                return .none
            }
        }
    }
}
```

Every line here was already explained in Chapter 5, Section 5.3 — if
anything looks unfamiliar, that's the section to re-read. The one new
thing worth calling out: `Result<String, Error>` conforms to `Equatable`
only if `Error` does (it doesn't, by default), so in a real project you
would typically either constrain the error type to something `Equatable`
(e.g. a custom `enum NumberFactError: Error, Equatable`) or mark the
`State` comparison to ignore it. For this first app, keep `Result<String, Error>`
as shown — you'll fix this properly with a custom error type in Chapter 8.

## 7.5 `CounterView.swift` — the View

**Why this file exists:** this is the *only* file allowed to import
SwiftUI layout code and describe pixels. It has no logic of its own.

```swift
import ComposableArchitecture
import SwiftUI

struct CounterView: View {
    @Bindable var store: StoreOf<CounterFeature>

    var body: some View {
        VStack(spacing: 20) {
            Text("\(store.count)")
                .font(.system(size: 64, weight: .bold))
                .contentTransition(.numericText())

            HStack(spacing: 24) {
                Button {
                    store.send(.decrementButtonTapped)
                } label: {
                    Image(systemName: "minus.circle.fill")
                        .font(.largeTitle)
                }

                Button {
                    store.send(.incrementButtonTapped)
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.largeTitle)
                }
            }

            Divider()

            Group {
                if store.isLoadingFact {
                    ProgressView()
                } else if let fact = store.fact {
                    Text(fact)
                        .multilineTextAlignment(.center)
                        .padding(.horizontal)
                } else {
                    Text("No fact loaded yet.")
                        .foregroundStyle(.secondary)
                }
            }
            .frame(minHeight: 60)

            Button("Fetch Fact") {
                store.send(.fetchFactButtonTapped)
            }
            .buttonStyle(.borderedProminent)
        }
        .padding()
        .animation(.default, value: store.count)
    }
}

#Preview {
    CounterView(
        store: Store(initialState: CounterFeature.State()) {
            CounterFeature()
        }
    )
}
```

**Line by line, focusing on what's new:**

- `@Bindable var store: StoreOf<CounterFeature>` — `@Bindable` allows this
  View to create two-way SwiftUI bindings (`$store.someField`) directly
  against the Store's state, if needed later (this simple screen doesn't
  use one yet, but it's the modern default declaration style for a TCA
  root view). `StoreOf<CounterFeature>` is shorthand for
  `Store<CounterFeature.State, CounterFeature.Action>`.
- `Text("\(store.count)")` — reads state directly off the store, as
  explained in Chapter 5.
- `.contentTransition(.numericText())` — a pure SwiftUI visual detail
  (animates digit changes); this belongs in the View because it is not
  business logic, just presentation.
- `Button { store.send(.decrementButtonTapped) } label: { ... }` — sends
  an action; never mutates `store.count` directly.
- `if store.isLoadingFact { ProgressView() } else if let fact = store.fact { ... } else { ... }`
  — purely declarative rendering based on current state; no logic
  decisions are made here beyond "what to show for this state."
- `#Preview { CounterView(store: Store(initialState: CounterFeature.State()) { CounterFeature() }) }`
  — builds a real `Store` for the preview. Because we did not override
  any dependency, this preview will use `NumberFactClient.previewValue`
  automatically (Section 7.3) — meaning previews are safe, fast, and
  network-free.

## 7.6 `CounterDemoApp.swift` — the entry point

**Why this file exists:** exactly one place in the whole app creates the
*root* `Store`. Every other Store in the app (for child features) will be
derived from this one via scoping (Chapter 10), never created
independently.

```swift
import ComposableArchitecture
import SwiftUI

@main
struct CounterDemoApp: App {
    var body: some Scene {
        WindowGroup {
            CounterView(
                store: Store(initialState: CounterFeature.State()) {
                    CounterFeature()
                }
            )
        }
    }
}
```

- `@main` — marks this as the app's entry point (standard Swift).
- `Store(initialState: CounterFeature.State()) { CounterFeature() }` —
  creates the one and only root Store for this simple app, with default
  starting state (`count: 0, fact: nil, isLoadingFact: false`) and the
  real reducer. Because this is the actual running app (not a preview or
  a test), `@Dependency(\.numberFactClient)` inside `CounterFeature`
  resolves to `NumberFactClient.liveValue` here — real network calls.

## 7.7 Why each file exists — summary table

| File | Contains | Does NOT contain |
|---|---|---|
| `NumberFactClient.swift` | Dependency interface + live/preview/test implementations | Any reference to `CounterFeature` or SwiftUI |
| `CounterFeature.swift` | State, Action, reducer logic | Any SwiftUI `View` code |
| `CounterView.swift` | SwiftUI layout, reading state, sending actions | Any business logic or validation |
| `CounterDemoApp.swift` | Root `Store` creation, app entry point | Any feature-specific logic |

This separation is small-scale now, but it is the exact same shape you
will use in Chapter 15 for a full production folder structure — this
chapter is that structure's smallest possible example.

## 7.8 Running it

Build and run. Tapping `+`/`-` should update the count instantly (no
network involved — this confirms the synchronous reducer path). Tapping
"Fetch Fact" should show a spinner, then a real fact about the current
number (this confirms the effect → live dependency → response → reducer
path). If you disconnect your network and tap "Fetch Fact," you should
see the "Couldn't load a fact" fallback text — this confirms the
`.failure` branch of `factResponse` works.

## 7.9 Chapter Summary

- A minimal TCA app needs at least: one dependency file, one feature
  file (State/Action/Reducer), one View file, and one app entry point
  that creates the root `Store`.
- Each file has exactly one job, matching the separation of concerns
  established in Chapters 1, 2, and 5.
- Previews and tests never accidentally hit the network, because
  `previewValue` and `testValue` are separate, deliberately safe
  implementations.
- Only one `Store` is created directly, at the app's root — every other
  Store in a larger app comes from scoping this one (Chapter 10).

## 7.10 Check Your Understanding

1. Why does `NumberFactClient` live in its own file, separate from
   `CounterFeature`?
2. What would go wrong if `CounterView` mutated `store.count` directly
   instead of sending `.incrementButtonTapped`?
3. Which dependency value (`liveValue`, `previewValue`, or `testValue`)
   runs when you open `CounterView` in Xcode's canvas, and why?
4. Where, in this whole project, is the only place a `Store` is created
   from scratch?
5. What would you need to add to `CounterFeature.Action` to make
   `Result<String, Error>` fully `Equatable`, and why would that matter?

---

**Previous:** [Chapter 6 — Internal Working](06-Internal-Working.md)
**Next:** [Chapter 8 — More Examples](08-More-Examples.md)
