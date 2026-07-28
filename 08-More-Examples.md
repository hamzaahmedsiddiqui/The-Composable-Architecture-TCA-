# Chapter 8 — More Examples

Chapter 7 built one complete app end to end. This chapter builds nine
more, back to back, each one introducing at least one new TCA idea you
have not used yet. Basics that were already fully explained in Chapters
5-7 (what `@Reducer` means, why `send` is used, etc.) are not re-explained
line by line here — instead, each example calls out **what's new**, then
gives the full code so you can see it in context.

## 8.1 Todo List — introduces `IdentifiedArrayOf` and list operations

**What's new:** managing a *list* of independent, editable items, each
with its own small piece of state, and routing an action to "the one item
the user interacted with."

### The problem with plain `[Todo]`

If `State` held `var todos: [Todo]` (a plain Swift `Array`), toggling one
todo's "done" flag would require finding it by looping and checking
`id == someID`, which is slow and easy to get wrong (e.g. using an index
that becomes stale if the array is reordered while the operation is in
flight). TCA solves this with `IdentifiedArrayOf<Todo>` — an array-like
collection, from the `swift-identified-collections` library, where every
element must conform to `Identifiable`, and elements can be looked up,
updated, or removed **by ID**, safely, even if the collection has been
resorted or mutated in between two actions.

```swift
import ComposableArchitecture
import Foundation

struct Todo: Equatable, Identifiable {
    let id: UUID
    var title: String
    var isDone = false
}

@Reducer
struct TodoListFeature {
    @ObservableState
    struct State: Equatable {
        var todos: IdentifiedArrayOf<Todo> = []
        var newTodoTitle = ""
    }

    enum Action: BindableAction {
        case addButtonTapped
        case binding(BindingAction<State>)
        case todoCheckboxToggled(id: Todo.ID)
        case deleteButtonTapped(id: Todo.ID)
    }

    var body: some ReducerOf<Self> {
        BindingReducer()
        Reduce { state, action in
            switch action {
            case .addButtonTapped:
                guard !state.newTodoTitle.isEmpty else { return .none }
                state.todos.append(Todo(id: UUID(), title: state.newTodoTitle))
                state.newTodoTitle = ""
                return .none

            case .binding:
                return .none

            case let .todoCheckboxToggled(id):
                state.todos[id: id]?.isDone.toggle()
                return .none

            case let .deleteButtonTapped(id):
                state.todos.remove(id: id)
                return .none
            }
        }
    }
}

struct TodoListView: View {
    @Bindable var store: StoreOf<TodoListFeature>

    var body: some View {
        List {
            HStack {
                TextField("New todo", text: $store.newTodoTitle)
                Button("Add") { store.send(.addButtonTapped) }
            }
            ForEach(store.todos) { todo in
                HStack {
                    Button {
                        store.send(.todoCheckboxToggled(id: todo.id))
                    } label: {
                        Image(systemName: todo.isDone ? "checkmark.circle.fill" : "circle")
                    }
                    Text(todo.title)
                        .strikethrough(todo.isDone)
                }
                .swipeActions {
                    Button("Delete", role: .destructive) {
                        store.send(.deleteButtonTapped(id: todo.id))
                    }
                }
            }
        }
    }
}
```

**Key new ideas:**

- `IdentifiedArrayOf<Todo>` — declared as `[]` (an empty literal works
  because it conforms to `ExpressibleByArrayLiteral`).
- `state.todos[id: id]?.isDone.toggle()` — the special `[id:]` subscript
  looks up an element by its `Identifiable.id`, safely returning `nil` if
  it's gone (for example, deleted by another action that arrived a moment
  earlier), and lets you mutate it in place if found.
- `state.todos.remove(id: id)` — removes by ID, not by index.
- `enum Action: BindableAction` + `case binding(BindingAction<State>)` +
  `BindingReducer()` in the `body` — this is TCA's built-in tool for
  simple two-way-bound fields like a text field. `$store.newTodoTitle`
  in the View creates a binding that, when changed, automatically sends
  a `.binding(...)` action, which `BindingReducer()` applies to state for
  you, with zero manual code. Full details in Chapter 14.

## 8.2 Shopping Cart — introduces derived/computed state and multi-action coordination

**What's new:** computing values (like a total price) from other state,
and coordinating several small state changes from one user action.

```swift
struct CartItem: Equatable, Identifiable {
    let id: UUID
    var name: String
    var price: Decimal
    var quantity: Int
}

@Reducer
struct ShoppingCartFeature {
    @ObservableState
    struct State: Equatable {
        var items: IdentifiedArrayOf<CartItem> = []

        // Computed, not stored — always in sync, can never drift.
        var subtotal: Decimal {
            items.reduce(0) { $0 + $1.price * Decimal($1.quantity) }
        }
        var isEmpty: Bool { items.isEmpty }
    }

    enum Action {
        case incrementQuantity(id: CartItem.ID)
        case decrementQuantity(id: CartItem.ID)
        case removeItem(id: CartItem.ID)
    }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case let .incrementQuantity(id):
                state.items[id: id]?.quantity += 1
                return .none

            case let .decrementQuantity(id):
                guard let quantity = state.items[id: id]?.quantity else { return .none }
                if quantity <= 1 {
                    state.items.remove(id: id)
                } else {
                    state.items[id: id]?.quantity -= 1
                }
                return .none

            case let .removeItem(id):
                state.items.remove(id: id)
                return .none
            }
        }
    }
}
```

**Key new idea:** `subtotal` and `isEmpty` are **computed properties**,
not stored fields. This is the best-practice call-out from Chapter 5,
Section 5.1: never store a value that can be derived from other state,
because a stored, duplicated value can drift out of sync (you would have
to remember to update `subtotal` in every single action that touches
`items` — a computed property makes that entire category of bug
impossible).

## 8.3 Weather App — introduces multiple loading states and a richer error type

**What's new:** modeling "not started / loading / loaded / failed" as an
explicit type instead of loose booleans, and defining a custom,
`Equatable` error type so `State` can stay fully `Equatable` (solving the
loose end left in Chapter 7).

```swift
enum LoadingState<Value: Equatable>: Equatable {
    case idle
    case loading
    case loaded(Value)
    case failed(String)
}

struct Weather: Equatable {
    var cityName: String
    var temperatureCelsius: Double
    var condition: String
}

struct WeatherClient {
    var fetch: (String) async throws -> Weather
}

@Reducer
struct WeatherFeature {
    @ObservableState
    struct State: Equatable {
        var city = ""
        var weather: LoadingState<Weather> = .idle
    }

    enum Action {
        case searchButtonTapped
        case weatherResponse(Result<Weather, EquatableError>)
    }

    @Dependency(\.weatherClient) var weatherClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .searchButtonTapped:
                guard !state.city.isEmpty else { return .none }
                state.weather = .loading
                let city = state.city
                return .run { send in
                    await send(.weatherResponse(
                        Result { try await weatherClient.fetch(city) }
                            .mapError(EquatableError.init)
                    ))
                }

            case let .weatherResponse(.success(weather)):
                state.weather = .loaded(weather)
                return .none

            case let .weatherResponse(.failure(error)):
                state.weather = .failed(error.localizedDescription)
                return .none
            }
        }
    }
}
```

**Key new ideas:**

- `LoadingState<Value>` — a small, reusable, generic enum that models
  every screen's "what state is this async thing in" question, instead of
  juggling `isLoading: Bool` + `data: Value?` + `errorMessage: String?`
  as three separate, independently-mutable properties (the exact Chapter
  1 problem: nothing stops `isLoading == true && data != nil` at the same
  time). One enum, one valid state at a time — the compiler enforces it.
- `EquatableError` — a small helper type (ships in TCA's supporting
  libraries) that wraps any `Error` in an `Equatable`-conforming box, so
  actions carrying errors, and `State` containing them, can still be
  compared with `==` — required for `TestStore` assertions (Chapter 13).

## 8.4 Login — introduces validation-as-computed-state and disabling UI based on State

**What's new:** replacing Chapter 1's tangled `LoginView` with the TCA
version, and showing how "is the button disabled" becomes a pure function
of state, not logic in the View.

```swift
@Reducer
struct LoginFeature {
    @ObservableState
    struct State: Equatable {
        var email = ""
        var password = ""
        var isLoading = false
        var errorMessage: String?

        var isFormValid: Bool {
            email.contains("@") && password.count >= 8
        }
    }

    enum Action: BindableAction {
        case binding(BindingAction<State>)
        case loginButtonTapped
        case loginResponse(Result<AuthToken, EquatableError>)
    }

    @Dependency(\.authClient) var authClient

    var body: some ReducerOf<Self> {
        BindingReducer()
        Reduce { state, action in
            switch action {
            case .binding:
                state.errorMessage = nil
                return .none

            case .loginButtonTapped:
                guard state.isFormValid else { return .none }
                state.isLoading = true
                let email = state.email
                let password = state.password
                return .run { send in
                    await send(.loginResponse(
                        Result { try await authClient.login(email, password) }
                            .mapError(EquatableError.init)
                    ))
                }

            case let .loginResponse(.success(token)):
                state.isLoading = false
                // In a real app: send a .delegate(.didLogIn(token)) action here.
                // See Chapter 10 for delegate actions.
                return .none

            case let .loginResponse(.failure(error)):
                state.isLoading = false
                state.errorMessage = error.localizedDescription
                return .none
            }
        }
    }
}

struct LoginView: View {
    @Bindable var store: StoreOf<LoginFeature>

    var body: some View {
        Form {
            TextField("Email", text: $store.email)
                .textInputAutocapitalization(.never)
            SecureField("Password", text: $store.password)

            if let errorMessage = store.errorMessage {
                Text(errorMessage).foregroundStyle(.red)
            }

            Button {
                store.send(.loginButtonTapped)
            } label: {
                if store.isLoading {
                    ProgressView()
                } else {
                    Text("Log In")
                }
            }
            .disabled(!store.isFormValid || store.isLoading)
        }
    }
}
```

Compare this directly to the `LoginView` from Chapter 1. Every piece of
logic that was buried in a button closure is now in a reducer, testable
by calling it directly (Chapter 13 writes exactly this test). The View is
back to being a pure description of layout, exactly as promised in
Chapter 4.

## 8.5 API Fetching (generic list screen) — introduces pagination-ready state shape

**What's new:** a reusable shape for "fetch a list from an API," designed
so pagination (Chapter 21) can be added later without restructuring.

```swift
struct Post: Equatable, Identifiable, Decodable {
    let id: Int
    let title: String
}

struct PostsClient {
    var fetchPage: (_ page: Int) async throws -> [Post]
}

@Reducer
struct PostsFeature {
    @ObservableState
    struct State: Equatable {
        var posts: IdentifiedArrayOf<Post> = []
        var status: LoadingState<Never> = .idle // .loaded(Never) never happens; used only for idle/loading/failed here
        var currentPage = 1
    }

    enum Action {
        case onAppear
        case postsResponse(Result<[Post], EquatableError>)
    }

    @Dependency(\.postsClient) var postsClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                guard state.posts.isEmpty else { return .none }
                state.status = .loading
                let page = state.currentPage
                return .run { send in
                    await send(.postsResponse(
                        Result { try await postsClient.fetchPage(page) }
                            .mapError(EquatableError.init)
                    ))
                }

            case let .postsResponse(.success(posts)):
                state.status = .idle
                state.posts.append(contentsOf: posts)
                return .none

            case let .postsResponse(.failure(error)):
                state.status = .failed(error.localizedDescription)
                return .none
            }
        }
    }
}
```

**Key new idea:** `onAppear` guards with `state.posts.isEmpty` so
re-appearing (e.g. navigating back to this screen) does not re-fetch data
that is already loaded — a common, easy-to-forget production detail.

## 8.6 Image Downloader — introduces effect cancellation by ID

**What's new:** cancelling an in-flight effect when a newer request makes
the old one obsolete — critical for anything triggered repeatedly, like
scrolling a list of thumbnails.

```swift
struct ImageClient {
    var download: (URL) async throws -> Data
}

@Reducer
struct ImageDownloaderFeature {
    @ObservableState
    struct State: Equatable {
        var url: URL?
        var imageData: Data?
        var isLoading = false
    }

    enum Action {
        case urlChanged(URL?)
        case imageResponse(Result<Data, EquatableError>)
    }

    @Dependency(\.imageClient) var imageClient
    private enum CancelID { case download }

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case let .urlChanged(url):
                state.url = url
                state.imageData = nil
                guard let url else {
                    return .cancel(id: CancelID.download)
                }
                state.isLoading = true
                return .run { send in
                    await send(.imageResponse(
                        Result { try await imageClient.download(url) }
                            .mapError(EquatableError.init)
                    ))
                }
                .cancellable(id: CancelID.download, cancelInFlight: true)

            case let .imageResponse(.success(data)):
                state.isLoading = false
                state.imageData = data
                return .none

            case .imageResponse(.failure):
                state.isLoading = false
                return .none
            }
        }
    }
}
```

**Key new ideas:**

- `private enum CancelID { case download }` — a small, private type used
  purely as a unique, stable identifier for an in-flight effect. Using an
  enum case (rather than a raw string) avoids typos and namespace
  collisions between features.
- `.cancellable(id: CancelID.download, cancelInFlight: true)` —
  `cancelInFlight: true` means "if an effect with this same ID is already
  running, cancel it before starting this new one." This is exactly what
  you want when the URL changes rapidly (e.g. fast scrolling) — you only
  care about the *latest* image, not every one you scrolled past.
- `.cancel(id: CancelID.download)` — explicitly cancels a running effect,
  used here when the URL becomes `nil`. Chapter 11 covers cancellation in
  full depth, including debounce/throttle built on this same mechanism.

## 8.7 Search Screen — introduces debouncing user input

**What's new:** waiting for the user to pause typing before firing a
network request, instead of firing one request per keystroke.

```swift
@Reducer
struct SearchFeature {
    @ObservableState
    struct State: Equatable {
        var query = ""
        var results: [String] = []
    }

    enum Action: BindableAction {
        case binding(BindingAction<State>)
        case searchResponse([String])
    }

    @Dependency(\.searchClient) var searchClient
    private enum CancelID { case search }

    var body: some ReducerOf<Self> {
        BindingReducer()
        Reduce { state, action in
            switch action {
            case .binding(\.query):
                let query = state.query
                guard !query.isEmpty else {
                    state.results = []
                    return .cancel(id: CancelID.search)
                }
                return .run { send in
                    let results = try await searchClient.search(query)
                    await send(.searchResponse(results))
                }
                .debounce(id: CancelID.search, for: .milliseconds(300), scheduler: DispatchQueue.main)

            case .binding:
                return .none

            case let .searchResponse(results):
                state.results = results
                return .none
            }
        }
    }
}
```

**Key new ideas:**

- `case .binding(\.query):` — modern TCA lets you pattern-match a binding
  action down to a *specific* field using a key path, so you can react
  right when `query` changes, without a separate manual action.
- `.debounce(id:for:scheduler:)` — waits 300 milliseconds after the
  *last* call before actually running the effect; if another call comes
  in before that time is up, the wait restarts. This is the standard fix
  for "search as you type" screens, and it is built directly on the same
  cancellation system from Section 8.6.

## 8.8 Profile Screen — introduces loading-on-appear plus editing a copy of state

**What's new:** a screen that loads data when it appears, then lets the
user edit fields locally before saving — a very common real-world shape.

```swift
struct Profile: Equatable {
    var name: String
    var bio: String
}

struct ProfileClient {
    var fetch: () async throws -> Profile
    var save: (Profile) async throws -> Void
}

@Reducer
struct ProfileFeature {
    @ObservableState
    struct State: Equatable {
        var profile: Profile?
        var isLoading = false
        var isSaving = false
    }

    enum Action {
        case onAppear
        case profileResponse(Result<Profile, EquatableError>)
        case nameChanged(String)
        case bioChanged(String)
        case saveButtonTapped
        case saveResponse(Result<Void, EquatableError>)
    }

    @Dependency(\.profileClient) var profileClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                state.isLoading = true
                return .run { send in
                    await send(.profileResponse(
                        Result { try await profileClient.fetch() }.mapError(EquatableError.init)
                    ))
                }

            case let .profileResponse(.success(profile)):
                state.isLoading = false
                state.profile = profile
                return .none

            case .profileResponse(.failure):
                state.isLoading = false
                return .none

            case let .nameChanged(name):
                state.profile?.name = name
                return .none

            case let .bioChanged(bio):
                state.profile?.bio = bio
                return .none

            case .saveButtonTapped:
                guard let profile = state.profile else { return .none }
                state.isSaving = true
                return .run { send in
                    await send(.saveResponse(
                        Result { try await profileClient.save(profile) }.mapError(EquatableError.init)
                    ))
                }

            case .saveResponse:
                state.isSaving = false
                return .none
            }
        }
    }
}
```

**Key new idea:** `state.profile?.name = name` — because `profile` is an
`Optional<Profile>`, TCA lets you mutate *through* the optional directly
with `?.`, which does nothing if `profile` is still `nil` (e.g., editing
attempted before load finished — impossible in practice here since the
UI would not show editable fields until `profile` is non-nil, but the
reducer stays safe regardless).

## 8.9 Settings Screen — introduces combining several small reducers into one parent

**What's new:** a first, small taste of composition (fully covered in
Chapter 10) — a Settings screen made of a couple of independent
sub-features.

```swift
@Reducer
struct NotificationSettingsFeature {
    @ObservableState
    struct State: Equatable {
        var pushEnabled = true
        var emailEnabled = false
    }
    enum Action: BindableAction {
        case binding(BindingAction<State>)
    }
    var body: some ReducerOf<Self> {
        BindingReducer()
    }
}

@Reducer
struct AccountSettingsFeature {
    @ObservableState
    struct State: Equatable {
        var displayName = ""
    }
    enum Action: BindableAction {
        case binding(BindingAction<State>)
    }
    var body: some ReducerOf<Self> {
        BindingReducer()
    }
}

@Reducer
struct SettingsFeature {
    @ObservableState
    struct State: Equatable {
        var notifications = NotificationSettingsFeature.State()
        var account = AccountSettingsFeature.State()
    }

    enum Action {
        case notifications(NotificationSettingsFeature.Action)
        case account(AccountSettingsFeature.Action)
    }

    var body: some ReducerOf<Self> {
        Scope(state: \.notifications, action: \.notifications) {
            NotificationSettingsFeature()
        }
        Scope(state: \.account, action: \.account) {
            AccountSettingsFeature()
        }
    }
}
```

**Key new idea:** `Scope(state: \.notifications, action: \.notifications) { NotificationSettingsFeature() }`
plugs a whole independent feature (with its own State, Action, and logic)
into a slice of the parent's State and a case of the parent's Action.
Neither sub-feature knows the other exists, and neither knows about
`SettingsFeature` at all — this is real composability, and Chapter 10
explains exactly how state and action routing work here, including how
to let a child talk back to its parent.

## 8.10 Chapter Summary

| Example | New concept introduced |
|---|---|
| Todo List | `IdentifiedArrayOf`, ID-based lookup/removal, `BindingReducer` |
| Shopping Cart | Computed/derived state instead of duplicated stored state |
| Weather | `LoadingState<Value>` enum, `EquatableError` |
| Login | Validation as computed state; disabling UI from State, not View logic |
| API Fetching | Guarding against redundant re-fetches; pagination-ready shape |
| Image Downloader | `.cancellable(id:cancelInFlight:)`, `.cancel(id:)` |
| Search | `.debounce(id:for:scheduler:)`, binding to a specific key path |
| Profile | Load-then-edit-a-copy pattern; mutating through Optional state |
| Settings | `Scope`, composing independent sub-features into one parent |

## 8.11 Check Your Understanding

1. Why is `IdentifiedArrayOf` safer than a plain `[Todo]` array when
   actions can arrive out of order relative to a stale index?
2. Why should `subtotal` in the Shopping Cart example never be a stored
   `var` instead of a computed property?
3. What specific Chapter 1 problem does `LoadingState<Value>` solve that
   three separate booleans/optionals do not?
4. In the Image Downloader example, what does `cancelInFlight: true` do,
   and why does the Search example need debounce instead of just
   cancellation alone?
5. In the Settings example, does `NotificationSettingsFeature` need to
   know that `SettingsFeature` exists? Why or why not?

---

**Previous:** [Chapter 7 — Building the First App](07-Building-the-First-App.md)
**Next:** [Chapter 9 — Navigation](09-Navigation.md)
