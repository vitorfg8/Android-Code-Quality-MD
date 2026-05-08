# Android Code Quality Instructions

## Severity levels

- `[CRITICAL]` — Causes crashes, data loss, or broken structured concurrency. Never violate.
- `[ERROR]`    — Produces incorrect behavior, leaks, or untestable code. Never violate.
- `[WARNING]`  — Detekt violation. Fix before committing.
- `[STYLE]`    — Convention and readability. Apply consistently.

When rules conflict, higher severity wins.
`@Suppress` requires a one-line justification comment. Never suppress `[CRITICAL]` or `[ERROR]`.

---

## 1. Nullability

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never use `!!`. Use `?.`, `?:`, `requireNotNull()`, or `checkNotNull()`. |
| `[ERROR]`    | Never use `Any` as a type. Create a domain-specific type instead. |
| `[WARNING]`  | Always declare explicit types for parameters, return values, and public properties. |
| `[STYLE]`    | Prefer `val` over `var`. Treat `var` as a deliberate, justified exception. |

```kotlin
// avoid
val name = data!!.toString()

// prefer
val name = data?.toString() ?: "Unknown"
```

---

## 2. Immutability

| Severity | Rule |
|---|---|
| `[ERROR]`   | Never expose `MutableStateFlow`, `MutableLiveData`, or any mutable collection publicly. |
| `[WARNING]` | Use immutable collection types (`List`, `Map`, `Set`) as parameters and return types. |
| `[STYLE]`   | Use `data class` with `val` properties. Produce modified versions with `copy()`. |

```kotlin
// avoid
val items = MutableStateFlow<List<Item>>(emptyList())

// prefer
private val _items = MutableStateFlow<List<Item>>(emptyList())
val items: StateFlow<List<Item>> = _items.asStateFlow()
```

---

## 3. Naming

Names must reveal intent. If a name requires a comment to be understood, the name is wrong.
Names must be searchable, pronounceable, and free of mental-mapping requirements.

| Element | Convention | Severity |
|---|---|---|
| Class / Interface / Object / Enum | `PascalCase` | `[WARNING]` |
| Function / variable / property | `camelCase` | `[WARNING]` |
| `const val` / top-level `val` | `UPPER_SNAKE_CASE` | `[WARNING]` |
| Package | `lowercase`, no underscores | `[WARNING]` |
| Boolean property | Must start with `is`, `has`, `are`, `can`, `should` | `[WARNING]` |
| File name | Must match primary declaration name | `[WARNING]` |

Additional rules `[STYLE]`:
- Functions must start with a verb: `fetchUser()`, `saveOrder()`, `buildRequest()`.
- Pick one word per concept and use it consistently across the codebase: either `fetch` or `get`, never both.
- No arbitrary abbreviations. Accepted: `API`, `URL`, `ID`, `DB`, `i`/`j`, `err`, `ctx`.
- No generic class suffixes: `Util`, `Helper`, `Manager` (unless they are standard Android/Jetpack types).
- No name shadowing: never reuse a variable name in an inner scope.
- No disinformation: `userList` must be a `List`, not a `Set` or custom collection.
- No encoded prefixes: never `mUserName`, `sInstance`. Let the type and IDE communicate that.

> **Compose-only naming** (skip if project does not use Compose — see Section 13):
> - `@Composable` functions returning `Unit` → `PascalCase` noun: `UserCard`, `LoginScreen`.
> - `@Composable` functions returning a value → `camelCase`: `rememberScrollState()`.
> - `@Composable` factories that internally `remember {}` → prefix `remember`.
> - `CompositionLocal` keys → `Local` prefix as adjective: `LocalTheme`, `LocalAnalytics`.
> - Enum values in Compose APIs → `PascalCase`, not `UPPER_SNAKE_CASE`.

---

## 4. Functions

A function must do one thing, at one level of abstraction, with no side effects.

| Severity | Rule |
|---|---|
| `[WARNING]` | Max 20 instructions per function. Max 60 lines (Detekt threshold). |
| `[WARNING]` | Max 6 parameters. Beyond that, introduce a `data class`. |
| `[WARNING]` | Max 4 levels of nesting. Extract inner blocks to named functions. |
| `[WARNING]` | Never use `break@label`, `continue@label`, or `return@label`. |
| `[WARNING]` | Never use a boolean flag parameter to control branching — split into two functions. |
| `[ERROR]`   | No side effects: a function must not silently mutate state beyond its declared purpose. |
| `[STYLE]`   | Command-query separation: a function either does something (command) or answers a question (query) — never both. |
| `[STYLE]`   | Use guard clauses / early returns. Valid conditions first, main logic last. |
| `[STYLE]`   | Maintain a single level of abstraction per function. Do not mix low-level I/O with business logic. |
| `[STYLE]`   | Use default parameter values instead of null checks or overloads at call sites. |
| `[STYLE]`   | DRY: extract any logic that appears more than once into a named function. |

```kotlin
// avoid — boolean flag collapses two functions into one
fun load(id: String, forceRefresh: Boolean) {
    if (forceRefresh) fetchFromNetwork(id) else fetchFromCache(id)
}

// prefer — two focused functions, each doing one thing
fun loadCached(id: String): User? = cache.get(id)
fun fetchFresh(id: String): User = api.getUser(id)
```

```kotlin
// avoid — mixed abstraction levels, no early return
fun processOrder(order: Order?) {
    if (order != null) {
        if (order.isValid) {
            if (order.items.isNotEmpty()) { /* business logic */ }
        }
    }
}

// prefer — flat, guard-first
fun processOrder(order: Order?) {
    order ?: return
    if (!order.isValid) return
    if (order.items.isEmpty()) return
    /* business logic */
}
```

---

## 5. Clean Code Principles

These principles come from *Clean Code* (Robert C. Martin) and apply at every layer.

### 5.1 Don't Repeat Yourself (DRY) `[WARNING]`
Every piece of knowledge must have a single, unambiguous representation.
Duplicated logic scattered across files is the root cause of divergent bugs when requirements change.
Extract any repeated behavior into a named function, extension, or class.

### 5.2 Command-Query Separation `[STYLE]`
A function that returns a value must not change observable state.
A function that changes state must return `Unit`.
Mixing both produces hidden side effects that callers cannot anticipate.

```kotlin
// avoid — sets state and returns a result
fun addAndGetSize(item: Item): Int {
    list.add(item)
    return list.size
}

// prefer — two separate responsibilities
fun addItem(item: Item) { list.add(item) }
fun getSize(): Int = list.size
```

### 5.3 Law of Demeter `[WARNING]`
A method must call only methods of:
- objects it received as parameters,
- objects it created itself,
- objects it holds as fields.

Never chain calls into objects retrieved from other objects ("train wrecks").

```kotlin
// avoid — reaching into internal structure of a retrieved object
val city = user.getAddress().getCity().getName()

// prefer — ask for what you need directly
val city = user.getCityName()
```

### 5.4 No Null Returns or Null Parameters `[ERROR]`
- Never return `null` from a function when an empty collection, `Optional`-equivalent, or sealed type is possible.
- Never accept a nullable parameter and then branch on it inside the function — make null the caller's problem through type-safe APIs.

```kotlin
// avoid
fun findUser(id: String): User? = db.find(id)
fun process(user: User?) { if (user != null) { /* ... */ } }

// prefer
fun findUser(id: String): User = db.find(id) ?: throw UserNotFoundException(id)
// or use a sealed result type
fun findUser(id: String): DataResult<User>
```

### 5.5 Error Handling Is One Thing `[STYLE]`
Extract `try/catch` blocks into their own function. A function that handles an error must not also contain business logic.

```kotlin
// avoid — error handling mixed with business logic
fun saveUser(user: User) {
    try {
        validate(user)
        repository.save(user)
        notifyAnalytics(user)
    } catch (e: ValidationException) { ... }
}

// prefer — error handling isolated
fun trySaveUser(user: User): DataResult<Unit> = try {
    saveUser(user)
    DataResult.Success(Unit)
} catch (e: ValidationException) {
    DataResult.Failure(DomainError.Validation(e.message))
}

fun saveUser(user: User) {
    validate(user)
    repository.save(user)
    notifyAnalytics(user)
}
```

### 5.6 Comments `[STYLE]`
Good code is self-documenting. A comment that restates what the code already says is noise.

Write a comment only when it cannot be expressed in code:
- Legal notices.
- Warning of non-obvious consequences.
- Explanation of intent behind a counter-intuitive decision.
- `TODO` with a ticket reference.

Never write:
- Comments that repeat the code (`// increment i` above `i++`).
- Commented-out code — delete it; version control preserves history.
- Comments that will become stale as the code evolves without anyone updating them.
- Section dividers inside a function — if you need one, the function is too long.

---

## 6. SOLID

Apply pragmatically. Domain and business-critical layers require strict compliance.
UI/presentation layers do not require the same abstraction level — forcing SOLID uniformly produces overengineering.

| Principle | Severity | Core rule |
|---|---|---|
| S — Single Responsibility | `[WARNING]` | One business reason to change — one actor owns it |
| O — Open/Closed | `[WARNING]` | Extend via composition or new types; never edit stable core logic |
| L — Liskov Substitution | `[ERROR]` | Every subtype must be fully substitutable for its declared type |
| I — Interface Segregation | `[WARNING]` | Clients must not depend on operations they do not use |
| D — Dependency Inversion | `[ERROR]` | High-level modules depend on abstractions, never on low-level details |

Violations at **architectural boundaries** (cross-layer coupling) → `[ERROR]`.
Violations **within a single layer** → `[WARNING]`.

---

### S — Single Responsibility `[WARNING]`

One class, one actor: one stakeholder group whose requirements drive change.
SRP is about **cohesion of purpose**, not size. A `UserRepository` with many methods can
respect SRP. A two-method class can violate it. Never conflate SRP with complexity limits
(those are Detekt thresholds in Section 11.1).

Rules:
- Never mix UI formatting, business rules, persistence, and networking in one class.
- Diagnostic: who would ask me to change this? If more than one distinct team would, split it.
- Excessive size is a symptom of an SRP violation, not SRP itself.

```kotlin
// avoid — Finance and HR both own this class; a payroll change risks breaking hour reporting
class EmployeeService {
    fun calculatePay(employee: Employee): Money  // Finance
    fun reportHours(employee: Employee): String  // HR
    fun save(employee: Employee)                 // DBA
}

// prefer
class PayCalculator { fun calculatePay(employee: Employee): Money }
class HourReporter  { fun reportHours(employee: Employee): String }
class EmployeeStore { fun save(employee: Employee) }
```

---

### O — Open/Closed `[WARNING]`

Extend behavior through composition, polymorphism, or new implementations — never by editing
stable, tested core logic. In Kotlin, prefer `sealed class`, extension functions, higher-order
functions, and delegation over explicit interface hierarchies.

Never create an interface with a single real implementation "for OCP" — that is premature
abstraction. Apply OCP only where variation is already demonstrated, not preemptively.

```kotlin
// avoid — adding a channel requires editing and retesting this class
class NotificationSender {
    fun send(type: String, message: String) {
        when (type) { "push" -> sendPush(message); "email" -> sendEmail(message) }
    }
}

// prefer — new channels extend without modifying existing code
fun interface NotificationChannel { fun send(message: String) }
class PushChannel  : NotificationChannel { override fun send(message: String) { /* ... */ } }
class EmailChannel : NotificationChannel { override fun send(message: String) { /* ... */ } }
class NotificationSender(private val channels: List<NotificationChannel>) {
    fun send(message: String) = channels.forEach { it.send(message) }
}
```

---

### L — Liskov Substitution `[ERROR]`

Every subtype must be fully substitutable for its declared type without callers knowing
which concrete implementation they received.

An override violates LSP when it:
- throws where the base type never throws
- returns `null` where the base type guarantees non-null
- ignores a parameter the base type uses
- strengthens a precondition (rejects valid inputs the base accepts)
- weakens a postcondition (delivers less than the base promises)

Diagnostic: if caller code uses `is` or `as` to branch on the concrete type, LSP is violated —
the type check is the caller compensating for a broken substitution.

```kotlin
// avoid — Square mutates height when width is set; caller's area calculation silently breaks
open class Rectangle(open var width: Int, open var height: Int) { fun area() = width * height }
class Square(side: Int) : Rectangle(side, side) {
    override var width: Int = side
        set(value) { field = value; super.height = value } // contract broken
}

// prefer — do not inherit where substitution fails
interface Shape { fun area(): Int }
data class Rectangle(val width: Int, val height: Int) : Shape { override fun area() = width * height }
data class Square(val side: Int)                       : Shape { override fun area() = side * side }
```

---

### I — Interface Segregation `[WARNING]`

Clients must not depend on operations they do not use. Depending on an unused operation
creates coupling: the client must be updated and redeployed whenever that operation changes.

In modern Android the problem appears less as fat interfaces and more as **god use cases**,
**god repositories**, and **ViewModels injected with a full service to call one method**.

```kotlin
// avoid — read-only screen depends on write and export operations it never calls
interface UserRepository {
    fun fetchById(id: String): User; fun fetchAll(): List<User>
    fun save(user: User); fun delete(id: String); fun exportCsv(): ByteArray
}

// prefer — each caller depends only on what it uses
interface UserReader   { fun fetchById(id: String): User; fun fetchAll(): List<User> }
interface UserWriter   { fun save(user: User); fun delete(id: String) }
interface UserExporter { fun exportCsv(): ByteArray }

class UserListViewModel(private val reader: UserReader) : ViewModel()
class UserRepositoryImpl : UserReader, UserWriter, UserExporter { /* ... */ }
```

---

### D — Dependency Inversion `[ERROR]`

High-level modules must not depend on low-level implementation details.
Both depend on abstractions. Abstractions must not depend on details — details depend on abstractions.

| Rule | Severity |
|---|---|
| Inject all dependencies via constructor | `[ERROR]` |
| Never instantiate a dependency inside a class body | `[ERROR]` |
| Never use a service locator inside business logic | `[ERROR]` |
| `import com.app.data.*` inside domain or presentation → forbidden | `[ERROR]` |

```kotlin
// avoid — ViewModel coupled to a concrete low-level class
class LoginViewModel {
    private val repo = UserRepositoryImpl(RetrofitClient.api, AppDatabase.dao)
}

// prefer — depends only on abstractions; wiring is the DI graph's responsibility
class LoginViewModel(
    private val loginUseCase: LoginUseCase,        // interface owned by domain layer
    private val ioDispatcher: CoroutineDispatcher, // injected; never hardcoded
) : ViewModel()
```

**Architecture rules that follow from DIP** `[ERROR]`:
- Repository **interfaces** → **domain** layer.
- Repository **implementations** → **data** layer.
- ViewModels depend on **use case interfaces** only — never on concrete repositories or data sources.
- Domain layer must have **zero imports** from data, remote, or presentation packages.

---

## 7. Classes

| Severity | Rule |
|---|---|
| `[STYLE]`   | Prefer composition over inheritance. Use delegation (`by`) and interfaces. |
| `[WARNING]` | Use `sealed class` / `sealed interface` for exhaustive states. Use `when` without `else`. |
| `[STYLE]`   | Use `object` for true stateless singletons only. |
| `[STYLE]`   | Use `@JvmInline value class` for domain primitives wrapping a single value to avoid primitive obsession. |

```kotlin
@JvmInline
value class Email(val value: String) {
    init { require(value.contains('@')) { "Invalid email: $value" } }
}
```

---

## 8. Scope Functions

| Function | Context object | Returns | Use when |
|---|---|---|---|
| `let` | `it` | Lambda result | Null-safe block; scoped transformation |
| `run` | `this` | Lambda result | Config + compute result in one block |
| `with` | `this` | Lambda result | Multiple calls on an existing non-null object |
| `apply` | `this` | The object | Builder / object configuration |
| `also` | `it` | The object | Side effects (logging, assertions) |

Rules `[STYLE]`:
- Never chain more than 2 scope functions in a single expression.
- Name the parameter explicitly when `it` is ambiguous.
- Never use a scope function solely to avoid a local variable.

```kotlin
// apply — builder pattern
val request = NetworkRequest().apply { url = endpoint; timeout = TIMEOUT_MS }

// also — side effect without altering the chain
return repository.fetchUser(id).also { logger.log("Fetched ${it.id}") }

// with — grouped calls on an existing binding
with(binding) { titleLabel.text = state.title; progressBar.isVisible = state.isLoading }

// let — null-safe block with named parameter
order?.let { o -> validate(o); submit(o) }

// run — scoped computation returning a value
val isValid = run { prefs.token != null && prefs.expiry > now() }
```

---

## 9. Error Handling

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never catch `CancellationException` without rethrowing. |
| `[ERROR]`    | Never swallow exceptions silently. Always log or rethrow with context. |
| `[ERROR]`    | Never `return` from a `finally` block — it discards the in-flight exception. |
| `[ERROR]`    | Never throw from `equals()`, `hashCode()`, `toString()`, or `finalize()`. |
| `[ERROR]`    | Never return `null` to signal failure — use `Result<T>` or a sealed type. |
| `[ERROR]`    | Never accept a nullable parameter and branch on it inside — push null handling to the caller. |
| `[WARNING]`  | Never catch `Exception`, `Throwable`, or `RuntimeException` broadly (only at top-level boundaries). |
| `[WARNING]`  | Never `throw Exception(...)` or `throw RuntimeException(...)`. Throw a named type. |
| `[WARNING]`  | Every thrown exception must have a descriptive message or cause. |
| `[WARNING]`  | Use `check()` instead of `throw IllegalStateException(...)`. |
| `[WARNING]`  | Use `require()` instead of `throw IllegalArgumentException(...)`. |
| `[WARNING]`  | Use `error()` for unreachable branches instead of `throw IllegalStateException("unreachable")`. |
| `[STYLE]`    | Define exception types in terms of the caller's needs, not the implementation's internal errors. |
| `[STYLE]`    | Extract `try/catch` into its own function. Error handling is one thing. |

```kotlin
// avoid
try { cache.evict(key) } catch (e: Exception) { }

// prefer
try {
    cache.evict(key)
} catch (ignored: CacheException) {
    logger.warn("Cache eviction failed for key=$key", ignored)
}
```

---

## 10. Coroutines

| Severity | Rule |
|---|---|
| `[CRITICAL]` | Never use `GlobalScope`. |
| `[CRITICAL]` | Never use `runCatching` around `suspend` code — swallows `CancellationException`. |
| `[ERROR]`    | Always inject `CoroutineDispatcher`. Never hardcode `Dispatchers.IO` or `Dispatchers.Main` inside a class. |
| `[ERROR]`    | Never call `runBlocking` in production Android code. |
| `[ERROR]`    | Never use `Thread.sleep()` — use `delay()`. |
| `[ERROR]`    | Wrap `suspend` calls inside `finally` with `withContext(NonCancellable) { }`. |
| `[WARNING]`  | Functions returning `Flow` must not be marked `suspend`. |
| `[WARNING]`  | Remove `suspend` from functions that do not call other suspend functions. |
| `[WARNING]`  | No blocking I/O on `Main` or `Default` dispatcher — use `withContext(ioDispatcher)`. |
| `[STYLE]`    | Convert to `StateFlow` at the ViewModel boundary via `stateIn()`. |
| `[STYLE]`    | Use `flowOn(ioDispatcher)` to move upstream emissions off main thread. |
| `[STYLE]`    | Collect in XML UI with `repeatOnLifecycle(STARTED)`. Collect in Compose with `collectAsStateWithLifecycle()`. |

```kotlin
// avoid
class UserRepository {
    suspend fun fetch() = withContext(Dispatchers.IO) { api.getUser() }
}

// prefer
class UserRepository(private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO) {
    suspend fun fetch() = withContext(ioDispatcher) { api.getUser() }
}
```

Coroutine testing `[ERROR]`: always `runTest { }`, never raw coroutines in `@Test`.
Inject `UnconfinedTestDispatcher` or `StandardTestDispatcher`. Drive time with `advanceUntilIdle()`.

---

## 11. Detekt — Complete Rule Reference

### 11.1 Complexity `[WARNING]`

| Rule | Limit | Action |
|---|---|---|
| `CyclomaticComplexMethod` | 10 paths | Split the function |
| `CognitiveComplexMethod` | 15 | Early returns; extract blocks |
| `LongMethod` | 60 lines | Extract sub-steps |
| `LongParameterList` | 6 params | Introduce a params `data class` |
| `NestedBlockDepth` | 4 levels | Extract inner blocks |
| `TooManyFunctions` | Per-class / per-file limits | Split by responsibility |
| `LargeClass` | 200 instructions / 10 methods / 10 props | Apply SRP |
| `StringLiteralDuplication` | 3+ occurrences | Extract to named constant |
| `ComplexCondition` | 4 boolean ops | Extract to named boolean function |
| `LabeledExpression` | Any | Eliminate via extraction |

### 11.2 Style `[WARNING]`

| Rule | Requirement |
|---|---|
| `MagicNumber` | All literals except -1, 0, 1 → named constant |
| `WildcardImport` | Never `import com.example.*` |
| `MaxLineLength` | 120 characters |
| `MayBeConst` | Top-level `val` eligible for `const val` must be `const val` |
| `UnusedPrivateMember` | Remove all dead private code |
| `UnnecessaryLet` | Do not use `let` where a direct call suffices |
| `UseCheckOrError` | `check()` / `error()` — not manual throws |
| `UseRequire` | `require()` — not manual throws |
| `ModifierOrder` | Follow Kotlin modifier order convention |
| `ReturnCount` | Limit total return points per function |
| `NewLineAtEndOfFile` | Every file ends with a newline |

### 11.3 Potential Bugs `[ERROR]`

| Rule | Requirement |
|---|---|
| `UnsafeCallOnNullableType` | Never `!!` |
| `UnreachableCode` | No code after `return`, `throw`, `break`, `continue` |
| `EqualsWithHashCodeExist` | Always implement `hashCode()` alongside `equals()` |
| `ImplicitDefaultLocale` | Always pass `Locale` to `String.format()`, `lowercase()`, `uppercase()` |
| `WrongEqualsTypeParameter` | `equals()` parameter must be `Any?` |
| `UnreachableCatchBlock` | No catch blocks that can never be reached |

### 11.4 Exceptions `[ERROR]`

| Rule | Requirement |
|---|---|
| `SwallowedException` | Never silent catch — always log or rethrow |
| `TooGenericExceptionCaught` | Never catch `Exception`, `Throwable`, `RuntimeException` broadly |
| `TooGenericExceptionThrown` | Never `throw Exception(...)` or `throw RuntimeException(...)` |
| `ThrowingExceptionsWithoutMessageOrCause` | Every exception needs message or cause |
| `ReturnFromFinally` | Never `return` from `finally` |
| `ExceptionRaisedInUnexpectedLocation` | No throws in `toString`, `hashCode`, `equals`, `finalize` |
| `RethrowCaughtException` | Do not catch just to rethrow identically |
| `ObjectExtendsThrowable` | `object` declarations must not extend `Throwable` |

### 11.5 Empty Blocks `[WARNING]`

Never leave empty: `EmptyCatchBlock`, `EmptyFunctionBlock`, `EmptyClassBlock`, `EmptyIfBlock`,
`EmptyElseBlock`, `EmptyFinallyBlock`, `EmptyTryBlock`, `EmptyForBlock`, `EmptyWhileBlock`, `EmptyWhenBlock`.

Intentionally empty catch → name the parameter `ignored` and add a comment explaining why:
```kotlin
try { cache.evict(key) } catch (ignored: CacheException) { /* non-critical: proceed without cache */ }
```

### 11.6 Coroutines

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

### 11.7 Performance `[WARNING]`

| Rule | Requirement |
|---|---|
| `ArrayPrimitive` | Use `IntArray`, `FloatArray` instead of `Array<Int>` |
| `SpreadOperator` | Avoid `*` spread in hot paths — copies the full array |
| `ForEachOnRange` | Use `for` on ranges, not `forEach` |

---

## 12. Android Resources

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

- Never hardcode user-visible text in `.kt` or `.xml` files. Always use `@string/`.
- Format args → positional notation: `%1$s`, `%2$d` — not `%s`, `%d`.
- Quantity strings → `<plurals>` with at least `one` and `other`. Never `if/else` in Kotlin.
- Escape: `\'` apostrophe · `\"` quote · `\n` newline.
- Always pass an explicit `Locale` to `String.format()`, `lowercase()`, `uppercase()`.
- Alternative translations → `res/values-<locale>/strings.xml`, not runtime `if (locale == ...)`.

```xml
<!-- avoid -->
<string name="count">%d items</string>

<!-- prefer -->
<plurals name="item_count">
    <item quantity="one">%d item</item>
    <item quantity="other">%d items</item>
</plurals>
```

```kotlin
// avoid
val msg = if (count == 1) "$count item" else "$count items"

// prefer
val msg = resources.getQuantityString(R.plurals.item_count, count, count)
```

### Colors & Dimensions `[WARNING]`

- Raw palette colors → `res/values/colors.xml` (e.g., `color_blue_500`).
- Reference theme attributes (`?attr/colorPrimary`) in layouts, never raw palette colors.
- All dimensions → `res/values/dimens.xml`. Text → `sp`. Everything else → `dp`.
- Dark mode → `res/values-night/`. Tablets → `res/values-w600dp/`. Never runtime checks.

### Drawables & Layouts `[WARNING]`

- Icons → vector drawables (`<vector>`). Never PNG for icons.
- Stateful drawables → `<selector>`. Never set drawables programmatically on boolean state.
- Layouts → flat. Max 2–3 `ViewGroup` nesting levels. Use `ConstraintLayout` for complexity.
- Repeated view groups → extract with `<include>`.
- Component layout always inside a `ViewGroup` → use `<merge>` as root.
- Never `wrap_content` as the height of a `RecyclerView`.

---

## 13. Jetpack Compose — Code Quality

Detection: `build.gradle.kts` contains `compose = true` or `androidx.compose`.
Skip this section if the project does not use Jetpack Compose.

### 13.1 Composable design `[ERROR]`

- A `@Composable` must either emit UI content or return a value — never both.
- Every `@Composable` that affects layout must expose `modifier: Modifier = Modifier`.
  Apply the modifier to the outermost layout element only.

```kotlin
// avoid — emits UI and returns a control object
@Composable fun InputField(): InputState { ... }

// prefer — hoisted state as parameter; modifier applied once at root
@Composable
fun InputField(state: InputState, modifier: Modifier = Modifier) {
    TextField(value = state.text, modifier = modifier)
}
```

### 13.2 State hoisting `[ERROR]`

- Never own mutable state in a leaf composable that is shared with its parent or siblings.
- Pass data down as parameters. Pass events up as lambdas.
- Stateful logic lives in ViewModel, not inside `@Composable` functions.

```kotlin
// avoid — state owned by the leaf
@Composable fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// prefer — state hoisted to the caller
@Composable fun Counter(count: Int, onIncrement: () -> Unit) {
    Button(onClick = onIncrement) { Text("$count") }
}
```

### 13.3 Stability `[ERROR]`

- `data class` with all `val` properties → inferred stable automatically. Prefer this.
- Never pass `List`, `Map`, or `Set` directly as composable parameters — unstable by the compiler.
  Wrap in an `@Immutable data class` or use `kotlinx.collections.immutable`.
- Annotate with `@Stable` or `@Immutable` only when the contract is fully met.
  A broken stability contract causes silent incorrect rendering.

```kotlin
// avoid — List is unstable; recomposition never skipped
@Composable fun UserList(users: List<User>) { ... }

// prefer — @Immutable wrapper makes the parameter stable
@Immutable data class UserListState(val users: List<User>)
@Composable fun UserList(state: UserListState) { ... }
```

### 13.4 Recomposition performance `[WARNING]`

| Problem | Rule |
|---|---|
| Expensive work in composable body | Wrap in `remember(key) { }` |
| Derived state recomputes more than necessary | Use `derivedStateOf { }` |
| Frequently changing state (scroll, animation) | Use lambda-based modifiers: `Modifier.offset { }` |
| Items in `LazyColumn` / `LazyRow` | Always provide `key = { item -> item.id }` |
| Backwards write | Never write to state already read in the same composition frame |

```kotlin
// avoid — rebuilds path on every recomposition
@Composable fun Chart(data: List<Point>) {
    val path = buildPath(data)
    Canvas { drawPath(path, paint) }
}

// prefer — rebuilds only when data changes
@Composable fun Chart(data: List<Point>) {
    val path = remember(data) { buildPath(data) }
    Canvas { drawPath(path, paint) }
}

// avoid — reads scroll offset in composition phase → full recomposition on every scroll frame
Box(Modifier.offset(y = scrollState.value.dp))

// prefer — reads in layout phase only → no recomposition
Box(Modifier.offset { IntOffset(0, scrollState.value) })

// avoid — recomposes on every index change
val showButton = listState.firstVisibleItemIndex > 0

// prefer — recomposes only when the boolean flips
val showButton by remember { derivedStateOf { listState.firstVisibleItemIndex > 0 } }
```

### 13.5 CompositionLocal `[STYLE]`

- Use only for genuinely ambient, cross-cutting data: theme, analytics, locale.
- Never use to avoid passing parameters through a few levels — that is regular parameter passing.
- Prefer `staticCompositionLocalOf` when the value changes infrequently.

---

## 14. Testing

| Severity | Rule |
|---|---|
| `[ERROR]`   | Never launch raw coroutines in `@Test`. Always use `runTest { }`. |
| `[ERROR]`   | Use the mock library already present in the project. Never introduce a new one. |
| `[WARNING]` | Write unit tests for every public use case, ViewModel, and mapper function. |
| `[STYLE]`   | Unit tests → Arrange–Act–Assert. Integration tests → Given–When–Then. |
| `[STYLE]`   | Name test variables: `inputEmail`, `mockRepo`, `actualResult`, `expectedState`. |
| `[STYLE]`   | Inject `UnconfinedTestDispatcher` or `StandardTestDispatcher`. Use `advanceUntilIdle()`. |

```kotlin
@Test
fun `emits error state when repository throws`() = runTest {
    // Arrange
    val mockRepo = mockk<UserRepository>()
    val viewModel = UserViewModel(mockRepo, UnconfinedTestDispatcher())
    coEvery { mockRepo.fetchUser(any()) } throws NetworkException("timeout")

    // Act
    viewModel.loadUser("user-123")

    // Assert
    with(viewModel.uiState.value) {
        assertThat(hasError).isTrue()
        assertThat(isLoading).isFalse()
    }
}
```