# Mozilla Firefox for Android — Project-Specific Patterns

This reference is loaded when reviewing code in the Firefox for Android (Fenix) codebase.
It covers Mozilla-specific conventions that differ from general Android best practices.

---

## Mozilla State Pattern

Firefox uses a custom unidirectional state management pattern built around `Store`, `Action`, `Reducer`, and `Middleware`. This is distinct from MVI/Redux-style libs like Orbit or MVI-Core.

### Core types

```kotlin
// State is a data class — always immutable, always copied on change
data class BrowserState(
    val tabs: List<TabSessionState> = emptyList(),
    val selectedTabId: String? = null,
)

// Actions are sealed classes — one per state mutation type
sealed class BrowserAction : Action {
    data class AddTabAction(val tab: TabSessionState) : BrowserAction()
    data object RemoveAllTabsAction : BrowserAction()
}

// Reducer is a pure function — no side effects, no coroutines
fun browserStateReducer(state: BrowserState, action: BrowserAction): BrowserState =
    when (action) {
        is BrowserAction.AddTabAction -> state.copy(tabs = state.tabs + action.tab)
        is BrowserAction.RemoveAllTabsAction -> state.copy(tabs = emptyList())
    }
```

### Review flags for Mozilla State

- **Reducer doing IO or launching coroutines** — 🔴 Blocking. Reducers must be pure and synchronous.
- **State class with mutable fields** — 🔴 Blocking. All state must be `val`, not `var`.
- **State class not a `data class`** — 🟡 Warning. Non-data-class state can't be diffed correctly by `observe`.
- **Action dispatched from a Composable** — 🟡 Warning. Actions should flow from ViewModel/Middleware, not UI. The composable should call a lambda passed from the ViewModel.
- **Direct state mutation** — 🔴 Blocking. Never mutate state directly; always go through `store.dispatch(action)`.
- **Middleware with side effects that aren't tested** — 🟡 Warning. Middleware is the intentional side-effect layer; it must have unit tests covering each action it handles.

### Observing state

```kotlin
// Correct: observe in Fragment using viewLifecycleOwner
store.flowScoped(viewLifecycleOwner) { flow ->
    flow.mapNotNull { it.selectedTab }
        .distinctUntilChanged()
        .collect { tab -> updateUI(tab) }
}

// Wrong: observing without lifecycle scoping
store.observe(this) { state -> updateUI(state) }  // 'this' is the Fragment, not viewLifecycleOwner
```

---

## Proto DataStore Migration Patterns

Firefox is migrating from `SharedPreferences` to `Proto DataStore`. Flag any code that regresses this.

### Correct usage

```kotlin
// Injected via Hilt — never instantiate DataStore directly in a class body
@Inject lateinit var dataStore: DataStore<AppSettings>

// Reading — always via Flow, never blocking
val setting: Flow<Boolean> = dataStore.data
    .catch { e -> if (e is IOException) emit(AppSettings.getDefaultInstance()) else throw e }
    .map { it.someFlag }

// Writing — suspend function, never runBlocking
suspend fun updateFlag(value: Boolean) {
    dataStore.updateData { current -> current.toBuilder().setSomeFlag(value).build() }
}
```

### Review flags for DataStore

- **`runBlocking { dataStore.data.first() }`** — 🔴 Blocking. This will ANR if called on the main thread and deadlock in tests. Must be a `suspend` call or consumed as a `Flow`.
- **DataStore accessed outside a coroutine scope** — 🔴 Blocking.
- **`SharedPreferences` still being written in a class that has a DataStore counterpart** — 🟡 Warning. Flag as migration regression.
- **Missing `.catch { }` on `dataStore.data`** — 🟡 Warning. DataStore can throw `IOException` on corrupt/missing files; uncaught will crash the collection.
- **Proto schema change without migration** — 🟡 Warning. Removing or renaming a field without a migration strategy causes silent data loss on upgrade.

---

## Jetpack Compose Migration Conventions

Firefox is incrementally migrating from `Fragment + View` to Compose. Mixed codebases have specific pitfalls.

### Interop boundary rules

```kotlin
// Correct: ComposeView inside a Fragment
class MyFragment : Fragment() {
    override fun onCreateView(...) = ComposeView(requireContext()).apply {
        setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
        setContent { MyScreen(viewModel = viewModel()) }
    }
}
```

- **`ViewCompositionStrategy` not set on `ComposeView`** — 🔴 Blocking in Fragment context. Without it, the Composition is disposed on every `onDestroyView`, causing memory leaks and state loss during back stack operations.
- **Accessing `FragmentManager` from inside a Composable** — 🟡 Warning. Pass navigation callbacks as lambdas; composables shouldn't know about Fragment transactions.
- **`LocalContext.current as Activity`** — 🟡 Warning. This cast will fail in preview and test; use `LocalContext.current as? Activity ?: return` and handle null.

### State bridging (LiveData → State)

```kotlin
// Correct for migration period
val uiState by viewModel.uiState.collectAsStateWithLifecycle()

// Wrong: collectAsState() without lifecycle awareness
val uiState by viewModel.uiState.collectAsState()
// ^^ won't stop collecting when the app is backgrounded
```

---

## Hilt Scoping in Firefox

Firefox uses a multi-activity architecture. Pay attention to scope boundaries.

- **`@Singleton` for anything that holds tab or session state** — 🟡 Warning. Singletons survive across browser restarts within a process; verify that's the intent.
- **`@ActivityScoped` vs `@ActivityRetainedScoped`** — `@ActivityScoped` dies with the Activity on rotation; `@ActivityRetainedScoped` survives rotation but dies on back press. Verify the intended lifetime.
- **`@ViewModelScoped`** — correct for anything that should live as long as the ViewModel. Prefer this over `@ActivityRetainedScoped` unless the object is needed across multiple ViewModels.

---

## Testing Conventions

- Firefox uses **JUnit 4** (not 5) with **MockK** for mocking. Don't suggest JUnit 5 migrations.
- **`MainCoroutineRule`** with `TestCoroutineDispatcher` is the project standard for coroutine tests — don't replace with `StandardTestDispatcher` without checking compatibility.
- **`TestStoreMiddleware`** is available for testing store actions without dispatching to real state.
- GeckoView-dependent code requires device tests (`@UiThreadTest` / Espresso) — pure business logic tests should be pure JUnit with no GeckoView dependency.

---

## Naming Conventions

| Pattern | Convention |
|---|---|
| Screen composable | `FooScreen.kt` — one top-level `@Composable fun FooScreen(...)` |
| ViewModel | `FooViewModel` — one per screen, in `ui/` package |
| State class | `FooFragmentState` (legacy) or `FooUiState` (Compose migration) |
| Action | `FooAction` sealed class |
| Reducer | `FooReducer` object with `invoke` operator |
| Middleware | `FooMiddleware` implementing `Middleware<S, A>` |
| Repository | `FooRepository` interface + `DefaultFooRepository` impl |
