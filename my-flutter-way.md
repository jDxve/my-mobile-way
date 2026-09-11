<h1 align="center">My Flutter Way</h1>
<p align="center"><em><strong>Flutter Production Playbook</strong></em></p>

<p align="center">
  Feature-first over the official Flutter architecture &middot; Typed networking &middot;
  Riverpod notifiers &middot; <code>Result&lt;T&gt;</code> in data, <code>DataState&lt;T&gt;</code> in UI &middot;
  Testing &middot; CI-enforced boundaries
  <br />
  <em>My conventions for building production-grade Flutter apps.</em>
</p>

<p align="center">
  <img alt="Flutter" src="https://img.shields.io/badge/Flutter-02569B?logo=flutter&logoColor=white" />
  <img alt="Dart 3" src="https://img.shields.io/badge/Dart-3.x-0175C2?logo=dart&logoColor=white" />
  <img alt="Riverpod" src="https://img.shields.io/badge/State-Riverpod-4B32C3" />
  <img alt="Architecture" src="https://img.shields.io/badge/Architecture-Feature--first%20%2B%20Layered-2E7D32" />
  <img alt="Lints" src="https://img.shields.io/badge/Lints-Enforced%20in%20CI-EF6C00" />
</p>

<p align="center">
  <b>Author</b> : <b>JOHN DAVE BAÑAS</b>
</p>

---

This is how I structure and write Flutter apps: not just the architecture, but the
conventions, error handling, tests, and lints that keep it honest. Follow it when scaffolding
a new app, adding a feature, or reviewing code.

It is deliberately built **on top of the official Flutter architecture guidance**, not as an
alternative to it. Section 0 maps every piece of this document onto the Flutter team's
vocabulary and states the two places I diverge, with reasons. Everything else is an
implementation of that guidance with a specific stack.

> **Status of the code samples.** Snippets are minimal extracts from the reference
> implementation, with imports and unrelated members omitted. A `/* ... */` or a trailing
> comment marks an elision; those blocks show the shape, not a file you can paste and compile.
> Configuration blocks (YAML, Gradle, manifest) are the exception and are given in full,
> because half a config is worse than none. Anything illustrative rather than real is labelled
> **conceptual** inline.

## Contents

| | |
| --- | --- |
| [0. Relationship to the Official Flutter Architecture](#0-relationship-to-the-official-flutter-architecture) | The mapping onto the official guidance, and the two deviations |
| [1. Architecture Philosophy](#1-architecture-philosophy) | Why feature-first, and what the boundaries buy |
| [2. Folder Structure](#2-folder-structure) | Where every file goes, at two scales |
| [3. Core Layer](#3-core-layer) | `Result` vs `DataState`, error handling, auth refresh |
| [4. Common Layer](#4-common-layer) | Shared resources, navigation guard, global error hooks |
| [5. Feature Anatomy](#5-feature-anatomy-worked-example-auth) | One vertical slice end to end |
| [6. State Management with Riverpod](#6-state-management-with-riverpod) | Provider graph, notifier conventions, lifecycle |
| [7. Testing](#7-testing) | Four seams, four kinds of test |
| [8. Tooling and Enforcement](#8-tooling-and-enforcement) | What a machine checks so review does not have to |
| [9. Naming and File Conventions](#9-naming-and-file-conventions) | File, class, and provider naming |
| [10. New Feature Checklist](#10-new-feature-checklist) | The order to build a slice in, and the rules that never bend |
| [11. Production and Release Concerns](#11-production-and-release-concerns) | Flavors, CI pipeline, release hardening |

**Reading order.** New to the document: §0, §1, §5. That is the architecture and one worked
feature. Implementing it: §3 (core), §10 (checklist), §8 (lints). Reviewing someone's PR
against it: §10's "rules that never bend" is the checklist.

---

## Stack

- **State / notifiers**: `flutter_riverpod` (v3). `Notifier` / `AsyncNotifier` + `NotifierProvider`. No BLoC, no GetX.
- **Networking**: `dio` + `retrofit`. Retrofit for typed calls; raw Dio for nullable, 204, or dynamic-shape endpoints.
- **Data-layer result**: hand-written sealed `Result<T>` (`Ok` / `Err`). Two cases, because a completed call has exactly two outcomes.
- **UI state**: `freezed` sealed union `DataState<T>` (idle / loading / success / error). Owned by notifiers only. Freezed is used **only** for `DataState` and error envelopes; DTOs, domain models, and composite notifier state are hand-written plain Dart.
- **Storage**: `flutter_secure_storage` for tokens and session, `shared_preferences` for preferences and cache.
- **Sizing**: `flutter_screenutil` behind a `Dimensions` facade. A default, not an invariant (see §4).
- **Theming**: `ThemeExtension<AppColors>` plus a `BuildContext` extension for ergonomic color access.
- **Testing**: `flutter_test` + `mocktail` (no codegen, matching the hand-written ethos). DI is exercised through `ProviderContainer` with providers overridden. Never a real Dio, real keystore, or real socket.

---

## 0. Relationship to the Official Flutter Architecture

The Flutter team publishes an architecture guide that splits an app into a **UI layer** and a
**Data layer**, with four components. The dependency direction is:

```
View  ->  Notifier  ->  Repository  ->  Service
```

This playbook implements that guide. The mapping is one-to-one:

| Official guidance | This playbook | Notes |
| --- | --- | --- |
| **View** | `presentation/screens/` + `presentation/components/` | Thin. Widgets, layout, gesture handlers. |
| **Notifier** | **`Notifier`** / `AsyncNotifier` in `presentation/notifier/` | The guide's UI-state component. I call it a **Notifier**: it is the Riverpod class name and the Flutter convention. |
| **Command** | Public intent methods on the notifier | `login()`, `removeMember()`. See §5. |
| **Repository** | `data/repository/<Feature>RepositoryImpl` | Single source of truth. Owns caching and retry. |
| **Service** | `data/services/<Feature>ApiService` (Retrofit) | Wraps one endpoint group. Stateless. |
| **Domain (optional)** | `domain/models/` + `domain/repository/` | Models and the repository contract. No use-case classes until one is shared. |
| **`Result<T>`** | `core/result/result.dart` | `Ok` / `Err`. Returned by every repository. |

The guide describes that component as holding the state the view needs so it can rebuild
without losing data, and exposing callbacks (commands) that the view attaches to event handlers.
A Riverpod `Notifier` does exactly this. The guide's own samples use `ChangeNotifier`; the guide
itself is state-management agnostic.

### The two places I deviate, and why

**1. Feature-first folders instead of layer-first.** The official sample app groups by layer at
the top level (`lib/ui/`, `lib/data/`, `lib/domain/`). I group by feature, with the layers
inside each feature. The reason is deletion and ownership: a feature is the unit of work, and
removing one should be removing one directory, not touching four. This is a folder decision,
not an architecture decision. Every dependency arrow in the table above still points the same
direction. Past roughly six features, layer-first top-level folders start costing more than
they save.

**2. Riverpod `Notifier` instead of `ChangeNotifier`.** Riverpod covers both the UI-state role
and dependency injection with one tool, which removes a second DI mechanism from the stack. `ChangeNotifier` plus `provider` would satisfy the guide equally well; this is
a stack preference, and the boundaries it enforces are stricter, not looser.

### One thing I explicitly adopt from the guide

The guide separates **the result of a data operation** from **the state of a screen**. A
repository returns success or failure. "Loading" and "idle" are things a *Notifier* knows, not
things a finished network call can be. One four-case `DataState<T>` used everywhere means
every repository signature advertises two outcomes it can never produce, so the two are split:
`Result<T>` below the repository boundary, `DataState<T>` above it. See §3.

---

## 1. Architecture Philosophy

**Mental model:** feature-first on the outside, layered on the inside.
The app is a set of self-contained features. Each feature owns its full vertical slice
(`data`, `domain`, `presentation`). Cross-cutting infrastructure lives in `core`;
reusable UI and helpers live in `common`.

**Why feature-first:** a feature is the unit of work. Everything needed to build, test, or
delete a feature sits in one folder. You never hunt across top-level `models/`, `services/`,
`screens/` buckets. Deleting a feature is deleting one directory.

**Why layering inside a feature:** the UI must not know how data is fetched, and the data
layer must not know how data is displayed. The `domain` layer (plain Dart models plus an
abstract repository interface) is the contract between them. Presentation depends on the
domain interface, never on Dio, Retrofit, or DTOs.

**Dependency direction (always inward):**

```
presentation      ->  domain   <-  data
(View + Notifier)   (contract)   (Repository + Service)
```

- `presentation` depends on `domain` (models plus the abstract repository) and on `core` / `common`.
- `data` depends on `domain` (it implements the interface and maps DTOs to domain models).
- `domain` depends only on `core/result/` (`Result`, `AppException`). No Flutter, no Dio, no HTTP types in any domain signature.
- Nothing in a feature imports another feature's `data` or `presentation` internals. Shared cross-feature contracts go through `domain` or `common`.

**On the word "pure".** The domain layer is **platform-independent and transport-independent**:
no Flutter, no Dio, no HTTP, no `dart:io`. It is not dependency-free, and I do not claim it is.
It depends on two project-level types, `Result` and `AppException`, which live under
`core/result/` precisely because they are not network types. If you need a genuinely
dependency-free domain, extract it to its own Dart package; for an app of this size that is
ceremony without a payoff.

**What goes where:**

- **`core/`**: app-wide infrastructure with no UI. Config, the Dio factory and interceptors, `Result`, `AppException`, the repository guard, theme state, storage keys, launch bootstrap. If two unrelated apps could share it verbatim, it is probably core.
- **`common/`**: shared presentation-side building blocks. Reusable widgets, resources (theme, strings, dimensions, assets), the navigation guard, UI utils and validators. Depends on `core`; never on a feature.
- **`features/<name>/`**: one product capability, full vertical slice. Depends on `core` and `common`, not on sibling features.

**Views are minimal.** A screen composes components and wires notifiers. It holds no
business logic and no inline widget tree of significance. The guide allows a view to own
purely visual logic: animation state, layout decisions based on screen size or orientation,
and simple routing. Everything touching data belongs in the notifier.

---

## 2. Folder Structure

### The standard tier (what this document describes)

```
lib/
  main.dart              # bootstrap: error hooks, splash, orientation, launch init, theme preload, runApp
  main_shell.dart        # bottom-nav shell (IndexedStack) + tab-state notifier
  router.dart            # onGenerateRoute, named routes, global navigatorKey, transitions

  core/                          # app-wide infrastructure, no UI
    config/
      app_config.dart            # baseUrl per env, timeouts, secure-url assert
    result/                      # pure Dart: imports nothing but dart:core
      result.dart                # Result<T>: Ok<T> | Err<T>  (data-layer outcome)
      app_exception.dart         # AppException: no Dio, no Flutter, no HTTP
    reporting/
      reporter.dart              # crash-reporter facade (see below)
    network/                     # everything that knows about Dio lives here
      dio_error_mapper.dart      # mapDioException(): the one DioException -> AppException site
      repository_guard.dart      # RepositoryGuard mixin: guard() wraps a call into Result
      dio_factory.dart           # dioProvider: BaseOptions + interceptor stack
      auth_interceptor.dart      # bearer inject + single-flight 401 refresh/retry
      logging_interceptor.dart   # pretty request/response logs (debug only, redacts auth)
      reporting_interceptor.dart # forwards surviving DioExceptions to the crash reporter
      secure_storage.dart        # secureStorageProvider + safeRead guard
      models/
        error_response.dart      # freezed error envelope
    state/
      data_state.dart            # DataState<T> freezed union (UI state, notifiers only)
    theme/
      theme_notifier.dart        # ThemeSettings + ThemeNotifier (persisted light/dark/system)
    utils/
      storage_keys.dart          # secure-storage key constants + session key list + cache keys
      launch_utils.dart          # first-install wipe, onboarding flags

  common/                        # shared presentation building blocks
    models/                      # tiny cross-feature value types (e.g. SimpleOption)
    navigation/
      auth_guard.dart            # AuthGuard widget: gate routes on auth state
    resources/
      app_theme.dart             # color primitives, AppColors ThemeExtension, AppTheme, context ext
      strings.dart               # all user-facing copy as static const
      dimensions.dart            # Dimensions.pxNN sizing facade
      assets.dart                # asset path constants
    utils/
      ui_utils.dart              # errorToMessage(), validators, formatters
      constants.dart
      animations/
    widgets/                     # reusable widgets (buttons, inputs, toast, shimmer, ...)

  features/
    <feature>/
      data/
        dto/                     # request/response DTOs: fromJson + toJson + toDomain()
        services/                # Retrofit @RestApi abstract class (+ .g.dart)
        source/                  # local source: TTL cache, DAO, prefs wrapper
        repository/              # <Feature>RepositoryImpl + repositoryProvider
      domain/
        models/                  # plain Dart domain models
        repository/              # abstract interface class <Feature>Repository
      presentation/
        notifier/                # Notifier<State> + NotifierProvider
        screens/                 # thin view screens (routeName lives here)
        components/              # feature-scoped widgets
```

Note `core/result/` and `core/state/`. Filing `Result` and `AppException` under
`core/network/` is what makes the domain layer *look* like it depends on infrastructure. They
are transport-neutral types, so they sit where that is obvious from the import line.

Small or leaf features (splash, onboarding) may collapse to `screens/` and `components/` only.
A feature with no networking may omit `data/` and `domain/` entirely.

### The small-app tier (start here below roughly four features)

Full vertical slices are overhead on a three-screen app. Start layer-first inside one package
and promote later. The layering rules are identical; only the folders differ.

```
lib/
  core/           # config, result, network, theme, utils
  common/         # widgets, resources
  data/
    dto/
    services/
    repository/
  domain/
    models/
    repository/
  presentation/
    notifier/
    screens/
    components/
```

**Promote to feature-first when any of these becomes true**, not before:

- more than about four or five product capabilities, so `presentation/screens/` stops being scannable;
- more than one developer working in parallel and colliding in the same folders;
- you find yourself prefixing filenames to disambiguate (`order_list_notifier.dart` vs `product_list_notifier.dart` in one directory);
- a change to one capability keeps forcing edits in four sibling directories.

Promotion is mechanical: create `features/<name>/`, move the matching files, fix imports. The
boundary lint in §8 starts paying for itself at exactly this point, because cross-feature
imports only become possible once features exist.

---

## 3. Core Layer

### Config (`core/config/app_config.dart`)

Values come from the build, with a dev fallback. `String.fromEnvironment` is `const`, so the
same compiled code runs in every environment and nothing sensitive is committed. A startup
assert fails fast if a non-dev build has an empty or cleartext base URL. Flavors and the
build invocation are in §11.1.

```dart
enum AppEnv { dev, staging, prod }

const String _devBaseUrl = 'http://192.168.1.100:4000/api/v1'; // local only

const String _envName = String.fromEnvironment('APP_ENV', defaultValue: 'dev');
const String baseUrl = String.fromEnvironment('BASE_URL', defaultValue: _devBaseUrl);

const AppEnv appEnv = _envName == 'prod'
    ? AppEnv.prod
    : _envName == 'staging'
        ? AppEnv.staging
        : AppEnv.dev;

const Duration kConnectTimeout = Duration(seconds: 10);
const Duration kSendTimeout = Duration(seconds: 30);
const Duration kReceiveTimeout = Duration(seconds: 30);

void assertSecureBaseUrl() {
  assert(
    baseUrl.isNotEmpty,
    'baseUrl is empty for appEnv=$appEnv. Pass BASE_URL via --dart-define-from-file. '
    'An empty base URL must fail at boot, not mid-request.',
  );
  assert(
    appEnv == AppEnv.dev || baseUrl.startsWith('https://'),
    'Non-dev build must use an https baseUrl. appEnv=$appEnv baseUrl="$baseUrl".',
  );
}
```

The non-empty check comes first on purpose: a missing `BASE_URL` define fails at boot with a
clear message instead of surfacing as a confusing connection error on the first request.

Asserts are stripped in release. They catch the mistake in dev, staging, and profile builds,
which is where a misconfigured define actually gets made. If you want the guard in release
too, promote it to a thrown `StateError` in `main()` before `runApp`.

### Crash reporter (`core/reporting/reporter.dart`)

`reporter` appears throughout this document. It is a thin facade over Crashlytics or Sentry, so
that swapping vendors touches one file and so tests can substitute a no-op.

```dart
abstract interface class Reporter {
  void recordError(Object error, StackTrace? stack, {bool fatal, List<Object> information});
  void recordFlutterError(FlutterErrorDetails details);
}

/// Set once in main(), before runApp. Defaults to a no-op so tests and the
/// dev loop never need a real vendor SDK.
Reporter reporter = const NoopReporter();
```

A top-level mutable rather than a provider on purpose: `FlutterError.onError` and
`PlatformDispatcher.onError` are installed before any `ProviderScope` exists, so the reporter
has to be reachable without a `Ref`. It is the only global in this document, and it is
write-once at boot.

### Result (`core/result/result.dart`)

The data-layer outcome type. A finished call either produced a value or it failed. Two cases,
no more. Repositories return `Future<Result<T>>`; nothing below the repository knows what a
loading spinner is.

```dart
sealed class Result<T> {
  const Result();
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Err<T> extends Result<T> {
  const Err(this.exception);
  final AppException exception;
}
```

Hand-written, not freezed: it is ten lines and Dart 3 sealed classes already give exhaustive
`switch`. `exception` is non-nullable, so no consumer ever writes `exception!`.

```dart
// Consumption is always an exhaustive switch. No casts, no null checks.
final message = switch (result) {
  Ok(:final value) => 'Loaded ${value.length} items',
  Err(:final exception) => errorToMessage(exception),
};
```

### AppException (`core/result/app_exception.dart`)

The error type carried by `Err`. **Genuinely plain Dart**: this file imports nothing but
`dart:core`. It does not know Dio exists.

```dart
enum ErrorType { timeout, connection, server, unauthorized, unknown }

class AppException implements Exception {
  const AppException({
    this.statusCode,
    this.message,
    this.type = ErrorType.unknown,
  });

  final int? statusCode;
  final String? message;
  final ErrorType type;

  /// Non-transport failures (JSON shape changed, mapping threw, cache corrupt).
  factory AppException.unexpected(Object error) =>
      AppException(message: error.toString(), type: ErrorType.unknown);

  @override
  String toString() => 'AppException($type, status: $statusCode, message: $message)';
}
```

A `AppException.fromDioException` factory on this class would contradict the
transport-independence claim, because a class with a `DioException` parameter imports Dio.
**The mapping lives on the transport side instead**, which is where the knowledge belongs and
what makes the claim honest.

### Dio error mapper (`core/network/dio_error_mapper.dart`)

The one place a Dio type becomes an `AppException`. In `core/network/`, not `core/result/`,
because knowing about Dio is a network-layer job. Swap the HTTP client and you replace this
file and nothing else.

```dart
AppException mapDioException(DioException e) {
  final status = e.response?.statusCode;
  return AppException(
    statusCode: status,
    message: _extractServerMessage(e.response?.data),
    type: switch (e.type) {
      DioExceptionType.connectionTimeout ||
      DioExceptionType.sendTimeout ||
      DioExceptionType.receiveTimeout =>
        ErrorType.timeout,
      DioExceptionType.connectionError => ErrorType.connection,
      _ => switch (status) {
          401 || 403 => ErrorType.unauthorized,
          != null when status! >= 500 => ErrorType.server,
          _ => ErrorType.unknown,
        },
    },
  );
}

/// Pulls the server's own message out of a JSON error envelope, if there is one.
String? _extractServerMessage(Object? data) => switch (data) {
      {'message': final String m} => m,
      {'error': {'message': final String m}} => m,
      _ => null,
    };
```

Parsing the response body belongs here too: it runs once, during the conversion, not at the UI
edge.

### Repository guard (`core/network/repository_guard.dart`)

A mixin that collapses the repetitive try/catch in every repository method. It catches
`DioException` **and** everything else, because a DTO whose JSON shape changed throws a
`TypeError`, not a `DioException`, and an uncaught one of those escapes into the notifier as
a raw crash.

```dart
mixin RepositoryGuard {
  Future<Result<T>> guard<T>(
    Future<T> Function() request, {
    Result<T> Function(DioException error)? onDioError,
  }) async {
    try {
      return Ok(await request());
    } on DioException catch (e) {
      return onDioError?.call(e) ?? mapError<T>(e);
    } catch (e, stack) {
      // Mapping/parsing failures are bugs, not user errors: report, then degrade.
      reporter.recordError(e, stack, fatal: false);
      return Err(AppException.unexpected(e));
    }
  }

  Result<T> mapError<T>(DioException error) => Err(mapDioException(error));
}
```

The guard sits in `core/network/` for the same reason the mapper does: it catches
`DioException`, so it is transport-aware by definition. `core/result/` stays free of Dio, which
is what lets the domain layer depend on it.

`onDioError` handles status codes that are not really failures, such as a 404 meaning "no data
yet":

```dart
@override
Future<Result<Household?>> fetchMyHousehold() => guard(
      () async => (await _apiService.getMyHousehold())?.toDomain(),
      onDioError: (e) =>
          e.response?.statusCode == 404 ? const Ok(null) : mapError(e),
    );
```

### DataState (`core/state/data_state.dart`)

The **UI state** type. Owned by notifiers, never returned by a repository. Four cases,
because a screen genuinely has four: nothing requested yet, in flight, loaded, failed.

```dart
@freezed
sealed class DataState<T> with _$DataState<T> {
  const factory DataState.idle() = DataIdle<T>;
  const factory DataState.loading() = DataLoading<T>;
  const factory DataState.success({required T data}) = DataSuccess<T>;
  const factory DataState.error({required AppException exception}) = DataError<T>;
}

extension DataStateX<T> on DataState<T> {
  bool get isLoading => this is DataLoading<T>;
  T? get valueOrNull => this is DataSuccess<T> ? (this as DataSuccess<T>).data : null;

  /// The one conversion point between the data layer and the UI layer.
  static DataState<T> from<T>(Result<T> result) => switch (result) {
        Ok(:final value) => DataState.success(data: value),
        Err(:final exception) => DataState.error(exception: exception),
      };
}
```

`exception` is `required`, so `DataError` always carries one. Making it nullable forces a null
check at every call site for a state that can never lack an exception.

**The rule, stated precisely.** Repositories return `Result<T>`. Notifiers hold
`DataState<T>` (or a composite state class containing `DataState<T>` fields) and are the only
place `idle` and `loading` are produced. Local reads, fire-and-forget operations, and anything
whose failure the UI does not surface may return a plain `Future<T>` or `Future<void>`. Not
every async call needs a result wrapper; every *networked operation whose outcome the user
sees* does.

### Secure storage (`core/network/secure_storage.dart`)

One provider for the single `FlutterSecureStorage` instance, following the same rule that
governs `dioProvider`: one provider next to the thing it exposes. Nothing constructs
`FlutterSecureStorage()` inline, which keeps it overridable with a mock in tests instead of
reaching for a real keystore.

```dart
final secureStorageProvider = Provider<FlutterSecureStorage>((ref) {
  return const FlutterSecureStorage(
    // Android: plugin defaults. AES-GCM with RSA-OAEP key wrapping, Keystore-backed.
    // Do NOT pass encryptedSharedPreferences: the Jetpack Security backend it selected
    // was deprecated by Google, the parameter was removed in v11, and existing data is
    // migrated automatically (migrateOnAlgorithmChange defaults to true).
    aOptions: AndroidOptions(migrateWithBackup: true),
    // iOS: readable only after first unlock, never synced to iCloud or restored to a new device.
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );
});
```

> **Version note.** `flutter_secure_storage` changed its Android backend in v10 and removed
> `encryptedSharedPreferences` in v11. Pin the major version in `pubspec.yaml` and re-check
> this block when you bump it. `migrateWithBackup: true` makes the v9-to-v10 cipher migration
> crash-resistant and can be dropped once your install base has migrated.

**Android: disable auto-backup for the secure store.** Android's Google Drive backup will
restore encrypted blobs onto a device whose Keystore cannot unwrap them, producing
`InvalidKeyException: Failed to unwrap key`. Exclude the plugin's preferences file, or turn
auto-backup off:

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:allowBackup="false"
    android:fullBackupContent="false"
    android:dataExtractionRules="@xml/data_extraction_rules">
```

**Android: guard the cold-start read anyway.** An OS update, a device-to-device migration, or a
restore can still leave the store holding keys the current Keystore cannot decrypt. Every
`read` then throws, and because it happens during bootstrap the app cold-crashes on launch with
no user-recoverable path. The plugin's `resetOnError` now defaults to true and handles most of
this, but it does not cover a `MissingPluginException` or a failure in your own decode step,
and "the app cannot start" is not a failure mode to leave to a default.

```dart
Future<String?> safeRead(FlutterSecureStorage storage, String key) async {
  try {
    return await storage.read(key: key);
  } on PlatformException catch (e, stack) {
    reporter.recordError(e, stack, fatal: false);
    await storage.deleteAll(); // unreadable store -> reset, do not crash the launch
    return null;
  }
}
```

Route the initial token read (`getStoredToken`, which seeds `authStateProvider`) through this
guard so a corrupted store degrades to "logged out", never to a crash loop.

**iOS: the Keychain survives uninstall.** Keychain items are not deleted when the app is
removed, so a reinstall can silently resurrect a previous user's tokens. Pair this with the
first-install wipe in `launch_utils.dart`: on the first launch after install, detected via a
`shared_preferences` flag which *is* cleared on uninstall, call `deleteAll()` on secure storage
before seeding any session. `first_unlock` accessibility also keeps items off iCloud Keychain
backups so they cannot restore onto a different device.

### Storage keys (`core/utils/storage_keys.dart`)

All secure-storage keys as `const String`. A `kSessionStorageKeys` list defines exactly what
gets cleared on logout, so ending a session never blindly wipes all secure storage. Per-user
cache keys are namespaced by user id via helper functions.

```dart
const String kAccessToken = 'access_token';
const String kRefreshToken = 'refresh_token';
// ...
const List<String> kSessionStorageKeys = [kAccessToken, kRefreshToken /* ... */];

String cacheProfileKey(String userId) => 'cache_${userId}_profile';
```

### Dio factory (`core/network/dio_factory.dart`)

A `Provider<Dio>`. Builds `BaseOptions` from config, then stacks interceptors in order:
logging (debug only), auth, reporting. Reporting goes **last** so it only sees failures that
survived the 401 refresh and retry. The auth interceptor is fully injected (token reader,
refresh function, expiry handler), which keeps it testable and free of circular imports.

```dart
final sessionExpiredProvider = Provider<StreamController<void>>((ref) {
  final controller = StreamController<void>.broadcast();
  ref.onDispose(controller.close);
  return controller;
});

final navigatorKeyProvider =
    Provider<GlobalKey<NavigatorState>>((ref) => GlobalKey<NavigatorState>());

/// A bare Dio with no interceptors, used for the refresh call itself.
/// Without this, a 401 from /auth/refresh re-enters AuthInterceptor and loops.
final refreshClientProvider = Provider<Dio>((ref) => Dio(BaseOptions(
      baseUrl: baseUrl,
      connectTimeout: kConnectTimeout,
      receiveTimeout: kReceiveTimeout,
    )));

final dioProvider = Provider<Dio>((ref) {
  final storage = ref.watch(secureStorageProvider);
  final sessionExpired = ref.watch(sessionExpiredProvider);
  final navKey = ref.watch(navigatorKeyProvider);
  final refreshClient = ref.watch(refreshClientProvider);

  final dio = Dio(BaseOptions(
    baseUrl: baseUrl,
    headers: {'Accept': 'application/json'},
    connectTimeout: kConnectTimeout,
    sendTimeout: kSendTimeout,
    receiveTimeout: kReceiveTimeout,
  ));

  dio.interceptors.addAll([
    if (kDebugMode) LoggerInterceptor(),
    AuthInterceptor(
      retryClient: Dio(BaseOptions(
        connectTimeout: kConnectTimeout,
        sendTimeout: kSendTimeout,
        receiveTimeout: kReceiveTimeout,
      )),
      readToken: () => safeRead(storage, kAccessToken),
      doRefresh: () async {
        final refreshToken = await safeRead(storage, kRefreshToken);
        if (refreshToken == null) return null;
        try {
          final res = await refreshClient.post<Map<String, dynamic>>(
            '/auth/refresh',
            data: {'refreshToken': refreshToken},
          );
          final access = res.data?['accessToken'] as String?;
          final rotated = res.data?['refreshToken'] as String?;
          if (access == null) return null;
          await storage.write(key: kAccessToken, value: access);
          if (rotated != null) {
            await storage.write(key: kRefreshToken, value: rotated);
          }
          return access;
        } on DioException {
          return null; // refresh itself failed -> caller expires the session
        }
      },
      onExpired: () async {
        await Future.wait(kSessionStorageKeys.map((k) => storage.delete(key: k)));
        sessionExpired.add(null);
        navKey.currentState?.pushNamedAndRemoveUntil('/login', (_) => false);
      },
    ),
    ReportingInterceptor(),
  ]);

  return dio;
});
```

`onExpired` awaits its deletes. Fire-and-forget deletes race the navigation, and a token that
outlives the redirect is the kind of bug that only shows up on a slow device.

### Auth interceptor (`core/network/auth_interceptor.dart`)

Attaches the bearer on every request. On 401: refresh **once per token generation**, replay the
original request with a dedicated retry client, and expire the session only if the retry itself
returns 401. Network errors after a good refresh must not log the user out.

> **The invariant is "exactly one refresh per token generation", not "QueuedInterceptor
> serializes".** Those are different guarantees, and this is the single most common mistake in
> Flutter auth code. `QueuedInterceptor` does serialize `onError` callbacks: the second 401 is
> not processed until the first finishes. It does **not** prevent the second one from
> refreshing again, because that request was already dispatched with the stale token and the
> interceptor has no memory of what just happened. Serialization alone gives you N sequential
> refreshes for N concurrent 401s, which with a rotating refresh token means the second refresh
> presents an already-consumed token and logs the user out.
>
> The fix is a **generation counter**. Each outgoing request is stamped with the token
> generation it was sent under. On 401, a request whose generation is behind the current one
> knows a sibling already refreshed, so it re-reads the token and replays instead of
> refreshing. Because `QueuedInterceptor` serializes `onError`, the counter needs no lock.
>
> A counter is used rather than comparing token strings because a server that reissues an
> identical access token would defeat string comparison and trigger a second refresh.
>
> **The counter alone is not enough, and this is the trap.** It only advances on a *successful*
> refresh. When the refresh token is rejected, the generation never moves, so the second
> concurrent 401 sees the same generation it was sent under, calls `doRefresh()` again, fails
> again, and expires the session again. N concurrent 401s become N refresh attempts and N
> expirations, which is the exact storm the design is supposed to prevent. Two more pieces of
> state close it: a **failed-generation marker** so one failure per token era is enough, and an
> **expire-once latch** so the teardown runs at most once per session.

```dart
class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor({
    required this.readToken,
    required this.doRefresh,
    required this.onExpired,
    Dio? retryClient,
  }) : _retryClient = retryClient ?? Dio();

  final Future<String?> Function() readToken;
  final Future<String?> Function() doRefresh;
  final Future<void> Function() onExpired;
  final Dio _retryClient;

  /// Incremented once per successful refresh. Serialized by QueuedInterceptor,
  /// so none of this mutable state needs a lock.
  int _generation = 0;

  /// The generation whose refresh already failed. Prevents a second attempt.
  int? _failedGeneration;

  /// Latch: the session teardown runs at most once.
  bool _sessionExpired = false;

  static const _kGenKey = 'authGen';
  static const _kReplayed = 'authReplayed';

  @override
  void onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final token = await readToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    options.extra[_kGenKey] = _generation; // stamp the token era
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode != 401) return handler.next(err);

    // A replay that 401s again means the fresh token is genuinely rejected. Stop.
    if (err.requestOptions.extra[_kReplayed] == true) {
      await _expireOnce();
      return handler.next(err);
    }

    final sentGen = err.requestOptions.extra[_kGenKey] as int? ?? _generation;

    final String? token;
    if (sentGen < _generation) {
      // A sibling 401 already refreshed while this request was in flight.
      // Reuse its token: do not refresh again.
      token = await readToken();
    } else if (_failedGeneration == _generation) {
      // The refresh for this token era already failed. Do not retry it per request.
      await _expireOnce();
      return handler.next(err);
    } else {
      token = await doRefresh();
      if (token == null) {
        _failedGeneration = _generation; // one failure per era is enough
      } else {
        _generation++;
      }
    }

    if (token == null) {
      await _expireOnce();
      return handler.next(err);
    }

    try {
      handler.resolve(await _retryClient.fetch(_replayOptions(err, token)));
    } on DioException catch (retryErr) {
      if (retryErr.response?.statusCode == 401) await _expireOnce();
      handler.next(err);
    }
  }

  Future<void> _expireOnce() async {
    if (_sessionExpired) return;
    _sessionExpired = true;
    await onExpired();
  }

  /// Call after a successful login. Without this the interceptor stays latched
  /// in the expired state for the process lifetime and a second session can
  /// never be torn down.
  void onSessionStarted() {
    _sessionExpired = false;
    _failedGeneration = null;
    _generation++; // invalidate any request still in flight from the old session
  }

  RequestOptions _replayOptions(DioException err, String token) {
    final original = err.requestOptions;
    return original.copyWith(
      headers: {...original.headers, 'Authorization': 'Bearer $token'},
      extra: {...original.extra, _kReplayed: true},
    );
  }
}
```

**`onSessionStarted()` must be wired up.** The expire-once latch is per interceptor instance,
and `dioProvider` is keep-alive, so the instance outlives a logout. Call it from
`AuthStateNotifier.setAuthenticated`, via a provider that exposes the interceptor:

```dart
final authInterceptorProvider = Provider<AuthInterceptor>((ref) { /* built in dio_factory */ });

// in AuthStateNotifier
void setAuthenticated(AuthToken token) {
  ref.read(authInterceptorProvider).onSessionStarted();
  state = AsyncData(token);
}
```

Forgetting this is a silent bug: login, logout, log in again, and the second session's expiry
never fires.

**Two limitations to know about, because they bite in production.**

- **Streamed and multipart bodies cannot be replayed.** `RequestOptions.data` holding a
  `FormData` or a `Stream` is single-consumption; re-sending it throws or uploads an empty
  body. This is a **policy**, not a warning: the interceptor replays idempotent JSON bodies and
  refuses everything else, so a silent empty upload is impossible.

  ```dart
  bool _isReplayable(RequestOptions o) =>
      o.data is! FormData && o.data is! Stream;

  // in onError, before attempting the replay:
  if (!_isReplayable(err.requestOptions)) {
    // Token is refreshed and persisted; the next attempt will carry it.
    // The caller retries the upload rather than us replaying a consumed body.
    return handler.next(err);
  }
  ```

  For upload features, refresh proactively before the request (below) so the 401 path is never
  reached, or have the repository rebuild the `FormData` and call again on `ErrorType.unauthorized`.
- **Proactive refresh beats reactive refresh.** Refreshing on a 401 means every expiry costs
  the user one wasted round trip. If your token carries an `exp`, check it in `onRequest` and
  refresh when it is within, say, thirty seconds of expiring. The 401 path then becomes the
  fallback for clock skew and server-side revocation rather than the normal case. The
  generation counter is what makes adding this safe.

### Logging interceptor (`core/network/logging_interceptor.dart`)

Debug-only pretty logging. Always redacts `Authorization` headers and `/auth/` request and
response bodies. Registered only under `kDebugMode`, so it cannot leak a token into a release
logcat.

### Reporting interceptor (`core/network/reporting_interceptor.dart`)

Registered after `AuthInterceptor`, so it only sees failures that survived the refresh and
retry. Forwards the request path, status code, and a redacted breadcrumb to the crash reporter.
Never the token, never an `/auth/` body. Non-actionable cases are filtered out so the dashboard
stays signal:

```dart
class ReportingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    final status = err.response?.statusCode;
    final isOffline = err.type == DioExceptionType.connectionError;
    final isExpectedEmpty = status == 404;
    final isCancelled = err.type == DioExceptionType.cancel;

    if (!isOffline && !isExpectedEmpty && !isCancelled) {
      reporter.recordError(
        err,
        err.stackTrace,
        fatal: false,
        information: [
          'path: ${err.requestOptions.path}',
          'method: ${err.requestOptions.method}',
          'status: $status',
        ],
      );
    }
    handler.next(err);
  }
}
```

The rule: the user sees a toast, and we see the stack trace. Neither substitutes for the other,
and reporting happens in exactly one place rather than sprinkled through repositories.

### Theme notifier (`core/theme/theme_notifier.dart`)

An immutable `ThemeSettings` (followDevice plus isDark) with a `mode` getter, driven by a
`Notifier`. Preferences persist to `shared_preferences` and are loaded once before `runApp`.

```dart
final themeProvider =
    NotifierProvider<ThemeNotifier, ThemeSettings>(ThemeNotifier.new);

class ThemeNotifier extends Notifier<ThemeSettings> {
  @override
  ThemeSettings build() => ThemeSettings.defaults;

  Future<void> load() async { /* read prefs into state */ }

  Future<void> setDarkMode(bool value) async {
    state = state.copyWith(isDark: value);
    await _save();
  }
}
```

---

## 4. Common Layer

### Resources

**`app_theme.dart`**: raw `Color` primitives at the top, an `AppColors extends
ThemeExtension<AppColors>` holding semantic roles (surface, textPrimary, error), light and dark
instances, `AppTheme.light` / `AppTheme.dark`, and a `BuildContext` extension for ergonomic
access. Widgets read colors via `context.primaryColor`, never raw constants.

```dart
class AppColors extends ThemeExtension<AppColors> {
  const AppColors({required this.surface, required this.primary /* ... */});
  final Color surface;
  final Color primary;
  // copyWith + lerp over every field
}

extension ThemeColors on BuildContext {
  AppColors get _c => Theme.of(this).extension<AppColors>()!;
  bool get isDark => Theme.of(this).brightness == Brightness.dark;
  Color get primaryColor => _c.primary;
  Color get surfaceColor => _c.surface;
  Color get textPrimaryColor => _c.textPrimary;
  Color get errorColor => _c.error;
}
```

**`strings.dart`**: all user-facing copy as `static const String` on a `Strings` class. No
hardcoded strings in widgets. Retire this for ARB or `slang` the moment a second locale appears.

**`assets.dart`**: asset path constants.

**`dimensions.dart`**: the sizing facade.

```dart
abstract final class Dimensions {
  static double get px4 => 4.w;
  static double get px8 => 8.w;
  static double get px16 => 16.w;
  static double get sp14 => 14.sp;
  // ...
}
```

> **`flutter_screenutil` is a default, not an invariant.** The facade is the convention; the
> package behind it is a choice. Widgets call `Dimensions.px16` and never `.w` or `.sp`
> directly, which means swapping to plain constants is a one-file change:
>
> ```dart
> static double get px16 => 16; // no scaling, same call sites
> ```
>
> For many apps `const EdgeInsets.all(16)` is simply correct, and linear scaling by screen
> width makes a tablet look like a magnified phone. Use ScreenUtil when your designs are
> pixel-specified against one reference device and visual fidelity across phone sizes matters.
> Drop it when you are targeting tablets, foldables, or web, where breakpoint-based layout beats
> uniform scaling. The facade exists so that decision stays reversible.
>
> **Never scale fonts by disabling text scaling.** Pinning `MediaQuery` text scaling to 1.0 in
> `main.dart` overrides the user's accessibility setting and is the single most common
> accessibility failure in Flutter apps. If large text
> breaks a layout, fix the layout: `FittedBox`, `Flexible`, scrollable content. If you must
> bound it, clamp rather than disable:
>
> ```dart
> MediaQuery.withClampedTextScaling(
>   minScaleFactor: 1.0,
>   maxScaleFactor: 1.4,
>   child: child!,
> )
> ```

### Navigation guard (`common/navigation/auth_guard.dart`)

A `ConsumerWidget` wrapping protected routes. Watches the async auth state and renders the
child only when a token is present; otherwise redirects to login after the frame.

```dart
class AuthGuard extends ConsumerWidget {
  const AuthGuard({required this.child, super.key});
  final Widget child;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ref.watch(authStateProvider).when(
          loading: () => const Scaffold(body: SizedBox.shrink()),
          error: (_, __) => _redirectToLogin(context),
          data: (token) => token == null ? _redirectToLogin(context) : child,
        );
  }

  Widget _redirectToLogin(BuildContext context) {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      Navigator.of(context)
          .pushNamedAndRemoveUntil(LoginScreen.routeName, (_) => false);
    });
    return const Scaffold(body: SizedBox.shrink());
  }
}
```

Wired in the router:
`case MainShell.routeName: return _buildFadeRoute(const AuthGuard(child: MainShell()));`

### UI utils (`common/utils/ui_utils.dart`)

Pure functions. `errorToMessage(Object?)` is the single place an `AppException` becomes copy,
plus validators and formatters. No Dio types reach this layer.

```dart
String errorToMessage(
  Object? error, {
  String fallback = 'Something went wrong. Please try again.',
}) {
  if (error is! AppException) return fallback;

  return switch (error.type) {
    ErrorType.timeout => 'Request timed out. Please try again.',
    ErrorType.connection => 'No internet connection. Please check your network.',
    ErrorType.unauthorized => 'Your session has expired. Please log in again.',
    ErrorType.server => 'Server error. Please try again later.',
    ErrorType.unknown => switch (error.statusCode) {
        400 => error.message ?? 'Invalid request. Please check your information.',
        _ => error.message ?? fallback,
      },
  };
}
```

Server messages are surfaced only for 400, where they are validation feedback written for the
user. For anything else the server's wording is for engineers, not customers.

### Global error hooks (`main.dart`)

The reporting interceptor covers network errors. Everything else, a bad `build`, a failed
layout, an un-awaited future that throws off the widget tree, needs two catch-alls installed
once before `runApp`:

- **`FlutterError.onError`**: framework, build, and render errors. In debug keep the red screen
  via `FlutterError.presentError`; in release forward to the crash reporter.
- **`PlatformDispatcher.instance.onError`**: the top-level catch for async errors that escape
  the framework, such as platform-channel failures or errors from futures outside a widget.
  Return `true` to mark them handled.

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();

  FlutterError.onError = (details) {
    if (kReleaseMode) {
      reporter.recordFlutterError(details);
    } else {
      FlutterError.presentError(details);
    }
  };

  PlatformDispatcher.instance.onError = (error, stack) {
    reporter.recordError(error, stack, fatal: true);
    return true;
  };

  assertSecureBaseUrl();
  // ...launch init, theme preload, runApp
}
```

These two cover overlapping but not identical surfaces to `runZonedGuarded`. They are the
current recommendation for most apps and are what this playbook uses; `runZonedGuarded` is
still the answer when you need zone-scoped context such as per-request tracing, and some crash
reporters install their own zone. Do not run both mechanisms for the same errors and then
wonder about duplicate reports.

**What this does and does not give you.** Together with the reporting interceptor, the three
main application error surfaces (sync framework, async platform, network) route into one
dashboard. It does **not** capture native crashes, OS process kills, failures that occur before
the reporter initializes, or errors in code paths that deliberately swallow them. Claiming
total coverage would be false; the honest claim is that no *application-level* Dart error is
invisible by default.

### Top-level wiring

- **`main.dart`**: install the error hooks, `assertSecureBaseUrl()`, preserve the splash, lock orientation, `initializeLaunch()`, preload theme through an `UncontrolledProviderScope` container, then `runApp`. Wraps `MaterialApp` in `ScreenUtilInit(designSize: Size(390, 844))`, reads `navigatorKeyProvider` for the navigator key, and sets `onGenerateRoute: generateRoute`.
- **`main_shell.dart`**: bottom-nav shell with `IndexedStack` over the tab screens and a `Notifier<int>` tracking tab history, for back-navigation and double-back-to-exit.
- **`router.dart`**: reads `navigatorKeyProvider`, a named-route `switch` in `generateRoute`, custom `PageRouteBuilder` transitions, protected routes wrapped in `AuthGuard`, and a default case that toasts "page not found".

### Navigator 1.0 by default, with explicit `go_router` triggers

This uses imperative Navigator 1.0 (`onGenerateRoute` plus `Navigator.pushNamed`) on purpose:
for a bottom-tab app with a handful of pushed detail screens it is less machinery, no codegen,
and no new mental model. **Migrate to `go_router` the moment any one of these becomes true**,
not before:

- **Deep links or universal links.** The OS hands you a URL (`myapp://order/42`) that must resolve to a specific screen stack. Reconstructing that by hand is where Navigator 1.0 stops being worth it.
- **Web or URL-addressable routes.** You ship a Flutter Web target and the address bar must reflect and drive navigation.
- **Nested or persistent shells.** Per-tab navigation stacks that survive tab switches, with a persistent bottom bar over nested routes. Navigator 1.0 can only fake this with brittle nested `Navigator`s.
- **Centralized declarative redirects.** Auth gating that must run as a route-level policy, once redirect logic sprawls across many routes and needs one authority.

When you migrate, the seams already exist: `navigatorKeyProvider` maps onto `go_router`'s
`navigatorKey`, `AuthGuard` maps onto a top-level `redirect`, and route names stay
`static const String routeName` on the screens.

---

## 5. Feature Anatomy (worked example: `auth`)

A feature is a vertical slice. Flow: **service, then repository, then notifier, then view**,
with `domain` as the contract in the middle.

```
View (screen + components)
      |  watches state, calls commands
Notifier                        <- holds DataState<T>
      |  calls
Repository (domain interface)   <- returns Result<T>, single source of truth
      |  calls
Service (Retrofit) + local source (cache)
```

### data/services (Retrofit)

Typed API surface. Abstract `@RestApi` class, `part '<name>.g.dart'` generated by build_runner.
One service per resource. Stateless: it wraps the endpoint and returns DTOs, nothing else.

```dart
part 'auth_service.g.dart';

@RestApi()
abstract class AuthApiService {
  factory AuthApiService(Dio dio, {String baseUrl}) = _AuthApiService;

  @POST('/auth/login')
  Future<LoginResponseDto> login(@Body() LoginRequestDto dto);

  @POST('/auth/logout')
  Future<void> logout();
}
```

For nullable responses, 204s, dynamic paths, or bulk mutations, signatures like
`Future<HouseholdDto?>` and `Future<void>` are fine; the repository's `guard` handles the rest.

### data/dto

Plain classes, no freezed. `fromJson` for responses, `toJson` for requests, and a `toDomain()`
that maps to the domain model. **DTOs never leak past the repository.**

```dart
class LoginRequestDto {
  const LoginRequestDto({
    required this.email,
    required this.password,
    this.clientType = 'mobile',
  });

  final String email;
  final String password;
  final String clientType;

  Map<String, dynamic> toJson() => {
        'email': email,
        'password': password,
        'clientType': clientType,
      };
}

class LoginResponseDto {
  const LoginResponseDto({required this.accessToken, required this.user /* ... */});

  final String accessToken;
  final UserDto user;

  factory LoginResponseDto.fromJson(Map<String, dynamic> json) => LoginResponseDto(
        accessToken: json['accessToken'] as String,
        user: UserDto.fromJson(json['user'] as Map<String, dynamic>),
      );

  AuthToken toDomain() => AuthToken(accessToken: accessToken, user: user.toDomain() /* ... */);
}
```

### data/source (local)

The local half of a repository's data. A TTL cache over `shared_preferences`, a `drift` DAO, or
a prefs wrapper. It lives in `data/`, **not** in `common/services/`, because it is a data source
and only its repository may touch it.

```dart
final profileCacheProvider = Provider<ProfileLocalSource>(
  (ref) => ProfileLocalSource(),
);

class ProfileLocalSource {
  static const _ttl = Duration(minutes: 10);

  Future<Profile?> read(String userId) async { /* value + `${key}_ts`, null if stale */ }
  Future<void> write(String userId, Profile profile) async { /* ... */ }
  Future<void> clear(String userId) async { /* ... */ }
}
```

### domain/models

Plain Dart. No JSON, no Dio, no Flutter. Immutable, `const` constructors, `required` fields.

```dart
class AuthToken {
  const AuthToken({
    required this.accessToken,
    required this.refreshToken,
    required this.expiresIn,
    required this.tokenType,
    required this.user,
  });

  final String accessToken;
  final String refreshToken;
  final int expiresIn;
  final String tokenType;
  final AuthUser user;
}
```

### domain/repository (contract)

An `abstract interface class`. Networked operations whose outcome the UI surfaces return
`Future<Result<T>>`. Local reads and fire-and-forget operations return a plain `Future`.

```dart
abstract interface class AuthRepository {
  Future<Result<AuthToken>> login({
    required String email,
    required String password,
    required bool rememberMe,
  });

  Future<void> logout();
  Future<AuthToken?> getStoredToken();
}
```

### data/repository (impl plus provider)

`with RepositoryGuard implements <Feature>Repository`. Dependencies injected via constructor; a
`Provider` wires the concrete service from `dioProvider`.

**The repository is the single source of truth.** It decides fresh versus cached, owns retry
and polling, maps DTO to domain, and returns `Result`. A notifier asks for data and does not
know where it came from. Cache logic never lives in a notifier.

```dart
final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepositoryImpl(
    apiService: AuthApiService(ref.watch(dioProvider)),
    secureStorage: ref.watch(secureStorageProvider),
  );
});

class AuthRepositoryImpl with RepositoryGuard implements AuthRepository {
  AuthRepositoryImpl({
    required AuthApiService apiService,
    required FlutterSecureStorage secureStorage,
  })  : _apiService = apiService,
        _secureStorage = secureStorage;

  final AuthApiService _apiService;
  final FlutterSecureStorage _secureStorage;

  @override
  Future<Result<AuthToken>> login({
    required String email,
    required String password,
    required bool rememberMe,
  }) async {
    try {
      final response = await _apiService.login(
        LoginRequestDto(email: email, password: password),
      );
      final token = response.toDomain();
      await _persistToken(token, rememberMe: rememberMe);
      return Ok(token);
    } on DioException catch (e) {
      await _rollbackSession();
      return mapError(e);
    } catch (e, stack) {
      // Mapping or storage failure after a 200: still a failed login for the user.
      reporter.recordError(e, stack, fatal: false);
      await _rollbackSession();
      return Err(AppException.unexpected(e));
    }
  }

  Future<void> _rollbackSession() async {
    await Future.wait(kSessionStorageKeys.map((k) => _secureStorage.delete(key: k)));
  }
}
```

The rollback catches **everything**, not just `DioException`. A 200 whose body fails to map, or
a keystore write that throws, would otherwise leave a half-written session on disk. Partial
writes are rolled back whatever threw.

A repository that combines remote and local sources, which is the shape the official guidance
means by "single source of truth":

```dart
class ProfileRepositoryImpl with RepositoryGuard implements ProfileRepository {
  ProfileRepositoryImpl(this._apiService, this._cache);

  final ProfileApiService _apiService;
  final ProfileLocalSource _cache;

  @override
  Future<Result<Profile>> getProfile(String userId, {bool forceRefresh = false}) async {
    if (!forceRefresh) {
      final cached = await _cache.read(userId);
      if (cached != null) return Ok(cached);
    }
    return guard(() async {
      final profile = (await _apiService.getProfile(userId)).toDomain();
      await _cache.write(userId, profile);
      return profile;
    });
  }
}
```

The notifier calls `getProfile(userId)` and gets data. Whether it came from disk or the
network is not its problem. `forceRefresh: true` is what pull-to-refresh passes.

### presentation/notifier

State lives in a `Notifier` or `AsyncNotifier`. Public methods are **commands**: verbs the view
calls. Three patterns, in order of reach. **Default to Pattern 1 and escalate only when a
concrete trigger forces it.**

```
Does the screen have ONE async action (login, submit, fetch)?   -> Pattern 1
Does the state seed from storage or react to a stream?          -> Pattern 2
Does the screen have MULTIPLE independent async slices AND
  per-item loading flags AND optimistic updates?                -> Pattern 3

None of these -> start with Pattern 1. You can always promote later.
```

#### Pattern 1: simple action notifier (the default)

State **is** a `DataState<T>`. Idle initially, then loading, then the mapped repository result.
This is the right shape for most features; reach past it only when the list above says so.

```dart
final loginNotifierProvider =
    NotifierProvider<LoginNotifier, DataState<AuthToken>>(LoginNotifier.new);

class LoginNotifier extends Notifier<DataState<AuthToken>> {
  @override
  DataState<AuthToken> build() => const DataState.idle();

  /// Command. Called from a gesture handler in the view.
  Future<void> login({
    required String email,
    required String password,
    required bool rememberMe,
  }) async {
    if (state.isLoading) return; // double-tap guard
    state = const DataState.loading();

    final result = await ref.read(authRepositoryProvider).login(
          email: email,
          password: password,
          rememberMe: rememberMe,
        );

    if (result case Ok(:final value)) {
      ref.read(authStateProvider.notifier).setAuthenticated(value);
    }
    state = DataStateX.from(result); // the one Result -> DataState conversion
  }

  void reset() => state = const DataState.idle();
}
```

The `isLoading` guard matters: without it, a user double-tapping Login fires two requests and
the second overwrites the first's state.

#### Pattern 2: session or seeded notifier

`AsyncNotifier<T?>` seeding from storage and reacting to a stream. Use this only when the
initial value comes from an async source or must react to an external event.

```dart
final authStateProvider =
    AsyncNotifierProvider<AuthStateNotifier, AuthToken?>(AuthStateNotifier.new);

class AuthStateNotifier extends AsyncNotifier<AuthToken?> {
  @override
  Future<AuthToken?> build() {
    final sub = ref.watch(sessionExpiredProvider).stream.listen((_) {
      if (state.value != null) state = const AsyncData(null);
    });
    ref.onDispose(sub.cancel);
    return ref.read(authRepositoryProvider).getStoredToken();
  }

  void setAuthenticated(AuthToken token) => state = AsyncData(token);
}
```

#### Pattern 3: composite-state notifier (reach for this last)

A hand-written immutable state class with `copyWith` and derived getters. Mutations funnel
through one `_runCommand` skeleton that flips a flag, awaits, maps to `String?` (null means
success), and clears the flag in `finally`. This is a hand-rolled version of the Command
pattern the official guide describes: a wrapper that handles running, complete, and error
around one method.

> **Gate. You need this only when ALL of these are true:**
>
> - the screen manages **multiple independent `DataState` fields** in one state class;
> - there are **per-item loading flags** that must coexist with the main data state;
> - **optimistic local mutations** must update the UI before the server confirms.
>
> If you cannot check all three, Pattern 1 with a second notifier is simpler. This is the most
> copy-paste-prone pattern and must not become the default.

```dart
class FamilyState {
  const FamilyState({
    this.householdState = const DataState.idle(),
    this.removingMemberIds = const {},
  });

  final DataState<Household?> householdState;
  final Set<String> removingMemberIds;

  Household? get household => householdState.valueOrNull;

  FamilyState copyWith({
    DataState<Household?>? householdState,
    Set<String>? removingMemberIds,
  }) =>
      FamilyState(
        householdState: householdState ?? this.householdState,
        removingMemberIds: removingMemberIds ?? this.removingMemberIds,
      );
}

class FamilyNotifier extends Notifier<FamilyState> {
  late final FamilyRepository _repository;

  @override
  FamilyState build() {
    _repository = ref.watch(familyRepositoryProvider);
    ref.watch(authStateProvider); // rebuild (and reset) on logout
    return const FamilyState();
  }

  /// Returns null on success, or a user-facing message on failure.
  Future<String?> _runCommand<T>(
    Future<Result<T>> Function() action, {
    void Function()? onStart,
    void Function()? onFinish,
    FutureOr<void> Function(T value)? onSuccess,
  }) async {
    onStart?.call();
    try {
      final result = await action();
      switch (result) {
        case Ok(:final value):
          await onSuccess?.call(value);
          return null;
        case Err(:final exception):
          return errorToMessage(exception);
      }
    } finally {
      onFinish?.call();
    }
  }

  Future<String?> removeMember(String id) => _runCommand(
        () => _repository.removeMember(id),
        onStart: () => state = state.copyWith(
          removingMemberIds: {...state.removingMemberIds, id},
        ),
        onFinish: () => state = state.copyWith(
          removingMemberIds: state.removingMemberIds.where((e) => e != id).toSet(),
        ),
        onSuccess: (_) { /* optimistic local removal */ },
      );
}

final familyProvider = NotifierProvider<FamilyNotifier, FamilyState>(FamilyNotifier.new);
```

The `switch` is exhaustive because `Result` is sealed. No `is` checks, no casts.

### presentation/screens (the view)

The screen declares `routeName`, builds the scaffold, and delegates to components. It does not
build large widget trees inline. Loads kick off in a post-frame callback guarded by the idle
state.

```dart
class LoginScreen extends StatelessWidget {
  const LoginScreen({super.key});
  static const String routeName = '/login';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: backgroundGradient(
        context: context,
        child: SafeArea(
          child: SingleChildScrollView(
            child: Padding(
              padding: EdgeInsets.symmetric(
                horizontal: Dimensions.px24,
                vertical: Dimensions.px64,
              ),
              child: const Column(
                mainAxisSize: MainAxisSize.min,
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Center(child: LoginTitle()),
                  LoginForm(),
                ],
              ),
            ),
          ),
        ),
      ),
    );
  }
}
```

### presentation/components (react to state)

Components consume the notifier. Side effects (navigation, toasts) live in `ref.listen`;
`errorToMessage` converts the carried exception to copy at this edge.

```dart
@override
Widget build(BuildContext context, WidgetRef ref) {
  ref.listen<DataState<AuthToken>>(loginNotifierProvider, (_, next) {
    switch (next) {
      case DataSuccess():
        ref.read(loginNotifierProvider.notifier).reset();
        Navigator.pushNamedAndRemoveUntil(context, MainShell.routeName, (_) => false);
      case DataError(:final exception):
        showToast(context, errorToMessage(exception), type: ToastType.error);
        ref.read(loginNotifierProvider.notifier).reset();
      case DataIdle() || DataLoading():
        break;
    }
  });

  final state = ref.watch(loginNotifierProvider);
  return LoginButton(
    loading: state.isLoading,
    onPressed: () => ref.read(loginNotifierProvider.notifier).login(/* ... */),
  );
}
```

The `switch` lists all four cases explicitly rather than using `default`, so adding a fifth
state to `DataState` becomes a compile error here instead of a silently ignored branch.

---

## 6. State Management with Riverpod

Riverpod is both the DI container and the notifier layer. One rule governs all of it: **a
provider lives next to the thing it exposes, and the provider graph points the same way the
layers do.** Outer depends on inner, never sideways.

### The provider graph mirrors the layers

```
secureStorageProvider ---+
navigatorKeyProvider ----+--> dioProvider --> <feature>RepositoryProvider --> <feature>NotifierProvider --> Widget
sessionExpiredProvider --+                          (data)                         (notifier)
refreshClientProvider ---+
<feature>LocalSourceProvider ---------------------->/

sessionExpiredProvider --------> authStateProvider
```

- `secureStorageProvider`, `dioProvider`, `sessionExpiredProvider`, `navigatorKeyProvider`, `refreshClientProvider`: core infrastructure as `Provider<T>`. Constructed once, injected everywhere. No global mutable state; everything is provider-scoped and overridable in tests.
- `<feature>RepositoryProvider`: DI for the data layer. A `Provider<<Feature>Repository>` that reads `dioProvider` and any local source, returning the concrete `Impl`. Presentation depends on this **interface** provider, never on the `Impl` type.
- `<feature>NotifierProvider`: the notifier. It reads the repository provider; nothing below it can see it.
- A widget only ever watches notifier providers, plus a couple of app-wide ones such as `authStateProvider` and `themeProvider`. It never watches a repository or `dioProvider` directly.

If a widget reaches past its notifier to a repository or Dio, the graph is broken the same way
an import that skips a layer is, and the §8 boundary lint is what catches it.

### Which provider type, and when

| Provider | Use for | Example |
| --- | --- | --- |
| `Provider<T>` | Stateless DI, constructed once | `dioProvider`, `secureStorageProvider`, `<feature>RepositoryProvider` |
| `NotifierProvider<N, S>` | Per-screen or per-action notifier | `loginNotifierProvider` (Pattern 1), `familyProvider` (Pattern 3) |
| `AsyncNotifierProvider<N, T>` | State seeded from an async source or stream | `authStateProvider` (Pattern 2) |

No `StateProvider` or `ChangeNotifierProvider` for feature state: they hide the transitions the
UI needs to see. `FutureProvider` is fine for a read-only derived value with no commands
attached, but a screen with actions gets a `Notifier`.

### `watch` vs `read` vs `listen`

- **`ref.watch`**: reactive dependency. Only inside a `build()` (widget `build`, `Notifier.build`, or a provider body). Watching `authStateProvider` inside a notifier's `build()` is how a feature resets on logout.
- **`ref.read`**: one-shot, non-reactive. Only inside callbacks and event handlers, to fire a command. Never `read` in `build` (you would miss updates); never `watch` in a callback (you would rebuild-thrash).
- **`ref.listen`**: side effects that are not UI. Navigation, toasts, dialogs. It lives in a widget `build` but returns nothing. This is the one place a `DataError` becomes a toast.

### Notifier conventions

- `build()` returns the **initial** state and does setup only: `watch` dependencies, subscribe to streams with `ref.onDispose` to cancel. No network call fires from an action notifier's `build()`; the view kicks off the load in a post-frame callback guarded by the idle state.
- Public methods are **commands**: verbs the view calls. They mutate `state`; the UI reacts. Notifiers never return widgets and never touch `BuildContext`.
- Cross-notifier writes go through `.notifier`, never by mutating another provider's state directly.
- Reset ephemeral state with a `reset()` command or `ref.invalidate(provider)`. App-wide teardown flows through the session-expired stream, not manual resets scattered across notifiers.

### Lifecycle and scope

Providers are **kept alive by default**, because the app is tab-based over an `IndexedStack`:
every tab stays mounted for the app's life, so its notifier should too. Disposing it would
drop scroll position, filters, and loaded data on every tab switch.

Add `.autoDispose` when **all** of these hold:

- the provider is scoped to a route that is pushed and popped (a detail screen, a modal, a wizard step), not a persistent tab;
- its state is per-visit and stale on return, so rebuilding from scratch is correct rather than lossy;
- it holds something that must be released promptly: a `StreamSubscription`, a large list, a controller.

```dart
final orderDetailProvider =
    NotifierProvider.autoDispose<OrderDetailNotifier, DataState<Order>>(
  OrderDetailNotifier.new,
);
```

Two escape hatches: `ref.keepAlive()` inside an `autoDispose` provider pins it after a
successful load (dispose while loading, cache once it succeeds), and `ref.invalidate(provider)`
forces a keep-alive provider to rebuild on demand, for example on pull-to-refresh, without
changing its scope.

**If you cannot point to a push/pop route AND a leak or staleness reason, leave it keep-alive.**

Overrides are the test and bootstrap seam:
`ProviderContainer(overrides: [<feature>RepositoryProvider.overrideWithValue(mock)])`. Because
every dependency is a provider, nothing needs a real Dio or keystore to test.

---

## 7. Testing

The layering exists so each piece is testable in isolation, and this is where it pays off. Four
seams, four kinds of test: repository, notifier, interceptor, and the layer boundaries
themselves. `mocktail` for doubles, `ProviderContainer` for DI. Never a real Dio, real keystore,
or real socket. Mock the boundary directly below the thing under test.

The official guidance puts this at the "strongly recommend" tier: write unit tests for every
service, repository, and notifier.

### Repository test: mock the service, assert `Result` branches

The repository's job is to call the service, map DTO to domain, wrap in `Result`, and undo
partial writes on failure. Assert both branches.

```dart
class _MockAuthApi extends Mock implements AuthApiService {}
class _MockStorage extends Mock implements FlutterSecureStorage {}

void main() {
  setUpAll(() {
    registerFallbackValue(const LoginRequestDto(email: '', password: ''));
  });

  late _MockAuthApi api;
  late _MockStorage storage;
  late AuthRepositoryImpl repo;

  setUp(() {
    api = _MockAuthApi();
    storage = _MockStorage();
    repo = AuthRepositoryImpl(apiService: api, secureStorage: storage);
    when(() => storage.write(key: any(named: 'key'), value: any(named: 'value')))
        .thenAnswer((_) async {});
    when(() => storage.delete(key: any(named: 'key'))).thenAnswer((_) async {});
  });

  test('login returns Ok and persists the access token on 200', () async {
    when(() => api.login(any())).thenAnswer((_) async => _loginResponseDto);

    final result = await repo.login(email: 'a@b.com', password: 'pw', rememberMe: true);

    expect(result, isA<Ok<AuthToken>>());
    verify(() => storage.write(key: kAccessToken, value: _loginResponseDto.accessToken))
        .called(1);
  });

  test('login returns Err and rolls back the partial write on failure', () async {
    when(() => api.login(any())).thenThrow(DioException(
      requestOptions: RequestOptions(path: '/auth/login'),
      response: Response(
        requestOptions: RequestOptions(path: '/auth/login'),
        statusCode: 401,
      ),
    ));

    final result = await repo.login(email: 'a@b.com', password: 'bad', rememberMe: false);

    expect(result, isA<Err<AuthToken>>());
    expect((result as Err<AuthToken>).exception.statusCode, 401); // mapped, not raw Dio
    verify(() => storage.delete(key: kAccessToken)).called(1);    // rollback fired
  });

  test('login rolls back when mapping throws after a 200', () async {
    when(() => api.login(any())).thenThrow(TypeError()); // shape changed server-side

    final result = await repo.login(email: 'a@b.com', password: 'pw', rememberMe: false);

    expect(result, isA<Err<AuthToken>>());
    verify(() => storage.delete(key: kAccessToken)).called(1);
  });
}
```

The third test is the one people skip. It pins the behavior that a non-Dio failure still
produces `Err` and still rolls back, rather than escaping as an uncaught exception.

### Repository test: cache is consulted before the network

```dart
test('getProfile serves the cache and does not hit the network', () async {
  when(() => cache.read('u1')).thenAnswer((_) async => _profile);

  final result = await repo.getProfile('u1');

  expect(result, isA<Ok<Profile>>());
  verifyNever(() => api.getProfile(any()));
});

test('getProfile bypasses the cache when forceRefresh is true', () async {
  when(() => cache.read('u1')).thenAnswer((_) async => _profile);
  when(() => api.getProfile('u1')).thenAnswer((_) async => _profileDto);
  when(() => cache.write(any(), any())).thenAnswer((_) async {});

  await repo.getProfile('u1', forceRefresh: true);

  verify(() => api.getProfile('u1')).called(1);
});
```

### Notifier test: `ProviderContainer` with the repository overridden

The notifier owns exactly one thing: the idle, loading, result transition. No widgets, no
`WidgetTester`.

```dart
class _MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late _MockAuthRepository repo;
  late ProviderContainer container;

  setUp(() {
    repo = _MockAuthRepository();
    when(() => repo.getStoredToken()).thenAnswer((_) async => null);
    container = ProviderContainer(
      overrides: [authRepositoryProvider.overrideWithValue(repo)],
    );
    addTearDown(container.dispose);
  });

  test('login emits idle, loading, success', () async {
    when(() => repo.login(
          email: any(named: 'email'),
          password: any(named: 'password'),
          rememberMe: any(named: 'rememberMe'),
        )).thenAnswer((_) async => Ok(_token));

    final emitted = <DataState<AuthToken>>[];
    container.listen(
      loginNotifierProvider,
      (_, next) => emitted.add(next),
      fireImmediately: true,
    );

    await container
        .read(loginNotifierProvider.notifier)
        .login(email: 'a@b.com', password: 'pw', rememberMe: false);

    expect(emitted.map((s) => s.runtimeType), [
      DataIdle<AuthToken>,
      DataLoading<AuthToken>,
      DataSuccess<AuthToken>,
    ]);
  });

  test('a second login while loading is ignored', () async {
    when(() => repo.login(
          email: any(named: 'email'),
          password: any(named: 'password'),
          rememberMe: any(named: 'rememberMe'),
        )).thenAnswer((_) async {
      await Future<void>.delayed(const Duration(milliseconds: 50));
      return Ok(_token);
    });

    final notifier = container.read(loginNotifierProvider.notifier);
    await Future.wait([
      notifier.login(email: 'a@b.com', password: 'pw', rememberMe: false),
      notifier.login(email: 'a@b.com', password: 'pw', rememberMe: false),
    ]);

    verify(() => repo.login(
          email: any(named: 'email'),
          password: any(named: 'password'),
          rememberMe: any(named: 'rememberMe'),
        )).called(1); // the double-tap guard held
  });
}
```

### Interceptor test: two concurrent 401s, exactly one refresh

This is the test that justifies the networking design, and it is written against the
**generation counter**, not against queue semantics. The fake token store is mutable, because
the second request must be able to observe the rotated token: with a constant `readToken` the
replay can never succeed and the test proves nothing.

```dart
class _HeaderAdapter implements HttpClientAdapter {
  @override
  Future<ResponseBody> fetch(
    RequestOptions options,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    final refreshed = options.headers['Authorization'] == 'Bearer new';
    return ResponseBody.fromString(
      '{}',
      refreshed ? 200 : 401,
      headers: {
        Headers.contentTypeHeader: [Headers.jsonContentType],
      },
    );
  }

  @override
  void close({bool force = false}) {}
}

void main() {
  test('two concurrent 401s trigger exactly one refresh and both replay', () async {
    var refreshCount = 0;
    var stored = 'old'; // mutable: the rotation must be observable

    final retryClient = Dio()..httpClientAdapter = _HeaderAdapter();
    final dio = Dio(BaseOptions(baseUrl: 'https://test'))
      ..httpClientAdapter = _HeaderAdapter()
      ..interceptors.add(AuthInterceptor(
        readToken: () async => stored,
        doRefresh: () async {
          refreshCount++;
          await Future<void>.delayed(const Duration(milliseconds: 20)); // widen the race
          return stored = 'new';
        },
        onExpired: () async {},
        retryClient: retryClient,
      ));

    final responses = await Future.wait([dio.get('/a'), dio.get('/b')]);

    expect(refreshCount, 1);                                       // single-flight held
    expect(responses.map((r) => r.statusCode), everyElement(200)); // both replayed
  });

  test('three concurrent 401s with a dead refresh token expire the session once', () async {
    var refreshCount = 0;
    var expiredCount = 0;

    final dio = Dio(BaseOptions(baseUrl: 'https://test'))
      ..httpClientAdapter = _HeaderAdapter()
      ..interceptors.add(AuthInterceptor(
        readToken: () async => 'old',
        doRefresh: () async {
          refreshCount++;
          await Future<void>.delayed(const Duration(milliseconds: 20));
          return null; // refresh token rejected
        },
        onExpired: () async => expiredCount++,
        retryClient: Dio()..httpClientAdapter = _HeaderAdapter(),
      ));

    await Future.wait([
      expectLater(dio.get('/a'), throwsA(isA<DioException>())),
      expectLater(dio.get('/b'), throwsA(isA<DioException>())),
      expectLater(dio.get('/c'), throwsA(isA<DioException>())),
    ]);

    expect(refreshCount, 1);  // the failed-generation marker held
    expect(expiredCount, 1);  // the expire-once latch held
  });

  test('a new session can expire again after onSessionStarted', () async {
    // Expire once, call onSessionStarted(), expire again.
    // Without the reset the latch stays closed and the second expiry never fires.
  });

  test('a multipart body is not replayed', () async {
    // 401 on a FormData request: the token is refreshed and persisted,
    // but the request is not re-sent with a consumed body.
  });

  test('a non-401 error passes through untouched', () async {
    // Adapter returns 500; assert refreshCount == 0 and the error propagates.
  });
}
```

The first of these is the test that matters, and it is the easiest one to write wrongly. Fire a
single request and `expiredCount == 1` passes trivially while the concurrent case silently
expires three times and attempts three refreshes.

What each test pins down: the repository maps outcomes and rolls back, the notifier drives the
state machine and nothing else, the interceptor collapses a refresh storm into one call and
expires cleanly when refresh genuinely fails. If a future change breaks any of those
invariants, one of these fails.

### Golden rule for what to test

Test **invariants**, not coverage percentages. "The repository rolls back a partial write" is an
invariant. "`AuthRepositoryImpl` has 94% line coverage" is a number. The coverage floor in CI
exists to catch whole files that nobody tested, not to be optimized.

---

## 8. Tooling and Enforcement

The dependency rules above are only real if CI can fail on them. Three things carry that weight:
`analysis_options.yaml` for style and imports, `riverpod_lint` for state-layer misuse, and a
**real custom lint package** for the layering rules the analyzer cannot express on its own.

### `analysis_options.yaml`

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  plugins:
    - custom_lint # hosts riverpod_lint + the app_lints package below
  errors:
    always_use_package_imports: error # relative imports fail the build, not just warn
    missing_provider_scope: error     # from riverpod_lint: a launch-time bug, not a nit

linter:
  rules:
    always_use_package_imports: true
    require_trailing_commas: true
    prefer_const_constructors: true
    prefer_const_constructors_in_immutables: true
    prefer_const_literals_to_create_immutables: true
    prefer_final_fields: true
    unnecessary_this: true
    sort_pub_dependencies: true

custom_lint:
  rules:
    # from app_lints (tool/lints)
    - layer_boundary
    - no_cross_feature_imports
    # from riverpod_lint
    - avoid_public_notifier_properties
    - provider_dependencies
    - notifier_extends
```

Promoting `always_use_package_imports` to `error` is deliberate: the boundary lint keys off
`package:` paths, so a stray relative import is not a style nit, it is a hole in the enforcement.

### Boundary lint: the actual implementation, not pseudo-config

> **This is the part most architecture READMEs get wrong.** `custom_lint` rules are Dart
> classes in a package. The `custom_lint:` block in `analysis_options.yaml` can **enable** rules
> and pass options to them; it cannot **define** what a rule matches. A YAML block that appears
> to declare `from` and `to` path globs will be silently ignored by the analyzer unless a Dart
> rule you wrote reads those options. If you copy a config like that from a blog post, you get
> zero enforcement and a green build.

The rules live in a local package at `tool/lints`, wired in as a path dependency:

```yaml
# pubspec.yaml
dev_dependencies:
  custom_lint: ^0.7.0
  riverpod_lint: ^3.0.0
  app_lints:
    path: tool/lints
```

```dart
// tool/lints/lib/app_lints.dart
import 'package:analyzer/error/error.dart';
import 'package:analyzer/error/listener.dart';
import 'package:custom_lint_builder/custom_lint_builder.dart';

PluginBase createPlugin() => _AppLints();

class _AppLints extends PluginBase {
  @override
  List<LintRule> getLintRules(CustomLintConfigs configs) => const [
        LayerBoundary(),
        NoCrossFeatureImports(),
      ];
}

/// A feature's presentation layer may not import that feature's data layer.
class LayerBoundary extends DartLintRule {
  const LayerBoundary() : super(code: _code);

  static const _code = LintCode(
    name: 'layer_boundary',
    problemMessage:
        'presentation must go through the domain contract, not data/.',
    errorSeverity: ErrorSeverity.ERROR,
  );

  @override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    final self = _segments(resolver.source.uri);
    if (self == null || !self.contains('presentation')) return;

    context.registry.addImportDirective((node) {
      // Resolve against the library, so relative imports and aliases are
      // normalised to a package URI before matching.
      final target = node.libraryImport?.importedLibrary?.source.uri;
      final seg = _segments(target);
      if (seg == null) return;
      if (seg.contains('features') && seg.contains('data')) {
        reporter.atNode(node, _code);
      }
    });
  }
}

/// Path segments of a `package:` URI, or null for dart:/http/unresolved imports.
/// Matching on segments rather than substrings avoids false hits from a
/// directory literally named `metadata` or a package called `mydata`, and is
/// separator-independent on Windows.
List<String>? _segments(Uri? uri) =>
    uri == null || uri.scheme != 'package' ? null : uri.pathSegments;

/// No feature may import another feature's internals.
class NoCrossFeatureImports extends DartLintRule {
  const NoCrossFeatureImports() : super(code: _code);

  static const _code = LintCode(
    name: 'no_cross_feature_imports',
    problemMessage:
        'cross-feature import; extract the shared piece to common/ or core/.',
    errorSeverity: ErrorSeverity.ERROR,
  );

  @override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    final own = _featureOf(_segments(resolver.source.uri));
    if (own == null) return;

    context.registry.addImportDirective((node) {
      final target = _featureOf(
        _segments(node.libraryImport?.importedLibrary?.source.uri),
      );
      if (target != null && target != own) {
        reporter.atNode(node, _code);
      }
    });
  }
}

/// The segment immediately after `features`, or null if the URI is not inside one.
String? _featureOf(List<String>? seg) {
  if (seg == null) return null;
  final i = seg.indexOf('features');
  return i == -1 || i + 1 >= seg.length ? null : seg[i + 1];
}
```

Roughly a hundred lines, written once, and every PR that points an import arrow the wrong way
goes red before review.

**Why segments and resolved URIs, not substrings.** The obvious version of this rule does
`path.contains('/data/')` on the raw import string. That breaks in four ways: a directory named
`metadata` or a package named `mydata` matches by substring; a relative import never contains
the package path at all; Windows paths use backslashes; and generated `part` files are
attributed to their parent library. Resolving `node.libraryImport?.importedLibrary?.source.uri`
gives a normalised `package:` URI regardless of how the import was written, and comparing path
*segments* removes the substring hazard. Unresolvable imports return null and are skipped
rather than flagged, so a rule failure is never a false positive.

**Test the rules.** `custom_lint` ships a test harness; point it at a fixture file that
deliberately violates each rule and assert the expected diagnostics. A lint you have not tested
is a lint that silently stopped working three refactors ago.

**Off-the-shelf alternative.** If you would rather not maintain a lint package, the
`import_lint` package does the same job with declarative configuration and its own YAML section.
It is a legitimate substitute. What is not legitimate is publishing YAML that looks like it
configures a rule which does not exist.

### `riverpod_lint`

Runs through the same `custom_lint` plugin, so it costs nothing extra once the plugin is
enabled, and it catches Riverpod mistakes that compile cleanly and misbehave at runtime. The
ones that earn their keep here:

- **`missing_provider_scope`**: the app must be wrapped in a `ProviderScope`. Promoted to `error` above, because a missing scope is a launch failure.
- **`avoid_public_notifier_properties`**: a `Notifier`'s only public surface is its commands and `state`. Public mutable fields do not trigger rebuilds. This is the §6 "public methods are commands" rule, machine-checked.
- **`provider_dependencies`** and **`notifier_extends`**: keep provider and notifier declarations in the shape the rest of this document assumes.

Between the boundary lints (structure), `always_use_package_imports` (imports), and
`riverpod_lint` (state), `flutter analyze` fails on all three classes of drift, so the
conventions hold even on a branch nobody reviewed closely.

---

## 9. Naming and File Conventions

**Files and folders:** `snake_case.dart`, `snake_case/` directories. One primary class per file;
the filename matches its subject (`auth_repository_impl.dart` holds `AuthRepositoryImpl`).

**Layer suffixes:**

| Thing | File | Type |
| --- | --- | --- |
| Service | `<feature>_service.dart` | `<Feature>ApiService` (Retrofit abstract class) |
| DTO | `<thing>_dto.dart` | `<Thing>RequestDto` / `<Thing>ResponseDto` |
| Local source | `<thing>_local_source.dart` | `<Thing>LocalSource` |
| Domain model | `<thing>.dart` | `<Thing>` (no suffix) |
| Repository contract | `<feature>_repository.dart` | `abstract interface class <Feature>Repository` |
| Repository impl | `<feature>_repository_impl.dart` | `<Feature>RepositoryImpl` |
| Notifier | `<thing>_notifier.dart` | `<Thing>Notifier` + `<thing>NotifierProvider` |
| Screen | `<name>_screen.dart` | `<Name>Screen` with `static const String routeName` |

**Providers:** `camelCase` ending in `Provider`. One provider declared next to the thing it
exposes.

**Constants:** global constants prefixed `k` (`kAccessToken`, `kConnectTimeout`). Private
file-level constants start with `_k`.

**Strings, colors, dimensions:** never inline. Copy via `Strings.x`, colors via
`context.xColor`, sizes via `Dimensions.pxNN`.

**Imports** (ordered, blank line between groups):

1. Dart SDK (`dart:async`)
2. Flutter and third-party packages
3. Local `package:` imports

Always `package:` imports inside `lib/`, never `../../` relatives. Absolute paths make the layer
a file lives in obvious from its import line, survive file moves, and are what the boundary lint
keys off. Enforced, not preferred.

**Style:** `const` constructors wherever possible; `required` named parameters; immutable state
with manual `copyWith`; expression bodies for one-liners; comments that explain **why**, not
what. Prefer Dart 3 pattern matching over `is` checks and casts when consuming `Result` or
`DataState`.

---

## 10. New Feature Checklist

Adding a networked feature `<feature>`:

1. **Folders**: `features/<feature>/{data/{dto,services,source,repository},domain/{models,repository},presentation/{notifier,screens,components}}`.
2. **Domain models** (`domain/models/`): plain Dart, immutable, `const`, `required` fields. No JSON.
3. **Repository contract** (`domain/repository/`): `abstract interface class` returning `Future<Result<T>>` for networked operations the UI surfaces, plain `Future<T>` for local or fire-and-forget.
4. **DTOs** (`data/dto/`): plain classes with `fromJson`, `toJson`, `toDomain()`.
5. **Service** (`data/services/`): Retrofit `@RestApi` abstract class with `part '<feature>_service.g.dart'`.
6. **build_runner**: `dart run build_runner build --delete-conflicting-outputs`.
7. **Local source** (`data/source/`), only if the feature caches or persists. It belongs to this repository and nothing else may touch it.
8. **Repository impl** (`data/repository/`): `with RepositoryGuard implements <Feature>Repository`. Use `guard(...)`; use explicit try/catch plus `mapError` where there are side effects to roll back, and catch non-Dio throws too. Declare `<feature>RepositoryProvider` reading `dioProvider`.
9. **Notifier** (`presentation/notifier/`): **default to Pattern 1** (`Notifier<DataState<T>>`). Pattern 2 only for state seeded from storage or a stream. Pattern 3 only when all three gate conditions in §5 hold. Declare its `NotifierProvider` in the same file. `ref.watch(authStateProvider)` inside `build()` if it should reset on logout.
10. **Screen** (`presentation/screens/`): thin view with `static const String routeName`; compose components; kick off loads in a post-frame callback guarded by the idle state.
11. **Components** (`presentation/components/`): every non-trivial widget. `ref.listen` for navigation and toasts, `errorToMessage(exception)` for copy.
12. **Strings and assets**: add copy to `Strings`, paths to `assets.dart`. No inline literals.
13. **Route and guard**: register in `router.dart`; wrap protected routes in `AuthGuard`.
14. **Shell**: if it is a tab, add it to `main_shell.dart`'s `IndexedStack`.
15. **Tests** (`test/features/<feature>/`): a repository test asserting the `Ok` branch, the `Err` branch, and the rollback on a non-Dio throw; a notifier test on a `ProviderContainer` with the repository overridden. Anything with retry or refresh behavior gets an interceptor-style test.
16. **Analyze**: `flutter analyze` clean. `package:` imports only, no presentation-to-data import, no cross-feature import.

**Rules that never bend.** DTOs stay in `data`. Domain models stay platform-independent.
Presentation depends only on the domain interface. Repositories return `Result` and are the
single source of truth for their data, including cache. Notifiers own `DataState` and are the
only place `loading` and `idle` exist. Errors travel as `AppException` and become strings exactly
once, at the UI edge, via `errorToMessage`. Views orchestrate, components render. Imports are
`package:` and never cross a boundary the lint forbids.

---

## 11. Production and Release Concerns

Everything above is application architecture. This section is the operational layer: the
difference between clean code and a shippable release. It also states, on the record, what this
playbook leaves out, so an omission reads as a decision rather than an oversight.

### 11.1 Environments and flavors

Config comes from the build, not from a committed constant. Editing source and recompiling to
switch environments means staging and production are different artifacts, and any URL or key
lives in version control.

- **Build flavors**: Android `productFlavors` (`dev`, `staging`, `prod`) and matching iOS schemes give distinct application IDs, icons, and names, so all three can sit on one device.
- **`--dart-define-from-file`**: per-environment JSON supplies values at build time, read via `String.fromEnvironment`, so the same compiled code runs everywhere.

```json
// config/prod.json  (git-ignored if it carries anything sensitive)
{
  "APP_ENV": "prod",
  "BASE_URL": "https://api.example.com/v1",
  "SENTRY_DSN": "..."
}
```

```bash
flutter build appbundle --flavor prod --dart-define-from-file=config/prod.json
```

`assertSecureBaseUrl()` still runs and still catches an empty or cleartext URL, now guarding the
injected value at boot.

**A dart-define is not a secret store.** Defines are compiled into the binary and are
recoverable from it. They are correct for base URLs, DSNs, public keys, and feature flags. They
are wrong for anything whose disclosure matters: those belong on your server, behind an
authenticated endpoint. If a value must never appear on a device, do not ship it to the device.

### 11.2 CI pipeline

The lints and tests only mean something if a machine runs them on every PR. That is what makes
"CI-enforced boundaries" true rather than aspirational.

```yaml
# .github/workflows/ci.yaml
name: ci
on:
  pull_request:
  push: { branches: [main] }

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { channel: stable }

      - run: flutter pub get
      - run: dart format --set-exit-if-changed .

      - name: Generate code
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Fail if generated code is stale or hand-edited
        run: |
          if ! git diff --exit-code --stat; then
            echo "::error::Generated files differ from committed output."
            echo "Run: dart run build_runner build --delete-conflicting-outputs"
            exit 1
          fi

      - run: flutter analyze          # includes custom_lint + riverpod_lint
      - run: flutter test --coverage

      - name: Enforce coverage floor
        run: |
          sudo apt-get install -y lcov
          pct=$(lcov --summary coverage/lcov.info 2>&1 | grep -oP 'lines.*?\K[0-9.]+(?=%)')
          echo "coverage: $pct%"
          awk "BEGIN { exit !($pct >= 70) }"
```

The `git diff` step is the one people leave out, and it is what makes "generated code stays
synchronized" a rule rather than a hope. Without it, CI regenerates silently and a hand-edited
`.g.dart` sails through review. With it, stale or edited generated output fails the build.

Release builds (`--flavor prod`, signing, store upload) run on tags in a separate job.

### 11.3 Release hardening

- **Obfuscation and symbolication**: build with `--obfuscate --split-debug-info=build/symbols`, archive the symbol files, and upload them to the crash reporter so stack traces stay readable.
- **Transport security**: HTTPS is asserted at boot. For a high-value target, add certificate pinning at the Dio layer. The single-Dio design makes that a one-place change. Pin to a CA or to a backup key as well as the leaf, or a certificate rotation bricks every installed app.
- **Store shrinking**: keep R8 and resource shrinking on for Android release. JSON is code-generated, so there is usually no reflection path to keep.
- **No secrets in logs**: the logging interceptor redacts `Authorization` and `/auth/` bodies and is `kDebugMode`-only. Keep it that way.
- **Backup exclusion**: confirm the secure-storage preferences file is excluded from Android auto-backup (§3), or restores will produce unwrappable keys on the target device.

### 11.4 Deliberately out of scope

Each of these is a "reach for this when the requirement is real" pointer, not an accidental gap.

| Concern | When you need it | Reach for |
| --- | --- | --- |
| **Localization** | more than one locale | `flutter_localizations` + ARB, or `slang`; retire the `static const Strings` |
| **Offline-first / local DB** | data must survive offline or sync | `drift` or `isar` behind the repository, plus a migration story. A TTL cache is not a database |
| **Pagination** | long server-driven lists | a paged notifier state (cursor or page) plus an infinite-scroll component |
| **Analytics and feature flags** | product measurement, gradual rollout | an analytics facade and a remote-config provider, injected like any other `Provider` |
| **Push and deep links** | re-engagement, URL entry points | FCM/APNs plus `go_router` (the §4 deep-link trigger) |
| **Accessibility audit** | inclusive UI and store compliance | `Semantics`, dynamic text (already respected, see §4), contrast, audited per screen |
| **State restoration** | survive OS process death mid-flow | `RestorationMixin` on the few flows that warrant it |
| **Background sync** | work that continues past app exit | `workmanager` behind a domain-level scheduler contract |

Adopting any of these bolts onto the existing seams: a new `Provider`, a source behind a
repository, a component in `presentation/`. That is the point of the architecture. Production
concerns are additions, never rewrites.