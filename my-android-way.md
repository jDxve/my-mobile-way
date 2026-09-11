<h1 align="center">My Android Way</h1>
<p align="center"><em><strong>Jetpack Compose Production Playbook</strong></em></p>

<p align="center">
  Google's recommended architecture, made concrete &middot; Unidirectional data flow &middot;
  ViewModel + StateFlow &middot; Typed networking &middot; Testing &middot; Compile-time boundaries
  <br />
  <em>The exact conventions I ship in production.</em>
</p>

<p align="center">
  <img alt="Kotlin" src="https://img.shields.io/badge/Kotlin-2.x-7F52FF?logo=kotlin&logoColor=white" />
  <img alt="Compose" src="https://img.shields.io/badge/Jetpack%20Compose-4285F4?logo=jetpackcompose&logoColor=white" />
  <img alt="State" src="https://img.shields.io/badge/State-ViewModel%20%2B%20StateFlow-4B32C3" />
  <img alt="DI" src="https://img.shields.io/badge/DI-Hilt-2E7D32" />
  <img alt="Boundaries" src="https://img.shields.io/badge/Boundaries-Compile--time-EF6C00" />
</p>

<p align="center">
  <b>Author</b> : <b>JOHN DAVE BAÑAS</b>
</p>

---

This is how I structure and write every Android app. Not just the architecture, but the
conventions, error handling, tests, and enforcement that keep it honest. Follow it when
scaffolding a new app, adding a feature, or reviewing code.

It follows Google's recommended app architecture. Where the official guidance offers a choice,
this document makes the choice and says why. Where it stops at principle, this document gives
the code.

## Contents

1. [Architecture Philosophy](#1-architecture-philosophy)
2. [Module and Folder Structure](#2-module-and-folder-structure)
3. [Core Layer](#3-core-layer)
4. [Design System Layer](#4-design-system-layer)
5. [Feature Anatomy (worked example: auth)](#5-feature-anatomy-worked-example-auth)
6. [State Management](#6-state-management)
7. [Testing](#7-testing)
8. [Tooling and Enforcement](#8-tooling-and-enforcement)
9. [Naming and File Conventions](#9-naming-and-file-conventions)
10. [New Feature Checklist](#10-new-feature-checklist)
11. [Production and Release Concerns](#11-production-and-release-concerns)

---

## Stack

- **UI**: Jetpack Compose + Material 3. Single Activity, no XML layouts, no Fragments.
- **State**: `ViewModel` + `StateFlow`. Compose runtime (`remember`/`mutableStateOf`) for UI-only state. No LiveData, no RxJava, no third-party MVI framework.
- **DI**: Hilt. Constructor injection everywhere; modules only for types you do not own.
- **Async**: Coroutines + Flow. Injected `CoroutineDispatcher` via qualifiers, never `Dispatchers.IO` at the call site.
- **Networking**: Retrofit + OkHttp + `kotlinx.serialization`. Retrofit for typed calls; OkHttp `Authenticator` for token refresh.
- **Result type**: a hand-written `sealed interface DataState<out T>` (Idle / Loading / Success / Error). No codegen.
- **Models**: plain `data class`. No annotation processing outside Hilt, Room, and serialization.
- **Storage**: `DataStore` (Preferences) for settings and cache metadata; `DataStore` + Tink for tokens. `Room` when a real database is warranted.
- **Navigation**: Navigation Compose with type-safe `@Serializable` routes.
- **Theming**: `MaterialTheme` + a `CompositionLocal` for custom semantic tokens.
- **Testing**: JUnit + MockK + Turbine + `kotlinx-coroutines-test` + MockWebServer. Compose UI tests via `createComposeRule`.

---

## 1. Architecture Philosophy

### 1.1 The two principles everything else follows from

**Separation of concerns.** UI code changes for one set of reasons (a redesign, a new
interaction) and data code changes for another (a new endpoint, a schema migration). Anything
that mixes them changes for both, so it breaks twice as often. The UI must not know how data
is fetched, and the data layer must not know how data is displayed.

**Drive the UI from data models.** The screen renders a model that exists independently of it.
Never store application state in a UI component, because the system can destroy that component
at any time: a rotation, a language change, a low-memory kill.

### 1.2 The layers

Google's guide names three:

```
 UI layer   ──────▶  Domain layer  ◀──────  Data layer
(Compose +           (models +              (repository impl,
 ViewModel)           contracts)             remote + local sources)
```

- **UI layer** displays state and forwards user events upward. It depends on the domain layer's models and contracts, never on the data layer.
- **Domain layer** is the contract in the middle: pure Kotlin models plus the repository interfaces that describe what the app can do. It depends on nothing. No Android framework types, no Retrofit, no `Context`, no Compose in any signature. A domain package should compile as a pure Kotlin/JVM module.
- **Data layer** implements those contracts. It owns the network, the database, and the mapping between wire formats and domain models.

**Dependencies point inward.** The data layer depends on the domain layer because it implements
it. The UI layer depends on the domain layer because it consumes it. The domain layer depends
on neither.

> **A note on where the repository lives.** The official guide places repositories in the data
> layer and treats the domain layer as optional, for use cases only. This document splits the
> difference deliberately: the repository *interface* lives in the domain layer, the
> *implementation* in the data layer. That costs one extra file per repository and buys a UI
> layer that cannot see Retrofit even by accident, plus a trivially fakeable seam for tests.
> Use cases stay optional. Add one only when logic is genuinely shared by several ViewModels or
> is complex enough to deserve its own tests; a use case that forwards a single call to a
> single repository is indirection with no payoff.

### 1.3 Unidirectional data flow

State flows down, events flow up. The ViewModel is the only thing that decides what the state
is; the UI never mutates it.

```
      ┌──────────────── state ────────────────┐
      │                                       ▼
 ViewModel                                 Composable
      ▲                                       │
      └──────────────── events ───────────────┘
```

This is what makes the UI a pure function of state, which in turn is what makes previews,
screenshot tests, and UI tests possible without mocking half the app.

### 1.4 Feature-first on the outside

The app is a set of self-contained features. Each feature owns its full vertical slice
(`data` → `domain` → `ui`) in one place. Everything needed to build, test, or delete a feature
sits in one directory. No feature reaches into another feature's internals; shared contracts go
through `core`.

The alternative (top-level `models/`, `repositories/`, `screens/` buckets) means every change
touches four directories and deleting a feature is an archaeology exercise.

### 1.5 Composables are minimal

A screen composable composes components and renders state. It holds no business logic, no
`Context`-dependent work beyond resource lookup, and no direct repository access.

Every screen splits in two: a **stateful route** that owns the ViewModel and wires effects, and
a **stateless screen** that takes state in and sends events out. This is the single highest-value
convention in the document. It is what makes `@Preview` work, what makes UI tests need no DI,
and what keeps navigation decisions out of feature code.

---

## 2. Module and Folder Structure

### 2.1 Start here: a single module

Most apps should start as one module. The layering above is about *dependency direction*, not
about Gradle, and packages express it perfectly well:

```
app/src/main/kotlin/<pkg>/
  core/
    common/            # DataState, AppException, dispatcher qualifiers
    network/           # OkHttp/Retrofit setup, interceptors, Authenticator
    datastore/         # DataStore instances, TokenStore
    designsystem/      # theme, tokens, spacing, reusable composables
  feature/
    auth/
      data/            # Retrofit API, DTOs, repository impl, Hilt module
      domain/          # models, repository interface
      ui/              # route, screen, ViewModel, components
      navigation/      # route types + NavGraphBuilder extension
    home/
  MainActivity.kt
  App.kt
```

Every convention in this document works unchanged in this layout. What you give up is §8.1:
boundaries are enforced by Konsist tests rather than by the compiler.

### 2.2 Promote to modules when a trigger fires

Split into Gradle modules when one of these is actually true, not before:

- **Build times hurt.** Incremental builds are recompiling the world because everything is one compilation unit.
- **More than one developer** is working in the codebase and boundaries need to hold without review catching every slip.
- **Roughly eight or more features**, where package discipline has visibly started to erode.
- **A second target** needs the code: an instant app, a Wear or TV app, a shared library.

Then the layout becomes:

```
settings.gradle.kts
build-logic/                       # convention plugins (see §8.4)
app/                               # Application, MainActivity, NavHost, DI wiring only

core/
  model/                           # pure Kotlin (kotlin("jvm")) domain models
  common/                          # DataState, AppException, dispatcher qualifiers
  network/                         # OkHttp/Retrofit factory, interceptors, Authenticator
  datastore/                       # DataStore instances, TokenStore, preference keys
  designsystem/                    # theme, tokens, typography, spacing, reusable composables
  testing/                         # MainDispatcherRule, fakes, fixtures

feature/
  auth/
    src/main/kotlin/<pkg>/feature/auth/
      data/
        remote/                    # Retrofit interface + DTOs   (internal)
        repository/                # AuthRepositoryImpl          (internal)
        di/                        # Hilt module binding impl to interface
      domain/
        model/                     # pure Kotlin models          (public)
        repository/                # AuthRepository interface    (public)
      ui/
        login/
          LoginRoute.kt            # stateful: owns the ViewModel
          LoginScreen.kt           # stateless: state in, events out
          LoginViewModel.kt
          components/
      navigation/
        AuthNavigation.kt          # route types + NavGraphBuilder extension
  home/
```

**One module per feature, not three.** Splitting a single feature into `:feature:auth:data` /
`:domain` / `:ui` sub-modules buys three build files and nothing else, unless that feature's
domain genuinely needs to compile for a non-Android target.

**Visibility is the intra-module boundary.** Inside a feature module, everything in `data/` is
`internal`. The only public surface is the domain interface, the domain models, the route types,
and the `NavGraphBuilder` extension. Because `internal` is module-scoped in Kotlin, a DTO or a
repository implementation is physically invisible to the rest of the app.

A feature with no networking omits `data/` and `domain/` entirely.

---

## 3. Core Layer

### 3.1 Config and flavors

Environment config comes from the build, never from a committed Kotlin constant. Gradle flavors
are cheap, so there is no reason to hardcode a URL and recompile to switch.

```kotlin
// app/build.gradle.kts
android {
    buildFeatures { buildConfig = true }

    flavorDimensions += "env"
    productFlavors {
        create("dev") {
            dimension = "env"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            buildConfigField("String", "BASE_URL", "\"http://10.0.2.2:4000/api/v1/\"")
        }
        create("prod") {
            dimension = "env"
            buildConfigField("String", "BASE_URL", "\"https://api.example.com/v1/\"")
        }
    }
}
```

Distinct application IDs mean dev and prod can sit on one device at the same time. `10.0.2.2`
is the emulator's alias for the host machine's localhost; a physical device on the same LAN uses
the machine's IP.

Add a `staging` flavor when a staging backend actually exists. Two flavors until then.

Cleartext HTTP is blocked by the platform by default. Allow it **only** in the dev flavor, via a
flavor-specific network security config:

```xml
<!-- app/src/dev/res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

A prod build then physically cannot talk cleartext. That is a manifest-level guarantee, stronger
than any runtime check you could forget to call.

Secrets (API keys, DSNs) come from a git-ignored `secrets.properties` read by Gradle, or from the
CI secret store. Never `buildConfigField` with a real key out of a committed file.

### 3.2 Result type (`core/common/DataState.kt`)

The project-wide result type for operations whose success, failure, and in-flight states the UI
must all render.

```kotlin
sealed interface DataState<out T> {
    data object Idle : DataState<Nothing>
    data object Loading : DataState<Nothing>
    data class Success<out T>(val data: T) : DataState<T>
    data class Error(val exception: AppException) : DataState<Nothing>
}
```

`out T` is what lets `DataState.Loading` be assignable to `DataState<AuthToken>` without a cast,
and what lets `Idle`, `Loading`, and `Error` be singletons rather than allocating per call.

**Use it where the caller needs those semantics; do not wrap reflexively.** A fire-and-forget
call (`suspend fun logout()`) and an observable stream (`fun observeSession(): Flow<AuthToken?>`)
are both cleaner unwrapped. Wrapping everything is ceremony, not consistency.

Small helpers keep call sites flat:

```kotlin
inline fun <T> DataState<T>.onSuccess(block: (T) -> Unit): DataState<T> {
    if (this is DataState.Success) block(data)
    return this
}

inline fun <T, R> DataState<T>.mapData(transform: (T) -> R): DataState<R> = when (this) {
    is DataState.Success -> DataState.Success(transform(data))
    is DataState.Error -> this
    DataState.Idle -> DataState.Idle
    DataState.Loading -> DataState.Loading
}

val <T> DataState<T>.dataOrNull: T? get() = (this as? DataState.Success)?.data
val DataState<*>.isLoading: Boolean get() = this is DataState.Loading
```

Consume it with an exhaustive `when`. Because it is sealed, adding a branch later turns every
consumer into a compile error until it is handled. That is the point.

### 3.3 AppException (`core/common/AppException.kt`)

The transport-agnostic error type carried by `DataState.Error`. Pure Kotlin. The data boundary
converts platform exceptions into this, so nothing above the repository ever sees an
`HttpException` or an `IOException`.

```kotlin
enum class ErrorType { TIMEOUT, CONNECTION, UNAUTHORIZED, SERVER, UNKNOWN }

data class AppException(
    val statusCode: Int? = null,
    val serverMessage: String? = null,
    val type: ErrorType = ErrorType.UNKNOWN,
    override val cause: Throwable? = null,
) : Exception(serverMessage, cause)
```

The mapping lives in `core/network`, since it needs Retrofit types:

```kotlin
fun Throwable.toAppException(): AppException = when (this) {
    is AppException -> this
    is SocketTimeoutException -> AppException(type = ErrorType.TIMEOUT, cause = this)
    is UnknownHostException,
    is ConnectException,
    is IOException -> AppException(type = ErrorType.CONNECTION, cause = this)
    is HttpException -> AppException(
        statusCode = code(),
        serverMessage = extractServerMessage(this),
        type = when {
            code() == 401 -> ErrorType.UNAUTHORIZED
            code() >= 500 -> ErrorType.SERVER
            else -> ErrorType.UNKNOWN
        },
        cause = this,
    )
    else -> AppException(cause = this)
}
```

`extractServerMessage` reads `response()?.errorBody()?.string()` **once** and parses the error
envelope. That call consumes the stream, so a second read returns an empty string. Read it here
and nowhere else.

### 3.4 The guard (`core/common/SafeCall.kt`)

One top-level inline function collapses the repetitive `try/catch` in every repository method.

```kotlin
suspend inline fun <T> safeCall(
    crossinline onError: (Throwable) -> DataState<T>? = { null },
    crossinline block: suspend () -> T,
): DataState<T> = try {
    DataState.Success(block())
} catch (e: CancellationException) {
    throw e
} catch (e: Throwable) {
    onError(e) ?: DataState.Error(e.toAppException())
}
```

**The `CancellationException` rethrow is not optional.** Coroutine cancellation is delivered as
an exception. A bare `catch (e: Throwable)` swallows it, and the coroutine machinery breaks in
ways that are painful to debug: a ViewModel that will not clear, a `stateIn` that never stops
collecting, a `withTimeout` that silently does nothing. Every catch-all in the codebase rethrows
it first.

The `onError` hook handles status codes that are not really failures:

```kotlin
override suspend fun fetchMyHousehold(): DataState<Household?> = safeCall(
    onError = { e ->
        if (e is HttpException && e.code() == 404) DataState.Success(null) else null
    },
) {
    api.getMyHousehold()?.toDomain()
}
```

### 3.5 Dispatchers (`core/common/Dispatchers.kt`)

Never call `Dispatchers.IO` inside a repository. Inject it, so tests can swap in a
`TestDispatcher` and control virtual time.

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class IoDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatchersModule {
    @Provides @IoDispatcher
    fun providesIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
}
```

Repositories take `@IoDispatcher private val io: CoroutineDispatcher` and wrap work in
`withContext(io)`.

ViewModels do **not** specify a dispatcher. `viewModelScope` is main-bound by design, and the
suspend function it calls is responsible for its own threading. A suspend function that blocks
its caller's thread is a bug in that function, not in the caller.

### 3.6 Networking (`core/network`)

One Hilt module builds the whole stack. `@Singleton` throughout: one `OkHttpClient` for the app,
sharing its connection pool and thread pool.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides @Singleton
    fun okHttpClient(
        authInterceptor: AuthInterceptor,
        tokenAuthenticator: TokenAuthenticator,
    ): OkHttpClient = OkHttpClient.Builder()
        .connectTimeout(10, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .addInterceptor(authInterceptor)
        .authenticator(tokenAuthenticator)
        .apply {
            if (BuildConfig.DEBUG) {
                addInterceptor(
                    HttpLoggingInterceptor().apply {
                        level = HttpLoggingInterceptor.Level.BODY
                        redactHeader("Authorization")
                        redactHeader("Cookie")
                    },
                )
            }
        }
        .build()

    @Provides @Singleton
    fun json(): Json = Json {
        ignoreUnknownKeys = true     // a new server field must never crash the app
        explicitNulls = false
        coerceInputValues = true
    }

    @Provides @Singleton
    fun retrofit(client: OkHttpClient, json: Json): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.BASE_URL)
        .client(client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
}
```

`ignoreUnknownKeys = true` is a production requirement, not a convenience. Without it, the day
the backend adds a field, every already-installed client starts throwing.

The logging interceptor is `BuildConfig.DEBUG`-gated **and** redacts the auth header. Both.
`redactHeader` still logs the header name with a masked value, which is what you want: you can
confirm the header was attached without leaking the token into logcat.

### 3.7 Auth: interceptor and Authenticator

Two pieces with two jobs. OkHttp has a purpose-built hook for each, and conflating them is where
most token-refresh implementations go wrong.

**`AuthInterceptor` attaches the bearer. That is all it does.**

```kotlin
@Singleton
class AuthInterceptor @Inject constructor(
    private val tokenStore: TokenStore,
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        if (request.header(NO_AUTH) != null) {
            return chain.proceed(request.newBuilder().removeHeader(NO_AUTH).build())
        }
        val token = runBlocking { tokenStore.accessToken() }
        val authed = if (token != null) {
            request.newBuilder().header("Authorization", "Bearer $token").build()
        } else {
            request
        }
        return chain.proceed(authed)
    }

    companion object { const val NO_AUTH = "No-Auth" }
}
```

`runBlocking` here is correct, not a smell: OkHttp runs interceptors on its own background
dispatcher, never on the main thread. The `No-Auth` header lets login and refresh opt out, which
is how you avoid attaching a stale token to the very request trying to replace it.

**`TokenAuthenticator` handles 401.** OkHttp calls it automatically, only on a 401, and replays
the request with whatever you return.

```kotlin
@Singleton
class TokenAuthenticator @Inject constructor(
    private val tokenStore: TokenStore,
    private val refreshApi: Provider<AuthRefreshApi>,
    private val sessionEvents: SessionEventBus,
) : Authenticator {

    private val mutex = Mutex()

    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.retryCount() >= 1) return null

        val staleToken = response.request.header("Authorization")?.removePrefix("Bearer ")

        val newToken = runBlocking {
            mutex.withLock {
                val current = tokenStore.accessToken()
                if (current != null && current != staleToken) return@withLock current
                refreshOrNull()
            }
        }

        if (newToken == null) {
            runBlocking { tokenStore.clearSession() }
            sessionEvents.expire()
            return null
        }

        return response.request.newBuilder()
            .header("Authorization", "Bearer $newToken")
            .build()
    }

    private suspend fun refreshOrNull(): String? {
        val refresh = tokenStore.refreshToken() ?: return null
        return runCatching { refreshApi.get().refresh(RefreshRequestDto(refresh)) }
            .onSuccess { tokenStore.save(it.toDomain()) }
            .map { it.accessToken }
            .getOrNull()
    }
}

private fun Response.retryCount(): Int {
    var count = 0
    var prior = priorResponse
    while (prior != null) { count++; prior = prior.priorResponse }
    return count
}
```

Three things are doing real work here:

- **`Mutex` + the stale-token check.** Ten concurrent requests can 401 at once. The mutex serializes them; the first refreshes, the other nine wake up, see that the stored token no longer matches the one they sent, and reuse it. One refresh, ten replays. §7.4 pins this.
- **`retryCount() >= 1`.** OkHttp keeps calling `authenticate` as long as you keep returning a request. Returning `null` on the second attempt is what prevents an infinite refresh loop against a server that 401s regardless.
- **`Provider<AuthRefreshApi>`.** The refresh API is built from Retrofit, which is built from the `OkHttpClient`, which needs the `Authenticator`. `Provider` defers resolution past construction and breaks the cycle, which would otherwise be a Hilt build failure. The refresh call itself carries the `No-Auth` header.

`SessionEventBus` is a `@Singleton` wrapping a `MutableSharedFlow<Unit>` with
`extraBufferCapacity = 1` and `tryEmit`. The root composable collects it and navigates to login.
No global mutable state, no `NavController` held in a singleton.

### 3.8 Token storage (`core/datastore`)

`EncryptedSharedPreferences` and the whole `androidx.security:security-crypto` library were
deprecated at 1.1.0-alpha07 in April 2025, with no further releases planned. Do not start a new
project on it. The current answer is **DataStore + Tink**, with the keyset wrapped by a master
key from the Android Keystore.

```kotlin
@Singleton
class TokenStore @Inject constructor(
    private val dataStore: DataStore<Preferences>,
    private val crypto: Aead,                       // Tink, keyset wrapped by Android Keystore
) {
    private val accessKey = stringPreferencesKey("access_token")
    private val refreshKey = stringPreferencesKey("refresh_token")

    suspend fun accessToken(): String? = read(accessKey)
    suspend fun refreshToken(): String? = read(refreshKey)

    suspend fun save(token: AuthToken) {
        dataStore.edit {
            it[accessKey] = crypto.encryptToBase64(token.accessToken)
            it[refreshKey] = crypto.encryptToBase64(token.refreshToken)
        }
    }

    suspend fun clearSession() {
        dataStore.edit { it.remove(accessKey); it.remove(refreshKey) }
    }

    private suspend fun read(key: Preferences.Key<String>): String? =
        runCatching { dataStore.data.first()[key]?.let(crypto::decryptFromBase64) }
            .getOrElse { clearSession(); null }
}
```

**Two platform pitfalls this guards against.**

- **Keyset corruption on cold start.** An OS update, a device-to-device migration, or a cloud restore can leave the Keystore unable to decrypt an existing keyset. Every read then throws, and because it happens during startup the app cold-crashes on launch with no recovery path for the user. The `runCatching` above degrades to "logged out" instead. Route the initial session read through it.
- **Auto Backup resurrecting a dead session.** Auto Backup will happily back up a DataStore file whose encryption key is device-bound, restoring undecryptable garbage onto a new device. Exclude it:

```xml
<!-- res/xml/backup_rules.xml, referenced from android:dataExtractionRules -->
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="file" path="datastore/" />
    </cloud-backup>
</data-extraction-rules>
```

App data is wiped on uninstall, so a reinstall starts genuinely clean. No first-launch wipe is
needed.

---

## 4. Design System Layer

`core/designsystem` holds theme, tokens, and reusable composables. It depends on Compose and
nothing else. No feature, no repository, no `DataState`.

### 4.1 Theme and semantic colors

Material 3's `ColorScheme` covers most roles. For roles it does not have (success, warning, a
brand accent), add a `CompositionLocal` rather than threading colors through parameters or
reaching for raw constants.

```kotlin
@Immutable
data class ExtendedColors(
    val success: Color,
    val warning: Color,
    val onSuccess: Color,
)

val LocalExtendedColors = staticCompositionLocalOf<ExtendedColors> {
    error("ExtendedColors not provided. Wrap the content in AppTheme.")
}

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S ->
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    CompositionLocalProvider(
        LocalExtendedColors provides if (darkTheme) DarkExtended else LightExtended,
        LocalSpacing provides Spacing(),
    ) {
        MaterialTheme(colorScheme = colorScheme, typography = AppTypography, content = content)
    }
}

val MaterialTheme.extended: ExtendedColors
    @Composable @ReadOnlyComposable get() = LocalExtendedColors.current
```

`staticCompositionLocalOf` over `compositionLocalOf`: theme values rarely change, and the static
variant skips the invalidation bookkeeping. Use the dynamic variant only for values that
genuinely change at runtime.

**Decide on `dynamicColor` deliberately.** Material You gives you free system integration and
costs you brand control. If the app has a brand palette that must hold, pass
`dynamicColor = false` and mean it.

### 4.2 Spacing

`dp` is already density-independent and `sp` already respects the user's font scale, so there is
no scaling layer to build. What you want is a spacing scale, so that layout rhythm is a decision
made once rather than per screen.

```kotlin
@Immutable
data class Spacing(
    val xs: Dp = 4.dp,
    val sm: Dp = 8.dp,
    val md: Dp = 16.dp,
    val lg: Dp = 24.dp,
    val xl: Dp = 32.dp,
)

val LocalSpacing = staticCompositionLocalOf { Spacing() }
```

Used as `Modifier.padding(MaterialTheme.spacing.md)`.

**Scope:** the scale governs *layout* spacing, the gaps between elements. A component's own
internal dimensions (a 1.dp divider, a 20.dp icon, a 56.dp bar height) are that component's
business and belong as `private val`s next to it. Forcing every number through the scale turns
a design system into ceremony.

Adapt to screen size with `WindowSizeClass`, never by scaling. A tablet should get a different
layout, not the same layout with bigger numbers.

### 4.3 Strings

All copy lives in `res/values/strings.xml`, read via `stringResource(R.string.x)`. This is how
plurals, argument reordering, and locale fallback work on the platform, so localization costs
nothing later instead of becoming a migration.

Never build a user-facing sentence by concatenating strings. Use positional arguments:
`<string name="items_selected">%1$d of %2$d selected</string>`.

### 4.4 Error copy

`AppException` becomes a user-facing string in exactly one place, at the UI edge. Take a
`Context` so the function works both inside composition and inside a `LaunchedEffect`:

```kotlin
fun AppException.asMessage(context: Context): String = when {
    type == ErrorType.TIMEOUT -> context.getString(R.string.error_timeout)
    type == ErrorType.CONNECTION -> context.getString(R.string.error_offline)
    type == ErrorType.UNAUTHORIZED -> context.getString(R.string.error_session_expired)
    type == ErrorType.SERVER -> context.getString(R.string.error_server)
    else -> serverMessage ?: context.getString(R.string.error_generic)
}

@Composable
fun AppException.asMessage(): String = asMessage(LocalContext.current)
```

A ViewModel must **never** produce a user-facing string. It has no `Context` and should not have
one. It emits an `AppException` or a `UiText` sealed type; the UI resolves it.

### 4.5 Observability

`asMessage()` produces copy. It is not where an error goes to die. Reporting is a separate,
one-way concern wired in one place: an OkHttp `EventListener` or a reporting interceptor
registered **after** the `Authenticator` in the chain, so it only sees failures that survived the
refresh and replay. It forwards the request path, status code, and a redacted breadcrumb. Never
the token, never an `/auth/` body. Filter out offline `IOException`s and deliberate
404-as-empty so the dashboard stays signal.

Uncaught errors need a catch-all installed in the `Application` class. In practice Crashlytics
or Sentry installs one for you; the rule that matters is that you chain rather than replace an
existing handler.

```kotlin
@HiltAndroidApp
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        val previous = Thread.getDefaultUncaughtExceptionHandler()
        Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
            reporter.recordFatal(throwable)
            previous?.uncaughtException(thread, throwable)
        }
    }
}
```

The rule: the user sees a snackbar, and you see the stack trace. Neither substitutes for the
other, and reporting happens in one place, not sprinkled through repositories.

---

## 5. Feature Anatomy (worked example: `auth`)

Flow: **api → repository → ViewModel → route → screen**, with `domain` as the contract in the
middle.

### 5.1 `domain/model`

```kotlin
data class AuthToken(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Int,
    val user: AuthUser,
)

data class AuthUser(val id: String, val email: String, val displayName: String)
```

Pure Kotlin. No `@Serializable`, no Android imports, no nullable-everything. If the server may
omit a field, the DTO carries the nullability and `toDomain()` decides what the domain-level
truth is.

### 5.2 `data/remote`

```kotlin
internal interface AuthApi {
    @Headers("No-Auth: true")
    @POST("auth/login")
    suspend fun login(@Body body: LoginRequestDto): LoginResponseDto

    @POST("auth/logout")
    suspend fun logout()
}

@Serializable
internal data class LoginRequestDto(
    val email: String,
    val password: String,
    val clientType: String = "mobile",
)

@Serializable
internal data class LoginResponseDto(
    val accessToken: String,
    val refreshToken: String,
    val expiresIn: Int,
    val user: UserDto,
) {
    fun toDomain() = AuthToken(accessToken, refreshToken, expiresIn, user.toDomain())
}
```

`internal` on every DTO and on the API interface. A DTO that escapes the module is a design
error the compiler can catch.

Note the relative path (`auth/login`, no leading slash) and the trailing slash on `BASE_URL`.
Retrofit resolves these as URLs: a leading slash discards the base URL's path segment, which is
a classic way to silently lose your `/api/v1` prefix.

### 5.3 `domain/repository`

```kotlin
interface AuthRepository {
    suspend fun login(email: String, password: String): DataState<AuthToken>
    suspend fun logout()
    fun observeSession(): Flow<AuthToken?>
}
```

`suspend` returning `DataState<T>` when the caller must render loading, success, and error.
Plain `suspend` for fire-and-forget. `Flow<T>` for anything observable.

Avoid `Flow<DataState<T>>` for a simple one-shot call. That combination earns its complexity
only when you are genuinely emitting cached-then-fresh.

### 5.4 `data/repository` and DI

```kotlin
internal class AuthRepositoryImpl @Inject constructor(
    private val api: AuthApi,
    private val tokenStore: TokenStore,
    @IoDispatcher private val io: CoroutineDispatcher,
) : AuthRepository {

    override suspend fun login(email: String, password: String): DataState<AuthToken> =
        withContext(io) {
            safeCall { api.login(LoginRequestDto(email, password)).toDomain() }
                .onSuccess { tokenStore.save(it) }
        }

    override suspend fun logout() = withContext(io) {
        runCatching { api.logout() }        // best effort; local cleanup happens regardless
        tokenStore.clearSession()
    }

    override fun observeSession(): Flow<AuthToken?> = tokenStore.observeSession()
}
```

`logout` clears locally even when the network call fails. A user who taps logout on a plane must
end up logged out.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class AuthDataModule {

    @Binds @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository

    companion object {
        @Provides @Singleton
        fun provideAuthApi(retrofit: Retrofit): AuthApi = retrofit.create()
    }
}
```

`@Binds` over `@Provides` for interface-to-implementation: less generated code, clearer intent.
The module is `internal`, so the binding is invisible outside the feature even though the
resulting `AuthRepository` is injectable app-wide.

### 5.5 `ui` — ViewModel

```kotlin
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val repository: AuthRepository,
) : ViewModel() {

    private val _state = MutableStateFlow<DataState<AuthToken>>(DataState.Idle)
    val state: StateFlow<DataState<AuthToken>> = _state.asStateFlow()

    private val _events = Channel<LoginEvent>(Channel.BUFFERED)
    val events: Flow<LoginEvent> = _events.receiveAsFlow()

    fun login(email: String, password: String) {
        if (_state.value is DataState.Loading) return
        viewModelScope.launch {
            _state.value = DataState.Loading
            val result = repository.login(email, password)
            _state.value = result
            when (result) {
                is DataState.Success -> _events.send(LoginEvent.NavigateHome)
                is DataState.Error -> _events.send(LoginEvent.ShowError(result.exception))
                else -> Unit
            }
        }
    }
}

sealed interface LoginEvent {
    data object NavigateHome : LoginEvent
    data class ShowError(val exception: AppException) : LoginEvent
}
```

The `isLoading` early return is the double-submit guard. Do not rely on the button's disabled
state alone; a fast double tap can slip between recompositions.

### 5.6 `ui` — route (stateful)

```kotlin
@Composable
fun LoginRoute(
    onLoggedIn: () -> Unit,
    viewModel: LoginViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    val snackbarHostState = remember { SnackbarHostState() }
    val context = LocalContext.current

    ObserveAsEvents(viewModel.events) { event ->
        when (event) {
            LoginEvent.NavigateHome -> onLoggedIn()
            is LoginEvent.ShowError ->
                snackbarHostState.showSnackbar(event.exception.asMessage(context))
        }
    }

    LoginScreen(
        state = state,
        snackbarHostState = snackbarHostState,
        onSubmit = viewModel::login,
    )
}

@Composable
fun <T> ObserveAsEvents(flow: Flow<T>, onEvent: suspend (T) -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(flow, lifecycleOwner) {
        lifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            flow.collect(onEvent)
        }
    }
}
```

`collectAsStateWithLifecycle()`, never `collectAsState()`. The plain version keeps collecting
while the app is backgrounded, which burns battery and can deliver a navigation event to a screen
the user cannot see.

`repeatOnLifecycle(STARTED)` around events is the same protection: a navigation event fired while
backgrounded would act on a detached `NavController`.

The route takes navigation as lambdas (`onLoggedIn: () -> Unit`), never a `NavController`. This
keeps the feature independent of the app's navigation graph and keeps the route previewable.

### 5.7 `ui` — screen (stateless)

```kotlin
@Composable
fun LoginScreen(
    state: DataState<AuthToken>,
    snackbarHostState: SnackbarHostState,
    onSubmit: (email: String, password: String) -> Unit,
    modifier: Modifier = Modifier,
) {
    var email by rememberSaveable { mutableStateOf("") }
    var password by rememberSaveable { mutableStateOf("") }

    Scaffold(
        modifier = modifier,
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        Column(
            modifier = Modifier
                .padding(padding)
                .padding(MaterialTheme.spacing.lg)
                .verticalScroll(rememberScrollState()),
        ) {
            LoginTitle()
            Spacer(Modifier.height(MaterialTheme.spacing.xl))
            AppTextField(
                value = email,
                onValueChange = { email = it },
                label = stringResource(R.string.email),
            )
            Spacer(Modifier.height(MaterialTheme.spacing.md))
            AppPasswordField(value = password, onValueChange = { password = it })
            Spacer(Modifier.height(MaterialTheme.spacing.lg))
            AppButton(
                text = stringResource(R.string.sign_in),
                loading = state.isLoading,
                onClick = { onSubmit(email, password) },
            )
        }
    }
}

@Preview @Composable
private fun LoginScreenPreview() = AppTheme {
    LoginScreen(DataState.Idle, SnackbarHostState(), { _, _ -> })
}
```

**Text field state stays in the composable, not the ViewModel.** `rememberSaveable` survives
configuration change and process death through the saved-instance bundle, which is everything a
form field needs. Hoisting every keystroke into a `StateFlow` costs a recomposition per character
and buys nothing. Hoist it only when the ViewModel must *react* to the value: live validation
against the server, a debounced search query.

A stateless screen with a `@Preview` is the payoff for the route/screen split. Every screen gets
at least one preview, and screens with meaningful loading or error states get one each.

### 5.8 `navigation`

Type-safe routes, so a typo is a compile error rather than a runtime crash.

```kotlin
@Serializable data object LoginRoute
@Serializable data class OrderDetailRoute(val orderId: String)

fun NavGraphBuilder.authGraph(onLoggedIn: () -> Unit) {
    composable<LoginRoute> {
        LoginRoute(onLoggedIn = onLoggedIn)
    }
}
```

Arguments come back typed via `backStackEntry.toRoute<OrderDetailRoute>()`. Pass **IDs only**,
never whole objects. The back stack is persisted to a bundle, and a large object there is a
`TransactionTooLargeException` waiting for a slow day.

The app module assembles the graph and owns every cross-feature navigation decision:

```kotlin
NavHost(navController, startDestination = LoginRoute) {
    authGraph(onLoggedIn = { navController.navigate(HomeRoute) { popUpTo(0) } })
    homeGraph(onOrderClick = { navController.navigate(OrderDetailRoute(it)) })
}
```

Session expiry is handled once, at the root: collect `SessionEventBus` and navigate to login with
`popUpTo(0)`. No per-screen auth guard is needed, because a feature is reachable only through the
graph the app module built.

---

## 6. State Management

State flows down, events flow up, and the ViewModel is the only thing that decides what the
state is.

### 6.1 The three tiers, and how to pick

```
Does anything outside this composable care about the value?     NO  → remember / rememberSaveable
Does it belong to the screen and survive configuration change?  YES → ViewModel + StateFlow
Does it come from a repository and change over time?            YES → repository Flow → stateIn
```

Getting tier 1 wrong is the most common over-architecture mistake. A dropdown's expanded flag
living in a ViewModel is not "more architectural," it is a leak of UI concerns into a
lifecycle-scoped object that now has to be reset, tested, and reasoned about.

Prefer `rememberSaveable` over `remember` whenever losing the value on rotation would annoy the
user. It costs nothing extra for primitives.

### 6.2 Three ViewModel shapes

**Shape 1: single action.** State *is* a `DataState<T>`. See §5.5. This covers most screens;
start here.

**Shape 2: observed stream.** State comes from a repository `Flow`. No manual loading flag;
`stateIn` supplies the initial value.

```kotlin
@HiltViewModel
class OrdersViewModel @Inject constructor(
    repository: OrderRepository,
) : ViewModel() {

    val state: StateFlow<DataState<List<Order>>> = repository.observeOrders()
        .map<List<Order>, DataState<List<Order>>> { DataState.Success(it) }
        .catch { emit(DataState.Error(it.toAppException())) }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = DataState.Loading,
        )
}
```

**`WhileSubscribed(5_000)` is the number that matters.** It stops the upstream flow five seconds
after the last collector goes away. On rotation the ViewModel survives but the composable is
briefly destroyed, so a zero timeout would tear down and restart the database query or network
subscription every time the user turns the phone. Five seconds covers that gap and still releases
resources when the user actually leaves. `SharingStarted.Eagerly` keeps it running for the
ViewModel's whole life; use it only when you want exactly that.

**Shape 3: composite UI state.** A `data class` holding several slices plus per-item flags.

```kotlin
data class FamilyUiState(
    val household: DataState<Household?> = DataState.Idle,
    val removingMemberIds: Set<String> = emptySet(),
    val isRefreshing: Boolean = false,
)

@HiltViewModel
class FamilyViewModel @Inject constructor(
    private val repository: FamilyRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(FamilyUiState())
    val state: StateFlow<FamilyUiState> = _state.asStateFlow()

    private val _events = Channel<FamilyEvent>(Channel.BUFFERED)
    val events: Flow<FamilyEvent> = _events.receiveAsFlow()

    fun removeMember(id: String) {
        viewModelScope.launch {
            _state.update { it.copy(removingMemberIds = it.removingMemberIds + id) }
            try {
                when (val result = repository.removeMember(id)) {
                    is DataState.Error -> _events.send(FamilyEvent.ShowError(result.exception))
                    else -> refresh()
                }
            } finally {
                _state.update { it.copy(removingMemberIds = it.removingMemberIds - id) }
            }
        }
    }
}
```

Always `_state.update { }`, never `_state.value = _state.value.copy(...)`. `update` is
compare-and-set; the naive form has a read-modify-write race under concurrent coroutines.

Use this shape as soon as a screen has more than one independent piece of state. `data class` +
`copy` is cheap, and two separate `StateFlow`s that must stay consistent with each other are
harder to reason about than one that cannot drift.

### 6.3 One-off events

State is for *what to render*. Events are for *what to do once*: navigate, show a snackbar,
launch an intent. Never model these as state, or they replay on every rotation.

`Channel` + `receiveAsFlow` is the right primitive: it buffers while nothing is collecting and
delivers each event exactly once. `SharedFlow` with `replay = 0` drops events fired while
backgrounded, which is usually wrong for navigation.

### 6.4 Scoping

- `hiltViewModel()` inside a `composable<Route>` block scopes the ViewModel to that back-stack entry. It is created on navigate and cleared on pop, which is the right default and needs no opt-in.
- To share a ViewModel across a nested graph (a multi-step wizard), scope it to the parent entry: `hiltViewModel(remember(entry) { navController.getBackStackEntry(WizardGraph) })`.
- App-wide state (session, theme) lives in a `@Singleton` repository, not a ViewModel. ViewModels read it; they do not own it.
- **Bottom navigation**: Navigation Compose pops and recreates tab destinations by default. Preserve per-tab state with `saveState = true` / `restoreState = true` in the navigate options, plus `popUpTo(graph.findStartDestination().id) { saveState = true }`.

### 6.5 `SavedStateHandle`

Process death is a real state on Android and cheap to handle. Anything the user typed or selected
that would be painful to redo goes through it:

```kotlin
private val query = savedStateHandle.getStateFlow("query", "")
fun setQuery(value: String) { savedStateHandle["query"] = value }
```

Route arguments arrive here automatically, which is how `toRoute<OrderDetailRoute>()` works inside
a ViewModel.

### 6.6 Invariants

Violating any of these produces a bug, not a style disagreement.

- A ViewModel never imports `androidx.compose.*`, never holds a `Context`, never holds a `NavController`, never returns a composable.
- A composable never calls a repository.
- `MutableStateFlow` is always private; the public surface is `StateFlow` via `asStateFlow()`.
- Collect with `collectAsStateWithLifecycle()`, not `collectAsState()`.
- Mutate composite state with `update { }`, not read-modify-write.
- Every catch-all rethrows `CancellationException` first.
- Never launch app-scoped work in `viewModelScope`. Work that must finish after the user leaves the screen (an upload, a logout call) belongs to a `@Singleton` injected `CoroutineScope` built from `SupervisorJob() + Dispatchers.Default`, or to `WorkManager` if it must survive process death.

The ViewModel's public API is intent methods (verbs: `login`, `removeMember`, `refresh`) plus
`state` and `events`. Public mutable properties are a bug.

---

## 7. Testing

The layering exists so each piece is testable in isolation. Four seams, four kinds of test:
repository, ViewModel, Authenticator, screen. MockK for doubles, Turbine for flows,
`kotlinx-coroutines-test` for virtual time, MockWebServer for HTTP. Never a real socket, never a
real keystore, never `Thread.sleep`.

### 7.1 The dispatcher rule

`viewModelScope` dispatches to `Dispatchers.Main`, which does not exist in a JVM unit test. Every
ViewModel test needs this:

```kotlin
class MainDispatcherRule(
    private val dispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(dispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}
```

`UnconfinedTestDispatcher` runs coroutines eagerly, which keeps state assertions
straightforward. Switch to `StandardTestDispatcher` and drive it with `advanceUntilIdle()` when
you need to assert on an intermediate state precisely.

### 7.2 Repository test: mock the API, assert the branches

The repository's job is to call the API, map DTO to domain, wrap in `DataState`, and persist or
roll back. Assert every branch.

```kotlin
class AuthRepositoryImplTest {

    private val api = mockk<AuthApi>()
    private val tokenStore = mockk<TokenStore>(relaxed = true)
    private val repository = AuthRepositoryImpl(api, tokenStore, UnconfinedTestDispatcher())

    @Test
    fun `login returns Success and persists the token`() = runTest {
        coEvery { api.login(any()) } returns loginResponseDto

        val result = repository.login("a@b.com", "pw")

        assertIs<DataState.Success<AuthToken>>(result)
        coVerify(exactly = 1) { tokenStore.save(match { it.accessToken == "abc" }) }
    }

    @Test
    fun `login maps a 401 to an UNAUTHORIZED AppException and persists nothing`() = runTest {
        coEvery { api.login(any()) } throws HttpException(
            Response.error<Unit>(401, "{}".toResponseBody("application/json".toMediaType())),
        )

        val result = repository.login("a@b.com", "wrong")

        val error = assertIs<DataState.Error>(result).exception
        assertEquals(401, error.statusCode)
        assertEquals(ErrorType.UNAUTHORIZED, error.type)
        coVerify(exactly = 0) { tokenStore.save(any()) }
    }

    @Test
    fun `login does not swallow cancellation`() = runTest {
        coEvery { api.login(any()) } throws CancellationException("cancelled")
        assertFailsWith<CancellationException> { repository.login("a@b.com", "pw") }
    }

    @Test
    fun `logout clears the session even when the network call fails`() = runTest {
        coEvery { api.logout() } throws IOException("offline")

        repository.logout()

        coVerify(exactly = 1) { tokenStore.clearSession() }
    }
}
```

The cancellation test looks pedantic and is not. It is the regression test for the most damaging
mistake a catch-all guard can make, and it costs three lines.

### 7.3 ViewModel test: Turbine on the state flow

The ViewModel owns one thing, the `Idle → Loading → result` transition. Assert the sequence and
nothing else.

```kotlin
class LoginViewModelTest {

    @get:Rule val mainDispatcherRule = MainDispatcherRule()

    private val repository = mockk<AuthRepository>()

    @Test
    fun `login emits Idle then Loading then Success`() = runTest {
        coEvery { repository.login(any(), any()) } coAnswers {
            delay(50)
            DataState.Success(authToken)
        }
        val viewModel = LoginViewModel(repository)

        viewModel.state.test {
            assertEquals(DataState.Idle, awaitItem())
            viewModel.login("a@b.com", "pw")
            assertEquals(DataState.Loading, awaitItem())
            assertIs<DataState.Success<AuthToken>>(awaitItem())
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `a second login while loading is ignored`() = runTest {
        coEvery { repository.login(any(), any()) } coAnswers {
            delay(100)
            DataState.Success(authToken)
        }
        val viewModel = LoginViewModel(repository)

        viewModel.login("a@b.com", "pw")
        viewModel.login("a@b.com", "pw")
        advanceUntilIdle()

        coVerify(exactly = 1) { repository.login(any(), any()) }
    }

    @Test
    fun `an error emits a ShowError event`() = runTest {
        coEvery { repository.login(any(), any()) } returns
            DataState.Error(AppException(statusCode = 401, type = ErrorType.UNAUTHORIZED))
        val viewModel = LoginViewModel(repository)

        viewModel.events.test {
            viewModel.login("a@b.com", "wrong")
            assertIs<LoginEvent.ShowError>(awaitItem())
        }
    }
}
```

`delay(50)` inside `runTest` is virtual time. The test does not actually wait.

### 7.4 Authenticator test: concurrent 401s, exactly one refresh

This is the test that justifies the networking design. Ten requests 401 at once; the mutex must
collapse that into a single refresh and ten replays.

```kotlin
class TokenAuthenticatorTest {

    private val server = MockWebServer()

    @Test
    fun `ten concurrent 401s trigger exactly one refresh and all replay`() = runTest {
        val refreshCount = AtomicInteger(0)

        server.dispatcher = object : Dispatcher() {
            override fun dispatch(request: RecordedRequest): MockResponse =
                if (request.getHeader("Authorization") == "Bearer new") {
                    MockResponse().setResponseCode(200).setBody("{}")
                } else {
                    MockResponse().setResponseCode(401)
                }
        }
        server.start()

        val tokenStore = FakeTokenStore(access = "old", refresh = "r")
        val authenticator = TokenAuthenticator(
            tokenStore = tokenStore,
            refreshApi = Provider {
                AuthRefreshApi {
                    refreshCount.incrementAndGet()
                    delay(50)                       // widen the race window
                    RefreshResponseDto(accessToken = "new", refreshToken = "r2")
                }
            },
            sessionEvents = SessionEventBus(),
        )

        val client = OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor(tokenStore))
            .authenticator(authenticator)
            .build()

        val responses = (1..10).map {
            async(Dispatchers.IO) {
                client.newCall(Request.Builder().url(server.url("/x")).build()).execute()
            }
        }.awaitAll()

        assertEquals(1, refreshCount.get())
        assertTrue(responses.all { it.code == 200 })
    }

    @Test
    fun `a 401 on the retry gives up instead of looping`() = runTest {
        server.dispatcher = object : Dispatcher() {
            override fun dispatch(request: RecordedRequest) = MockResponse().setResponseCode(401)
        }
        server.start()
        // ... same wiring

        assertEquals(401, response.code)
        assertEquals(1, refreshCount.get())      // one refresh, then retryCount stops it
    }

    @Test
    fun `a failed refresh clears the session and emits expiry`() = runTest {
        // refresh throws; assert tokenStore.clearSession() ran and sessionEvents emitted once
    }
}
```

### 7.5 Screen test

The route/screen split means the screen takes plain arguments, so a UI test needs no Hilt, no
navigation, and no ViewModel.

```kotlin
class LoginScreenTest {
    @get:Rule val composeRule = createComposeRule()

    @Test
    fun `the submit button is disabled while loading`() {
        composeRule.setContent {
            AppTheme { LoginScreen(DataState.Loading, SnackbarHostState(), { _, _ -> }) }
        }
        composeRule.onNodeWithTag("login_submit").assertIsNotEnabled()
    }
}
```

Use `testTag` or resolved string resources for anything the copy team might change. A UI test
that breaks on a wording tweak is a tax, not a safety net.

### 7.6 What each test pins down

The repository maps outcomes and cleans up. The ViewModel drives the state machine and nothing
else. The Authenticator collapses a refresh storm into one call and refuses to loop. The screen
renders every state branch. If a change breaks one of those invariants, exactly one test goes
red, and its name reads as a bug report.

---

## 8. Tooling and Enforcement

Conventions are only real if a machine fails on them.

### 8.1 Boundaries

In a **multi-module** project the compiler does most of the work:

- **`:feature:auth` does not depend on `:feature:home`.** Not "should not." Does not. The import will not resolve, because the dependency is not declared in `build.gradle.kts`. A cross-feature import is a build failure found in seconds, before review.
- **`internal` is module-scoped in Kotlin.** Everything in a feature's `data/` package is `internal`, so a DTO, a Retrofit interface, or a repository implementation is physically invisible outside its module.
- **A pure-Kotlin `:core:model` cannot import Android.** Apply `kotlin("jvm")` instead of the Android library plugin and the framework is simply not on the classpath. Domain purity becomes a build setting.

What is left is the intra-module rule (`ui` must not touch `data` inside one feature). In a
**single-module** project, all of it is left. Either way Konsist covers it as an ordinary unit
test:

```kotlin
class ArchitectureTest {

    @Test
    fun `ui does not depend on data`() {
        Konsist.scopeFromProject()
            .files
            .withPackage("..feature..ui..")
            .assertFalse { file -> file.imports.any { it.name.contains(".data.") } }
    }

    @Test
    fun `domain has no android or retrofit imports`() {
        Konsist.scopeFromProject()
            .files
            .withPackage("..domain..")
            .assertFalse { file ->
                file.imports.any {
                    it.name.startsWith("android.") ||
                        it.name.startsWith("androidx.") ||
                        it.name.startsWith("retrofit2.")
                }
            }
    }

    @Test
    fun `no feature imports another feature`() {
        Konsist.scopeFromProject()
            .files
            .withPackage("..feature..")
            .assertTrue { file ->
                val own = file.packagee?.name?.substringAfter("feature.")?.substringBefore(".")
                file.imports.none { it.name.contains(".feature.") && !it.name.contains(".feature.$own.") }
            }
    }

    @Test
    fun `no ViewModel exposes a MutableStateFlow`() {
        Konsist.scopeFromProject()
            .classes()
            .withAllParentsOf(ViewModel::class)
            .properties()
            .withPublicOrDefaultModifier()
            .assertFalse { it.hasType { type -> type.name.contains("MutableStateFlow") } }
    }
}
```

These run in `./gradlew test` alongside everything else. No plugin, no IDE integration required.

### 8.2 Static analysis

- **ktlint** (via Spotless or the ktlint Gradle plugin) for formatting. Formatting is never a review topic; `./gradlew spotlessCheck` decides.
- **detekt** for complexity, long parameter lists, and code smells. Add `detekt-compose` rules for Compose-specific issues: missing `Modifier` parameter, `Modifier` not the first optional parameter, mutable state not remembered, composable returning a value.
- **Android Lint** with the checks that matter promoted to errors:

```kotlin
android {
    lint {
        warningsAsErrors = true
        abortOnError = true
        disable += setOf("GradleDependency")   // noisy, not actionable in CI
    }
}
```

### 8.3 Compose-specific checks

- **Strong skipping mode** (default with the Kotlin 2.0+ Compose compiler) handles most stability issues, but unstable parameters still cost recompositions. Enable compiler metrics periodically and read the report:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    metricsDestination = layout.buildDirectory.dir("compose_compiler")
}
```

- Annotate models used as composable parameters with `@Immutable` when they hold only `val`s of stable types. A `List<T>` parameter is unstable; use `ImmutableList` from `kotlinx.collections.immutable`, or accept the recomposition knowingly.
- Every public composable takes `modifier: Modifier = Modifier` as its first optional parameter, and applies it to its outermost layout node.

### 8.4 Convention plugins

Past roughly four modules, copy-pasted `build.gradle.kts` blocks become the main source of drift.
Move shared config into `build-logic` convention plugins:

```kotlin
// build-logic/convention/src/main/kotlin/AndroidFeatureConventionPlugin.kt
class AndroidFeatureConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply("myapp.android.library")
        pluginManager.apply("myapp.android.hilt")
        dependencies {
            add("implementation", project(":core:designsystem"))
            add("implementation", project(":core:common"))
            add("testImplementation", project(":core:testing"))
        }
    }
}
```

A new feature module's build file is then three lines, which is what makes the module-per-feature
rule cheap enough to actually follow. Skip this entirely while the project is single-module.

---

## 9. Naming and File Conventions

**Files:** `PascalCase.kt`, matching the primary declaration (`AuthRepositoryImpl.kt`). A file
holding several small related declarations (extensions, a set of previews) takes a descriptive
plural name (`DataStateExtensions.kt`).

**Packages:** all lowercase, no underscores. `com.example.app.feature.auth.data.remote`.

**Layer suffixes:**

| Thing | File | Declaration |
|---|---|---|
| Retrofit interface | `AuthApi.kt` | `internal interface AuthApi` |
| DTO | `LoginDto.kt` | `internal data class LoginRequestDto` / `LoginResponseDto` |
| Domain model | `AuthToken.kt` | `data class AuthToken` (no suffix) |
| Repository contract | `AuthRepository.kt` | `interface AuthRepository` |
| Repository impl | `AuthRepositoryImpl.kt` | `internal class AuthRepositoryImpl` |
| ViewModel | `LoginViewModel.kt` | `@HiltViewModel class LoginViewModel` |
| UI state | `LoginUiState.kt` | `data class LoginUiState` |
| Stateful route | `LoginRoute.kt` | `@Composable fun LoginRoute` |
| Stateless screen | `LoginScreen.kt` | `@Composable fun LoginScreen` |
| Hilt module | `AuthDataModule.kt` | `internal abstract class AuthDataModule` |

**Composables:** `PascalCase`, noun-phrased, returning `Unit`. `LoginForm`, not `buildLoginForm`.
Parameter order: required data, then `modifier: Modifier = Modifier`, then other optionals, then
a trailing lambda for content.

**Constants:** `SCREAMING_SNAKE_CASE` in a `companion object`, or a `private const val` at file
level.

**Flows:** the private backing field is `_state`, the public one is `state`. No `Flow` or
`StateFlow` suffix in the name.

**Booleans:** `is` / `has` / `should` prefix. `isLoading`, `hasUnsavedChanges`.

**Tests:** backtick-quoted sentences describing behavior, not method names.
``fun `login maps a 401 to an UNAUTHORIZED AppException`()``. A failing test name should read as
a bug report.

**Style:**

- `val` over `var`. A `var` in a class body needs a reason.
- Expression bodies for one-liners.
- Exhaustive `when` over `if/else` chains on sealed types. Never add an `else` branch to a `when` over a sealed type, or you lose the compile error when a branch is added later.
- Named arguments whenever a call has more than two parameters, or any boolean.
- Trailing commas everywhere: stable diffs, and the formatter expects them.
- Comments explain *why*, never *what*. KDoc on public API only.
- Scope functions used for their meaning: `let` for nullable transforms, `apply` for configuration, `also` for side effects, `run` for scoped computation. Not interchangeably, and never nested more than one deep.

---

## 10. New Feature Checklist

Adding a networked feature `<feature>`:

1. **Location**: a new package under `feature/`, or a new Gradle module if the project has passed a §2.2 trigger.
2. **Domain models** (`domain/model/`): plain `data class`, immutable, no Android imports, no `@Serializable`.
3. **Repository contract** (`domain/repository/<Feature>Repository.kt`): `suspend` returning `DataState<T>` where the UI renders loading/error, plain `suspend` for fire-and-forget, `Flow<T>` for streams.
4. **DTOs** (`data/remote/`): `internal @Serializable data class` with a `toDomain()`. Nullable where the server is nullable.
5. **API** (`data/remote/<Feature>Api.kt`): `internal interface`, `suspend` functions, relative paths with no leading slash.
6. **Repository impl** (`data/repository/`): `internal class`, constructor injection, `withContext(io)`, `safeCall { }`.
7. **DI** (`data/di/<Feature>DataModule.kt`): `internal abstract class` with `@Binds` for the interface and a `companion object` `@Provides` for the Retrofit API.
8. **ViewModel** (`ui/<screen>/`): `@HiltViewModel`, private `MutableStateFlow`, public `StateFlow`, `Channel` for events. Shape 1 by default (§6.2); `stateIn` for observed streams; a composite `UiState` once the screen has more than one independent slice.
9. **Route** (`ui/<screen>/<Screen>Route.kt`): owns the ViewModel via `hiltViewModel()`, collects with `collectAsStateWithLifecycle()`, handles effects via `ObserveAsEvents`, takes navigation as lambdas.
10. **Screen** (`ui/<screen>/<Screen>Screen.kt`): stateless, takes state and callbacks, holds only UI-local `rememberSaveable` state. At least one `@Preview`.
11. **Components** (`ui/<screen>/components/`): every non-trivial composable, with `modifier` as the first optional parameter.
12. **Strings**: add to `res/values/strings.xml`. No inline literals, no string concatenation for sentences.
13. **Navigation** (`navigation/<Feature>Navigation.kt`): `@Serializable` route types and a `NavGraphBuilder.<feature>Graph()` extension taking navigation lambdas. Wire it in the app module.
14. **Tests**: a repository test covering success, error, and cancellation; a ViewModel test with Turbine covering the state sequence. Anything with retry behavior gets a MockWebServer test.
15. **Verify**: `./gradlew test spotlessCheck detekt lint` passes clean, architecture tests included.

**Defaults, and when they bend.** Everything in this checklist is a default. Skip a step when you
can say what you are buying: a feature with no network needs no DTOs or repository; a screen with
no state needs no ViewModel; a throwaway internal tool may not need previews for every state.

**Invariants, which do not bend.** DTOs stay `internal` and stay in `data`. Domain models stay
pure Kotlin. The UI layer depends only on the domain contract. Errors travel as `AppException`
inside `DataState` and become strings only at the UI edge. ViewModels never see a `Context` and
never return a composable. Every catch-all rethrows `CancellationException`.

---

## 11. Production and Release Concerns

Everything above is application architecture. This section is the operational layer, plus an
explicit statement of what this playbook leaves out, so an omission reads as a decision.

### 11.1 Signing and release builds

Signing config comes from a git-ignored properties file locally and from CI secrets in the
pipeline. A keystore never enters version control.

```kotlin
signingConfigs {
    create("release") {
        val props = Properties().apply {
            rootProject.file("keystore.properties")
                .takeIf { it.exists() }?.inputStream()?.use(::load)
        }
        storeFile = props.getProperty("storeFile")?.let(::file)
        storePassword = props.getProperty("storePassword")
        keyAlias = props.getProperty("keyAlias")
        keyPassword = props.getProperty("keyPassword")
    }
}

buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        signingConfig = signingConfigs.getByName("release")
    }
}
```

Enroll in Play App Signing so a lost upload key is recoverable. Ship an App Bundle (`.aab`), not
an APK.

### 11.2 R8 and keep rules

R8 shrinks, obfuscates, and optimizes. It also breaks reflection-based code silently, and a
release-only crash is the worst kind.

`kotlinx.serialization` is compile-time, so it needs far fewer keep rules than reflective JSON
libraries, which is part of why it is the default here. Retrofit ships `consumer-rules.pro`, so
its interfaces are handled. What still needs attention:

- Any model reached only through reflection.
- Enum `valueOf` on a serialized name.
- `@Keep` on anything a native library or a WebView JS bridge resolves by name.

**Always run a release build against the real backend before shipping.** `./gradlew
installProdRelease` and click through the app. A debug build proves nothing about R8.

Upload the mapping file to the crash reporter (the Crashlytics Gradle plugin does it
automatically) so stack traces stay readable.

### 11.3 Baseline profiles

A baseline profile tells ART which code paths to AOT-compile at install time, typically cutting
cold start and first-scroll jank meaningfully with no code change. It is the highest-leverage
performance item available, and it needs no architectural change to adopt.

```kotlin
// :baselineprofile module
@Test
fun generate() = baselineProfileRule.collect(packageName = "com.example.app") {
    startActivityAndWait()
    device.findObject(By.res("orders_list")).fling(Direction.DOWN)
}
```

Add this once you have a startup path worth protecting, and regenerate it when that path changes
meaningfully. Pair it with a Macrobenchmark test asserting a cold-start budget, so a regression
fails CI rather than a review.

### 11.4 CI pipeline

```yaml
name: ci
on:
  pull_request:
  push: { branches: [main] }

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew spotlessCheck
      - run: ./gradlew detekt
      - run: ./gradlew lintProdRelease
      - run: ./gradlew testProdDebugUnitTest        # includes the architecture tests
      - run: ./gradlew assembleProdRelease          # prove R8 does not break the build

  instrumented:
    runs-on: ubuntu-latest
    steps:
      - uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          script: ./gradlew connectedProdDebugAndroidTest
```

Gradle's build cache and configuration cache keep this fast enough to run on every PR
(`org.gradle.caching=true`, `org.gradle.configuration-cache=true` in `gradle.properties`).
Release upload to Play runs on tags in a separate job, gated on the same checks.

Anything a machine can check (formatting, layering, lint, R8) is checked here, so review is spent
on design rather than drift.

### 11.5 Transport and runtime hardening

- **HTTPS only.** Enforced by the platform default plus the flavor-specific network security config in §3.1.
- **Certificate pinning** for high-value targets, via OkHttp's `CertificatePinner`. One place to change, because there is one `OkHttpClient`. Always pin a backup key and have a kill switch; a pin that outlives its certificate bricks the app in the field.
- **`FLAG_SECURE`** on screens showing sensitive data, to block screenshots and the recents-screen thumbnail.
- **No secrets in logs.** The logging interceptor is `BuildConfig.DEBUG`-gated and redacts the auth header. Strip remaining log calls from release with a ProGuard `assumenosideeffects` rule.
- **`android:allowBackup`** and the data-extraction rules reviewed deliberately, per §3.8.

### 11.6 Deliberately out of scope

Each of these is a "reach for it when the requirement is real" pointer, not an accidental gap.

| Concern | When you need it | Reach for |
|---|---|---|
| **Offline-first / local DB** | data must survive offline, or sync | Room behind the repository, a migration strategy, and `WorkManager` for sync |
| **Pagination** | long server-driven lists | Paging 3 + `collectAsLazyPagingItems()` |
| **Background work** | uploads, sync, anything that must outlive the process | `WorkManager`, never a bare `CoroutineScope` |
| **Push notifications** | re-engagement | FCM, a notification channel strategy, and the `POST_NOTIFICATIONS` runtime permission (API 33+) |
| **Deep links** | URL entry points | `navDeepLink` on the Compose destinations; App Links verification for `https://` |
| **Analytics / feature flags** | measurement, gradual rollout | a facade interface in `core/common`, implementations injected like anything else |
| **Accessibility** | inclusive UI and store compliance | `contentDescription`, `Modifier.semantics`, touch targets >= 48.dp, a TalkBack audit per screen |
| **Adaptive layouts** | tablets, foldables, desktop | `WindowSizeClass` + `NavigationSuiteScaffold`. Required for a Play tablet listing. |
| **Localization** | more than one locale | Already free via `strings.xml`. Add per-app language support (API 33+) and RTL layout testing. |
| **Use cases** | logic shared across several ViewModels, or complex enough to test on its own | A `<Verb><Noun>UseCase` class in `domain/usecase/`, injected into the ViewModel |
| **Multiplatform** | a shared iOS target | The domain layer is already portable at zero cost. The data layer is not: sharing it means Ktor instead of Retrofit and Koin or manual DI instead of Hilt, which is an up-front stack decision rather than a later refactor. |

Adopting any of these bolts onto the existing seams: a repository behind a domain contract, a
composable in `ui/`, a new module. Production concerns are additions, never rewrites. That is the
point of the architecture.
