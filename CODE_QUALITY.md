# GitHub Copilot — Android Code Quality Instructions

## How to read this file

Every rule carries a severity tag:

- `[CRITICAL]` — Causes crashes, data loss, or security issues. Never violate.
- `[ERROR]` — Breaks coroutines, leaks, or produces incorrect behavior. Never violate.
- `[WARNING]` — Detekt violation. Fix before committing.
- `[STYLE]` — Readability and convention. Apply consistently.

When rules conflict, higher severity wins.

---

## 1. Nullability

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never use `!!`. Use `?.`, `?:`, `requireNotNull()`, or `checkNotNull()`. |
| `[ERROR]` | Never use `Any` as a type. Create a specific domain type. |
| `[WARNING]` | Always declare explicit types for parameters, return values, and public properties. |
| `[STYLE]` | Use `val` over `var`. Treat `var` as a deliberate, justified exception. |

```kotlin
// ❌
val name = data!!.toString()

// ✅
val name = data?.toString() ?: "Unknown"
```

---

## 2. Immutability

| Severity | Rule |
|---|---|
| `[ERROR]` | Never expose `MutableStateFlow`, `MutableLiveData`, or `MutableList` publicly. |
| `[WARNING]` | Use immutable collection types (`List`, `Map`, `Set`) as return types and parameters. |
| `[STYLE]` | Use `data class` with `val` properties for DTOs and UI state. Use `copy()` to produce modified versions. |

```kotlin
// ❌
val items = MutableStateFlow<List<Item>>(emptyList())

// ✅
private val _items = MutableStateFlow<List<Item>>(emptyList())
val items: StateFlow<List<Item>> = _items.asStateFlow()
```

---

## 3. Naming

| Element | Convention | Severity |
|---|---|---|
| Class / Interface / Object / Enum | `PascalCase` | `[WARNING]` |
| Function / variable / property | `camelCase` | `[WARNING]` |
| `const val` / top-level `val` | `UPPER_SNAKE_CASE` | `[WARNING]` |
| Package | `lowercase`, no underscores | `[WARNING]` |
| Boolean property | Must start with `is`, `has`, `are`, `can`, `should` | `[WARNING]` |
| File name | Must match primary declaration name | `[WARNING]` |

Additional naming rules `[STYLE]`:
- Functions must start with a verb: `fetchUser()`, `saveOrder()`, `buildRequest()`.
- No arbitrary abbreviations. Accepted: `API`, `URL`, `ID`, `DB`, `i`/`j`, `err`, `ctx`.
- No generic class suffixes: `Util`, `Helper`, `Manager` (unless they are standard Android types).
- No name shadowing: never reuse a variable name in an inner scope.

> **Compose-only naming rules** (skip if project does not use Compose — see Section 10):
> - `@Composable` functions returning `Unit` → `PascalCase` noun: `UserCard`, `LoginScreen`.
> - `@Composable` functions returning a value → `camelCase`: `rememberScrollState()`.
> - `@Composable` factories that internally `remember {}` → prefix `remember`: `rememberPagerState()`.
> - `CompositionLocal` keys → `Local` prefix as adjective: `LocalTheme`, `LocalAnalytics`.
> - Enum values in Compose APIs → `PascalCase`, not `UPPER_SNAKE_CASE`.

---

## 4. Functions

| Severity | Rule |
|---|---|
| `[WARNING]` | Max 20 instructions per function. Max 60 lines (Detekt threshold). |
| `[WARNING]` | Max 6 parameters. Beyond that, introduce a `data class`. |
| `[WARNING]` | Max 4 levels of nesting. Extract inner blocks to named functions. |
| `[WARNING]` | Never use `break@label`, `continue@label`, or `return@label`. |
| `[STYLE]` | Use guard clauses / early returns. Valid conditions first, main logic last. |
| `[STYLE]` | Maintain a single level of abstraction per function. |
| `[STYLE]` | Use default parameter values instead of null checks or overloads at call sites. |
| `[STYLE]` | Never use a boolean parameter to control internal branching — split into two functions. |

```kotlin
// ❌
fun processOrder(order: Order?) {
    if (order != null) {
        if (order.isValid) {
            if (order.items.isNotEmpty()) { /* logic */ }
        }
    }
}

// ✅
fun processOrder(order: Order?) {
    order ?: return
    if (!order.isValid) return
    if (order.items.isEmpty()) return
    /* logic */
}
```

---

## 5. SOLID

Apply all five principles at every layer. Violations are `[ERROR]` when they cross layer boundaries, `[WARNING]` otherwise.

**S — Single Responsibility** `[WARNING]`
One class, one reason to change. Max: 200 instructions, 10 public methods, 10 properties.

**O — Open/Closed** `[WARNING]`
Add new implementations rather than modifying existing ones when adding behavior.

**L — Liskov Substitution** `[ERROR]`
Never override a function in a way that breaks the parent contract (throws where parent doesn't, returns null unexpectedly, ignores parameters).

**I — Interface Segregation** `[WARNING]`
Never define a single interface that bundles unrelated operations. Callers must not depend on methods they don't use.

```kotlin
// ❌
interface UserRepository { fun fetch(); fun save(); fun delete(); fun search() }

// ✅
interface UserReader { fun fetch(id: String): User }
interface UserWriter { fun save(user: User); fun delete(id: String) }
```

**D — Dependency Inversion** `[ERROR]`
- Repository interfaces → domain layer. Implementations → data layer.
- ViewModels depend on use case interfaces, never on concrete implementations.
- Always inject via constructor. Never use a service locator inside business logic.

---

## 6. Classes

| Severity | Rule |
|---|---|
| `[STYLE]` | Prefer composition over inheritance. Use delegation (`by`) and interfaces. |
| `[WARNING]` | Use `sealed class` / `sealed interface` for exhaustive domain states. Use `when` without `else`. |
| `[STYLE]` | Use `object` for true stateless singletons only. |
| `[STYLE]` | Use `value class` (`@JvmInline`) for domain primitives wrapping a single value. |

```kotlin
@JvmInline
value class Email(val value: String) {
    init { require(value.contains('@')) { "Invalid email: $value" } }
}
```

---

## 7. Scope Functions

| Function | Context object | Returns | Use when |
|---|---|---|---|
| `let` | `it` | Lambda result | Null-safe block; scoped transformation |
| `run` | `this` | Lambda result | Config + compute result in one block |
| `with` | `this` | Lambda result | Multiple calls on an existing non-null object |
| `apply` | `this` | The object | Builder / object configuration |
| `also` | `it` | The object | Side effects (logging, assertions) |

Rules `[STYLE]`:
- Never chain more than 2 scope functions in a single expression.
- Name the lambda parameter explicitly when `it` is ambiguous: `fetchUser()?.let { user -> user.validate() }`.
- Never use a scope function solely to avoid a local variable.

```kotlin
// apply — builder
val request = NetworkRequest().apply { url = endpoint; timeout = TIMEOUT_MS }

// also — side effect
return repo.fetchUser(id).also { logger.log("Fetched ${it.id}") }

// with — grouped calls on existing object
with(binding) { titleLabel.text = state.title; progressBar.isVisible = state.isLoading }

// let — null-safe block
user?.let { sendWelcomeEmail(it) }

// run — scoped computation
val isValid = run { prefs.token != null && prefs.expiry > now() }
```

---

## 8. Error Handling

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never catch `CancellationException` without rethrowing — breaks coroutine cancellation. |
| `[ERROR]` | Never swallow exceptions silently. Always log or rethrow. |
| `[ERROR]` | Never `return` from a `finally` block — discards exceptions. |
| `[ERROR]` | Never throw from `equals()`, `hashCode()`, `toString()`, or `finalize()`. |
| `[WARNING]` | Never catch `Exception`, `Throwable`, or `RuntimeException` broadly (only at top-level boundaries). |
| `[WARNING]` | Never `throw Exception(...)` or `throw RuntimeException(...)`. Throw a named type. |
| `[WARNING]` | Every exception must include a descriptive message or cause. |
| `[WARNING]` | Use `check()` instead of `throw IllegalStateException(...)`. |
| `[WARNING]` | Use `require()` instead of `throw IllegalArgumentException(...)`. |
| `[WARNING]` | Use `error()` instead of `throw IllegalStateException("unreachable")`. |
| `[STYLE]` | Use `Result<T>` or a sealed type for expected failures (network errors, validation). |

```kotlin
// ❌
try { cache.evict(key) } catch (e: Exception) { }

// ✅
try {
    cache.evict(key)
} catch (ignored: CacheException) {
    logger.warn("Cache eviction failed for $key", ignored)
}
```

---

## 9. Coroutines

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never use `GlobalScope`. Use `viewModelScope`, `lifecycleScope`, or an injected scope. |
| `[CRITICAL]` | Never use `runCatching` around `suspend` code — swallows `CancellationException`. |
| `[ERROR]` | Always inject `CoroutineDispatcher`. Never hardcode `Dispatchers.IO` or `Dispatchers.Main` inside a class. |
| `[ERROR]` | Never call `runBlocking` in production Android code — blocks the calling thread. |
| `[ERROR]` | Never use `Thread.sleep()` — use `delay()`. |
| `[ERROR]` | Wrap `suspend` calls inside `finally` with `withContext(NonCancellable) { }`. |
| `[WARNING]` | Functions returning `Flow` must not be marked `suspend`. |
| `[WARNING]` | Remove `suspend` from functions that do not call other suspend functions. |
| `[WARNING]` | No blocking I/O on `Main` or `Default` dispatcher — use `withContext(ioDispatcher)`. |
| `[STYLE]` | Convert to `StateFlow` only at the ViewModel boundary via `stateIn()`. |
| `[STYLE]` | Use `flowOn(ioDispatcher)` to move upstream emissions off main thread. |
| `[STYLE]` | Collect in XML UI with `repeatOnLifecycle(STARTED)`. Collect in Compose with `collectAsStateWithLifecycle()`. |

```kotlin
// ❌
class UserRepository {
    suspend fun fetch() = withContext(Dispatchers.IO) { api.getUser() }
}

// ✅
class UserRepository(private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) {
    suspend fun fetch() = withContext(ioDispatcher) { api.getUser() }
}
```

**Testing coroutines** `[ERROR]`:
- Always use `runTest { }`. Never launch raw coroutines in `@Test`.
- Inject `UnconfinedTestDispatcher` or `StandardTestDispatcher`.
- Drive time-based logic with `advanceUntilIdle()`.

---

## 10. Detekt — Complete Rule Reference

### 10.1 Complexity `[WARNING]`

| Rule | Limit | Action |
|---|---|---|
| `CyclomaticComplexMethod` | 10 paths | Split the function |
| `CognitiveComplexMethod` | 15 | Use early returns; extract blocks |
| `LongMethod` | 60 lines | Extract sub-steps into named functions |
| `LongParameterList` | 6 params | Introduce a `data class` for params |
| `NestedBlockDepth` | 4 levels | Extract inner blocks |
| `TooManyFunctions` | Per-class / per-file limits | Split by responsibility |
| `LargeClass` | 200 instructions / 10 methods / 10 props | Split by SRP |
| `StringLiteralDuplication` | ≥ 3 occurrences | Extract to a named constant |
| `ComplexCondition` | 4 boolean ops | Extract to a named boolean function |
| `LabeledExpression` | Any | Eliminate — use extraction instead |

### 10.2 Style `[WARNING]`

| Rule | Requirement |
|---|---|
| `MagicNumber` | All literals except -1, 0, 1 → named constant |
| `WildcardImport` | Never `import com.example.*` |
| `MaxLineLength` | 120 characters |
| `MayBeConst` | Top-level `val` that can be `const val` must be |
| `UnusedPrivateMember` | Remove all dead private code |
| `UnnecessaryLet` | Do not use `let` where a direct call suffices |
| `UseCheckOrError` | `check()` / `error()` — not manual `IllegalStateException` throws |
| `UseRequire` | `require()` — not manual `IllegalArgumentException` throws |
| `ModifierOrder` | Follow Kotlin modifier order convention |
| `ReturnCount` | Limit total return points per function |
| `NewLineAtEndOfFile` | Every file ends with a newline |
| `NoWildcardImports` | No star imports |

### 10.3 Potential Bugs `[ERROR]`

| Rule | Requirement |
|---|---|
| `UnsafeCallOnNullableType` | Never `!!` |
| `UnreachableCode` | No code after `return`, `throw`, `break`, `continue` |
| `EqualsWithHashCodeExist` | Always implement `hashCode()` alongside `equals()` |
| `ImplicitDefaultLocale` | Always pass `Locale` to `String.format()`, `lowercase()`, `uppercase()` |
| `WrongEqualsTypeParameter` | `equals()` parameter must be `Any?` |
| `UnreachableCatchBlock` | No catch blocks that can never be reached |

### 10.4 Exceptions `[ERROR]`

| Rule | Requirement |
|---|---|
| `SwallowedException` | Never silent catch — always log or rethrow |
| `TooGenericExceptionCaught` | Never catch `Exception`, `Throwable`, `RuntimeException` broadly |
| `TooGenericExceptionThrown` | Never `throw Exception(...)` or `throw RuntimeException(...)` |
| `ThrowingExceptionsWithoutMessageOrCause` | Every exception → message or cause |
| `ReturnFromFinally` | Never `return` from `finally` |
| `ExceptionRaisedInUnexpectedLocation` | No throws in `toString`, `hashCode`, `equals`, `finalize` |
| `RethrowCaughtException` | Do not catch just to rethrow identically without adding context |
| `ObjectExtendsThrowable` | `object` declarations must not extend `Throwable` |

### 10.5 Empty Blocks `[WARNING]`

Never leave empty: `EmptyCatchBlock`, `EmptyFunctionBlock`, `EmptyClassBlock`, `EmptyIfBlock`,
`EmptyElseBlock`, `EmptyFinallyBlock`, `EmptyTryBlock`, `EmptyForBlock`, `EmptyWhileBlock`, `EmptyWhenBlock`.

For an intentionally empty catch, name the parameter `ignored` and add a comment:
```kotlin
try { cache.evict(key) } catch (ignored: CacheException) { /* non-critical: proceed */ }
```

### 10.6 Coroutines `[CRITICAL]` / `[ERROR]`

| Rule | Severity |
|---|---|
| `GlobalCoroutineUsage` | `[CRITICAL]` |
| `RunCatchingCanSwallowCancellation` | `[CRITICAL]` |
| `InjectDispatcher` | `[ERROR]` |
| `InappropriateBlockingMethodCall` | `[ERROR]` |
| `SleepInsteadOfDelay` | `[ERROR]` |
| `SuspendFunInFinally` | `[ERROR]` |
| `SuspendFunWithFlowReturnType` | `[WARNING]` |
| `RedundantSuspendModifier` | `[WARNING]` |

### 10.7 Performance `[WARNING]`

| Rule | Requirement |
|---|---|
| `ArrayPrimitive` | Use `IntArray`, `FloatArray` instead of `Array<Int>` |
| `SpreadOperator` | Avoid `*` spread in hot paths — copies the array |
| `ForEachOnRange` | Use `for` on ranges, not `forEach` |
| `UnnecessaryTemporaryInstantiation` | Do not create wrapper objects when converting primitives to `String` |

### 10.8 Suppression

Use `@Suppress` only with a one-line justification comment.
**Never suppress** `[CRITICAL]` or `[ERROR]` rules.

```kotlin
@Suppress("LongMethod") // Single orchestration step — splitting degrades cohesion
fun buildComplexPayload(): Payload { ... }
```

---

## 11. Android Resources (XML projects only)

Skip this section if the project has no `res/` directory.

### Resource naming — always `snake_case`

| Type | Directory | Required prefix |
|---|---|---|
| Activity layout | `res/layout/` | `activity_` |
| Fragment layout | `res/layout/` | `fragment_` |
| List item layout | `res/layout/` | `item_` |
| Custom View layout | `res/layout/` | `view_` |
| Vector icon | `res/drawable/` | `ic_` |
| Background | `res/drawable/` | `bg_` |
| State list drawable | `res/drawable/` | `selector_` |
| Animation | `res/anim/` | `anim_` |
| Menu | `res/menu/` | `menu_` |

### Strings `[ERROR]`

- Never hardcode user-visible text in `.kt` files or `.xml` layouts. Always use `@string/`.
- Format args → positional notation: `%1$s`, `%2$d` (not `%s`, `%d`).
- Quantity-dependent text → `<plurals>`, not `if/else` in Kotlin. Always provide `one` and `other`.
- Escape: `\'` apostrophe · `\"` quote · `\n` newline.
- Styled text → `getText(R.string.x)`. Plain text → `getString(R.string.x)`.
- i18n: `Locale`-sensitive calls (`lowercase`, `uppercase`, `String.format`) must always receive an explicit `Locale`.
- Alternative translations → `res/values-<locale>/strings.xml`, not runtime `if (locale == ...)` checks.

```xml
<!-- ❌ -->
<string name="count">%d items</string>

<!-- ✅ -->
<plurals name="item_count">
    <item quantity="one">%d item</item>
    <item quantity="other">%d items</item>
</plurals>
```

### Colors & Dimensions `[WARNING]`

- Raw palette colors → `res/values/colors.xml` (e.g., `color_blue_500`).
- In layouts, reference theme attributes (`?attr/colorPrimary`), never raw palette colors directly.
- All dimensions → `res/values/dimens.xml`. Text sizes → `sp`. Everything else → `dp`.
- Never hardcode `16dp` in a layout — use `@dimen/spacing_md`.
- Dark mode → `res/values-night/colors.xml`, not runtime `if (isDarkMode)` checks.
- Tablets → `res/values-w600dp/`, not runtime screen-size checks.

### Drawables & Layouts `[WARNING]`

- Icons → vector drawables (`<vector>`). Never PNG for icons.
- Stateful drawables → `<selector>`. Never set drawables programmatically based on boolean state.
- Layouts → keep flat. Max 2–3 `ViewGroup` nesting levels. Use `ConstraintLayout` for complex layouts.
- Repeated view groups → extract with `<include>`.
- Layout used as a child component → use `<merge>` as root.
- Never use `wrap_content` as the height of a `RecyclerView`.

---

## 12. Jetpack Compose — Code Quality

**This section applies only if the project uses Jetpack Compose.**
Detection: `build.gradle.kts` contains `compose = true` or `androidx.compose`.

### 12.1 Composable design `[ERROR]`

- A `@Composable` function must either **emit UI content or return a value — never both**.
- Every `@Composable` that affects layout must expose `modifier: Modifier = Modifier` as a parameter.
  Apply the `modifier` to the **outermost layout element only**.

```kotlin
// ❌ — emits UI and returns a value
@Composable fun InputField(): InputState { ... }

// ✅ — hoisted state passed as parameter; modifier applied once
@Composable
fun InputField(state: InputState, modifier: Modifier = Modifier) {
    TextField(value = state.text, modifier = modifier, ...)
}
```

### 12.2 State hoisting `[ERROR]`

- Never own mutable state in a leaf composable that is shared with siblings or parent.
- Pass data down as parameters. Pass events up as lambdas.
- Stateful logic lives in ViewModel, not inside `@Composable` functions.

```kotlin
// ❌
@Composable fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// ✅
@Composable fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("$count") }
}
```

### 12.3 Stability `[ERROR]`

- `data class` with all `val` properties → inferred as stable automatically. Prefer this.
- Never pass `List`, `Map`, or `Set` directly as composable parameters — the Compose compiler
  treats them as unstable. Wrap in a `@Immutable data class` or use `kotlinx.collections.immutable`.
- Annotate with `@Stable` or `@Immutable` only when the contract is fully satisfied.
  A broken stability contract causes **silent incorrect rendering**.

```kotlin
// ❌ — List is unstable → recompositions not skipped
@Composable fun UserList(users: List<User>) { ... }

// ✅ — wrapped in @Immutable data class → stable
@Immutable data class UserListState(val users: List<User>)
@Composable fun UserList(state: UserListState) { ... }
```

### 12.4 Recomposition performance `[WARNING]`

| Rule | Pattern |
|---|---|
| Expensive work in composable body | Wrap in `remember(key) { }` |
| Source state changes more than derived | Use `derivedStateOf { }` |
| Frequently changing state (scroll, animation) | Use lambda-based modifiers: `Modifier.offset { }` |
| Items in `LazyColumn` / `LazyRow` | Always provide `key = { item -> item.id }` |
| Backwards write | Never write to state already read in the same composition frame |

```kotlin
// ❌ — recalculates on every recomposition
@Composable fun Chart(data: List<Point>) {
    val path = buildPath(data)
    Canvas { drawPath(path, paint) }
}

// ✅ — recalculates only when data changes
@Composable fun Chart(data: List<Point>) {
    val path = remember(data) { buildPath(data) }
    Canvas { drawPath(path, paint) }
}

// ❌ — reads scroll offset in composition phase → full recomposition on every scroll frame
Box(Modifier.offset(y = scrollState.value.dp))

// ✅ — reads in layout phase only → no recomposition
Box(Modifier.offset { IntOffset(0, scrollState.value) })

// ❌ — recomposes on every index change
val showButton = listState.firstVisibleItemIndex > 0

// ✅ — recomposes only when derived boolean flips
val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
```

### 12.5 `remember` and `CompositionLocal` `[STYLE]`

- Never compute an expensive value directly in a composable body — always `remember(key) { }`.
- Use `CompositionLocal` only for genuinely ambient, cross-cutting data (theme, analytics, locale).
  Never use it to avoid passing parameters through a few levels.
- Prefer `staticCompositionLocalOf` when the value changes infrequently.

---

## 13. Testing

| Severity | Rule |
|---|---|
| `[ERROR]` | Never launch raw coroutines in `@Test`. Always use `runTest { }`. |
| `[ERROR]` | Use the mock library already present in the project — never introduce a new one. |
| `[WARNING]` | Write unit tests for every public use case, ViewModel, and mapper function. |
| `[STYLE]` | Unit tests → Arrange–Act–Assert. Integration tests → Given–When–Then. |
| `[STYLE]` | Name test variables with role suffixes: `inputEmail`, `mockRepo`, `actualResult`, `expectedState`. |
| `[STYLE]` | Inject `UnconfinedTestDispatcher` or `StandardTestDispatcher`. Use `advanceUntilIdle()` for time-based logic. |

```kotlin
@Test
fun `emits error state when repository throws`() = runTest {
    val mockRepo = mockk<UserRepository>()
    val viewModel = UserViewModel(mockRepo, UnconfinedTestDispatcher())
    coEvery { mockRepo.fetchUser(any()) } throws NetworkException("timeout")

    viewModel.loadUser("user-123")

    with(viewModel.uiState.value) {
        assertThat(hasError).isTrue()
        assertThat(isLoading).isFalse()
    }
}
```
