<h1 align="center">My Flutter Way</h1>
<p align="center"><em><strong>Flutter Production Playbook</strong></em></p>

<p align="center">
  Feature-first over the official Flutter architecture &middot; Typed networking &middot;
  Riverpod notifiers &middot; <code>Result&lt;T&gt;</code> in data, <code>DataState&lt;T&gt;</code> in UI &middot;
  Session safety &middot; Testing &middot; CI-enforced boundaries
  <br />
  <em>My conventions for building production-grade Flutter apps.</em>
</p>

<p align="center">
  <img alt="Flutter" src="https://img.shields.io/badge/Flutter-02569B?logo=flutter&logoColor=white" />
  <img alt="Dart 3" src="https://img.shields.io/badge/Dart-3.x-0175C2?logo=dart&logoColor=white" />
  <img alt="Riverpod" src="https://img.shields.io/badge/State-Riverpod%203-4B32C3" />
  <img alt="Architecture" src="https://img.shields.io/badge/Architecture-Feature--first%20%2B%20Layered-2E7D32" />
  <img alt="Lints" src="https://img.shields.io/badge/Lints-Native%20analyzer%20plugins-EF6C00" />
  <img alt="Version" src="https://img.shields.io/badge/Playbook-2.0.0--rc.3-555555" />
</p>

<p align="center">
  <b>Author</b> : <b>JOHN DAVE BAÑAS</b>
</p>

---

This is how I structure and write Flutter apps: not just the architecture, but the
conventions, error handling, session lifecycle, tests, and lints that keep it honest. Follow it
when scaffolding a new app, adding a feature, or reviewing code.

It is deliberately built **on top of the official Flutter architecture guidance**, not as an
alternative to it. Section 0 maps every piece of this document onto the Flutter team's
vocabulary and states the two places I diverge, with reasons. Everything else is an
implementation of that guidance with a specific stack.

> **Status of this document.** Version `2.0.0-rc.3`. A version loses its `-rc` suffix only when
> the reference application (§12.4) passes its CI pipeline (§11.2) on GitHub at the pinned
> versions (§12.3), including the Android emulator and iOS simulator jobs. This version has been
> executed against the reference app from a clean clone: static analysis, both analyzer plugins,
> the lint fixture, 98 tests, the coverage floors, all 29 sabotages, the feature template, the
> release-config check, the OSV scan, and prod and dev Android builds. §12.6 records what ran,
> on what, and what has not run yet.

> **Status of the code samples.** Snippets are extracts from the reference implementation, with
> imports omitted. Where a block shows a whole file it is that file, byte for byte; a
> `/* ... */` or a trailing comment marks an elision. Configuration blocks
> (YAML, Gradle, manifest, workflow) and the scripts under `tool/` are given in full, because
> half a config is worse than none. Anything illustrative rather than real is labelled
> **conceptual** inline.

## Contents

| | |
| --- | --- |
| [Quick start](#quick-start) | Adopt it in a new app, add a feature, review a PR |
| [Rule index](#rule-index) | Every rule on one page: mandatory, conditional, or optional, and what enforces it |
| [0. Relationship to the Official Flutter Architecture](#0-relationship-to-the-official-flutter-architecture) | The mapping onto the official guidance, and the two deviations |
| [1. Architecture Philosophy](#1-architecture-philosophy) | Why feature-first, and what the boundaries buy |
| [2. Folder Structure](#2-folder-structure) | Where every file goes, at two scales |
| [3. Core Layer](#3-core-layer) | Config, `Result` vs `DataState`, errors, the session, the auth interceptor |
| [4. Common Layer](#4-common-layer) | Shared resources, error copy, global error hooks, wiring |
| [5. Feature Anatomy](#5-feature-anatomy-worked-example-auth) | One vertical slice end to end |
| [6. State Management with Riverpod](#6-state-management-with-riverpod) | Provider graph, notifier conventions, lifecycle |
| [7. Testing](#7-testing) | The test matrix, coverage policy, and the tests that pin the invariants |
| [8. Tooling and Enforcement](#8-tooling-and-enforcement) | What a machine checks so review does not have to |
| [9. Naming and File Conventions](#9-naming-and-file-conventions) | File, class, and provider naming |
| [10. New Feature Checklist](#10-new-feature-checklist) | The order to build a slice in, and the rules that never bend |
| [11. Production and Release](#11-production-and-release) | Flavors, CI, hardening, readiness gates, rollout and incidents |
| [12. Governance](#12-governance) | ADRs, exceptions, dependency policy, reference app, versioning, verification record |

**Reading order.** New to the document: Quick start, the Rule index, then §1 and §5. Implementing
it: §3 (core), §10 (checklist), §8 (lints), §11.2 (CI). Reviewing a PR: the Rule index. Changing
the playbook itself: §12.

---

## Quick start

**A new app (about an hour).** The reference app (§12.4) is the starter repository: it passes
every check in §11.2 as shipped.

1. Copy it, rename the package in `pubspec.yaml`, and replace `dev.jdxve.reference_app` in `android/app/build.gradle.kts`. Keep `features/auth`; delete `profile` and `orders` when you have your own.
2. Point `config/dev.json`, `staging.json` and `prod.json` at your backend. Staging and prod must be `https`; the release check refuses anything else.
3. Confirm the backend contract the client relies on: authentication runs before any handler, `/auth/refresh` returns 400/401/403 only as a verdict, and writes that carry `Idempotency-Key` are deduplicated (§5).
4. Add each capability with `dart run tool/new_feature.dart <snake_case_name>`. It writes the domain, data, root file, notifier, screen and their tests (§10).
5. Run what CI runs before you push:

```bash
flutter pub get && (cd tool/lints && dart pub get) && (cd tool/lints/fixture && dart pub get)
dart analyze --fatal-infos                 # at the root, no path: the only form that runs the plugins
dart run tool/check_lint_fixture.dart      # the plugins fire, and your app loads them
dart run tool/check_architecture.dart
dart run tool/check_release_config.dart config/staging.json config/prod.json
tool/gen_coverage_helper.sh && flutter test --coverage && dart run tool/check_coverage.dart
dart run tool/check_sabotage.dart
```

6. Turn on branch protection for the four CI jobs (§11.2), then work through the device checks in §11.4 before the first release.

**A three-screen app.** Start in the small-app tier (§2): same `core/`, no `features/` folders yet.
The Rule index says which rules wait until the first feature folder exists.

**A new feature in an existing app.** `dart run tool/new_feature.dart <name>`, then §10 from
step 2.

**Reviewing a PR.** Read the Rule index. Rules enforced by a lint, a script or a test have already
been checked by CI; spend review time on the rows marked *review*.

---

## Rule index

**Mandatory** applies to every app at every size. **Conditional** applies once its trigger is
true, and then it is mandatory. **Optional** is a recommendation. "Review" means no machine
checks it: those are the rows a reviewer owns.

| # | Rule | Level | Enforced by | § |
| --- | --- | --- | --- | --- |
| 1 | `presentation/` and `data/` of a feature meet only at `domain/` | Mandatory, from the first `features/` folder | `layer_boundary` | 1 |
| 2 | Another feature is reachable only through its `domain/` and root file | Mandatory, from the first `features/` folder | `no_cross_feature_imports` | 1 |
| 3 | `core/` and `common/` never import a feature | Mandatory | `shared_layer_independence` | 1 |
| 4 | `domain/` imports only its allowlist | Mandatory | `domain_purity` | 1 |
| 5 | No feature dependency cycles | Mandatory | `check_architecture.dart` | 1, 8 |
| 6 | An architecture suppression cites an ADR on the line above | Mandatory | `check_architecture.dart` | 12.2 |
| 7 | `package:` imports only inside `lib/` | Mandatory | `always_use_package_imports` | 9 |
| 8 | Repositories return `Result<T>`; only notifiers hold `DataState<T>` | Mandatory | types; review | 3 |
| 9 | An `AppException` becomes copy only in `errorToMessage` | Mandatory | exhaustive switch (compiler) | 4 |
| 10 | Server copy is shown only for `rejected`, one line, bounded | Mandatory | mapper test, sabotage | 3 |
| 11 | Only `SessionStore` touches credentials | Mandatory | review | 3 |
| 12 | Session: one key, memory first, scoped recovery, `resetOnError: false` | Mandatory | session suite, sabotage | 3 |
| 13 | Only a server verdict ends a session | Mandatory | interceptor and refresher suites, sabotage | 3 |
| 14 | Nothing is replayed unless sending it twice is safe | Mandatory | interceptor suite, sabotage | 3 |
| 15 | Body-authenticated endpoints carry `kSkipAuth` | Mandatory | repository test; review | 5 |
| 16 | A write that must survive a refresh carries one `Idempotency-Key` per user intent | Conditional: non-idempotent writes | notifier test, sabotage; backend contract | 5 |
| 17 | `if (!ref.mounted) return;` after every `await` in a notifier | Mandatory | review | 5 |
| 18 | Pattern 1 by default; Pattern 2 and 3 only past their gates | Conditional | review | 5 |
| 19 | Every data screen renders loading, success, empty, and error with retry | Mandatory | widget tests | 5, 7.8 |
| 20 | Accessibility guidelines in both themes; no overflow at 2.0x text | Mandatory | widget tests | 4, 7.8 |
| 21 | Text scaling is clamped, never disabled | Mandatory | `App.builder`; review | 4 |
| 22 | All copy in `Strings`; colors via `context`; sizes via `Dimensions` | Mandatory | review | 9 |
| 23 | Every invariant test names a sabotage that `check_sabotage.dart` applies | Mandatory | `check_sabotage.dart` | 7.3 |
| 24 | Coverage: 80% overall, 90% in `core/network` and `core/session` | Mandatory | `check_coverage.dart` | 7.2 |
| 25 | The critical auth flow and one business flow run headless and on devices | Mandatory | flow tests, `android` and `ios` jobs | 7.9 |
| 26 | `AppConfig` refuses a bad environment in every build mode | Mandatory | `AppConfig` tests, `check_release_config.dart` | 3, 11.1 |
| 27 | The secure store is excluded from backup and device transfer | Mandatory | `check_release_config.dart` | 3 |
| 28 | CI analyzes at the root and proves both plugins loaded | Mandatory | `check_lint_fixture.dart` | 8 |
| 29 | Generated code is committed and current | Mandatory | CI `git status --porcelain` | 11.2 |
| 30 | Feature-first folders | Conditional: past about four capabilities, or parallel developers | review | 2 |
| 31 | A local source and cache behind the repository | Conditional: data reused across screens or sessions | repository tests | 5 |
| 32 | `go_router` | Conditional: any trigger in §4 | review | 4 |
| 33 | Certificate pinning, with a backup pin | Conditional: a high-value target | review | 11.3 |
| 34 | The requirements in §11.6 (pagination, offline, l10n, profiling, ...) | Conditional: each has its trigger | as listed there | 11.6 |
| 35 | `flutter_screenutil` behind `Dimensions` | Optional | ADR-008 | 4 |
| 36 | Golden tests | Optional | CI OS only | 7.8 |

---

## Stack

- **State / notifiers**: `flutter_riverpod` (v3). `Notifier` / `AsyncNotifier` + `NotifierProvider`. No BLoC, no GetX.
- **Networking**: `dio` + `retrofit`. Retrofit for typed calls; raw Dio for nullable, 204, or dynamic-shape endpoints.
- **Data-layer result**: hand-written sealed `Result<T>` (`Ok` / `Err`). Two cases, because a completed call has exactly two outcomes.
- **UI state**: `freezed` sealed union `DataState<T>` (idle / loading / success / error). Owned by notifiers only. Freezed is used **only** for `DataState` and error envelopes; DTOs, domain models, and composite notifier state are hand-written plain Dart.
- **Session**: `flutter_secure_storage` behind a single `SessionStore` (§3) that is the only code allowed to touch credentials. `shared_preferences` for preferences and cache.
- **Idempotency**: `uuid` for `Idempotency-Key` headers on non-idempotent writes (§3, §5).
- **Sizing**: `flutter_screenutil` behind a `Dimensions` facade. A default, not an invariant (see §4).
- **Theming**: `ThemeExtension<AppColors>` plus a `BuildContext` extension for ergonomic color access.
- **Testing**: `flutter_test` + `mocktail` (no codegen, matching the hand-written ethos), `integration_test` for the same critical flow on a device. DI is exercised through `ProviderContainer` with providers overridden. Never a real Dio, real keystore, or real socket.
- **Enforcement**: native analyzer plugins (Dart 3.10+): a local `app_lints` plugin for the layering rules and `riverpod_lint`. Five Dart scripts under `tool/` do what a per-file linter cannot: prove the plugins load, find feature cycles, check release config, enforce coverage floors, and prove each invariant test can fail.

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
| **Repository** | `data/repository/<Feature>RepositoryImpl`, wired in `features/<f>/<f>.dart` | Single source of truth. Owns caching and retry. |
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
they save. Recorded as ADR-001 (§12.1).

**2. Riverpod `Notifier` instead of `ChangeNotifier`.** Riverpod covers both the UI-state role
and dependency injection with one tool, which removes a second DI mechanism from the stack.
`ChangeNotifier` plus `provider` would satisfy the guide equally well; this is a stack
preference, and the boundaries it enforces are stricter, not looser. Recorded as ADR-002.

### One thing I explicitly adopt from the guide

The guide separates **the result of a data operation** from **the state of a screen**. A
repository returns success or failure. "Loading" and "idle" are things a *Notifier* knows, not
things a finished network call can be. One four-case `DataState<T>` used everywhere means
every repository signature advertises two outcomes it can never produce, so the two are split:
`Result<T>` below the repository boundary, `DataState<T>` above it. See §3 and ADR-003.

---

## 1. Architecture Philosophy

**Mental model:** feature-first on the outside, layered on the inside.
The app is a set of self-contained features. Each feature owns its full vertical slice
(`data`, `domain`, `presentation`). Cross-cutting infrastructure lives in `core`;
reusable UI and helpers live in `common`.

**Why feature-first:** a feature is the unit of work. Everything needed to build, test, or
delete a feature sits in one folder. You never hunt across top-level `models/`, `services/`,
`screens/` buckets.

**Why layering inside a feature:** the UI must not know how data is fetched, and the data
layer must not know how data is displayed. The `domain` layer (plain Dart models plus an
abstract repository interface) is the contract between them. Presentation depends on the
domain interface, never on Dio, Retrofit, or DTOs.

**Dependency direction (always inward):**

```
presentation      ->  domain   <-  data
(View + Notifier)   (contract)   (Repository + Service)
```

Each rule below names the lint or script that enforces it (§8). A rule without an enforcer is
a convention, and this document says so where that is the case.

- `presentation` depends on `domain` (models plus the abstract repository), on its feature's root file, and on `core` / `common`. It never imports its own feature's `data/`. *Enforced by `layer_boundary`.*
- `data` depends on `domain` (it implements the interface and maps DTOs to domain models). It never imports `presentation/`. *Enforced by `layer_boundary`.*
- **One file per feature sees both sides**: `features/<f>/<f>.dart`, the feature's public API. It wires the `data/` implementation to the `domain/` contract as a provider typed by the interface, and exports what other features may use (ADR-009).
- `domain` depends only on `core/result/` (`Result`, `AppException`), other features' `domain/`, and a short allowlist of pure Dart libraries. No Flutter, no Dio, no `dart:io`. *Enforced by `domain_purity`.*
- A feature may import another feature's `domain/` and its root file, and nothing else of it. `data/` and `presentation/` are internals. *Enforced by `no_cross_feature_imports`.*
- `core/` and `common/` never import a feature. *Enforced by `shared_layer_independence`.*
- The dependency graph between features stays **acyclic**. If `orders` needs `billing` and `billing` needs `orders`, the shared concept belongs to one of them, or to a third feature, or to `common/`. *Enforced by `tool/check_architecture.dart`.*

**On the word "pure".** The domain layer is **platform-independent and transport-independent**:
no Flutter, no Dio, no HTTP, no `dart:io`. It is not dependency-free, and I do not claim it is.
It depends on two project-level types, `Result` and `AppException`, which live under
`core/result/` precisely because they are not network types. If you need a genuinely
dependency-free domain, extract it to its own Dart package; for an app of this size that is
ceremony without a payoff.

**What goes where:**

- **`core/`**: app-wide infrastructure with no UI. Config, the session, the HTTP client factory and interceptors, `Result`, `AppException`, the repository guard, theme state, storage keys, launch bootstrap. If two unrelated apps could share it verbatim, it is probably core.
- **`common/`**: shared presentation-side building blocks. Reusable widgets, resources (theme, strings, dimensions, assets), UI utils and validators. Depends on `core`; never on a feature.
- **`features/<name>/`**: one product capability, full vertical slice. Depends on `core`, `common`, and other features' `domain/` and root file only.
- **`features/<name>/<name>.dart`**: that feature's public API and wiring. Repository providers live here, typed by the domain interface, so no `presentation/` file ever sees an `Impl` (ADR-009). In rc.1 they lived in `data/repository/`, which every notifier imported: the playbook broke its own `layer_boundary` rule in its own worked example, and nothing noticed because the rules had never run.

**Views are minimal.** A screen composes components and wires notifiers. It holds no
business logic and no inline widget tree of significance. The guide allows a view to own
purely visual logic: animation state, layout decisions based on screen size or orientation,
and simple routing. Everything touching data belongs in the notifier.

---

## 2. Folder Structure

### The standard tier (what this document describes)

```
lib/
  main.dart              # bootstrap: config, reporter, error hooks, launch init, theme preload, runApp
  main_shell.dart        # bottom-nav shell (IndexedStack) + tab-state notifier
  router.dart            # onGenerateRoute, named routes, global navigatorKey, transitions

  core/                          # app-wide infrastructure, no UI
    config/
      app_config.dart            # AppConfig.fromEnvironment(): validated in every build mode
    result/                      # pure Dart: imports nothing but dart:core
      result.dart                # Result<T>: Ok<T> | Err<T>  (data-layer outcome)
      app_exception.dart         # AppException + ErrorType: no Dio, no Flutter, no HTTP
    reporting/
      reporter.dart              # crash-reporter facade + NoopReporter
    session/                     # the ONLY code that touches credentials
      secure_storage.dart        # secureStorageProvider (platform options)
      session_store.dart         # SessionStore: in-memory session, write-through, scoped recovery
      storage_keys.dart          # kSession (one key), legacy keys, kSessionStorageKeys
    network/                     # everything that knows about Dio lives here
      http_client.dart           # buildDio(), appConfigProvider, httpAdapterFactoryProvider
      request_flags.dart         # kSkipAuth, kIdempotencyKeyHeader
      dio_error_mapper.dart      # mapDioException(): the one DioException -> AppException site
      repository_guard.dart      # RepositoryGuard mixin: guard() wraps a call into Result
      token_refresher.dart       # TokenRefresher + RefreshOutcome (Refreshed | Rejected | Unavailable)
      auth_interceptor.dart      # bearer inject, single-flight refresh, session epochs, replay policy
      logging_interceptor.dart   # pretty request/response logs (debug only, redacts auth)
      reporting_interceptor.dart # forwards actionable failures, redacted, to the crash reporter
      dio_providers.dart         # SessionExpiry, authInterceptorProvider, dioProvider
      models/
        error_response.dart      # freezed error envelope
    state/
      data_state.dart            # DataState<T> freezed union (UI state, notifiers only)
    theme/
      theme_notifier.dart        # ThemeSettings + ThemeNotifier (persisted light/dark/system)
    utils/
      launch_utils.dart          # first-install wipe, onboarding flags, prefs cache keys

  common/                        # shared presentation building blocks; never imports features/
    models/                      # tiny cross-feature value types (e.g. SimpleOption)
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
      <feature>.dart             # public API: repository provider (typed by the interface) + exports
      data/
        dto/                     # request/response DTOs: fromJson + toJson + toDomain()
        services/                # Retrofit @RestApi abstract class (+ .g.dart)
        source/                  # local source: TTL cache, DAO, prefs wrapper
        repository/              # <Feature>RepositoryImpl (wired in <feature>.dart)
      domain/
        models/                  # plain Dart domain models
        repository/              # abstract interface class <Feature>Repository
      presentation/
        notifier/                # Notifier<State> + NotifierProvider
        screens/                 # thin view screens (routeName lives here)
        components/              # feature-scoped widgets
    auth/
      presentation/
        auth_guard.dart          # route gate; lives here because it reads auth state (see §4)
```

Note `core/result/`, `core/state/`, and `core/session/`. Filing `Result` and `AppException`
under `core/network/` is what makes the domain layer *look* like it depends on infrastructure.
They are transport-neutral types, so they sit where that is obvious from the import line.
Credentials get their own folder because exactly one class may touch them, and a folder with
one door is easy to review.

Small or leaf features (splash, onboarding) may collapse to `screens/` and `components/` only.
A feature with no networking may omit `data/` and `domain/` entirely.

### Outside `lib/`

```
.github/workflows/ci.yaml # verify, security, android, ios (§11.2)
config/                   # dev.json, staging.json, prod.json (--dart-define-from-file)
docs/adr/                 # architecture decision records (§12.1)
integration_test/         # the critical flow on a device; same body as test/flows/
test/
  helpers/pump_app.dart   # ProviderScope + ScreenUtilInit + theme wrapper for widget tests
  flows/                  # the critical flow headless, against a scripted backend (§7.9)
  core/ common/ features/ # mirror lib/
tool/
  lints/                  # app_lints: native analyzer plugin (lib/main.dart)
    fixture/              # a tiny package whose imports prove each rule fires, and only where it should
  check_lint_fixture.dart     # fixture expectations + a canary proving the app's own config loads both plugins
  check_architecture.dart     # feature cycles + justified suppressions
  check_release_config.dart   # release defines, Android manifest, backup rules
  check_coverage.dart         # coverage floors, read from lcov.info
  check_sabotage.dart         # applies every named sabotage; fails if its test still passes
  new_feature.dart            # scaffolds a feature from templates/; --check renders and tests it in CI
  templates/feature/          # the feature template (.tmpl files: not analyzed until rendered)
  gen_coverage_helper.sh      # makes untested files count as 0%, not as absent
```

### The small-app tier (start here below roughly four features)

Full vertical slices are overhead on a three-screen app. Start layer-first inside one package
and promote later. The layering rules are identical; only the folders differ.

```
lib/
  core/           # config, result, session, network, theme, utils
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
boundary lints in §8 start paying for themselves at exactly this point, because cross-feature
imports only become possible once features exist. Everything in `core/`, including the session
and the interceptor, is identical in both tiers: the small tier is smaller in folders, not in
safety.

**What is mandatory, and what is proportional.** The rules that protect correctness and
credentials apply at every size. Everything else is adopted when the app has the problem it
solves.

| Always | When the requirement is real |
| --- | --- |
| `SessionStore`, `AuthInterceptor`, `TokenRefresher` as written | A local source and cache (§5): only if the data is reused across screens or sessions |
| `Result` in data, `DataState` in UI, `errorToMessage` | Pattern 2 and Pattern 3 notifiers (§5): only past their gates |
| `AppConfig` validation and `check_release_config` | `flutter_screenutil` (ADR-008): only for pixel-specified phone designs |
| The four boundary rules (from the first `features/` folder) and the plugin canary | More than one integration flow: only for a second flow whose failure is an incident |
| The interceptor and session suites, and `check_sabotage` over them | Golden tests, performance budgets beyond the defaults in §11.4 |
| Four states per data screen, 2.0x text scale, semantics | Feature root files: none needed in the small tier, which has one `data/` |

---
## 3. Core Layer

### Config (`core/config/app_config.dart`)

Values come from the build via `--dart-define-from-file` (§11.1). The same compiled code runs in
every environment and nothing sensitive is committed. Validation **throws in every build mode**,
release included: a release build is exactly where a wrong base URL costs the most, and an
`assert` would be stripped from it.

```dart
enum AppEnv { dev, staging, prod }

const String _devBaseUrl = 'http://192.168.1.100:4000/api/v1'; // local only, dev only

final class AppConfig {
  const AppConfig({required this.env, required this.baseUrl});

  final AppEnv env;
  final Uri baseUrl;

  /// Reads the build's defines and refuses to start on a bad combination.
  factory AppConfig.fromEnvironment() => AppConfig.parse(
    envName: const String.fromEnvironment('APP_ENV'),
    rawUrl: const String.fromEnvironment('BASE_URL'),
  );

  /// The validation itself, separated so tests can exercise every branch.
  factory AppConfig.parse({
    required String envName,
    required String rawUrl,
    bool debug = kDebugMode,
  }) {
    final env = switch (envName) {
      'dev' => AppEnv.dev,
      'staging' => AppEnv.staging,
      'prod' => AppEnv.prod,
      '' when debug => AppEnv.dev, // a bare `flutter run` while developing
      _ => throw StateError('APP_ENV must be dev, staging or prod; got "$envName".'),
    };

    final url = Uri.tryParse(rawUrl.isEmpty && env == AppEnv.dev ? _devBaseUrl : rawUrl);
    if (url == null || !url.hasAuthority) {
      throw StateError(
        'BASE_URL is missing or not absolute for $env. '
        'Pass it with --dart-define-from-file=config/${env.name}.json.',
      );
    }
    if (env != AppEnv.dev && url.scheme != 'https') {
      throw StateError('$env requires an https BASE_URL; got "$url".');
    }
    return AppConfig(env: env, baseUrl: url);
  }
}

const Duration kConnectTimeout = Duration(seconds: 10);
const Duration kSendTimeout = Duration(seconds: 30);
const Duration kReceiveTimeout = Duration(seconds: 30);
```

Two failure modes the 1.x version allowed and this one does not:

- **A prod build with no `BASE_URL` shipped pointing at a LAN address.** The old `defaultValue`
  was the dev URL and the guard was an `assert`, so a release build with a missing define
  compiled, passed the stripped assert, and talked to `192.168.1.100`. The dev URL is now used
  only when the environment is dev.
- **A typo in `APP_ENV` silently meant dev.** `'production'` fell through to the dev branch.
  An unknown name now throws.

`AppConfig.parse` holds the validation so a test can drive every branch with an explicit
`debug` flag; `fromEnvironment` only reads the defines. The runtime throw is the backstop. The
primary guard runs before a build exists:
`tool/check_release_config.dart` in CI (§11.2) rejects a staging or prod config file that would
fail here.

### Crash reporter (`core/reporting/reporter.dart`)

`reporter` appears throughout this document. It is a thin facade over Crashlytics or Sentry, so
that swapping vendors touches one file and so tests can substitute a no-op.

```dart
abstract interface class Reporter {
  void recordError(
    Object error,
    StackTrace? stack, {
    bool fatal = false,
    List<Object> information = const [],
  });
  void recordFlutterError(FlutterErrorDetails details);
}

final class NoopReporter implements Reporter {
  const NoopReporter();
  @override
  void recordError(
    Object error,
    StackTrace? stack, {
    bool fatal = false,
    List<Object> information = const [],
  }) {}
  @override
  void recordFlutterError(FlutterErrorDetails details) {}
}

/// Set once in main(), before the error hooks. Defaults to a no-op so tests and the
/// dev loop never need a real vendor SDK.
Reporter reporter = const NoopReporter();
```

A top-level mutable rather than a provider on purpose: `FlutterError.onError` and
`PlatformDispatcher.onError` are installed before any `ProviderScope` exists, so the reporter
has to be reachable without a `Ref`. It is the only global in this document, and it is
write-once at boot.

**What the reporter may receive.** No tokens, no headers, no request or response bodies, no
query strings. Network failures are reduced to a `NetworkFailure` value before they reach it
(see the reporting interceptor). Configure the vendor SDK's scrubbing hook (`beforeSend` in
Sentry, custom keys discipline in Crashlytics) as a second line, because a `FormatException`
can carry a fragment of the payload it failed to parse.

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
enum ErrorType {
  timeout, // the request took too long
  connection, // no route to the server
  cancelled, // the caller or the session abandoned the request; never shown to the user
  unauthorized, // the session is over: refresh was rejected
  forbidden, // signed in, not allowed (403)
  rejected, // the server refused this input: 400, 409, 422, or 401 on an unauthenticated endpoint
  retryable, // transient and safe to retry: token refreshed but not replayed, or 429
  server, // 5xx
  unknown,
}

class AppException implements Exception {
  const AppException({
    this.statusCode,
    this.message,
    this.type = ErrorType.unknown,
    this.cause,
    this.stackTrace,
  });

  final int? statusCode;
  final ErrorType type;

  /// Server-supplied copy that is safe to show a user, and nothing else. Populated only
  /// for ErrorType.rejected. Never a Dart error string, never an exception's toString().
  final String? message;

  /// Diagnostics only. Goes to the crash reporter and the logs; never to the UI.
  final Object? cause;
  final StackTrace? stackTrace;

  /// Non-transport failures (JSON shape changed, mapping threw, cache corrupt).
  /// `message` stays null on purpose: there is no user-safe sentence to put in it.
  factory AppException.unexpected(Object error, [StackTrace? stack]) =>
      AppException(type: ErrorType.unknown, cause: error, stackTrace: stack);

  @override
  String toString() => 'AppException($type, status: $statusCode, message: $message, cause: $cause)';
}
```

**The invariant that keeps internals off the screen.** `message` holds only text a server
wrote for a user, and only the mapper writes it, and only for `rejected`. Everything a Dart
runtime produced, a `TypeError` from a changed JSON shape, a `FormatException` from a corrupt
cache, goes in `cause`, which no UI path reads. Putting `error.toString()` into `message` is how
an app ends up showing someone `type 'Null' is not a subtype of type 'String' in type cast`
in a toast. The split makes the rule checkable: a reviewer only has to ask whether anything
other than `mapDioException` assigns `message`.

**429 is `retryable`.** A rate limit is transient and the user's next tap may succeed; rc.1 fell
through to `unknown`. **Why `ErrorType` grew from five cases to nine.** In 1.x a wrong password (401 from
`/auth/login`), a permission failure (403), and a refresh that succeeded but could not replay an
upload all rendered as "Your session has expired. Please log in again." Each was a different
truth told with the same sentence. The enum now has one case per thing the user should be told,
and `errorToMessage` (§4) switches on it exhaustively, so a tenth case is a compile error at the
one place copy is chosen.

### Request flags (`core/network/request_flags.dart`)

```dart
/// RequestOptions.extra key. True on endpoints that authenticate by their body
/// (login, register, refresh, revoke, password reset): no bearer is attached, and a
/// 401 is a verdict on the input, not on the session.
const String kSkipAuth = 'skipAuth';

/// Sent on non-idempotent writes that must survive a token refresh (§3 replay policy).
const String kIdempotencyKeyHeader = 'Idempotency-Key';
```

Retrofit applies them declaratively: `@Extra({kSkipAuth: true})` on a method, and
`@Header(kIdempotencyKeyHeader) String key` as a parameter. See §5.

### Dio error mapper (`core/network/dio_error_mapper.dart`)

The one place a Dio type becomes an `AppException`. In `core/network/`, not `core/result/`,
because knowing about Dio is a network-layer job. Swap the HTTP client and you replace this
file and nothing else.

```dart
AppException mapDioException(DioException e) {
  final status = e.response?.statusCode;
  final skipAuth = e.requestOptions.extra[kSkipAuth] == true;

  final type = e.error is ReplayRefused
      ? ErrorType.retryable
      : switch (e.type) {
          DioExceptionType.cancel => ErrorType.cancelled,
          DioExceptionType.connectionTimeout ||
          DioExceptionType.sendTimeout ||
          DioExceptionType.receiveTimeout => ErrorType.timeout,
          DioExceptionType.connectionError => ErrorType.connection,
          _ => switch (status) {
            401 when skipAuth => ErrorType.rejected, // wrong password: about the input
            401 => ErrorType.unauthorized,
            403 => ErrorType.forbidden,
            429 => ErrorType.retryable,
            400 || 409 || 422 => ErrorType.rejected,
            final int s when s >= 500 => ErrorType.server,
            _ => ErrorType.unknown,
          },
        };

  return AppException(
    statusCode: status,
    type: type,
    // Only a rejection carries server copy; every other type gets generic copy in the UI.
    message: type == ErrorType.rejected ? _extractServerMessage(e.response?.data) : null,
    cause: e,
    stackTrace: e.stackTrace,
  );
}

/// Longest server sentence the UI will show. Anything longer is not a sentence for a user.
const int kMaxServerMessage = 160;

/// Pulls the server's own message out of a JSON error envelope, if there is one.
/// It is untrusted input on its way to the screen: one line of plain text, bounded, or nothing.
String? _extractServerMessage(Object? data) {
  final raw = switch (data) {
    {'message': final String m} => m,
    {'error': {'message': final String m}} => m,
    _ => null,
  };
  if (raw == null) return null;
  final text = raw
      .replaceAll(RegExp(r'[\x00-\x1F\x7F\u2028\u2029\u202A-\u202E\u2066-\u2069]+'), ' ')
      .trim();
  if (text.isEmpty || text.length > kMaxServerMessage) return null;
  return text;
}
```

**Server copy is untrusted input.** It reaches the screen, so it is constrained on the way in:
control characters, line and paragraph separators, and bidirectional overrides become spaces
(a `U+202E` can make a sentence read backwards); an empty or over-long message is dropped, and
the user sees the generic `rejected` copy instead. Flutter's `Text` renders no markup, so this is
about spoofing and layout, not injection. When the backend can send stable error codes, map
those to local `Strings` and treat the free-text message as the fallback: a compromised or
careless backend can still write a misleading sentence that passes every check here.

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
      return Err(AppException.unexpected(e, stack));
    }
  }

  Result<T> mapError<T>(DioException error) => Err(mapDioException(error));
}
```

The guard sits in `core/network/` for the same reason the mapper does: it catches
`DioException`, so it is transport-aware by definition. `core/result/` stays free of Dio, which
is what lets the domain layer depend on it.

`onDioError` handles status codes that are not really failures, such as a 404 meaning "no data
yet" (illustrative; the reference app's profile repository in §5 does exactly this):

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

  /// The one conversion point between the data layer and the UI layer.
  /// A cancelled request returns the screen to idle: nothing failed that the user should hear about.
  static DataState<T> from<T>(Result<T> result) => switch (result) {
    Ok(:final value) => DataState.success(data: value),
    Err(exception: AppException(type: ErrorType.cancelled)) => DataState<T>.idle(),
    Err(:final exception) => DataState.error(exception: exception),
  };
}

extension DataStateX<T> on DataState<T> {
  bool get isLoading => this is DataLoading<T>;
  T? get valueOrNull => switch (this) {
    DataSuccess(:final data) => data,
    _ => null,
  };
}
```

`exception` is `required`, so `DataError` always carries one. `from` is a static on the freezed
class itself (rc.1 put it on an extension, where its `<T>` shadowed the extension's). It returns
`DataState<T>.idle()` with an explicit type argument: a bare `const DataState.idle()` would infer
`DataIdle<Never>`, which passes `is DataIdle<T>` checks by covariance and hides the mistake.

**The rule, stated precisely.** Repositories return `Result<T>`. Notifiers hold
`DataState<T>` (or a composite state class containing `DataState<T>` fields) and are the only
place `idle` and `loading` are produced. Local reads, fire-and-forget operations, and anything
whose failure the UI does not surface may return a plain `Future<T>` or `Future<void>`. Not
every async call needs a result wrapper; every *networked operation whose outcome the user
sees* does.

### The session (`core/session/`)

Credentials are the highest-stakes data in the app, so they get one owner. **`SessionStore` is
the only code that reads or writes session keys.** Repositories, interceptors, and notifiers go
through it; nothing else holds a `FlutterSecureStorage` reference except the first-install wipe.
Three properties fall out of that:

1. **The session is one value under one key.** Access and refresh token are written together as
   one JSON string, so a crash during a rotation cannot leave a new access token beside an
   already-consumed refresh token. rc.1 wrote two keys; a process death between the writes meant
   the next cold start signed in, presented a dead refresh token, and logged the user out.
2. **The token lives in memory for the process.** The interceptor reads it on every request
   from a field, not from the keystore. A keystore read is a platform-channel round trip and,
   on some Android devices, slow enough to show up in request latency.
3. **Ending a session cannot fail.** Memory is cleared first and synchronously. Whatever the
   keystore does afterwards, the session is over the moment `clear()` is called.
4. **Recovery is scoped, and it is ours.** On a storage failure the store deletes the session
   keys it owns, one at a time, and nothing else. It never calls `deleteAll()`, and the plugin is
   not allowed to erase on its behalf (below).

**Secure storage holds session credentials and nothing else.** Anything that must survive a
sign-out (onboarding flags, theme, cached profile) belongs in `shared_preferences`. This makes
"clear the session" and "clear what is in secure storage" the same operation, which is what
keeps recovery narrow.

#### Platform options (`core/session/secure_storage.dart`)

```dart
final secureStorageProvider = Provider<FlutterSecureStorage>((ref) {
  return const FlutterSecureStorage(
    aOptions: AndroidOptions(
      // The plugin default (true) deletes a key it cannot decrypt and reports it as absent.
      // SessionStore owns recovery and must see the failure, so the plugin may not erase.
      resetOnError: false,
      migrateWithBackup: true,
    ),
    // Readable after first unlock, and bound to THIS device. Plain `first_unlock`
    // items are included in encrypted backups and can restore onto a new phone.
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device),
  );
});
```

> **`resetOnError: false` is load-bearing.** In v11 the Android default is `true`: when the
> plugin cannot decrypt a key it deletes that key and reports it as absent. `SessionStore` then
> sees `SessionMissing` instead of `SessionUnreadable`, so the contract row "read throws:
> nothing deleted" was silently false on Android in rc.1. A test in the reference app pins
> the option (`secure_storage_test.dart`).
>
> **Version note.** `flutter_secure_storage` changed its Android backend in v10 and removed
> `encryptedSharedPreferences` in v11. Do not pass that parameter. Pin the major version, and
> re-check this block, the backup rules below, and the `SessionStore` tests whenever it moves
> (§12.3). `migrateWithBackup: true` makes the v9-to-v10 cipher migration crash-resistant and
> can be dropped once your install base has migrated.

**Android: keep the secure store out of backup and device transfer.** A restored or transferred
blob lands on a device whose Keystore cannot unwrap it, producing
`InvalidKeyException: Failed to unwrap key`. For apps targeting Android 12 (API 31) and above,
`allowBackup="false"` stops cloud backup but **not** device-to-device transfer; that needs
`dataExtractionRules`.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:allowBackup="false"
    android:dataExtractionRules="@xml/data_extraction_rules"
    ... >
```

```xml
<!-- android/app/src/main/res/xml/data_extraction_rules.xml -->
<?xml version="1.0" encoding="utf-8"?>
<!-- flutter_secure_storage 11 writes three SharedPreferences files: the encrypted data, the
     wrapped key that decrypts it, and its algorithm/config markers. A restored key file
     cannot be unwrapped by the new device's Keystore, so all three stay off backups and
     device transfers. Re-check these names whenever the plugin's major version moves. -->
<data-extraction-rules>
    <cloud-backup>
        <exclude domain="sharedpref" path="FlutterSecureStorage.xml" />
        <exclude domain="sharedpref" path="FlutterSecureKeyStorage.xml" />
        <exclude domain="sharedpref" path="FlutterSecureStorageConfiguration.xml" />
    </cloud-backup>
    <device-transfer>
        <exclude domain="sharedpref" path="FlutterSecureStorage.xml" />
        <exclude domain="sharedpref" path="FlutterSecureKeyStorage.xml" />
        <exclude domain="sharedpref" path="FlutterSecureStorageConfiguration.xml" />
    </device-transfer>
</data-extraction-rules>
```

rc.1 excluded only `FlutterSecureStorage.xml`. The wrapped AES key lives in
`FlutterSecureKeyStorage.xml`; restored onto a new device, it is exactly the blob the Keystore
cannot unwrap. The three names come from the plugin's Android source at the pinned version.
Both manifest attributes and all six exclusions are checked in CI by
`tool/check_release_config.dart`.

**iOS: the Keychain survives uninstall.** Keychain items are not deleted when the app is
removed, so a reinstall can silently resurrect a previous user's tokens. On the first launch
after install, detected via a `shared_preferences` flag (which *is* cleared on uninstall),
`launch_utils.dart` calls `deleteAll()` on secure storage before anything reads it. This is the
one sanctioned `deleteAll()`: it runs before any session exists, and secure storage holds only
session data by the rule above.

#### Storage keys (`core/session/storage_keys.dart`)

```dart
/// The whole session, access and refresh token together, as one JSON value.
/// One key means one write: a crash cannot leave a new access token beside a consumed
/// refresh token.
const String kSession = 'session';

/// The 1.x and 2.0.0-rc.1 layout. Read once to migrate, then deleted with the session.
const List<String> kLegacySessionKeys = ['access_token', 'refresh_token'];

/// Every key SessionStore owns. Recovery and sign-out delete exactly these.
const List<String> kSessionStorageKeys = [kSession, ...kLegacySessionKeys];
```

Per-user cache keys live in `shared_preferences` and are namespaced by user id
(`'cache_${userId}_profile'`) in `launch_utils.dart` or the owning local source.

#### SessionStore (`core/session/session_store.dart`)

```dart
final class SessionTokens {
  const SessionTokens({required this.accessToken, required this.refreshToken});

  /// Throws FormatException on anything that is not a complete session.
  factory SessionTokens.decode(String raw) => switch (jsonDecode(raw)) {
    {'a': final String a, 'r': final String r} => SessionTokens(accessToken: a, refreshToken: r),
    _ => throw const FormatException('stored session is incomplete'),
  };

  final String accessToken;
  final String refreshToken;

  String encode() => jsonEncode({'a': accessToken, 'r': refreshToken});
}

/// Cold-start outcome. "Never signed in" and "storage would not answer" are different
/// facts with different telemetry, even though both start the app signed out.
sealed class SessionLoad {
  const SessionLoad();
}

final class SessionFound extends SessionLoad {
  const SessionFound(this.tokens);
  final SessionTokens tokens;
}

final class SessionMissing extends SessionLoad {
  const SessionMissing();
}

final class SessionUnreadable extends SessionLoad {
  const SessionUnreadable(this.error);
  final Object error;
}

final sessionStoreProvider = Provider<SessionStore>(
  (ref) => SessionStore(ref.watch(secureStorageProvider)),
);

class SessionStore {
  SessionStore(this._storage);

  final FlutterSecureStorage _storage;
  SessionTokens? _tokens;
  bool _persisting = false; // false when "remember me" was off
  Future<void> _tail = Future.value();

  SessionTokens? get tokens => _tokens;
  String? get accessToken => _tokens?.accessToken;

  Future<SessionLoad> load() async {
    try {
      final raw = await _storage.read(key: kSession);
      final tokens = raw != null ? SessionTokens.decode(raw) : await _readLegacy();
      if (tokens == null) return const SessionMissing();
      _persisting = true;
      return SessionFound(_tokens = tokens);
    } catch (e, stack) {
      // Device still locked after a reboot (transient), a keystore migration, or a corrupt
      // blob. Report and start signed out. Do NOT delete: the first case recovers on its own,
      // and the next sign-in overwrites the key anyway.
      reporter.recordError(e, stack, fatal: false);
      return SessionUnreadable(e);
    }
  }

  /// Sign-in. Memory first, so the session works for this process even if the keystore does not.
  Future<void> save(SessionTokens tokens, {required bool persist}) {
    _tokens = tokens;
    _persisting = persist;
    return persist ? _serial(() => _persist(tokens)) : _serial(_deleteKeys);
  }

  /// Token rotation. Applies only if [from] is still the current session, so a refresh
  /// that finishes after a sign-out or an account switch cannot resurrect the old session.
  Future<bool> rotate({required SessionTokens from, required SessionTokens to}) async {
    if (!identical(_tokens, from)) return false;
    _tokens = to;
    if (_persisting) await _serial(() => _persist(to));
    return true;
  }

  /// Sign-out and expiry. The session is over when this is called; storage cleanup is
  /// best effort and never throws.
  Future<void> clear() {
    _tokens = null;
    _persisting = false;
    return _serial(_deleteKeys);
  }

  /// Storage operations run one at a time, in call order, so a rotation that is still
  /// writing can never land after the sign-out that followed it. The chain swallows and
  /// reports anything an operation lets escape: one failure must not skip every later op.
  Future<void> _serial(Future<void> Function() op) =>
      _tail = _tail.then((_) => op()).catchError((Object e, StackTrace stack) {
        reporter.recordError(e, stack, fatal: false);
      });

  /// Lets a test put an operation on the chain that escapes its own error handling.
  @visibleForTesting
  Future<void> debugEnqueue(Future<void> Function() op) => _serial(op);

  /// One-time migration from the two-key layout. Rewrites it as one key, then drops the old ones.
  Future<SessionTokens?> _readLegacy() async {
    final access = await _storage.read(key: kLegacySessionKeys[0]);
    final refresh = await _storage.read(key: kLegacySessionKeys[1]);
    if (access == null || refresh == null) return null;
    final tokens = SessionTokens(accessToken: access, refreshToken: refresh);
    await _serial(() async {
      await _persist(tokens);
      for (final key in kLegacySessionKeys) {
        await _deleteKey(key);
      }
    });
    return tokens;
  }

  Future<void> _persist(SessionTokens t) async {
    try {
      await _storage.write(key: kSession, value: t.encode());
    } catch (e, stack) {
      // A failing write usually means a key the keystore can no longer unwrap.
      // Recovery: delete our keys (and only ours), retry once.
      reporter.recordError(e, stack, fatal: false);
      await _deleteKeys();
      try {
        await _storage.write(key: kSession, value: t.encode());
      } catch (e, stack) {
        // Still failing. The session lives in memory for this process; next launch signs in again.
        reporter.recordError(e, stack, fatal: false);
      }
    }
  }

  Future<void> _deleteKeys() async {
    for (final key in kSessionStorageKeys) {
      await _deleteKey(key); // one failed delete must not skip the rest
    }
  }

  Future<void> _deleteKey(String key) async {
    try {
      await _storage.delete(key: key);
    } catch (e, stack) {
      reporter.recordError(e, stack, fatal: false);
    }
  }
}
```

**What changed from 1.x and why.** The old `safeRead` called `deleteAll()` on any read
exception. On iOS with first-unlock accessibility, a background launch before the user has
unlocked the phone after a reboot throws on read; `deleteAll()` turned that transient state into
a forced sign-out. It also erased keys that were not session data, and if `deleteAll()` itself
threw, the "guard" crashed the launch it existed to protect.

**What changed in rc.2.** One key instead of two, with a one-time migration of the old pair.
And `_serial` cannot be poisoned: in rc.1 it was `_tail = _tail.then(op)`, safe only while
every operation caught its own errors. One future edit that let an error escape would have
turned the chain into an errored future, and every later storage call, sign-out's delete
included, would have been skipped silently for the life of the process.

The table below is the contract the `SessionStore` tests in §7 pin, one test per row:

| Situation | Outcome |
| --- | --- |
| No session keys | `SessionMissing`, signed out, nothing reported |
| Read throws | `SessionUnreadable`, signed out, reported, **nothing deleted** |
| Stored blob is corrupt | `SessionUnreadable`, left for the next sign-in to overwrite |
| Rotation | exactly one storage write |
| Write throws | session keys deleted (only ours), write retried once, session continues in memory |
| Keystore never answers on sign-out | the session is over at the call; memory does not wait for storage |
| Delete throws | remaining keys still deleted, reported, `clear()` completes |
| An operation escapes its own handling | later operations still run |
| Refresh finishes after sign-out | `rotate` returns false, old session not restored |
| Remember me off | tokens in memory only; rotations not persisted; stale keys deleted |
| A 1.x two-key session on disk | read once, rewritten as one key, old keys deleted |

`debugEnqueue` is the one test seam: it is the only way to put an operation that escapes its own
handling onto the chain, which is the case the chain's `catchError` exists for.

### HTTP clients (`core/network/http_client.dart`)

Every Dio in the app, including the refresh client and the replay client, is built by one
function from one adapter factory. Transport policy (timeouts, certificate pinning, proxy
rules) therefore cannot diverge between the client a request starts on and the client that
replays it. In 1.x the refresh and replay clients were bare `Dio()` instances: pinning added to
the main client would have been silently bypassed by the two requests that carry the most
sensitive tokens.

```dart
/// Overridden in main() with AppConfig.fromEnvironment(), and in tests with a fixture.
final appConfigProvider = Provider<AppConfig>(
  (ref) => throw UnimplementedError('appConfigProvider must be overridden in main()'),
);

/// The single transport seam. Certificate pinning goes here (IOHttpClientAdapter's
/// validateCertificate / createHttpClient). Flow tests override it with a scripted backend.
final httpAdapterFactoryProvider = Provider<HttpClientAdapter Function()>(
  (ref) => IOHttpClientAdapter.new,
);

Dio buildDio(AppConfig config, HttpClientAdapter adapter) => Dio(
  BaseOptions(
    baseUrl: config.baseUrl.toString(),
    headers: {'Accept': 'application/json'},
    connectTimeout: kConnectTimeout,
    sendTimeout: kSendTimeout,
    receiveTimeout: kReceiveTimeout,
  ),
)..httpClientAdapter = adapter;
```

### Token refresher (`core/network/token_refresher.dart`)

Calls `/auth/refresh` on an interceptor-free client (a 401 from the refresh endpoint must not
re-enter the auth interceptor) and **classifies** the outcome. The 1.x refresher returned
`null` for every failure, so a timeout on the refresh call logged the user out. The rule now:
**only a server verdict on the refresh token ends a session; a network failure never does.**

```dart
sealed class RefreshOutcome {
  const RefreshOutcome();
}

final class Refreshed extends RefreshOutcome {
  const Refreshed(this.accessToken);
  final String accessToken;
}

/// The server rejected the refresh token, or there is no session to refresh. Ends the session.
final class RefreshRejected extends RefreshOutcome {
  const RefreshRejected();
}

/// Offline, timed out, 5xx, 429, or a malformed response. The session stays; the caller sees why.
final class RefreshUnavailable extends RefreshOutcome {
  const RefreshUnavailable(this.error);
  final DioException error;
}

class TokenRefresher {
  TokenRefresher(this._client, this._session);

  final Dio _client; // built by buildDio, no interceptors
  final SessionStore _session;

  static const _verdicts = {400, 401, 403};

  Future<RefreshOutcome> call() async {
    final current = _session.tokens;
    if (current == null) return const RefreshRejected();
    try {
      final res = await _client.post<Map<String, dynamic>>(
        '/auth/refresh',
        data: {'refreshToken': current.refreshToken},
        options: Options(extra: {kSkipAuth: true}),
      );
      final access = res.data?['accessToken'];
      if (access is! String) throw const FormatException('refresh response has no accessToken');
      final rotated = res.data?['refreshToken'];
      final next = SessionTokens(
        accessToken: access,
        refreshToken: rotated is String ? rotated : current.refreshToken,
      );
      if (!await _session.rotate(from: current, to: next)) return const RefreshRejected();
      return Refreshed(access);
    } on DioException catch (e) {
      return _verdicts.contains(e.response?.statusCode)
          ? const RefreshRejected()
          : RefreshUnavailable(e);
    } on FormatException catch (e, stack) {
      reporter.recordError(e, stack, fatal: false);
      return RefreshUnavailable(
        DioException(
          requestOptions: RequestOptions(path: '/auth/refresh'),
          error: e,
          stackTrace: stack,
        ),
      );
    }
  }
}
```

### Auth interceptor (`core/network/auth_interceptor.dart`)

Attaches the bearer on every request that does not opt out with `kSkipAuth`. On a 401 it
refreshes **once per token generation**, replays the original request if the replay policy
allows it, and expires the session **only** on a server verdict. Six invariants, each pinned by
at least one test in §7.7, and each of those tests is proven able to fail by
`tool/check_sabotage.dart`:

| # | Invariant | Mechanism |
| --- | --- | --- |
| 1 | N concurrent 401s cause exactly one refresh | token **generation** counter |
| 2 | A rejected refresh is attempted once and tears down once | **expire-once** latch that also bumps the epoch, and no refresh once the session is over |
| 3 | A request from an old session is never refreshed for or replayed | session **epoch**, checked before and after the refresh |
| 4 | A network failure never logs the user out | `RefreshUnavailable` propagates the real error |
| 5 | A replay's own failure is what the caller sees | the replay's `DioException` is forwarded, not the original 401 |
| 6 | Nothing is sent twice unless it is safe to send twice | **replay policy**: idempotent method or `Idempotency-Key`, reusable body |

> **Why a generation counter, not "QueuedInterceptor serializes".** `QueuedInterceptor` does
> serialize `onError` callbacks: the second 401 is not processed until the first finishes. It
> does **not** stop the second one from refreshing again, because that request was dispatched
> with the stale token and the interceptor has no memory of what just happened. Serialization
> alone gives N sequential refreshes for N concurrent 401s, which with a rotating refresh token
> means the second refresh presents an already-consumed token and logs the user out. Each
> request is stamped with the generation it was sent under; a request whose generation is
> behind the current one knows a sibling already refreshed, and reuses that token. Because
> `onError` is serialized, the counter needs no lock.
>
> **Why a session epoch as well.** The generation counter answers "has the token changed since
> this request left?" It cannot answer "is this still the same session?" In 1.x, signing in
> bumped the generation, so a request sent by user A that came back 401 after user B signed in
> looked like an ordinary stale-token case: it read the current token, which was B's, and
> replayed A's request with B's credentials. The epoch is bumped on every sign-in, sign-out, and
> expiry; a 401 from an older epoch is dropped as cancelled, silently, without a refresh and
> without a replay. The epoch is checked again after the refresh call returns, because a
> sign-out can happen while the refresh is in flight.
>
> **Why `_expired` replaced the failed-generation marker.** rc.1 had two mechanisms for
> invariant 2: a `_failedGeneration` check and the epoch bump in `_expireOnce`. Each covered the
> other, so its test passed with either one deleted, including the deletion its own comment
> named; reproduced against rc.1's code, the test stays green with its sabotage applied. rc.2
> keeps one mechanism per job, each with its own test: the epoch bump quiets siblings already in
> flight (they become `StaleSession`, one caller hears "session expired" instead of three), and
> `if (_expired)` stops a request sent *after* the teardown from refreshing again. The second
> also closes a gap rc.1 had: a request sent while signed out whose 401 arrived after the next
> user's `save` but before `startSession` would have refreshed with the new user's token.

```dart
/// Marker errors, carried in DioException.error so the mapper can tell them apart.
final class ReplayRefused implements Exception {
  const ReplayRefused();
}

final class StaleSession implements Exception {
  const StaleSession();
}

class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor({
    required this.readToken,
    required this.refresh,
    required this.onExpired,
    required this._retryClient,
  });

  final Future<String?> Function() readToken;
  final Future<RefreshOutcome> Function() refresh;
  final Future<void> Function() onExpired;
  final Dio _retryClient;

  // All mutable state is touched only inside serialized callbacks or synchronous
  // session calls, so none of it needs a lock.
  int _generation = 0; // token era: +1 per successful refresh
  int _epoch = 0; // session era: +1 per sign-in, sign-out, and expiry
  bool _expired = false; // the current session is over: no refresh, no second teardown

  static const _kGen = 'auth.gen';
  static const _kEpoch = 'auth.epoch';
  static const _kReplayed = 'auth.replayed';

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    if (options.extra[kSkipAuth] == true) return handler.next(options);
    final token = await readToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    options.extra[_kGen] = _generation;
    options.extra[_kEpoch] = _epoch;
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    final o = err.requestOptions;
    if (err.response?.statusCode != 401 || o.extra[kSkipAuth] == true) {
      return handler.next(err); // not ours: wrong password, 403, 500, offline...
    }

    // Invariant 3: sent under a session that has since ended or been replaced.
    if (o.extra[_kEpoch] != _epoch) return handler.next(_stale(o));

    // Invariant 2: this session already ended. Never refresh for it again.
    if (_expired) return handler.next(err);

    // A replay that 401s again: the fresh token itself is rejected.
    if (o.extra[_kReplayed] == true) {
      await _expireOnce();
      return handler.next(err);
    }

    final sentGen = o.extra[_kGen] as int? ?? _generation;
    final String token;

    if (sentGen < _generation) {
      // Invariant 1: a sibling already refreshed while this request was in flight.
      final current = await readToken();
      if (current == null) return handler.next(_stale(o));
      token = current;
    } else {
      final outcome = await refresh();
      // Invariant 3, again: the user may have signed out while the refresh was in flight.
      if (o.extra[_kEpoch] != _epoch) return handler.next(_stale(o));
      switch (outcome) {
        case Refreshed(:final accessToken):
          _generation++;
          token = accessToken;
        case RefreshRejected():
          await _expireOnce();
          return handler.next(err);
        case RefreshUnavailable(:final error):
          // Invariant 4: the session is still valid. Report what actually happened.
          return handler.next(error);
      }
    }

    // Invariant 6: refreshed and persisted, but this request cannot safely be sent twice.
    // Surface it as retryable, never as an expired session; the caller's own retry
    // carries the new token.
    if (!_canReplay(o)) {
      return handler.next(
        DioException(requestOptions: o, response: err.response, error: const ReplayRefused()),
      );
    }

    try {
      handler.resolve(await _retryClient.fetch(_withToken(o, token)));
    } on DioException catch (retryErr) {
      // Invariant 5: only a 401 on the replay means the fresh token is rejected.
      if (retryErr.response?.statusCode == 401) await _expireOnce();
      handler.next(retryErr);
    } catch (e, stack) {
      // A non-Dio throw must not escape the interceptor, and must not masquerade as a 401.
      handler.next(DioException(requestOptions: o, error: e, stackTrace: stack));
    }
  }

  /// Call from AuthStateNotifier.setAuthenticated. Opens a new session era.
  void startSession() {
    _epoch++;
    _generation++;
    _expired = false;
  }

  /// Call from AuthStateNotifier.signOut, before clearing the session.
  /// Everything in flight becomes stale; nothing is left to tear down.
  void endSession() {
    _epoch++;
    _expired = true;
  }

  Future<void> _expireOnce() async {
    if (_expired) return;
    _expired = true;
    _epoch++; // other in-flight 401s of this session go quiet instead of toasting N times
    try {
      await onExpired();
    } catch (e, stack) {
      reporter.recordError(e, stack, fatal: false);
    }
  }

  /// Safe to send twice: a body that can be re-read, and either an idempotent method or an
  /// Idempotency-Key the server deduplicates on.
  bool _canReplay(RequestOptions o) {
    if (o.data is FormData || o.data is Stream) return false;
    const idempotent = {'GET', 'HEAD', 'OPTIONS', 'PUT', 'DELETE'};
    return idempotent.contains(o.method.toUpperCase()) ||
        o.headers.keys.any((k) => k.toLowerCase() == kIdempotencyKeyHeader.toLowerCase());
  }

  DioException _stale(RequestOptions o) =>
      DioException.requestCancelled(requestOptions: o, reason: const StaleSession());

  RequestOptions _withToken(RequestOptions o, String token) => o.copyWith(
    headers: {...o.headers, 'Authorization': 'Bearer $token'},
    extra: {...o.extra, _kReplayed: true},
  );
}
```

**The replay policy, and the backend contract behind it.** A POST that returned 401 was, on a
correctly built backend, never executed: authentication runs before the handler. That is a
property of the server, not of the client, and the client cannot verify it. So the client
does not assume it. A request is replayed only when sending it twice is harmless on its own
terms: an idempotent method, or a non-idempotent write that carries an `Idempotency-Key` the
server deduplicates on. Everything else is refreshed but not replayed, and reaches the UI as
`ErrorType.retryable` ("Please try again."), which is true. This is ADR-005.

`FormData` and `Stream` bodies are never replayed whatever the method: they are
single-consumption, and re-sending one either throws or uploads an empty body. The order of the
checks is part of the policy. The replay check comes *after* the refresh, so the caller's
retry carries a valid token and fails at most once; placed before the refresh, an upload after
token expiry would fail twice. The multipart test in §7 pins this.

**Wiring (`core/network/dio_providers.dart`).** The interceptor is its own provider so the auth
notifier can reach `startSession` and `endSession`. Navigation is not its job: `onExpired`
clears the session and notifies `SessionExpiry`, the auth state flips to signed out, and the
route guard (§4) is the single authority that sends the user to the login screen. In 1.x both
the interceptor and the guard navigated, which double-pushed the login route. `SessionExpiry`
wraps the stream controller so consumers get a stream and one way to emit, never the
controller itself (and `close_sinks` is satisfied without a suppression).

```dart
/// Session-expiry events, from the interceptor to AuthStateNotifier. Exposes the stream
/// and one way to emit, not the controller.
final class SessionExpiry {
  final _controller = StreamController<void>.broadcast();

  Stream<void> get stream => _controller.stream;
  void notify() => _controller.add(null);
  Future<void> dispose() => _controller.close();
}

final sessionExpiredProvider = Provider<SessionExpiry>((ref) {
  final expiry = SessionExpiry();
  ref.onDispose(expiry.dispose);
  return expiry;
});

final authInterceptorProvider = Provider<AuthInterceptor>((ref) {
  final config = ref.watch(appConfigProvider);
  final newAdapter = ref.watch(httpAdapterFactoryProvider);
  final session = ref.watch(sessionStoreProvider);
  final expired = ref.watch(sessionExpiredProvider);

  return AuthInterceptor(
    readToken: () async => session.accessToken, // memory, not the keystore
    refresh: TokenRefresher(buildDio(config, newAdapter()), session).call,
    retryClient: buildDio(config, newAdapter()),
    onExpired: () async {
      await session.clear(); // never throws
      expired.notify();
    },
  );
});

final dioProvider = Provider<Dio>((ref) {
  final dio = buildDio(ref.watch(appConfigProvider), ref.watch(httpAdapterFactoryProvider)());
  dio.interceptors.addAll([
    if (kDebugMode) LoggerInterceptor(),
    ref.watch(authInterceptorProvider),
    ReportingInterceptor(), // last: sees only failures that survived refresh and replay
  ]);
  return dio;
});
```

**Proactive refresh beats reactive refresh.** Refreshing on a 401 costs the user one wasted
round trip per expiry. If your access token carries an `exp`, check it in `onRequest` and
refresh when it is within, say, thirty seconds of expiring. The 401 path then becomes the
fallback for clock skew and server-side revocation. The generation counter and the epoch are
what make adding this safe.

### Logging interceptor (`core/network/logging_interceptor.dart`)

Debug-only logging. Redacts `Authorization` headers and every `/auth/` body in both directions,
and a test asserts that neither a bearer nor a password reaches `debugPrint`. Registered only
under `kDebugMode`, so it cannot leak into a release logcat. It is registered before
`AuthInterceptor`, so it logs headers as the caller set them, before the bearer is attached.

### Reporting interceptor (`core/network/reporting_interceptor.dart`)

Registered after `AuthInterceptor`, so it only sees failures that survived refresh and replay.
It reports **actionable** failures only, and reports a redacted value, never the
`DioException` itself: a `DioException`'s string form can include the full URI with its query
string, which is where search terms, emails, and occasionally tokens end up.

```dart
/// What reaches the crash reporter: method, a templated path, status, and type.
/// Never headers, bodies, query strings, or raw ids.
final class NetworkFailure implements Exception {
  NetworkFailure(DioException e)
    : method = e.requestOptions.method,
      path = e.requestOptions.uri.path.replaceAll(_id, '/:id'),
      status = e.response?.statusCode,
      type = e.type;

  // Numeric ids and UUIDs become ':id', which also groups the dashboard by endpoint.
  static final _id = RegExp(r'/(\d+|[0-9a-fA-F]{8}-[0-9a-fA-F-]{27})(?=/|$)');

  final String method;
  final String path;
  final int? status;
  final DioExceptionType type;

  @override
  String toString() => 'NetworkFailure($method $path, status: $status, type: ${type.name})';
}

class ReportingInterceptor extends Interceptor {
  @override
  void onError(DioException err, ErrorInterceptorHandler handler) {
    if (_actionable(err)) {
      reporter.recordError(NetworkFailure(err), err.stackTrace, fatal: false);
    }
    handler.next(err);
  }

  /// Offline, timeouts, cancellations, session expiry and 4xx verdicts are expected
  /// outcomes and stay off the dashboard. 5xx, certificate failures, and unknowns do not.
  static bool _actionable(DioException e) {
    final status = e.response?.statusCode;
    if (status != null) return status >= 500;
    return e.type == DioExceptionType.badCertificate || e.type == DioExceptionType.unknown;
  }
}
```

The rule: the user sees a message, and we see the stack trace. Neither substitutes for the
other, and network reporting happens in exactly one place rather than sprinkled through
repositories. Timeouts and offline rates are a metrics question, not a crash question; track
them as analytics events if you need them (§11.6).

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
### Threat model and limits

What the session and networking design protects against, and the test that proves each one.
Every test named here is also in `check_sabotage.dart`, so none of them can pass vacuously.

| Scenario | Mechanism | Test |
| --- | --- | --- |
| User B signs in while user A's request is in flight | session epoch | "an old session's 401 is dropped, never replayed with the new user's token" |
| Sign-out while `/auth/refresh` is in flight | epoch rechecked after refresh; `rotate` identity check | "a sign-out during the refresh call...", "a refresh that finishes after sign-out..." |
| Several requests 401 at once | token generation | "two concurrent 401s trigger exactly one refresh" |
| Refresh token dead | expire-once latch; no refresh after teardown | "a dead refresh token tears down once", "a request sent after expiry..." |
| Offline or 5xx during refresh | `RefreshUnavailable` keeps the session | "a refresh that times out keeps the session" |
| App killed mid-rotation | one storage write per rotation | "rotation is one write" |
| Keystore read fails, write fails, delete fails, or hangs | the `SessionStore` contract table | one test per row |
| Previous install's iOS keychain on reinstall | first-launch wipe | "first launch after install wipes a keychain..." |
| Tokens leave the device in a backup | backup exclusions; `this_device` accessibility | `check_release_config.dart`, `secure_storage_test.dart` |
| A non-idempotent write sent twice by a replay | replay policy | "a POST without an idempotency key is refreshed but not replayed" |
| Tokens or personal data in logs and crash reports | redaction | "the debug logger never prints a bearer...", "a 5xx is reported as a redacted NetworkFailure" |
| Spoofed, multi-line or oversized server copy | sanitization | "server copy is constrained before it can reach the screen" |

**What it does not protect against.** State these to anyone who relies on the design:

- **The device owner, or malware with root or accessibility rights.** Keystore and Keychain raise the cost of reading tokens; they do not make them secret from a compromised device. The access token also lives in process memory for the session.
- **A network attacker the device already trusts.** TLS proves the server presents a certificate the device accepts; a user-installed or corporate CA passes that test. Certificate pinning (§11.3) narrows it, can itself be bypassed on a rooted device, and turns a certificate rotation into an outage if there is no backup pin.
- **Backup paths outside the rules.** The Android exclusions cover cloud backup and device-to-device transfer as the platform defines them, not OEM backup tools or a rooted filesystem. `this_device` keeps iOS items from restoring onto another device; it does not hide them from the device they are on.
- **Exactly-once execution.** The client sends an idempotency key and refuses unsafe replays; only the backend can deduplicate (§5). Without that, a retried write can still execute twice.
- **Stolen refresh tokens.** Rotation, reuse detection and revocation are server responsibilities. The client honors a rotated token; it cannot detect that someone else used the old one.
- **What the server says.** Sanitized server copy is still the server's words.

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

Every semantic pair (text on surface, text on primary, error on surface) meets WCAG AA contrast
(4.5:1 for body text) in both themes. The widget tests in §7 check it with Flutter's
`textContrastGuideline`, so a palette change that breaks contrast fails a test.

**`strings.dart`**: all user-facing copy as `static const String` on a `Strings` class. No
hardcoded strings in widgets, including error copy. Retire this for ARB or `slang` the moment a
second locale appears.

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
> accessibility failure in Flutter apps. If large text breaks a layout, fix the layout:
> `FittedBox`, `Flexible`, scrollable content. If you must bound it, clamp rather than disable:
>
> ```dart
> MediaQuery.withClampedTextScaling(
>   minScaleFactor: 1.0,
>   maxScaleFactor: 1.4,
>   child: child!,
> )
> ```
>
> Every screen has a widget test at 2.0x text scale that asserts no overflow (§7.8).

### UI utils (`common/utils/ui_utils.dart`)

Pure functions. `errorToMessage` is the single place an `AppException` becomes copy, plus
validators and formatters. No Dio types reach this layer.

```dart
String errorToMessage(Object? error) {
  if (error is! AppException) return Strings.errorGeneric;

  return switch (error.type) {
    ErrorType.timeout => Strings.errorTimeout,
    ErrorType.connection => Strings.errorOffline,
    ErrorType.unauthorized => Strings.errorSessionExpired,
    ErrorType.forbidden => Strings.errorForbidden,
    // The one case where the server wrote the sentence for the user.
    ErrorType.rejected => error.message ?? Strings.errorRejected,
    ErrorType.retryable => Strings.errorTryAgain,
    ErrorType.server => Strings.errorServer,
    // cancelled never reaches the UI (DataState.from maps it to idle); unknown carries
    // its real cause in `cause`, already reported.
    ErrorType.cancelled || ErrorType.unknown => Strings.errorGeneric,
  };
}
```

There is no `_` branch. A new `ErrorType` is a compile error here, which is the point: the
decision about what a user reads is made explicitly, in one place, for every case.

**The catch-all that used to leak.** 1.x ended with `_ => error.message ?? fallback`.
`AppException.unexpected` has no status code, so a mapping failure landed in exactly that
branch, and if anything had put `error.toString()` into `message` the user read a Dart type
error. Two fixes hold the line together: the mapper writes `message` only for `rejected`, and
this switch renders it only for `rejected`. Either alone leaves the hole open the next time
someone edits the other.

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
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Order matters. A bad config must crash loudly, so it runs BEFORE the hooks that would
  // otherwise swallow the StateError and leave the app hanging on the splash screen.
  final config = AppConfig.fromEnvironment();
  reporter = await createReporter(config);

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

  final container = ProviderContainer(overrides: [appConfigProvider.overrideWithValue(config)]);
  await initializeLaunch(container.read(secureStorageProvider)); // first-install wipe (§3)
  await container.read(themeProvider.notifier).load();

  runApp(UncontrolledProviderScope(container: container, child: const App()));
}

/// Vendor SDK initialization (Sentry, Crashlytics) goes here. The reference app ships none.
Future<Reporter> createReporter(AppConfig config) async => const NoopReporter();
```

These two hooks cover overlapping but not identical surfaces to `runZonedGuarded`. They are the
current recommendation for most apps and are what this playbook uses; `runZonedGuarded` is
still the answer when you need zone-scoped context such as per-request tracing, and some crash
reporters install their own zone. Do not run both mechanisms for the same errors and then
wonder about duplicate reports.

**What this does and does not give you.** Together with the reporting interceptor, the three
main application error surfaces (sync framework, async platform, network) route into one
dashboard. It does **not** capture native crashes (enable the vendor's native SDK for those),
OS process kills, or errors in code paths that deliberately swallow them. The honest claim is
that no *application-level* Dart error is invisible by default.

### Route guard (lives in `features/auth/presentation/auth_guard.dart`)

The guard reads `authStateProvider`, which belongs to the auth feature. In 1.x it sat in
`common/navigation/`, which made `common` depend on a feature: the exact arrow §1 forbids, and
one the `shared_layer_independence` lint now rejects. It belongs to auth; `router.dart`, which is
top-level and may import features, wraps protected routes with it.

It is the **single authority** for leaving a protected area. Session expiry, sign-out, and an
unreadable store all end the same way: `authStateProvider` becomes `signedOut`, the guard
rebuilds, and it redirects once.

```dart
class AuthGuard extends ConsumerWidget {
  const AuthGuard({required this.child, super.key});
  final Widget child;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final auth = ref.watch(authStateProvider);
    if (auth.isLoading && !auth.hasValue) return const _Blank(); // cold start only
    return auth.value == AuthStatus.signedIn ? child : const _RedirectToLogin();
  }
}

/// Redirects in initState, so a rebuild cannot push the login route twice.
class _RedirectToLogin extends StatefulWidget {
  const _RedirectToLogin();
  @override
  State<_RedirectToLogin> createState() => _RedirectToLoginState();
}

class _RedirectToLoginState extends State<_RedirectToLogin> {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (!mounted) return;
      Navigator.of(context).pushNamedAndRemoveUntil(LoginScreen.routeName, (_) => false);
    });
  }

  @override
  Widget build(BuildContext context) => const _Blank();
}

class _Blank extends StatelessWidget {
  const _Blank();
  @override
  Widget build(BuildContext context) => const Scaffold(body: SizedBox.shrink());
}
```

Wired in the router (`router.dart`):

```dart
Route<dynamic> generateRoute(RouteSettings settings) => switch (settings.name) {
  MainShell.routeName => _buildFadeRoute(settings, const AuthGuard(child: MainShell())),
  LoginScreen.routeName => _buildFadeRoute(settings, const LoginScreen()),
  _ => _buildFadeRoute(settings, const _NotFoundScreen()),
};
```

Every protected route is wrapped, including pushed detail routes. A detail route pushed on top
of a guarded shell is covered in practice (the shell's guard clears the stack), but "in
practice" is not a rule; wrap it.

### Top-level wiring

- **`main.dart`**: as above. Boots and hands a container to `App`.
- **`app.dart`**: `App` wraps `MaterialApp` in `ScreenUtilInit(designSize: Size(390, 844))`, reads `navigatorKeyProvider` and `themeProvider`, sets `onGenerateRoute: generateRoute`, and clamps text scaling to 1.0 to 2.0 in `builder`. Tests pump `App` directly with overrides, so the flow test (§7.9) exercises the real router and guard.
- **`main_shell.dart`**: bottom-nav shell with `IndexedStack` over the tab screens and a `Notifier<int>` tracking tab history, for back-navigation and double-back-to-exit.
- **`router.dart`**: declares `navigatorKeyProvider`, a named-route `switch` in `generateRoute`, custom `PageRouteBuilder` transitions, protected routes wrapped in `AuthGuard`, and a default case that shows a "page not found" screen (not a toast: a route that does not exist is a bug, and a screen is reportable).

### Navigator 1.0 by default, with explicit `go_router` triggers

This uses imperative Navigator 1.0 (`onGenerateRoute` plus `Navigator.pushNamed`) on purpose:
for a bottom-tab app with a handful of pushed detail screens it is less machinery, no codegen,
and no new mental model (ADR-004). **Migrate to `go_router` the moment any one of these becomes
true**, not before:

- **Deep links or universal links.** The OS hands you a URL (`myapp://order/42`) that must resolve to a specific screen stack.
- **Web or URL-addressable routes.** You ship a Flutter Web target and the address bar must reflect and drive navigation.
- **Nested or persistent shells.** Per-tab navigation stacks that survive tab switches, with a persistent bottom bar over nested routes.
- **Centralized declarative redirects.** Auth gating that must run as a route-level policy, once redirect logic sprawls across many routes and needs one authority.

When you migrate, the seams already exist: `navigatorKeyProvider` maps onto `go_router`'s
`navigatorKey`, `AuthGuard` maps onto a top-level `redirect` that reads `authStateProvider`, and
route names stay `static const String routeName` on the screens.

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
Service (Retrofit) + local source (cache) + SessionStore (auth only)
```

### data/services (Retrofit)

Typed API surface. Abstract `@RestApi` class, `part '<name>.g.dart'` generated by build_runner.
One service per resource. Stateless: it wraps the endpoint and returns DTOs, nothing else.

```dart
@RestApi()
abstract class AuthApiService {
  factory AuthApiService(Dio dio, {String? baseUrl}) = _AuthApiService;

  /// Authenticates by body. A 401 here means wrong credentials, not an expired session.
  @POST('/auth/login')
  @Extra({kSkipAuth: true})
  Future<LoginResponseDto> login(@Body() LoginRequestDto dto);

  /// Revokes the refresh token server-side. Sent after the local session is already gone.
  @POST('/auth/revoke')
  @Extra({kSkipAuth: true})
  Future<void> revoke(@Body() RevokeRequestDto dto);
}
```

Every endpoint that authenticates by its body (login, register, refresh, revoke, password
reset) carries `kSkipAuth`. Forgetting it on login is how a wrong password becomes "Your
session has expired", which is why §10 lists it.

`baseUrl` is `String?`: the factory must match the constructor retrofit 4.10 generates.

**Non-idempotent writes carry an idempotency key** so they can be replayed safely after a
refresh (§3), and so a user's double submit or a flaky network does not create two orders:

```dart
  @POST('/orders')
  Future<OrderDto> createOrder(
    @Body() CreateOrderDto dto,
    @Header(kIdempotencyKeyHeader) String idempotencyKey,
  );
```

The key is generated once per user intent and reused if that same intent is retried. A new key
per HTTP attempt defeats the purpose. The notifier owns the intent, so the notifier owns the
key; two tests pin it (a retry after a failure reuses the key, a success retires it):

```dart
class PlaceOrderNotifier extends Notifier<DataState<Order>> {
  /// One key per user intent. A retry after a failure is the same intent and reuses it;
  /// only a success (or leaving the screen) starts a new one.
  String? _intentKey;

  @override
  DataState<Order> build() => const DataState.idle();

  Future<void> placeOrder(String sku) async {
    if (state.isLoading) return;
    state = const DataState.loading();
    final key = _intentKey ??= const Uuid().v4();

    final result = await ref
        .read(orderRepositoryProvider)
        .placeOrder(sku: sku, idempotencyKey: key);
    if (!ref.mounted) return;
    if (result is Ok) _intentKey = null;
    state = DataState.from(result);
  }
}
```

**What the backend must do with the key.** A header proves nothing on its own; exactly-once
execution is a server property. Agree on this contract with the backend team and record it in
ADR-005:

1. Scope keys per user and per endpoint, and keep them at least as long as the longest client retry window (24 hours is a common default).
2. The same key with the same body returns the **stored** response, status and body, without executing again.
3. The same key with a different body is rejected (`422`), never executed.
4. A duplicate that arrives while the first is still executing waits for it or gets `409`; it never runs in parallel.
5. A request without a key on an endpoint that requires one is rejected, so a client bug fails loudly.

The client cannot verify any of this. Verify it once against staging: send the same `POST` twice
with one key and assert one record and two identical responses; send it again with a changed
body and assert a rejection. Until that passes, the replay policy is only as safe as the
backend happens to be.

For nullable responses, 204s, dynamic paths, or bulk mutations, signatures like
`Future<HouseholdDto?>` and `Future<void>` are fine; the repository's `guard` handles the rest.

### data/dto

Plain classes, no freezed. `fromJson` for responses, `toJson` for requests, and a `toDomain()`
that maps to the domain model. **DTOs never leak past the repository.**

```dart
class LoginRequestDto {
  const LoginRequestDto({required this.email, required this.password, this.clientType = 'mobile'});

  final String email;
  final String password;
  final String clientType;

  Map<String, dynamic> toJson() => {'email': email, 'password': password, 'clientType': clientType};
}

class RevokeRequestDto {
  const RevokeRequestDto({required this.refreshToken});

  final String refreshToken;

  Map<String, dynamic> toJson() => {'refreshToken': refreshToken};
}

class UserDto {
  const UserDto({required this.id, required this.email});

  final String id;
  final String email;

  factory UserDto.fromJson(Map<String, dynamic> json) =>
      UserDto(id: json['id'] as String, email: json['email'] as String);

  AuthUser toDomain() => AuthUser(id: id, email: email);
}

class LoginResponseDto {
  const LoginResponseDto({
    required this.accessToken,
    required this.refreshToken,
    required this.user,
  });

  final String accessToken;
  final String refreshToken;
  final UserDto user;

  factory LoginResponseDto.fromJson(Map<String, dynamic> json) => LoginResponseDto(
    accessToken: json['accessToken'] as String,
    refreshToken: json['refreshToken'] as String,
    user: UserDto.fromJson(json['user'] as Map<String, dynamic>),
  );

  AuthToken toDomain() =>
      AuthToken(accessToken: accessToken, refreshToken: refreshToken, user: user.toDomain());
}
```

### data/source (local)

The local half of a repository's data: a TTL cache, a `drift` DAO, or a prefs wrapper. It lives
in `data/`, **not** in `common/`, because it is a data source and only its repository may touch
it. Its lifetime is a wiring decision, made in the feature's root file (below).

```dart
/// A TTL cache for the signed-in user's profile. It lives exactly as long as the session:
/// profile.dart rebuilds it whenever auth state changes, so one account never reads another's.
/// A cache that must survive restarts goes in shared_preferences or drift, keyed by user id.
class ProfileLocalSource {
  ProfileLocalSource({DateTime Function()? clock}) : _now = clock ?? DateTime.now;

  static const ttl = Duration(minutes: 10);

  final DateTime Function() _now;
  Profile? _value;
  DateTime? _storedAt;

  Profile? read() {
    final at = _storedAt;
    if (at == null || _now().difference(at) > ttl) return null;
    return _value;
  }

  void write(Profile profile) {
    _value = profile;
    _storedAt = _now();
  }

  void clear() => _value = _storedAt = null;
}
```

### domain/models

Plain Dart. No JSON, no Dio, no Flutter. Immutable, `const` constructors, `required` fields.

```dart
class AuthUser {
  const AuthUser({required this.id, required this.email});

  final String id;
  final String email;
}

class AuthToken {
  const AuthToken({required this.accessToken, required this.refreshToken, required this.user});

  final String accessToken;
  final String refreshToken;
  final AuthUser user;
}

/// Whether a session exists. Who the user is belongs to the profile feature.
enum AuthStatus { signedIn, signedOut }
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

  /// Cold start. Never throws: an unreadable store is "signed out", reported.
  Future<AuthStatus> restoreSession();

  /// Ends the local session first, then revokes server-side, best effort. Never throws.
  Future<void> logout();
}
```

### data/repository (impl) and the feature root file (wiring)

`with RepositoryGuard implements <Feature>Repository`. Dependencies arrive through the
constructor (private named parameters, so callers write `apiService:` and the class stores
`_apiService` without an initializer list).

**The repository is the single source of truth.** It decides fresh versus cached, owns retry
and polling, maps DTO to domain, and returns `Result`. A notifier asks for data and does not
know where it came from. Cache logic never lives in a notifier.

```dart
class AuthRepositoryImpl with RepositoryGuard implements AuthRepository {
  AuthRepositoryImpl({required this._apiService, required this._session});

  final AuthApiService _apiService;
  final SessionStore _session;

  @override
  Future<Result<AuthToken>> login({
    required String email,
    required String password,
    required bool rememberMe,
  }) async {
    try {
      final token = (await _apiService.login(LoginRequestDto(email: email, password: password)))
          .toDomain();
      await _session.save(
        SessionTokens(accessToken: token.accessToken, refreshToken: token.refreshToken),
        persist: rememberMe,
      );
      return Ok(token);
    } on DioException catch (e) {
      await _session.clear();
      return mapError(e);
    } catch (e, stack) {
      // A 200 whose body fails to map is still a failed login for the user.
      reporter.recordError(e, stack, fatal: false);
      await _session.clear();
      return Err(AppException.unexpected(e, stack));
    }
  }

  @override
  Future<AuthStatus> restoreSession() async => switch (await _session.load()) {
    SessionFound() => AuthStatus.signedIn,
    SessionMissing() || SessionUnreadable() => AuthStatus.signedOut,
  };

  @override
  Future<void> logout() async {
    final refreshToken = _session.tokens?.refreshToken;
    await _session.clear(); // the device is signed out from this line on, whatever follows
    if (refreshToken == null) return;
    try {
      await _apiService
          .revoke(RevokeRequestDto(refreshToken: refreshToken))
          .timeout(const Duration(seconds: 5));
    } on DioException {
      // Offline sign-out is normal. The refresh token expires server-side on its own.
    } on TimeoutException {
      // Same.
    } catch (e, stack) {
      reporter.recordError(e, stack, fatal: false);
    }
  }
}
```

The provider that hands this to the rest of the app lives in the feature's root file, typed by
the interface. Presentation imports the root file and never sees `AuthRepositoryImpl`; other
features import it to read `authStateProvider` (ADR-009):

```dart
export 'package:reference_app/features/auth/domain/models/auth_token.dart';
export 'package:reference_app/features/auth/domain/repository/auth_repository.dart';
export 'package:reference_app/features/auth/presentation/notifier/auth_state_notifier.dart'
    show AuthStateNotifier, authStateProvider;

final authRepositoryProvider = Provider<AuthRepository>((ref) {
  return AuthRepositoryImpl(
    apiService: AuthApiService(ref.watch(dioProvider)),
    session: ref.watch(sessionStoreProvider),
  );
});
```

A failed login leaves no session behind whatever threw. Sign-out cannot be blocked by the
network or by the keystore: the local session is cleared first, and revocation uses the
refresh token captured before the clear, sent with `kSkipAuth` so it never touches the
interceptor's refresh path.

A repository that combines remote and local sources, which is the shape the official guidance
means by "single source of truth":

```dart
class ProfileRepositoryImpl with RepositoryGuard implements ProfileRepository {
  ProfileRepositoryImpl(this._apiService, this._cache);

  final ProfileApiService _apiService;
  final ProfileLocalSource _cache;

  @override
  Future<Result<Profile?>> getProfile({bool forceRefresh = false}) async {
    if (!forceRefresh) {
      final cached = _cache.read();
      if (cached != null) return Ok(cached);
    }
    return guard(
      () async {
        final profile = (await _apiService.getProfile()).toDomain();
        _cache.write(profile);
        return profile;
      },
      // 404 means "no profile yet", which is data, not a failure.
      onDioError: (e) => e.response?.statusCode == 404 ? const Ok(null) : mapError(e),
    );
  }
}
```

And its wiring. Watching `authStateProvider` here is what scopes the cache to one session: a
sign-out or an account switch rebuilds the repository with an empty cache, so one user can never
be served another's profile. A cache that must survive restarts belongs in `shared_preferences`
or `drift`, keyed by user id.

```dart
export 'package:reference_app/features/profile/domain/models/profile.dart';
export 'package:reference_app/features/profile/domain/repository/profile_repository.dart';

final profileRepositoryProvider = Provider<ProfileRepository>((ref) {
  ref.watch(authStateProvider); // a new session gets a new, empty cache
  return ProfileRepositoryImpl(ProfileApiService(ref.watch(dioProvider)), ProfileLocalSource());
});
```

The notifier calls `getProfile()` and gets data. Whether it came from memory or the network is
not its problem. `forceRefresh: true` is what pull-to-refresh passes.

### presentation/notifier

State lives in a `Notifier` or `AsyncNotifier`. Public methods are **commands**: verbs the view
calls. Three patterns, in order of reach. **Default to Pattern 1 and escalate only when a
concrete trigger forces it.**

```
Does the screen have ONE async action (login, submit, fetch)?   -> Pattern 1
Does the state seed from storage or react to a stream?          -> Pattern 2
Does the screen have MULTIPLE independent async slices AND
  per-item loading flags AND local mutations without refetch?   -> Pattern 3

None of these -> start with Pattern 1. You can always promote later.
```

**One rule for every pattern: check `ref.mounted` after every `await` before touching `state`.**
While a command awaits, the notifier can be disposed (an `autoDispose` screen was popped) or
rebuilt (it watches `authStateProvider` and the user signed out or switched accounts). Writing
state from the old instance is at best an exception and at worst one user's data landing in
another user's screen.

#### Pattern 1: simple action notifier (the default)

State **is** a `DataState<T>`. Idle initially, then loading, then the mapped repository result.

```dart
final loginNotifierProvider = NotifierProvider<LoginNotifier, DataState<AuthToken>>(
  LoginNotifier.new,
);

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

    final result = await ref
        .read(authRepositoryProvider)
        .login(email: email, password: password, rememberMe: rememberMe);
    if (!ref.mounted) return;

    if (result case Ok()) {
      await ref.read(authStateProvider.notifier).setAuthenticated();
      if (!ref.mounted) return;
    }
    state = DataState.from(result); // the one Result -> DataState conversion
  }

  void reset() => state = const DataState.idle();
}
```

The `isLoading` guard matters: without it, a user double-tapping Login fires two requests and
the second overwrites the first's state.

#### Pattern 2: session or seeded notifier

`AsyncNotifier` seeding from storage and reacting to a stream. Use this only when the initial
value comes from an async source or must react to an external event.

```dart
final authStateProvider = AsyncNotifierProvider<AuthStateNotifier, AuthStatus>(
  AuthStateNotifier.new,
);

class AuthStateNotifier extends AsyncNotifier<AuthStatus> {
  @override
  Future<AuthStatus> build() {
    final sub = ref.watch(sessionExpiredProvider).stream.listen((_) {
      if (state.value == AuthStatus.signedIn) state = const AsyncData(AuthStatus.signedOut);
    });
    ref.onDispose(sub.cancel);
    return ref.read(authRepositoryProvider).restoreSession(); // never throws
  }

  Future<void> setAuthenticated() async {
    // If the cold-start restoreSession() is still pending, let it finish first.
    // Otherwise its late result (signedOut) would overwrite this one.
    await future;
    if (!ref.mounted) return;
    ref.read(authInterceptorProvider).startSession(); // new session era
    state = const AsyncData(AuthStatus.signedIn);
  }

  Future<void> signOut() async {
    final repository = ref.read(authRepositoryProvider);
    ref.read(authInterceptorProvider).endSession(); // in-flight requests go stale now
    state = const AsyncData(AuthStatus.signedOut); // the guard leaves protected routes now
    await repository.logout(); // local clear, then best-effort revoke
  }
}
```

The order in `signOut` is the contract: stop trusting in-flight work, leave the protected UI,
then clean up. Nothing that can fail runs before the user is signed out.

`startSession()` must be called on every sign-in. The interceptor's expire-once latch is per
instance, and `dioProvider` is keep-alive, so the instance outlives a sign-out; without the call,
a second session in the same process can never be torn down. The §7 test "a new session can
expire again" pins it, and a notifier test verifies `setAuthenticated` makes the call.

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
> - **local mutations** must update the UI without a refetch.
>
> If you cannot check all three, Pattern 1 with a second notifier is simpler. This is the most
> copy-paste-prone pattern and must not become the default.

**Conceptual:** the reference app has no screen that passes this gate, which is the point of
the gate. `test/conceptual/family_compile_test.dart` compiles this block against the real
`Result`, `DataState` and Riverpod types, so it cannot drift into code that does not build.

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
  // A getter, not `late final`: build() runs again on every rebuild, and a second
  // assignment to a late final field throws.
  FamilyRepository get _repository => ref.read(familyRepositoryProvider);

  @override
  FamilyState build() {
    ref.watch(familyRepositoryProvider);
    ref.watch(authStateProvider); // from auth.dart: rebuild (and reset) on sign-out or account switch
    return const FamilyState();
  }

  /// Returns null on success, or a user-facing message on failure.
  Future<String?> _runCommand<T>(
    Future<Result<T>> Function() action, {
    void Function()? onStart,
    void Function()? onFinish,
    void Function(T value)? onSuccess,
  }) async {
    onStart?.call();
    try {
      final result = await action();
      if (!ref.mounted) return null; // nobody is listening any more
      switch (result) {
        case Ok(:final value):
          onSuccess?.call(value);
          return null;
        case Err(:final exception):
          return exception.type == ErrorType.cancelled ? null : errorToMessage(exception);
      }
    } finally {
      if (ref.mounted) onFinish?.call();
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
        onSuccess: (_) { /* remove the member from householdState locally; no refetch */ },
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
      body: SafeArea(
        child: SingleChildScrollView(
          padding: EdgeInsets.symmetric(horizontal: Dimensions.px24, vertical: Dimensions.px48),
          child: const LoginForm(),
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
  Widget build(BuildContext context) {
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

    final loading = ref.watch(loginNotifierProvider).isLoading;
    // ... the form fields, and a submit button disabled while `loading`
  }
```

The `switch` lists all four cases explicitly rather than using `default`, so adding a fifth
state to `DataState` becomes a compile error here instead of a silently ignored branch.

**Every data screen renders four states, and empty is one of them.** Loading, success, error
with a retry action, and an empty success that says what the user can do next. A blank screen
on an empty list is a bug the widget tests in §7.8 catch. The profile screen renders them with
one exhaustive switch:

```dart
          switch (ref.watch(profileNotifierProvider)) {
            DataIdle() || DataLoading() => const Center(child: CircularProgressIndicator()),
            DataSuccess(data: null) => const Text(Strings.profileEmpty),
            DataSuccess(:final data?) => Text(
              data.name,
              style: TextStyle(fontSize: Dimensions.sp24),
            ),
            DataError(:final exception) => Column(
              children: [
                Text(errorToMessage(exception)),
                SizedBox(height: Dimensions.px16),
                OutlinedButton(
                  onPressed: () => notifier.load(forceRefresh: true),
                  child: const Text(Strings.retry),
                ),
              ],
            ),
          },
```

**Semantics.** Icon-only buttons get a `tooltip` or `Semantics(label:)`, images get
`semanticLabel` or are excluded, and custom tappables are at least 48x48 logical pixels. The
widget tests check these with Flutter's accessibility guidelines.

---

## 6. State Management with Riverpod

Riverpod is both the DI container and the notifier layer. One rule governs all of it: **a
provider lives next to the thing it exposes, and the provider graph points the same way the
layers do.** Outer depends on inner, never sideways.

### The provider graph mirrors the layers

```
appConfigProvider ----------+
httpAdapterFactoryProvider -+--> authInterceptorProvider --> dioProvider --> <feature>RepositoryProvider --> <feature>NotifierProvider --> Widget
secureStorageProvider --> sessionStoreProvider --+                          (features/<f>/<f>.dart)          (notifier)
sessionExpiredProvider --------------------------+

sessionExpiredProvider, authInterceptorProvider, authRepositoryProvider --> authStateProvider
authStateProvider --> any provider whose data belongs to one session (profileRepositoryProvider)
```

- `appConfigProvider`, `httpAdapterFactoryProvider`, `secureStorageProvider`, `sessionStoreProvider`, `sessionExpiredProvider`, `authInterceptorProvider`, `dioProvider`: core infrastructure as `Provider<T>`. Constructed once, injected everywhere, overridable in tests. The only global is `reporter` (§3), for the reason given there.
- `<feature>RepositoryProvider`: DI for the data layer, declared in the feature's root file. A `Provider<<Feature>Repository>` that reads `dioProvider` and builds any local source, returning the concrete `Impl` typed as the interface. Presentation depends on this **interface** provider, never on the `Impl` type.
- `<feature>NotifierProvider`: the notifier. It reads the repository provider; nothing below it can see it.
- A widget only ever watches notifier providers, plus a couple of app-wide ones such as `authStateProvider` and `themeProvider`. It never watches a repository or `dioProvider` directly.

If a widget reaches past its notifier to a repository or Dio, the graph is broken the same way
an import that skips a layer is, and the §8 boundary lints catch the import that makes it
possible.

### Which provider type, and when

| Provider | Use for | Example |
| --- | --- | --- |
| `Provider<T>` | Stateless DI, constructed once | `dioProvider`, `sessionStoreProvider`, `<feature>RepositoryProvider` |
| `NotifierProvider<N, S>` | Per-screen or per-action notifier | `loginNotifierProvider` (Pattern 1), `familyProvider` (Pattern 3) |
| `AsyncNotifierProvider<N, T>` | State seeded from an async source or stream | `authStateProvider` (Pattern 2) |

No `StateProvider` or `ChangeNotifierProvider` for feature state: they hide the transitions the
UI needs to see, and Riverpod 3 moved both to its `legacy` import for that reason.
`FutureProvider` is fine for a read-only derived value with no commands attached, but a screen
with actions gets a `Notifier`.

### `watch` vs `read` vs `listen`

- **`ref.watch`**: reactive dependency. Only inside a `build()` (widget `build`, `Notifier.build`, or a provider body). Watching `authStateProvider` inside a notifier's `build()` is how a feature resets on sign-out.
- **`ref.read`**: one-shot, non-reactive. Only inside callbacks and event handlers, to fire a command. Never `read` in `build` (you would miss updates); never `watch` in a callback (you would rebuild-thrash).
- **`ref.listen`**: side effects that are not UI. Navigation, toasts, dialogs. It lives in a widget `build` but returns nothing. This is the one place a `DataError` becomes a toast.

### Notifier conventions

- `build()` returns the **initial** state and does setup only: `watch` dependencies, subscribe to streams with `ref.onDispose` to cancel. No network call fires from an action notifier's `build()`; the view kicks off the load in a post-frame callback guarded by the idle state.
- Public methods are **commands**: verbs the view calls. They mutate `state`; the UI reacts. Notifiers never return widgets and never touch `BuildContext`.
- After every `await` in a command: `if (!ref.mounted) return;` before touching `state` or `ref`.
- Dependencies are getters over `ref.read`, or locals in `build()`. Never `late final` fields assigned in `build()`.
- Cross-notifier writes go through `.notifier`, never by mutating another provider's state directly.
- Reset ephemeral state with a `reset()` command or `ref.invalidate(provider)`. App-wide teardown flows through `authStateProvider`, not manual resets scattered across notifiers.

### Lifecycle and scope

Providers are **kept alive by default**, because the app is tab-based over an `IndexedStack`:
every tab stays mounted for the app's life, so its notifier should too. Disposing it would
drop scroll position, filters, and loaded data on every tab switch. Riverpod 3 pauses listeners
in widgets that are not visible (an offstage tab), so a keep-alive tab provider costs memory,
not rebuilds.

Add `.autoDispose` when **all** of these hold:

- the provider is scoped to a route that is pushed and popped (a detail screen, a modal, a wizard step), not a persistent tab;
- its state is per-visit and stale on return, so rebuilding from scratch is correct rather than lossy;
- it holds something that must be released promptly: a `StreamSubscription`, a large list, a controller.

```dart
// illustrative
final orderDetailProvider =
    NotifierProvider.autoDispose<OrderDetailNotifier, DataState<Order>>(
  OrderDetailNotifier.new,
);
```

Two escape hatches: `ref.keepAlive()` inside an `autoDispose` provider pins it after a
successful load, and `ref.invalidate(provider)` forces a keep-alive provider to rebuild on
demand, for example on pull-to-refresh, without changing its scope.

**If you cannot point to a push/pop route AND a leak or staleness reason, leave it keep-alive.**

**Riverpod 3 retries failing providers automatically.** That is right for a flaky read and
wrong for anything whose failure is a decision. `authStateProvider` cannot fail by construction
(`restoreSession` never throws). For any other provider where a retry would repeat a side
effect, disable it on that provider rather than globally.

Overrides are the test and bootstrap seam:
`ProviderContainer(overrides: [<feature>RepositoryProvider.overrideWithValue(mock)])`. Because
every dependency is a provider, nothing needs a real Dio or keystore to test.

---
## 7. Testing

The layering exists so each piece is testable in isolation, and this is where it pays off. The
official guidance puts this at the "strongly recommend" tier: write unit tests for every
service, repository, and notifier. This section goes further and states what evidence each
layer must produce before a change merges.

`mocktail` for doubles, `ProviderContainer` for DI. Never a real Dio, real keystore, or real
socket. Mock the boundary directly below the thing under test.

### 7.1 The test matrix

| Layer | What must be proven | Where | Gate |
| --- | --- | --- | --- |
| Unit | Error mapping (every `ErrorType`), `errorToMessage` (every case), `DataState.from`, `AppConfig.parse` | `test/core/`, `test/common/` | every PR |
| Session | The `SessionStore` contract table in §3; the platform options it depends on | `test/core/session/` | every PR |
| Repository | `Ok` branch, `Err` branch, non-Dio throw, cache hit, bypass and expiry, logout order | `test/features/<f>/data/` | every PR |
| Networking | The six interceptor invariants, refresher classification, redaction in logs and reports | `test/core/network/` | every PR |
| Notifier | idle, loading, result transitions; double-tap guard; session era calls; idempotency intent | `test/features/<f>/presentation/` | every PR |
| Widget | Loading, success, empty, error states; accessibility in both themes; 2.0x text scale | `test/features/<f>/presentation/` | every PR |
| Flow | Sign-in, one token refresh mid-flow, sign-out, through the real router and guard | `test/flows/` headless; `integration_test/` on a device | every PR |
| Test honesty | Every invariant test fails under its named sabotage | `tool/check_sabotage.dart` | every PR |
| Architecture | Each lint fires where it should and only there; both plugins load in the app; no feature cycles | `tool/check_lint_fixture.dart`, `tool/check_architecture.dart` | every PR |
| Release | Release config valid, backup rules complete, prod bundle builds, iOS release compiles | `tool/check_release_config.dart`, `android` and `ios` jobs | every PR |

### 7.2 Coverage policy

Coverage is a guardrail against whole files nobody tested, not a score to optimize. Two floors,
both enforced in CI (§11.2):

- **80% of lines overall**, generated files excluded.
- **90% of lines in `lib/core/network/` and `lib/core/session/`**. This is where a bug signs
  someone out, signs the wrong person in, or leaks a token, so the floor is higher.

`flutter test --coverage` only reports files that some test imports. A file with no test at all
is not 0%, it is absent, and the overall figure silently overstates what is tested.
`tool/gen_coverage_helper.sh` writes a throwaway test that imports every library, so untested
files count:

```bash
#!/usr/bin/env bash
# Writes test/coverage_helper_test.dart, which imports every library under lib/.
# Without it, a file no test imports is absent from lcov.info and the coverage
# figure silently overstates what is tested.
set -euo pipefail
pkg=$(sed -n 's/^name:[[:space:]]*//p' pubspec.yaml | head -n1)
{
  echo '// GENERATED by tool/gen_coverage_helper.sh. Do not edit or commit.'
  echo '// ignore_for_file: unused_import'
  find lib -name '*.dart' -type f -print0 | sort -z | xargs -0 grep -L '^part of' \
    | sed "s#^lib/\(.*\)#import 'package:${pkg}/\1';#"
  echo 'void main() {}'
} > test/coverage_helper_test.dart
```

`part of` files are skipped because they cannot be imported; generated `.g.dart` and
`.freezed.dart` files are all parts, so they drop out automatically. Add
`test/coverage_helper_test.dart` to `.gitignore`.

The floors are checked by a Dart script rather than `lcov` and `grep -oP`: one fewer apt
package in CI, no regex over a tool's human-readable output, and it lists the worst files when
a floor fails. In the reference app: 85.5% overall, 93.5% in `core/network`, 98.3% in
`core/session`.

```dart
// Coverage floors (§7.2), read straight from coverage/lcov.info. Generated files excluded.
// Run after tool/gen_coverage_helper.sh and `flutter test --coverage`.
import 'dart:io';

const _floors = {
  'overall': (prefix: 'lib/', min: 80.0),
  'core/network': (prefix: 'lib/core/network/', min: 90.0),
  'core/session': (prefix: 'lib/core/session/', min: 90.0),
};

void main() {
  final hits = <String, (int found, int hit)>{};
  String? file;
  var found = 0, hit = 0;
  for (final line in File('coverage/lcov.info').readAsLinesSync()) {
    if (line.startsWith('SF:')) {
      file = line.substring(3).replaceAll(r'\', '/');
      file = file.substring(file.indexOf('lib/'));
      found = hit = 0;
    } else if (line.startsWith('DA:')) {
      found++;
      if (line.split(',')[1] != '0') hit++;
    } else if (line == 'end_of_record' && file != null) {
      if (!file.endsWith('.g.dart') && !file.endsWith('.freezed.dart')) hits[file] = (found, hit);
      file = null;
    }
  }

  var failed = false;
  for (final MapEntry(key: name, value: (:prefix, :min)) in _floors.entries) {
    final files = hits.entries.where((e) => e.key.startsWith(prefix));
    final f = files.fold(0, (a, e) => a + e.value.$1);
    final h = files.fold(0, (a, e) => a + e.value.$2);
    final pct = f == 0 ? 0.0 : 100 * h / f;
    final ok = f > 0 && pct >= min;
    failed |= !ok;
    stdout.writeln(
      '${ok ? 'ok  ' : 'FAIL'} $name: ${pct.toStringAsFixed(1)}% of $f lines (floor $min%)',
    );
  }
  if (failed) {
    final worst = hits.entries.toList()
      ..sort((a, b) => (a.value.$2 / a.value.$1).compareTo(b.value.$2 / b.value.$1));
    for (final e in worst.take(8)) {
      stderr.writeln('  ${e.key}: ${e.value.$2}/${e.value.$1}');
    }
    exitCode = 1;
  }
}
```

### 7.3 Test honesty

A test that cannot fail is worse than no test, because the suite reports coverage of an
invariant nobody is checking.

- **No empty bodies, no comment-only stubs.** An empty `test()` passes. If a case is not worth writing yet, delete it or mark it `skip: '<reason and ticket>'` so the runner says so out loud.
- **Every invariant test names its sabotage, and a machine applies it.** The comment above each test in §7.5 to §7.7 names the one change to production code that must make it fail. `tool/check_sabotage.dart` holds the same list as data: it applies each sabotage, runs the named test, and fails CI if the test still passes, if the sabotage no longer compiles, or if the named test no longer exists. In rc.1 this was a manual step in the PR description, and rc.1's own invariant-2 test survived its own sabotage (§3). The reference app currently kills 29 of 29.
- **Assert outcomes, not calls, where you can.** `verify(...).called(1)` is right for "the refresh happened exactly once"; it is wrong as the only assertion of a test about what the user sees.

```dart
typedef Sabotage = ({String file, String find, String replace, String test, String name});

const _interceptor = 'lib/core/network/auth_interceptor.dart';
const _session = 'lib/core/session/session_store.dart';
const _interceptorTest = 'test/core/network/auth_interceptor_test.dart';
const _sessionTest = 'test/core/session/session_store_test.dart';

const List<Sabotage> sabotages = [
  (
    file: _interceptor,
    find: 'if (sentGen < _generation) {',
    replace: 'if (sentGen < _generation && false) {',
    test: _interceptorTest,
    name: 'two concurrent 401s trigger exactly one refresh and both replay',
  ),
  // ... 28 more: one entry per invariant test
];

void main() {
  final survivors = <String>[];
  for (final s in sabotages) {
    final file = File(s.file);
    final original = file.readAsStringSync();
    // Whitespace-insensitive, so `dart format` does not silently retire a sabotage.
    final target = RegExp(s.find.trim().split(RegExp(r'\s+')).map(RegExp.escape).join(r'\s+'));
    if (!target.hasMatch(original)) {
      survivors.add('${s.file}: sabotage target no longer exists; update this table: ${s.find}');
      continue;
    }
    try {
      file.writeAsStringSync(original.replaceFirst(target, s.replace));
      final run = Process.runSync('flutter', [
        'test',
        s.test,
        '--plain-name',
        s.name,
      ], runInShell: true);
      final out = '${run.stdout}${run.stderr}';
      if (out.contains('No tests ran') || out.contains('No tests match')) {
        survivors.add('NO SUCH TEST: "${s.name}" in ${s.test}');
      } else if (out.contains('Compilation failed')) {
        survivors.add('INVALID: sabotage of ${s.file} does not compile, so it proves nothing');
      } else if (run.exitCode == 0) {
        survivors.add('SURVIVED: "${s.name}" still passes with ${s.file} sabotaged');
      } else {
        stdout.writeln('killed: ${s.name}');
      }
    } finally {
      file.writeAsStringSync(original);
    }
  }
  if (survivors.isEmpty) {
    stdout.writeln('sabotage: ok (${sabotages.length} of ${sabotages.length} killed)');
    return;
  }
  survivors.forEach(stderr.writeln);
  exitCode = 1;
}
```

### 7.4 Repository tests

The repository's job is to call the service, map DTO to domain, wrap in `Result`, and leave no
session behind on failure. Assert both branches and the non-Dio path.

```dart
class _MockAuthApi extends Mock implements AuthApiService {}

class _MockSession extends Mock implements SessionStore {}

const _loginResponse = LoginResponseDto(
  accessToken: 'a1',
  refreshToken: 'r1',
  user: UserDto(id: 'u1', email: 'a@b.com'),
);

DioException _dioError(
  String path, {
  required int status,
  Map<String, dynamic> extra = const {},
  Object? body,
}) {
  final options = RequestOptions(path: path, extra: {...extra});
  return DioException(
    requestOptions: options,
    response: Response(requestOptions: options, statusCode: status, data: body),
    type: DioExceptionType.badResponse,
  );
}

void main() {
  setUpAll(() {
    registerFallbackValue(const LoginRequestDto(email: '', password: ''));
    registerFallbackValue(const RevokeRequestDto(refreshToken: ''));
    registerFallbackValue(const SessionTokens(accessToken: '', refreshToken: ''));
  });

  late _MockAuthApi api;
  late _MockSession session;
  late AuthRepositoryImpl repo;

  setUp(() {
    api = _MockAuthApi();
    session = _MockSession();
    repo = AuthRepositoryImpl(apiService: api, session: session);
    when(() => session.save(any(), persist: any(named: 'persist'))).thenAnswer((_) async {});
    when(() => session.clear()).thenAnswer((_) async {});
  });

  test('login returns Ok and saves the session on 200', () async {
    when(() => api.login(any())).thenAnswer((_) async => _loginResponse);

    final result = await repo.login(email: 'a@b.com', password: 'pw', rememberMe: true);

    expect(result, isA<Ok<AuthToken>>());
    verify(() => session.save(any(), persist: true)).called(1);
  });

  // Sabotage: delete the `401 when skipAuth` arm in mapDioException.
  test('a wrong password is rejected with the server message, not "session expired"', () async {
    when(() => api.login(any())).thenThrow(
      _dioError(
        '/auth/login',
        status: 401,
        extra: {kSkipAuth: true},
        body: {'message': 'Incorrect email or password.'},
      ),
    );

    final result = await repo.login(email: 'a@b.com', password: 'bad', rememberMe: false);

    final exception = (result as Err<AuthToken>).exception;
    expect(exception.type, ErrorType.rejected);
    expect(errorToMessage(exception), 'Incorrect email or password.');
    verify(() => session.clear()).called(1);
  });

  test('a non-Dio throw after a 200 is Err and leaves no session', () async {
    when(() => api.login(any())).thenThrow(TypeError()); // response shape changed server-side

    final result = await repo.login(email: 'a@b.com', password: 'pw', rememberMe: false);

    expect(result, isA<Err<AuthToken>>());
    expect((result as Err<AuthToken>).exception.message, isNull); // no internals for the UI
    verify(() => session.clear()).called(1);
  });

  test('logout clears locally before revoking, and survives an offline revoke', () async {
    final order = <String>[];
    when(() => session.tokens).thenReturn(const SessionTokens(accessToken: 'a', refreshToken: 'r'));
    when(() => session.clear()).thenAnswer((_) async => order.add('clear'));
    when(() => api.revoke(any())).thenAnswer((_) async {
      order.add('revoke');
      throw _dioError('/auth/revoke', status: 503);
    });

    await repo.logout(); // must not throw

    expect(order, ['clear', 'revoke']);
  });
}
```

The second test is the regression for a real 1.x bug: `/auth/login` went through the refresh
path and its 401 rendered as an expired session. The third is the one people skip: it pins that
a non-Dio failure still produces `Err`, still clears, and still keeps `message` empty. The fourth
pins the sign-out order: the device is signed out before the network is asked anything.

Cache behavior, for any repository with a local source:

```dart
  test('a cache hit does not touch the network', () async {
    await repo.getProfile();
    final result = await repo.getProfile();

    expect((result as Ok<Profile?>).value?.name, 'Dave');
    verify(() => api.getProfile()).called(1);
  });

  test('forceRefresh bypasses the cache', () async {
    await repo.getProfile();
    await repo.getProfile(forceRefresh: true);

    verify(() => api.getProfile()).called(2);
  });

  test('an expired entry is refetched', () async {
    await repo.getProfile();
    now = now.add(ProfileLocalSource.ttl + const Duration(seconds: 1));
    await repo.getProfile();

    verify(() => api.getProfile()).called(2);
  });

  // ... and: 404 means no profile yet, Ok(null), not an error
```

### 7.5 Session tests

These pin the contract table in §3. Every row has a test; five are shown. `_Keystore` is an
in-memory fake whose writes and deletes can be made to fail, which is closer to the real thing
than stubbing each call.

```dart
class _MockStorage extends Mock implements FlutterSecureStorage {}

/// An in-memory keystore whose operations can be made to fail on demand.
class _Keystore extends Fake implements FlutterSecureStorage {
  final values = <String, String>{};
  bool Function(String key)? failWrite;
  bool Function(String key)? failDelete;

  @override
  Future<String?> read({
    required String key,
    Object? aOptions,
    Object? iOptions,
    Object? lOptions,
    Object? mOptions,
    Object? wOptions,
    Object? webOptions,
  }) async => values[key];

  @override
  Future<void> write({
    required String key,
    required String? value,
    Object? aOptions,
    Object? iOptions,
    Object? lOptions,
    Object? mOptions,
    Object? wOptions,
    Object? webOptions,
  }) async {
    if (failWrite?.call(key) ?? false) throw PlatformException(code: 'write');
    values[key] = value!;
  }

  @override
  Future<void> delete({
    required String key,
    Object? aOptions,
    Object? iOptions,
    Object? lOptions,
    Object? mOptions,
    Object? wOptions,
    Object? webOptions,
  }) async {
    if (failDelete?.call(key) ?? false) throw PlatformException(code: 'delete');
    values.remove(key);
  }
}

const _tokens = SessionTokens(accessToken: 'a1', refreshToken: 'r1');
const _rotated = SessionTokens(accessToken: 'a2', refreshToken: 'r2');

void main() {
  late _Keystore keystore;
  late SessionStore store;

  setUp(() {
    keystore = _Keystore();
    store = SessionStore(keystore);
  });

  // Sabotage: delete the session keys in load()'s catch.
  test('an unreadable store starts signed out, reports, and deletes nothing', () async {
    final storage = _MockStorage();
    when(() => storage.read(key: any(named: 'key')))
        .thenThrow(PlatformException(code: 'errSecInteractionNotAllowed'));

    expect(await SessionStore(storage).load(), isA<SessionUnreadable>());
    verifyNever(() => storage.delete(key: any(named: 'key')));
    verifyNever(() => storage.deleteAll());
  });

  // Sabotage: split _persist into two writes.
  test('rotation is one write, so a crash cannot leave a mixed pair on disk', () async {
    final storage = _MockStorage();
    when(
      () => storage.write(
        key: any(named: 'key'),
        value: any(named: 'value'),
      ),
    ).thenAnswer((_) async {});
    final s = SessionStore(storage);
    await s.save(_tokens, persist: true);
    clearInteractions(storage);

    await s.rotate(from: s.tokens!, to: _rotated);

    verify(() => storage.write(key: kSession, value: _rotated.encode())).called(1);
    verifyNoMoreInteractions(storage);
  });

  // Sabotage: in clear(), null the tokens only after the deletes complete.
  test('clear ends the session at once, even if the keystore never answers', () async {
    final storage = _MockStorage();
    when(
      () => storage.write(
        key: any(named: 'key'),
        value: any(named: 'value'),
      ),
    ).thenAnswer((_) async {});
    when(() => storage.delete(key: any(named: 'key'))).thenAnswer((_) => Completer<void>().future);
    final s = SessionStore(storage);
    await s.save(_tokens, persist: true);

    unawaited(s.clear()); // the keystore hangs; the session must already be over

    expect(s.accessToken, isNull);
  });

  // Sabotage: remove the catchError from _serial.
  test('an operation that escapes its own handling does not stall later storage work', () async {
    await store.save(_tokens, persist: true);

    unawaited(store.debugEnqueue(() async => throw StateError('escaped')).catchError((_) {}));
    await store.clear();

    expect(keystore.values, isNot(contains(kSession)));
  });

  // Sabotage: remove the identical() check in rotate().
  test('a refresh that finishes after sign-out cannot resurrect the session', () async {
    await store.save(_tokens, persist: true);
    final captured = store.tokens!;

    await store.clear();
    final applied = await store.rotate(from: captured, to: _rotated);

    expect(applied, isFalse);
    expect(store.accessToken, isNull);
    expect(keystore.values, isNot(contains(kSession)));
  });

  // ... and one test for each remaining row of the contract table in §3
}
```

### 7.6 Notifier tests

The notifier owns exactly one thing: the idle, loading, result transition. No widgets, no
`WidgetTester`. The interceptor is overridden with a mock, because `setAuthenticated` calls
`startSession()` on it and nothing here is about networking.

```dart
class _MockAuthRepository extends Mock implements AuthRepository {}

class _MockAuthInterceptor extends Mock implements AuthInterceptor {}

const _token = AuthToken(
  accessToken: 'a',
  refreshToken: 'r',
  user: AuthUser(id: 'u1', email: 'a@b.com'),
);

void main() {
  late _MockAuthRepository repo;
  late _MockAuthInterceptor interceptor;
  late ProviderContainer container;

  setUp(() {
    repo = _MockAuthRepository();
    interceptor = _MockAuthInterceptor();
    when(() => repo.restoreSession()).thenAnswer((_) async => AuthStatus.signedOut);
    container = ProviderContainer(
      overrides: [
        authRepositoryProvider.overrideWithValue(repo),
        authInterceptorProvider.overrideWithValue(interceptor),
      ],
    );
    addTearDown(container.dispose);
  });

  void stubLogin(Future<Result<AuthToken>> Function() answer) => when(
    () => repo.login(
      email: any(named: 'email'),
      password: any(named: 'password'),
      rememberMe: any(named: 'rememberMe'),
    ),
  ).thenAnswer((_) => answer());

  Future<void> login() => container
      .read(loginNotifierProvider.notifier)
      .login(email: 'a@b.com', password: 'pw', rememberMe: false);

  test('login emits idle, loading, success, signs the app in, and opens a session era', () async {
    stubLogin(() async => const Ok(_token));

    final emitted = <DataState<AuthToken>>[];
    container.listen(loginNotifierProvider, (_, next) => emitted.add(next), fireImmediately: true);

    await login();

    expect(emitted.map((s) => s.runtimeType), [
      DataIdle<AuthToken>,
      DataLoading<AuthToken>,
      DataSuccess<AuthToken>,
    ]);
    expect(container.read(authStateProvider).value, AuthStatus.signedIn);
    verify(() => interceptor.startSession()).called(1);
  });

  // Sabotage: delete the `if (state.isLoading) return;` guard.
  test('a second login while loading is ignored', () async {
    stubLogin(() async {
      await Future<void>.delayed(const Duration(milliseconds: 50));
      return const Ok(_token);
    });

    await Future.wait([login(), login()]);

    verify(
      () => repo.login(
        email: any(named: 'email'),
        password: any(named: 'password'),
        rememberMe: any(named: 'rememberMe'),
      ),
    ).called(1);
  });

  test('a cancelled login returns to idle without an error', () async {
    stubLogin(() async => const Err(AppException(type: ErrorType.cancelled)));

    await login();

    expect(container.read(loginNotifierProvider), isA<DataIdle<AuthToken>>());
  });

  test('sign-out ends the interceptor session before the local clear', () async {
    final order = <String>[];
    when(() => interceptor.endSession()).thenAnswer((_) => order.add('endSession'));
    when(() => repo.logout()).thenAnswer((_) async => order.add('logout'));
    await container.read(authStateProvider.future);

    await container.read(authStateProvider.notifier).signOut();

    expect(order, ['endSession', 'logout']);
    expect(container.read(authStateProvider).value, AuthStatus.signedOut);
  });
}
```

### 7.7 Interceptor tests

These justify the networking design. They are written against the invariants in §3, not
against queue semantics, and each states the sabotage that breaks it. One scripted fake server
serves every test: it records the `Authorization` header of each request it receives, so a
test can prove not only what the caller saw but what was actually sent.

```dart
/// Answers each request with a status chosen by [respond], and records what it was sent.
class _FakeServer implements HttpClientAdapter {
  _FakeServer(this.respond);
  final FutureOr<int> Function(RequestOptions o) respond;
  final sent = <String?>[];

  @override
  Future<ResponseBody> fetch(
    RequestOptions o,
    Stream<Uint8List>? requestStream,
    Future<void>? cancelFuture,
  ) async {
    sent.add(o.headers['Authorization'] as String?);
    return ResponseBody.fromString(
      '{}',
      await respond(o),
      headers: {
        Headers.contentTypeHeader: [Headers.jsonContentType],
      },
    );
  }

  @override
  void close({bool force = false}) {}
}

/// 200 only for the refreshed token, 401 for anything else.
int _acceptsNew(RequestOptions o) => o.headers['Authorization'] == 'Bearer new' ? 200 : 401;

({Dio dio, AuthInterceptor auth}) _client({
  required _FakeServer server,
  _FakeServer? replayServer,
  required Future<String?> Function() readToken,
  required Future<RefreshOutcome> Function() refresh,
  Future<void> Function()? onExpired,
}) {
  final auth = AuthInterceptor(
    readToken: readToken,
    refresh: refresh,
    onExpired: onExpired ?? () async {},
    retryClient: Dio()..httpClientAdapter = replayServer ?? server,
  );
  final dio = Dio(BaseOptions(baseUrl: 'https://test'))
    ..httpClientAdapter = server
    ..interceptors.add(auth);
  return (dio: dio, auth: auth);
}

Matcher _failsWith(int status) =>
    throwsA(isA<DioException>().having((e) => e.response?.statusCode, 'status', status));

Matcher _failsWithError<T>() =>
    throwsA(isA<DioException>().having((e) => e.error, 'error', isA<T>()));

/// What a caller saw: an HTTP status, or the marker error that replaced it.
Future<String> _outcome(Future<Response<dynamic>> request) async {
  try {
    return '${(await request).statusCode}';
  } on DioException catch (e) {
    return e.error is StaleSession ? 'stale' : '${e.response?.statusCode}';
  }
}

void main() {
  // Invariant 1. Sabotage: delete the `sentGen < _generation` branch.
  test('two concurrent 401s trigger exactly one refresh and both replay', () async {
    var refreshes = 0;
    var stored = 'old'; // mutable: the rotation must be observable to the second request
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => stored,
      refresh: () async {
        refreshes++;
        await Future<void>.delayed(const Duration(milliseconds: 20)); // widen the race
        return Refreshed(stored = 'new');
      },
    );

    final responses = await Future.wait([c.dio.get('/a'), c.dio.get('/b')]);

    expect(refreshes, 1);
    expect(responses.map((r) => r.statusCode), everyElement(200));
  });

  // Invariant 2, siblings. Sabotage: remove `_epoch++` from _expireOnce.
  test('a dead refresh token tears down once; in-flight siblings go quiet', () async {
    var refreshes = 0;
    var expirations = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        await Future<void>.delayed(const Duration(milliseconds: 20));
        return const RefreshRejected();
      },
      onExpired: () async => expirations++,
    );

    final outcomes = await Future.wait([
      _outcome(c.dio.get('/a')),
      _outcome(c.dio.get('/b')),
      _outcome(c.dio.get('/c')),
    ]);

    expect(refreshes, 1);
    expect(expirations, 1);
    // One caller learns the session expired; the others are cancelled, not told three times.
    expect(outcomes..sort(), ['401', 'stale', 'stale']);
  });

  // Invariant 2, after the fact. Sabotage: delete the `if (_expired)` line in onError.
  test('a request sent after expiry is not refreshed again', () async {
    var refreshes = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        return const RefreshRejected();
      },
    );

    await expectLater(c.dio.get('/a'), _failsWith(401));
    await expectLater(c.dio.get('/b'), _failsWith(401)); // sent after the teardown

    expect(refreshes, 1);
  });

  // Invariant 2, across sessions. Sabotage: delete the body of startSession().
  test('a new session can expire again after startSession', () async {
    var expirations = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async => const RefreshRejected(),
      onExpired: () async => expirations++,
    );

    await expectLater(c.dio.get('/a'), throwsA(isA<DioException>()));
    c.auth.startSession(); // the user signs in again
    await expectLater(c.dio.get('/b'), throwsA(isA<DioException>()));

    expect(expirations, 2);
  });

  // Invariant 3. Sabotage: delete the first epoch check in onError.
  test("an old session's 401 is dropped, never replayed with the new user's token", () async {
    final arrived = Completer<void>();
    final release = Completer<void>();
    var refreshes = 0;
    var current = 'userA';
    final server = _FakeServer((o) async {
      if (o.headers['Authorization'] == 'Bearer userA') {
        arrived.complete(); // A's request is on the wire
        await release.future; // and stays there until we say so
      }
      return 401;
    });
    final c = _client(
      server: server,
      readToken: () async => current,
      refresh: () async {
        refreshes++;
        return const Refreshed('userB-refreshed');
      },
    );

    final pending = c.dio.get('/a'); // sent as user A
    await arrived.future;
    current = 'userB';
    c.auth.startSession(); // user B signs in while A's request is in flight
    release.complete(); // A's request now comes back 401

    await expectLater(pending, _failsWithError<StaleSession>());
    expect(refreshes, 0);
    expect(server.sent, ['Bearer userA']); // nothing was ever sent with B's credentials
  });

  // Invariant 3, during the refresh. Sabotage: delete the epoch check after `await refresh()`.
  test('a sign-out during the refresh call drops the request instead of replaying it', () async {
    late AuthInterceptor auth;
    final server = _FakeServer(_acceptsNew);
    final c = _client(
      server: server,
      readToken: () async => 'old',
      refresh: () async {
        auth.endSession(); // the user signs out while /auth/refresh is in flight
        return const Refreshed('new');
      },
    );
    auth = c.auth;

    await expectLater(c.dio.get('/a'), _failsWithError<StaleSession>());
    expect(server.sent, ['Bearer old']); // no replay
  });

  // Invariant 4. Sabotage: treat RefreshUnavailable like RefreshRejected.
  test('a refresh that times out keeps the session and surfaces the timeout', () async {
    var expirations = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async => RefreshUnavailable(
        DioException(
          requestOptions: RequestOptions(path: '/auth/refresh'),
          type: DioExceptionType.connectionTimeout,
        ),
      ),
      onExpired: () async => expirations++,
    );

    await expectLater(
      c.dio.get('/a'),
      throwsA(
        isA<DioException>().having((e) => e.type, 'type', DioExceptionType.connectionTimeout),
      ),
    );
    expect(expirations, 0);
  });

  // Invariant 5. Sabotage: forward `err` instead of `retryErr` in the replay catch.
  test('a replay that fails with 500 surfaces the 500, not the original 401', () async {
    var expirations = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      replayServer: _FakeServer((_) => 500),
      readToken: () async => 'old',
      refresh: () async => const Refreshed('new'),
      onExpired: () async => expirations++,
    );

    await expectLater(c.dio.get('/a'), _failsWith(500));
    expect(expirations, 0);
  });

  // Invariant 5, the 401 case. Sabotage: delete the `_expireOnce()` call in the replay catch.
  test('a replay rejected with 401 ends the session', () async {
    var expirations = 0;
    final c = _client(
      server: _FakeServer((_) => 401),
      readToken: () async => 'old',
      refresh: () async => const Refreshed('new'),
      onExpired: () async => expirations++,
    );

    await expectLater(c.dio.get('/a'), _failsWith(401));
    expect(expirations, 1);
  });

  // Invariant 6. Sabotage: make _canReplay return true.
  test('a POST without an idempotency key is refreshed but not replayed', () async {
    var refreshes = 0;
    final server = _FakeServer(_acceptsNew);
    final c = _client(
      server: server,
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        return const Refreshed('new');
      },
    );

    await expectLater(c.dio.post('/orders', data: {'sku': 'x'}), _failsWithError<ReplayRefused>());
    expect(refreshes, 1); // the caller's retry will carry a valid token
    expect(server.sent, ['Bearer old']); // and nothing was sent twice
  });

  // Invariant 6, keyed writes. Sabotage: drop the Idempotency-Key check from _canReplay.
  test('a POST with an idempotency key is replayed', () async {
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async => const Refreshed('new'),
    );

    final res = await c.dio.post(
      '/orders',
      data: {'sku': 'x'},
      options: Options(headers: {kIdempotencyKeyHeader: 'k-1'}),
    );

    expect(res.statusCode, 200);
  });

  // Invariant 6, bodies. Sabotage: drop the FormData check from _canReplay.
  test('a multipart body refreshes the token but is not replayed', () async {
    var refreshes = 0;
    final c = _client(
      server: _FakeServer(_acceptsNew),
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        return const Refreshed('new');
      },
    );

    await expectLater(
      c.dio.put(
        '/avatar',
        data: FormData.fromMap({'file': 'x'}),
      ), // PUT: idempotent, still not replayable
      _failsWithError<ReplayRefused>(),
    );
    expect(refreshes, 1);
  });

  // Sabotage: drop the kSkipAuth check in onError.
  test('a 401 on a skipAuth endpoint passes through without a refresh', () async {
    var refreshes = 0;
    final c = _client(
      server: _FakeServer((_) => 401),
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        return const Refreshed('new');
      },
    );

    await expectLater(
      c.dio.post('/auth/login', options: Options(extra: {kSkipAuth: true})),
      _failsWith(401),
    );
    expect(refreshes, 0);
  });

  test('a non-401 error passes through untouched', () async {
    var refreshes = 0;
    final c = _client(
      server: _FakeServer((_) => 500),
      readToken: () async => 'old',
      refresh: () async {
        refreshes++;
        return const Refreshed('new');
      },
    );

    await expectLater(c.dio.get('/a'), _failsWith(500));
    expect(refreshes, 0);
  });
}
```

The first test is the one that matters most and the easiest to write wrongly. Fire a single
request and `refreshes == 1` passes trivially while the concurrent case refreshes three times.
The stale-session tests are the second most important: they are the only ones whose failure
means a request went out under the wrong account. The sibling test asserts what each caller
*saw* (one 401, two `StaleSession`), not only how often the refresh ran; counting alone could not
tell rc.1's two mechanisms apart.

The `TokenRefresher` gets its own suite with a fake server as its client: 401 from
`/auth/refresh` is `RefreshRejected`, 503 and a timeout are `RefreshUnavailable`, a 200 with no
`accessToken` is `RefreshUnavailable`, a sign-out during the call makes `rotate` refuse and the
outcome `RefreshRejected`, and no session means no network call at all.

### 7.8 Widget tests

Widget tests run the real component against a stubbed notifier. Two pieces of setup make them
reliable: a `pumpApp` helper that provides what every screen assumes (ScreenUtil, the theme
extension, a `ProviderScope`), and a stub notifier that can be pushed into any state.

```dart
// test/helpers/pump_app.dart
extension PumpApp on WidgetTester {
  /// Provides what every screen assumes: a ProviderScope, ScreenUtil, the theme extension,
  /// and a Scaffold for toasts. Returns the container so a test can drive notifiers.
  Future<ProviderContainer> pumpApp(
    Widget child, {
    List<Override> overrides = const [],
    double textScale = 1.0,
    ThemeData? theme,
  }) async {
    final container = ProviderContainer(overrides: overrides);
    addTearDown(container.dispose);
    await pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: ScreenUtilInit(
          designSize: const Size(390, 844),
          builder: (_, _) => MaterialApp(
            theme: theme ?? AppTheme.light,
            home: MediaQuery.withClampedTextScaling(
              minScaleFactor: textScale,
              maxScaleFactor: textScale,
              child: Scaffold(body: child),
            ),
          ),
        ),
      ),
    );
    await pump();
    return container;
  }
}
```

```dart
class _StubLogin extends LoginNotifier {
  _StubLogin(this._initial);
  final DataState<AuthToken> _initial;

  @override
  DataState<AuthToken> build() => _initial;

  @override
  Future<void> login({
    required String email,
    required String password,
    required bool rememberMe,
  }) async {}

  void emit(DataState<AuthToken> next) => state = next;
}

void main() {
  testWidgets('shows progress while loading', (tester) async {
    await tester.pumpApp(
      const LoginScreen(),
      overrides: [loginNotifierProvider.overrideWith(() => _StubLogin(const DataState.loading()))],
    );
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('a rejected login shows the server message', (tester) async {
    final container = await tester.pumpApp(
      const LoginScreen(),
      overrides: [loginNotifierProvider.overrideWith(() => _StubLogin(const DataState.idle()))],
    );

    (container.read(loginNotifierProvider.notifier) as _StubLogin).emit(
      const DataState.error(
        exception: AppException(type: ErrorType.rejected, message: 'Incorrect email or password.'),
      ),
    );
    await tester.pump();

    expect(find.text('Incorrect email or password.'), findsOneWidget);
  });

  for (final (name, theme) in [('light', AppTheme.light), ('dark', AppTheme.dark)]) {
    testWidgets('meets accessibility guidelines ($name)', (tester) async {
      final handle = tester.ensureSemantics();
      await tester.pumpApp(
        const LoginScreen(),
        theme: theme,
        overrides: [loginNotifierProvider.overrideWith(() => _StubLogin(const DataState.idle()))],
      );

      await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
      await expectLater(tester, meetsGuideline(iOSTapTargetGuideline));
      await expectLater(tester, meetsGuideline(labeledTapTargetGuideline));
      await expectLater(tester, meetsGuideline(textContrastGuideline));
      handle.dispose();
    });
  }

  testWidgets('lays out at 2.0x text scale without overflow', (tester) async {
    await tester.pumpApp(
      const LoginScreen(),
      textScale: 2.0,
      overrides: [loginNotifierProvider.overrideWith(() => _StubLogin(const DataState.idle()))],
    );
    expect(tester.takeException(), isNull); // a RenderFlex overflow surfaces here
  });
}
```

`Override` comes from `package:flutter_riverpod/misc.dart` in Riverpod 3; it is no longer
exported by the main library. The accessibility test runs in both themes, so a dark palette
that fails contrast fails a test. Every data screen has the same set: one test per `DataState`
(with empty as its own success case), the accessibility test, and the text-scale test. A stub
notifier keeps any counters outside the class: `riverpod_lint` rejects public notifier
properties in tests too. Golden tests are optional; if you adopt
them, generate goldens on CI's OS only, because font rasterization differs across platforms.

### 7.9 Flow tests, headless and on a device

The critical flow is written once, as a function, and run twice: headless on every PR in
`test/flows/`, and on an Android emulator and an iOS simulator in the `android` and `ios` CI jobs
through `integration_test/`. The backend is a scripted `HttpClientAdapter` injected through
`httpAdapterFactoryProvider`, and secure storage is an in-memory fake. The test pumps the real
`App`, so the router, the guard, the interceptor, the refresher, and `SessionStore` all run.

```dart
/// Sign in, survive one token refresh mid-flow, sign out. Shared by the headless run in
/// test/flows/ and the device run in integration_test/.
Future<void> signInRefreshSignOut(WidgetTester tester) async {
  final backend = ScriptedBackend()
    ..on(
      'POST /auth/login',
      (_) => (
        200,
        {
          'accessToken': 't1',
          'refreshToken': 'r1',
          'user': {'id': 'u1', 'email': 'a@b.com'},
        },
      ),
    )
    ..on(
      'GET /profile',
      (o) => _bearer(o) == 't2' ? (200, {'userId': 'u1', 'name': 'Dave'}) : (401, {}),
    )
    ..on('POST /auth/refresh', (_) => (200, {'accessToken': 't2', 'refreshToken': 'r2'}))
    ..on('POST /auth/revoke', (_) => (204, null));
  final storage = InMemorySecureStorage();

  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        appConfigProvider.overrideWithValue(
          AppConfig(env: AppEnv.dev, baseUrl: Uri.parse('https://test/api')),
        ),
        httpAdapterFactoryProvider.overrideWithValue(() => backend),
        secureStorageProvider.overrideWithValue(storage),
      ],
      child: const App(),
    ),
  );
  await tester.pumpAndSettle();
  expect(find.byType(LoginScreen), findsOneWidget); // no session: the guard redirected

  await tester.enterText(find.byKey(const Key('login.email')), 'a@b.com');
  await tester.enterText(find.byKey(const Key('login.password')), 'pw');
  await tester.tap(find.byKey(const Key('login.submit')));
  await tester.pumpAndSettle();

  expect(find.byType(MainShell), findsOneWidget);
  expect(find.text('Dave'), findsOneWidget); // loaded after exactly one refresh
  expect(backend.count('POST /auth/refresh'), 1);
  expect(backend.bearers('GET /profile'), ['t1', 't2']); // the 401, then the replay
  expect(storage.values.values.single, contains('t2')); // the rotation reached disk

  await tester.tap(find.byKey(const Key('shell.signOut')));
  await tester.pumpAndSettle();

  expect(find.byType(LoginScreen), findsOneWidget);
  expect(backend.count('POST /auth/revoke'), 1);
  expect(storage.values, isEmpty);
}
```

The device entry point is four lines:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

import '../test/flows/auth_flow.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  testWidgets('sign in, survive a token refresh mid-flow, sign out', signInRefreshSignOut);
}
```

The flow asserts which token each request carried (`['t1', 't2']`), not only that the profile
appeared: a backend that returns 401 to anything but `t2` would also pass if the first request
had gone out with no bearer at all.

Beyond auth, each app picks **one** business flow whose failure would be an incident (place an
order, submit a report) and gives it the same treatment. More than two or three flows usually
means widget tests are missing.

### 7.10 Golden rule for what to test

Test **invariants**, not coverage percentages. "A request sent as user A is never replayed as
user B" is an invariant. "`AuthInterceptor` has 94% line coverage" is a number. The coverage
floors exist to catch whole files nobody tested, not to be optimized.

---
## 8. Tooling and Enforcement

The dependency rules above are only real if CI can fail on them. Five things carry that weight,
and §11.2 runs every one of them on every PR:

| Tool | Catches | Runs as |
| --- | --- | --- |
| `analysis_options.yaml` | style, relative imports, unawaited futures, unresolved imports | `dart analyze` |
| `riverpod_lint` (native plugin) | Riverpod misuse that compiles and misbehaves | `dart analyze` |
| `app_lints` (native plugin, local) | the four layering rules | `dart analyze` |
| `tool/check_lint_fixture.dart` | a rule that silently stopped firing; a plugin the app silently stopped loading | `dart run` |
| `tool/check_architecture.dart` | feature cycles, unjustified suppressions | `dart run` |

> **The single most expensive mistake in this document's history, made twice.** 1.x said
> `flutter analyze` "includes custom_lint + riverpod_lint". It did not, so every boundary rule
> ran on the author's screen and nowhere else. rc.1 fixed that by adding `dart run custom_lint`
> to CI, and that stopped working too: `custom_lint` 0.8.1 requires `analyzer ^8`, while
> `riverpod_lint` 3.x moved to the SDK's native plugin system on `analyzer` 13 to 14. The two no
> longer resolve in one `pubspec`. rc.2 moves `app_lints` to a native plugin (ADR-007).
>
> Native plugins carry the same trap in a new shape. Verified on Flutter 3.47.8 / Dart 3.13.7:
> **`dart analyze` at the package root reports plugin diagnostics; `flutter analyze` does not,
> and neither does `dart analyze <some/subdirectory>`; and a misspelled plugin path is ignored
> without a warning.** Each of those leaves CI green with zero enforcement. So CI runs
> `dart analyze` with no path argument, and `tool/check_lint_fixture.dart` plants a canary
> violation in the app itself to prove both plugins actually loaded.

### `analysis_options.yaml`

```yaml
include: package:flutter_lints/flutter.yaml

formatter:
  page_width: 100

analyzer:
  exclude:
    - "**/*.g.dart"
    - "**/*.freezed.dart"
    - test/coverage_helper_test.dart
    - tool/lints/** # its own packages, analyzed by their own CI steps
    - build/**
    - android/**
    - ios/**
  errors:
    always_use_package_imports: error # the boundary lints key off package: paths
    unawaited_futures: error # fire-and-forget deletes raced navigation in 1.x

linter:
  rules:
    always_use_package_imports: true
    unawaited_futures: true
    cancel_subscriptions: true
    close_sinks: true
    avoid_print: true
    prefer_const_constructors: true
    prefer_const_constructors_in_immutables: true
    prefer_const_literals_to_create_immutables: true
    prefer_final_fields: true
    unnecessary_this: true
    sort_pub_dependencies: true

# Native analyzer plugins (Dart 3.10+). Reported by `dart analyze` run at the package root
# with no path argument; not by `flutter analyze` (Flutter 3.47.8). Only this root file may
# configure them.
plugins:
  app_lints:
    path: tool/lints
  riverpod_lint: 3.1.9
```

`plugins:` is honored only in the root options file of the package or workspace, never in a
nested one. `app_lints` registers its rules as *warning* rules, which are on by default:
forgetting a `diagnostics:` line cannot disable them. Plugin rule severity is set in the rule's
code (`ERROR`), not in `analyzer: errors:`, and suppressions name the plugin:
`// ignore: app_lints/no_cross_feature_imports`.

`formatter: page_width: 100` matches the code in this document. Under the default 80, `dart format`
splits `if (x) return y;` one-liners across lines and `curly_braces_in_flow_control_structures`
then fails `--fatal-infos`.

An unresolved import is an analyzer **error** (`uri_does_not_exist`), so `dart analyze` fails on
it before the boundary lints are consulted. That matters because the lints skip URIs they cannot
classify rather than guessing.

### Boundary lints: the implementation, not pseudo-config

> **This is the part most architecture READMEs get wrong.** A lint rule is Dart code. YAML can
> enable a rule; it cannot define what the rule matches. A config block that appears to declare
> `from` and `to` path globs is silently ignored unless code you wrote reads it.

The rules live in a local package at `tool/lints`, a native analyzer plugin. The analysis
server builds it from the path in `analysis_options.yaml`; the app does not depend on it.

```yaml
name: app_lints
publish_to: none
environment:
  sdk: ^3.13.0
dependencies:
  analysis_server_plugin: ^0.3.25
  analyzer: ^14.6.0
```

Every rule has the same shape: a file at location A may not import a URI at location B. The
base class does the plumbing once; each rule is a predicate.

```dart
// tool/lints/lib/main.dart
import 'package:analysis_server_plugin/plugin.dart';
import 'package:analysis_server_plugin/registry.dart';
import 'package:analyzer/analysis_rule/analysis_rule.dart';
import 'package:analyzer/analysis_rule/rule_context.dart';
import 'package:analyzer/analysis_rule/rule_visitor_registry.dart';
import 'package:analyzer/dart/ast/ast.dart';
import 'package:analyzer/dart/ast/visitor.dart';
import 'package:analyzer/error/error.dart';

final plugin = AppLintsPlugin();

class AppLintsPlugin extends Plugin {
  @override
  String get name => 'app_lints';

  @override
  void register(PluginRegistry registry) {
    // Warning rules are on by default: forgetting a config line cannot silently disable them.
    registry
      ..registerWarningRule(LayerBoundary())
      ..registerWarningRule(NoCrossFeatureImports())
      ..registerWarningRule(SharedLayerIndependence())
      ..registerWarningRule(DomainPurity());
  }
}

abstract class _ImportRule extends AnalysisRule {
  _ImportRule(this.diagnosticCode, String description)
    : super(name: diagnosticCode.lowerCaseName, description: description);

  @override
  final LintCode diagnosticCode;

  /// True when [self] importing [target] breaks this rule. [pkg] is this app's package name.
  bool violates(Uri self, Uri target, String pkg);

  @override
  void registerNodeProcessors(RuleVisitorRegistry registry, RuleContext context) {
    final visitor = _DirectiveVisitor(this, context);
    registry
      ..addImportDirective(this, visitor)
      ..addExportDirective(this, visitor); // re-exporting an internal is importing it
  }
}

class _DirectiveVisitor extends SimpleAstVisitor<void> {
  _DirectiveVisitor(this.rule, this.context);

  final _ImportRule rule;
  final RuleContext context;

  @override
  void visitImportDirective(ImportDirective node) => _check(node);

  @override
  void visitExportDirective(ExportDirective node) => _check(node);

  void _check(NamespaceDirective node) {
    final self = context.libraryElement?.uri;
    if (self == null || self.scheme != 'package' || self.pathSegments.isEmpty) return;
    final pkg = self.pathSegments.first;

    // Conditional imports carry extra URIs; each one is a real dependency.
    final raws = [node.uri.stringValue, for (final c in node.configurations) c.uri.stringValue];
    for (final raw in raws.whereType<String>()) {
      // Resolve against this library so relative and package: forms compare equal.
      if (rule.violates(self, self.resolve(raw), pkg)) {
        rule.reportAtNode(node);
        return;
      }
    }
  }
}

LintCode _code(String name, String message) =>
    LintCode(name, message, severity: DiagnosticSeverity.ERROR);

/// (feature, layer) for package:<pkg>/features/<feature>/<layer>/..., else null.
/// For the feature's root file (features/<feature>/<feature>.dart) the "layer" is that file name.
(String, String)? _slice(Uri u, String pkg) {
  final s = u.pathSegments;
  if (u.scheme != 'package' || s.length < 4 || s[0] != pkg || s[1] != 'features') return null;
  return (s[2], s[3]);
}

/// The top-level folder under lib/ (core, common, features, ...), for this package only.
String? _area(Uri u, String pkg) {
  final s = u.pathSegments;
  return u.scheme == 'package' && s.length > 1 && s[0] == pkg ? s[1] : null;
}

/// Inside one feature, presentation/ and data/ meet only at domain/.
class LayerBoundary extends _ImportRule {
  LayerBoundary()
    : super(
        _code(
          'layer_boundary',
          'presentation/ and data/ meet only at the domain/ contract; neither imports the other.',
        ),
        'Same-feature presentation and data layers may not import each other.',
      );

  @override
  bool violates(Uri self, Uri target, String pkg) {
    final from = _slice(self, pkg);
    final to = _slice(target, pkg);
    if (from == null || to == null || from.$1 != to.$1) return false;
    return (from.$2 == 'presentation' && to.$2 == 'data') ||
        (from.$2 == 'data' && to.$2 == 'presentation');
  }
}

/// True for package:<pkg>/features/<feature>/<file>.dart: the feature's public API file.
bool _isFeatureRoot(Uri u, String pkg) {
  final s = u.pathSegments;
  return u.scheme == 'package' && s.length == 4 && s[0] == pkg && s[1] == 'features';
}

/// Another feature's public surface is its domain/ and its root file; everything else is private.
class NoCrossFeatureImports extends _ImportRule {
  NoCrossFeatureImports()
    : super(
        _code(
          'no_cross_feature_imports',
          "Another feature's data/ and presentation/ are private. Depend on its domain/ "
              'or its root <feature>.dart, or move the shared piece to common/ or core/.',
        ),
        "Features may import only another feature's domain/ and root file.",
      );

  @override
  bool violates(Uri self, Uri target, String pkg) {
    final from = _slice(self, pkg);
    final to = _slice(target, pkg);
    return from != null &&
        to != null &&
        from.$1 != to.$1 &&
        to.$2 != 'domain' &&
        !_isFeatureRoot(target, pkg);
  }
}

/// core/ and common/ are below every feature and may not reach up into one.
class SharedLayerIndependence extends _ImportRule {
  SharedLayerIndependence()
    : super(
        _code(
          'shared_layer_independence',
          'core/ and common/ may not import a feature. Move the dependency down, '
              'or move this file into the feature.',
        ),
        'core/ and common/ may not depend on features/.',
      );

  @override
  bool violates(Uri self, Uri target, String pkg) =>
      const {'core', 'common'}.contains(_area(self, pkg)) && _area(target, pkg) == 'features';
}

/// domain/ is platform- and transport-independent: an allowlist, not a denylist.
class DomainPurity extends _ImportRule {
  DomainPurity()
    : super(
        _code(
          'domain_purity',
          'domain/ may import only dart:core/async/collection/math/typed_data, '
              "package:meta, package:collection, core/result/, and other features' domain/.",
        ),
        'domain/ depends only on an allowlist.',
      );

  static const _dart = {'core', 'async', 'collection', 'math', 'typed_data'};
  static const _packages = {'meta', 'collection'};

  @override
  bool violates(Uri self, Uri target, String pkg) {
    if (_slice(self, pkg)?.$2 != 'domain') return false;
    if (target.scheme == 'dart') return !_dart.contains(target.path);
    if (target.scheme != 'package' || target.pathSegments.isEmpty) return false;
    final s = target.pathSegments;
    if (s.first != pkg) return !_packages.contains(s.first);
    final to = _slice(target, pkg);
    if (to != null) return to.$2 != 'domain';
    return !(s.length > 2 && s[1] == 'core' && s[2] == 'result');
  }
}
```

About a hundred and seventy lines, written once, and every PR that points an import arrow the
wrong way goes red before review.

**Why textual URIs resolved against the library, not the element model.** The obvious version
does `path.contains('/data/')` on the raw import string. That breaks four ways: a directory named
`metadata` or a package named `mydata` matches by substring; a relative import never contains the
package path; Windows paths use backslashes; and conditional imports hide a second URI. Resolving
each URI string against the library's own `package:` URI with `Uri.resolve` normalises relative
and absolute forms to the same value, and comparing path *segments* removes the substring
hazard. The rules touch only AST directives, `Uri`, and `context.libraryElement?.uri`, the
smallest surface of an API that is still pre-1.0.

**The rules allow what §1 allows, and that alignment is the point.** A lint stricter than its
own document is the worst of both: the first legitimate `orders -> billing/domain` import gets
an `ignore:` comment, and once one `ignore:` is in the codebase the rule is advisory. The reverse
is just as bad, and rc.1 had it: its worked example imported `data/` from `presentation/`, which
its own lint forbade. Running the rules over the reference app is what found it.

### The fixture and the canary: proving the rules fire

A lint rule that matches nothing reports nothing, and a rule that reports nothing looks exactly
like a codebase with no violations. `tool/lints/fixture/` is a tiny Dart package that enables
the plugin and contains every case written out. Native plugins have no `expect_lint`, so the
fixture marks expectations with `// expect: <rule>` on the line above, and
`tool/check_lint_fixture.dart` compares them with `dart analyze --format=machine`: every
expectation must fire, nothing else may, and every rule must fire at least once.

```yaml
# The fixture imports things it never uses, on purpose. Only plugin diagnostics matter here.
analyzer:
  errors:
    unused_import: ignore
    duplicate_import: ignore

plugins:
  app_lints:
    path: ..
```

```dart
// tool/lints/fixture/lib/features/orders/presentation/imports.dart
// expect: layer_boundary
import 'package:fixture/features/orders/data/orders_dto.dart';

// expect: layer_boundary
import '../data/orders_dto.dart'; // relative form resolves to the same URI

import 'package:fixture/features/orders/domain/order.dart'; // allowed
// expect: no_cross_feature_imports
import 'package:fixture/features/billing/data/invoice_dto.dart';
// expect: no_cross_feature_imports
import 'package:fixture/features/billing/presentation/billing_notifier.dart';
// expect: no_cross_feature_imports
import 'package:fixture/core/result/result.dart'
    if (dart.library.io) 'package:fixture/features/billing/data/invoice_dto.dart';
import 'package:fixture/features/billing/domain/invoice.dart'; // allowed: public contract
import 'package:fixture/features/billing/billing.dart'; // allowed: public API file
import 'package:fixture/common/widgets/button.dart'; // allowed
import 'package:fixture/metadata/version.dart'; // allowed: not a substring match

import 'dart:async'; // allowed

// expect: no_cross_feature_imports
export 'package:fixture/features/billing/data/invoice_dto.dart';
```

```dart
// tool/lints/fixture/lib/features/billing/domain/invoice.dart
// expect: domain_purity
import 'dart:io';

// expect: domain_purity
import 'package:fixture/core/network/http_client.dart';
// expect: domain_purity
import 'package:fixture/features/billing/data/invoice_dto.dart';
// expect: domain_purity
import 'package:fixture/features/billing/billing.dart'; // the root wires data, so domain may not
import 'package:fixture/core/result/result.dart'; // allowed
import 'package:fixture/features/orders/domain/order.dart'; // allowed

import 'dart:collection'; // allowed

class Invoice {}
```

The fixture also proves a suppression silences the rule (an `ignore:` that did not work would
show up as an unexpected diagnostic):

```dart
// tool/lints/fixture/lib/features/legacy/presentation/suppressed.dart
// A justified exception must silence the rule; if it does not, the checker sees an extra diagnostic.
// ADR-012: the legacy importer reads billing DTOs until the v3 API ships.
// ignore: app_lints/no_cross_feature_imports
import 'package:fixture/features/billing/data/invoice_dto.dart';
```

The fixture proves the rules; it cannot prove the *app* loads them. So the same script then
writes a two-file canary under `lib/features/zz_lint_canary/` (a `presentation/` file importing
its `data/`, and a notifier with a public field), runs `dart analyze` on the whole app, requires
both `layer_boundary` and `avoid_public_notifier_properties` on the canary, and deletes it. With
the plugin path misspelled, both disappear (one plugin failing to load took the other down with
it in testing), and the check fails.

```dart
/// Plants one violation per plugin in this app, analyzes only that folder, removes it.
List<String> _canary() {
  final pkg = RegExp(
    r'^name:\s*(\S+)',
    multiLine: true,
  ).firstMatch(File('pubspec.yaml').readAsStringSync())!.group(1)!;
  final dir = Directory('lib/features/zz_lint_canary');
  try {
    File('${dir.path}/data/source.dart')
      ..createSync(recursive: true)
      ..writeAsStringSync('const int canary = 0;\n');
    File('${dir.path}/presentation/canary_notifier.dart')
      ..createSync(recursive: true)
      ..writeAsStringSync('''
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:$pkg/features/zz_lint_canary/data/source.dart';

class CanaryNotifier extends Notifier<int> {
  int exposed = canary;

  @override
  int build() => exposed;
}
''');
    // The whole package, no path argument: analyzing a subdirectory skips plugins entirely.
    final run = Process.runSync('dart', ['analyze', '--format=machine']);
    final out = '${run.stdout}${run.stderr}'
        .split('\n')
        .where((l) => l.contains('zz_lint_canary'))
        .join('\n')
        .toLowerCase();
    return [
      if (!out.contains('|layer_boundary|'))
        'app_lints did not load (check plugins: in analysis_options.yaml)',
      if (!out.contains('|avoid_public_notifier_properties|')) 'riverpod_lint did not load',
    ];
  } finally {
    if (dir.existsSync()) dir.deleteSync(recursive: true);
  }
}
```

**When a boundary lint is wrong.** Reproduce it in the fixture first: add the import that
misfired with the expectation you believe is right, and run `dart run tool/check_lint_fixture.dart`.
If the fixture agrees with you, fix the predicate and keep the new fixture line, so the false
positive cannot come back. If the fixture agrees with the lint, the import is the problem; if it
is a justified exception, §12.2 applies.

### `tool/check_architecture.dart`: what a per-file linter cannot see

A plugin rule sees one library at a time, so it cannot see a cycle that spans files, and it
cannot judge whether a suppression was justified. This script does both, with no dependencies.

```dart
// 1. The feature dependency graph is acyclic.
// 2. Every suppression of an architecture rule cites an ADR on the line above it.
// Run from the package root: dart run tool/check_architecture.dart
import 'dart:io';

const _rules = [
  'layer_boundary',
  'no_cross_feature_imports',
  'shared_layer_independence',
  'domain_purity',
];
final _directive = RegExp(r'''^\s*(?:import|export)\s+['"]([^'"]+)['"]''', multiLine: true);
final _suppression = RegExp('ignore(?:_for_file)?:.*\\bapp_lints/(?:${_rules.join('|')})\\b');
final _adr = RegExp(r'^\s*//\s*ADR-\d+');

void main() {
  final pubspec = File('pubspec.yaml').readAsStringSync();
  final pkg = RegExp(r'^name:\s*(\S+)', multiLine: true).firstMatch(pubspec)!.group(1)!;
  final graph = <String, Set<String>>{};
  final problems = <String>[];

  final files = Directory('lib')
      .listSync(recursive: true)
      .whereType<File>()
      .where((f) => f.path.endsWith('.dart'));

  for (final file in files) {
    final rel = file.path.replaceAll(r'\', '/').substring('lib/'.length);
    final self = Uri.parse('package:$pkg/$rel');
    final source = file.readAsStringSync();

    final from = _feature(self, pkg);
    if (from != null) {
      final edges = graph.putIfAbsent(from, () => <String>{});
      for (final m in _directive.allMatches(source)) {
        final to = _feature(self.resolve(m.group(1)!), pkg);
        if (to != null && to != from) edges.add(to);
      }
    }

    final lines = source.split('\n');
    for (var i = 0; i < lines.length; i++) {
      if (_suppression.hasMatch(lines[i]) && (i == 0 || !_adr.hasMatch(lines[i - 1]))) {
        problems.add(
          '${file.path}:${i + 1}: architecture rule suppressed without '
          'an "// ADR-NNN: reason" line directly above it (§12.2)',
        );
      }
    }
  }

  problems.addAll(_cycles(graph));
  if (problems.isEmpty) {
    stdout.writeln('architecture: ok (${graph.length} features, no cycles)');
    return;
  }
  problems.forEach(stderr.writeln);
  exitCode = 1;
}

String? _feature(Uri u, String pkg) {
  final s = u.pathSegments;
  return u.scheme == 'package' && s.length > 3 && s[0] == pkg && s[1] == 'features' ? s[2] : null;
}

/// Depth-first search. Reports each back edge as a readable path: "orders -> billing -> orders".
List<String> _cycles(Map<String, Set<String>> graph) {
  final state = <String, int>{}; // 1 = on the stack, 2 = finished
  final stack = <String>[];
  final found = <String>[];

  void visit(String node) {
    state[node] = 1;
    stack.add(node);
    for (final next in graph[node] ?? const <String>{}) {
      if (state[next] == 1) {
        found.add('feature cycle: ${[...stack.sublist(stack.indexOf(next)), next].join(' -> ')}');
      } else if (state[next] == null) {
        visit(next);
      }
    }
    stack.removeLast();
    state[node] = 2;
  }

  for (final node in graph.keys) {
    if (state[node] == null) visit(node);
  }
  return found;
}
```

The import regex does not see the second URI of a conditional import. That is an accepted gap:
conditional imports between features are vanishingly rare, and `no_cross_feature_imports` still
checks both URIs per file.

### `riverpod_lint`

Runs through the same analyzer as a native plugin, pinned by version in `analysis_options.yaml`,
and catches Riverpod mistakes that compile cleanly and misbehave at runtime. Its rules are on by
default; the canary relies on `avoid_public_notifier_properties`: a `Notifier`'s only public
surface is its commands and `state`, because public mutable fields do not trigger rebuilds. That
is the §6 "public methods are commands" rule, machine-checked.

> **Version note.** The native plugin API (`analysis_server_plugin`) is pre-1.0 and moves with
> the SDK. Plugins require Dart 3.10 (Flutter 3.38) or later. When the SDK pin moves, the
> fixture run and the canary are what prove the plugins still load; ADR-007 records when to
> revisit.

Between the boundary lints (structure), the architecture script (graph), the analyzer
(imports, futures), and `riverpod_lint` (state), the conventions hold even on a branch nobody
reviewed closely.

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
| Feature root | `features/<feature>/<feature>.dart` | `<feature>RepositoryProvider` + exports (ADR-009) |
| Notifier | `<thing>_notifier.dart` | `<Thing>Notifier` + `<thing>NotifierProvider` |
| Screen | `<name>_screen.dart` | `<Name>Screen` with `static const String routeName` |
| Test | mirrors the file under test, `_test.dart` | `test/features/auth/data/auth_repository_impl_test.dart` |
| ADR | `docs/adr/NNN-<slug>.md` | `ADR-NNN` in prose and in suppression comments |
| Suppression | `// ignore: app_lints/<rule>` | the line above cites the ADR (§12.2) |

**Providers:** `camelCase` ending in `Provider`. One provider declared next to the thing it
exposes, except repository providers, which live in the feature's root file so they can see
`data/` while presentation cannot.

**Constants:** global constants prefixed `k` (`kSession`, `kSkipAuth`). Private file-level
constants start with `_k`.

**Strings, colors, dimensions:** never inline. Copy via `Strings.x`, colors via
`context.xColor`, sizes via `Dimensions.pxNN`.

**Imports** (ordered, blank line between groups):

1. Dart SDK (`dart:async`)
2. Flutter and third-party packages
3. Local `package:` imports

Always `package:` imports inside `lib/`, never `../../` relatives. Absolute paths make the layer
a file lives in obvious from its import line and survive file moves. Enforced, not preferred.

**Style:** `const` constructors wherever possible; `required` named parameters; immutable state
with manual `copyWith`; expression bodies for one-liners; comments that explain **why**, not
what. Prefer Dart 3 pattern matching over `is` checks and casts when consuming `Result` or
`DataState`.

---

## 10. New Feature Checklist

Adding a networked feature `<feature>`. Start from the template:

```bash
dart run tool/new_feature.dart notes
```

It writes ten files, everything the steps below call for in its simplest form: a domain model
and contract, a DTO, a Retrofit service, the repository with `guard`, the root file with the
provider, a Pattern 1 notifier, a screen that renders all four states, and repository and widget
tests (four states, 2.0x text, accessibility). CI renders the template under a throwaway name,
runs `build_runner`, analyzes it with both plugins, and runs its tests on every PR, so the
template cannot rot into code that violates the rules it is meant to teach. Then adapt it:

```
lib/features/notes/
  notes.dart                                  # root: notesRepositoryProvider + exports
  domain/models/notes_item.dart
  domain/repository/notes_repository.dart
  data/dto/notes_item_dto.dart
  data/services/notes_service.dart            # run build_runner next
  data/repository/notes_repository_impl.dart
  presentation/notifier/notes_notifier.dart   # Pattern 1, load()
  presentation/screens/notes_screen.dart      # loading / success / empty / error + retry
test/features/notes/
  data/notes_repository_impl_test.dart        # Ok, Err(server), changed JSON shape
  presentation/notes_screen_test.dart         # four states, 2.0x text, accessibility
```

The steps, for what the template cannot decide for you:

1. **Folders**: `dart run tool/new_feature.dart <feature>`, or by hand: `features/<feature>/{data/{dto,services,source,repository},domain/{models,repository},presentation/{notifier,screens,components}}`.
2. **Domain models** (`domain/models/`): plain Dart, immutable, `const`, `required` fields. No JSON.
3. **Repository contract** (`domain/repository/`): `abstract interface class` returning `Future<Result<T>>` for networked operations the UI surfaces, plain `Future<T>` for local or fire-and-forget.
4. **DTOs** (`data/dto/`): plain classes with `fromJson`, `toJson`, `toDomain()`.
5. **Service** (`data/services/`): Retrofit `@RestApi` abstract class with `part '<feature>_service.g.dart'`. `@Extra({kSkipAuth: true})` on every endpoint that authenticates by body. `@Header(kIdempotencyKeyHeader)` on every non-idempotent write that must survive a refresh.
6. **build_runner**: `dart run build_runner build --delete-conflicting-outputs`, and commit the output.
7. **Local source** (`data/source/`), only if the feature caches or persists. It belongs to this repository and nothing else may touch it. Never secure storage: that is the session's.
8. **Repository impl** (`data/repository/`): `with RepositoryGuard implements <Feature>Repository`. Use `guard(...)`; use explicit try/catch plus `mapError` where there are side effects to undo, and catch non-Dio throws too.
9. **Root file** (`features/<feature>/<feature>.dart`): declare `<feature>RepositoryProvider` typed by the interface, reading `dioProvider`; `ref.watch(authStateProvider)` there if the repository caches per-user data. Export the domain types other features may use.
10. **Notifier** (`presentation/notifier/`): import the root file for the repository provider. **Default to Pattern 1**. Pattern 2 only for state seeded from storage or a stream. Pattern 3 only when all three gate conditions in §5 hold. `if (!ref.mounted) return;` after every await. `ref.watch(authStateProvider)` inside `build()` if it holds per-user data.
11. **Screen** (`presentation/screens/`): thin view with `static const String routeName`; compose components; kick off loads in a post-frame callback guarded by the idle state.
12. **Components** (`presentation/components/`): every non-trivial widget. Four states including empty. `ref.listen` for navigation and toasts, `errorToMessage(exception)` for copy. Semantics labels on icon-only controls.
13. **Strings and assets**: add copy to `Strings`, paths to `assets.dart`. No inline literals.
14. **Route and guard**: register in `router.dart`; wrap protected routes in `AuthGuard`.
15. **Shell**: if it is a tab, add it to `main_shell.dart`'s `IndexedStack`.
16. **Tests**: repository (`Ok`, `Err`, non-Dio throw, cache if any); notifier (transitions, double-tap); widget (four states, accessibility, 2.0x text). If it is the app's critical business flow, a flow test. Any test that pins an invariant gets a `// Sabotage:` comment and an entry in `tool/check_sabotage.dart`.
17. **Verify locally**: `dart analyze --fatal-infos` (not `flutter analyze`), `dart run tool/check_architecture.dart`, `flutter test`. CI runs the same and more.
18. **Deviation?** If any rule here does not fit, write the ADR first (§12.1). An `ignore:` without one fails CI.

**Rules that never bend.** DTOs stay in `data`. Domain models stay platform-independent.
Presentation depends only on the domain interface, reached through the feature's root file. Repositories return `Result` and are the
single source of truth for their data, including cache. Notifiers own `DataState`, are the only
place `loading` and `idle` exist, and check `ref.mounted` after every await. Errors travel as
`AppException` and become strings exactly once, at the UI edge, via `errorToMessage`. Only
`SessionStore` touches credentials. Only a server verdict ends a session; a network failure
never does. Nothing is sent twice unless it is safe to send twice. A test that pins an invariant
must be able to fail, and a machine checks that it can. Views orchestrate,
components render. Imports are `package:` and never cross a boundary the lints forbid.

---
## 11. Production and Release

Everything above is application architecture. This section is the operational layer: the
difference between clean code and a shippable release. Each subsection ends in something a
machine or a named person checks.

### 11.1 Environments and flavors

Config comes from the build, not from a committed constant. Editing source and recompiling to
switch environments means staging and production are different artifacts.

- **Build flavors**: Android `productFlavors` (`dev`, `staging`, `prod`) give distinct application IDs and names, so all three can sit on one device. Names go through `manifestPlaceholders`, not `resValue`: on the current Android Gradle Plugin, `resValue` in a flavor fails the build unless `buildFeatures.resValues` is turned on, which the prod bundle build caught. iOS schemes per flavor are added when iOS needs them; until then the iOS job builds the single scheme with `--dart-define-from-file`.
- **`--dart-define-from-file`**: per-environment JSON supplies values at build time, read by `AppConfig.fromEnvironment()` (§3).
- **Cleartext only in dev.** The dev base URL is `http://` on a LAN, so `android:usesCleartextTraffic="true"` goes in `android/app/src/dev/AndroidManifest.xml`, never in `src/main`. CI checks it.

```json
// config/prod.json  (git-ignored if it carries anything sensitive)
{
  "APP_ENV": "prod",
  "BASE_URL": "https://api.example.com/v1",
  "PUBLIC_SENTRY_DSN": ""
}
```

```bash
flutter build appbundle --flavor prod --dart-define-from-file=config/prod.json \
  --obfuscate --split-debug-info=build/symbols
```

**A dart-define is not a secret store.** Defines are compiled into the binary and are
recoverable from it. They are correct for base URLs, DSNs, public keys, and feature flags. They
are wrong for anything whose disclosure matters: those belong on your server, behind an
authenticated endpoint. Keys that are public by design are prefixed `PUBLIC_`; the release
check below rejects any other key whose name looks like a secret.

```dart
// tool/check_release_config.dart
// Usage: dart run tool/check_release_config.dart config/staging.json config/prod.json
// Fails on a release config that AppConfig would reject at boot, on a define that looks like
// a secret, and on an Android manifest that would leak the session into backups.
import 'dart:convert';
import 'dart:io';

final _secretish = RegExp(
  r'SECRET|PRIVATE|PASSWORD|API_?KEY|TOKEN|CREDENTIAL',
  caseSensitive: false,
);

/// Every SharedPreferences file flutter_secure_storage 11 writes (§3). Re-check on a major bump.
const _secureStorageFiles = [
  'FlutterSecureStorage.xml',
  'FlutterSecureKeyStorage.xml',
  'FlutterSecureStorageConfiguration.xml',
];

void main(List<String> configFiles) {
  final problems = <String>[];

  for (final path in configFiles) {
    final json = jsonDecode(File(path).readAsStringSync()) as Map<String, dynamic>;
    final env = json['APP_ENV'];
    final url = Uri.tryParse('${json['BASE_URL'] ?? ''}');
    if (env != 'staging' && env != 'prod') {
      problems.add('$path: APP_ENV must be staging or prod, got "$env"');
    }
    if (url == null || url.scheme != 'https' || !url.hasAuthority) {
      problems.add('$path: BASE_URL must be an absolute https URL');
    }
    for (final key in json.keys) {
      if (_secretish.hasMatch(key) && !key.startsWith('PUBLIC_')) {
        problems.add('$path: "$key" looks like a secret; defines are readable from the binary');
      }
    }
  }

  final manifest = File('android/app/src/main/AndroidManifest.xml');
  if (manifest.existsSync()) {
    final xml = manifest.readAsStringSync();
    if (!xml.contains('android:allowBackup="false"')) {
      problems.add('main AndroidManifest: android:allowBackup="false" is required (§3)');
    }
    if (!xml.contains('android:dataExtractionRules="@xml/data_extraction_rules"')) {
      problems.add('main AndroidManifest: dataExtractionRules is required for Android 12+ (§3)');
    }
    if (xml.contains('android:usesCleartextTraffic="true"')) {
      problems.add('main AndroidManifest: cleartext traffic belongs in the dev flavor only');
    }

    final rules = File('android/app/src/main/res/xml/data_extraction_rules.xml');
    final text = rules.existsSync() ? rules.readAsStringSync() : '';
    for (final section in ['cloud-backup', 'device-transfer']) {
      final body = RegExp('<$section>([\\s\\S]*?)</$section>').firstMatch(text)?.group(1) ?? '';
      for (final file in _secureStorageFiles) {
        if (!body.contains('path="$file"')) {
          problems.add('data_extraction_rules.xml: <$section> must exclude $file');
        }
      }
    }
  }

  if (problems.isEmpty) {
    stdout.writeln('release config: ok');
    return;
  }
  problems.forEach(stderr.writeln);
  exitCode = 1;
}
```

### 11.2 CI pipeline

The lints and tests only mean something if a machine runs them on every PR, and if the PR
cannot merge when they fail. Configure branch protection on `main` to require the `verify`,
`security`, `android`, and `ios` jobs. A PR must not pass merely because the code compiles.

```yaml
# .github/workflows/ci.yaml
name: ci
on:
  pull_request:
  push: { branches: [main] }

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  FLUTTER_VERSION: '3.47.8' # §12.3; bumped deliberately, in its own PR

# Supply chain: in the committed file every third-party action is pinned to a full commit
# SHA with the tag in a trailing comment (uses: owner/action@<40-char sha> # v4).
# Tags are shown here for readability. Dependabot proposes SHA updates.
jobs:
  verify:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true

      - run: flutter --version # recorded in the log, so a build is reproducible
      - name: Dependencies (app, lint plugin, lint fixture)
        run: |
          flutter pub get --enforce-lockfile
          (cd tool/lints && dart pub get --enforce-lockfile)
          (cd tool/lints/fixture && dart pub get --enforce-lockfile)
      - run: dart format --output=none --set-exit-if-changed lib test tool integration_test

      - name: Generated code is committed and current
        run: |
          dart run build_runner build --delete-conflicting-outputs
          if [ -n "$(git status --porcelain)" ]; then
            git status --porcelain
            echo "::error::Generated output differs from the commit (changed, missing, or hand-edited). Run build_runner and commit the result."
            exit 1
          fi

      # `dart analyze`, at the package root, with no path argument. On Flutter 3.47.8,
      # `flutter analyze` and `dart analyze <subdir>` both skip analyzer plugins silently.
      - name: Analyzer, boundary lints, riverpod_lint
        run: dart analyze --fatal-infos
      - name: Lint plugin package
        working-directory: tool/lints
        run: dart analyze --fatal-infos

      - name: Plugins are alive (fixture + root canary)
        run: dart run tool/check_lint_fixture.dart
      - name: Feature cycles and justified suppressions
        run: dart run tool/check_architecture.dart
      - name: Release configuration
        run: dart run tool/check_release_config.dart config/staging.json config/prod.json

      - name: Tests with coverage
        run: |
          tool/gen_coverage_helper.sh
          flutter test --coverage
      - name: Coverage floors
        run: dart run tool/check_coverage.dart
      - name: Every invariant test fails under its named sabotage
        run: dart run tool/check_sabotage.dart
      - name: Feature template renders, analyzes, and passes its tests
        run: dart run tool/new_feature.dart --check

  security:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 } # full history: a secret removed in a later commit is still leaked
      - name: Secrets in history
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Known-vulnerable dependencies (OSV, reads pubspec.lock)
        run: |
          go install github.com/google/osv-scanner/v2/cmd/osv-scanner@v2.7.0
          "$(go env GOPATH)/bin/osv-scanner" scan --lockfile=pubspec.lock

  android:
    runs-on: ubuntu-24.04
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: '21' }
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true
      - run: flutter pub get --enforce-lockfile
      - name: Prod release bundle (obfuscated, symbols split)
        run: |
          flutter build appbundle --flavor prod --release \
            --dart-define-from-file=config/prod.json \
            --obfuscate --split-debug-info=build/symbols
      - name: Enable KVM for the emulator
        run: |
          echo 'KERNEL=="kvm", GROUP="kvm", MODE="0666", OPTIONS+="static_node=kvm"' \
            | sudo tee /etc/udev/rules.d/99-kvm4all.rules
          sudo udevadm control --reload-rules
          sudo udevadm trigger --name-match=kvm
      - name: Critical flow on an emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          arch: x86_64
          script: flutter test integration_test --flavor dev --dart-define-from-file=config/dev.json

  ios:
    runs-on: macos-15
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: ${{ env.FLUTTER_VERSION }}
          channel: stable
          cache: true
      - run: flutter pub get --enforce-lockfile
      - name: Release build compiles for device (unsigned)
        run: flutter build ios --release --no-codesign --dart-define-from-file=config/prod.json
      - name: Critical flow on a simulator
        run: |
          UDID=$(xcrun simctl list devices available -j | python3 -c "import json,sys; print(next(d['udid'] for r in json.load(sys.stdin)['devices'].values() for d in r if d['name'].startswith('iPhone')))")
          xcrun simctl boot "$UDID"
          flutter test integration_test -d "$UDID" --dart-define-from-file=config/dev.json
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: pub
    directory: /
    schedule: { interval: weekly }
    groups:
      minor-and-patch:
        update-types: [minor, patch]
```

**What each step closes.**

- **`dart analyze`, at the root, not `flutter analyze`.** The only command that reported plugin diagnostics in testing (§8). Without it the boundary lints and `riverpod_lint` do not run in CI at all.
- **`check_lint_fixture.dart`.** Proves the rules still fire after an SDK or analyzer move, and that the app's own config loads both plugins. A misspelled plugin path is otherwise silent.
- **`git status --porcelain`, not `git diff --exit-code`.** `git diff` ignores untracked files, so a new `.g.dart` that was never committed would pass. `status` sees changed, deleted, and untracked files.
- **The coverage helper and `check_coverage.dart`.** Without the helper, untested files are absent from `lcov.info` and the overall figure is inflated.
- **`check_sabotage.dart`.** Proves every invariant test can fail. About ninety seconds on the reference app.
- **`new_feature.dart --check`.** Renders the feature template, generates its code, analyzes it with both plugins, and runs its tests. A template that no longer passes the rules would teach every new feature to break them.
- **OSV pinned to `v2.7.0`.** rc.1 shipped a placeholder version (`v2.x.y`), which fails the job as written. A placeholder in a config this document calls "given in full" is a bug.
- **Three pins for reproducibility.** The SDK version, `pubspec.lock` enforced with `--enforce-lockfile` (for the app, the plugin, and the fixture), and `app_lints` by path inside the repo. Miss one and "CI passed" stops being a statement about this commit.

Release builds run on tags in a separate workflow: `flutter build appbundle --flavor prod` and
`flutter build ipa`, both with obfuscation and split debug info; symbol upload to
the crash reporter; signing from the CI secret store; upload to the internal testing track.
The release workflow re-runs `verify` on the tagged commit rather than trusting the PR run.

### 11.3 Release hardening

- **Obfuscation and symbolication**: build with `--obfuscate --split-debug-info=build/symbols`, archive the symbol files per build number, and upload them to the crash reporter so stack traces stay readable. A release whose symbols were not uploaded is a release you cannot debug.
- **Transport security**: HTTPS is enforced at boot (§3) and in CI (§11.1). For a high-value target, add certificate pinning in `httpAdapterFactoryProvider`; because every client is built by `buildDio`, the refresh and replay requests are pinned too. Pin to an intermediate CA or include a backup key alongside the leaf, or a certificate rotation bricks every installed app.
- **Store shrinking**: keep R8 and resource shrinking on for Android release. JSON is code-generated, so there is usually no reflection path to keep.
- **No secrets in logs or reports**: the logging interceptor is `kDebugMode`-only and redacts auth; the reporting interceptor sends `NetworkFailure`, never a raw `DioException`; the vendor SDK's scrubbing hook is configured as a second line.
- **Backup exclusion**: `allowBackup="false"` and `dataExtractionRules` (§3), checked in CI.

### 11.4 Production readiness gates

A release candidate ships when every row has its evidence. "Evidence" means something a person
other than the author can look at: a CI job, a test, a dashboard, a signed-off checklist in the
release ticket.

**Security and privacy**

| Requirement | Acceptance | Evidence |
| --- | --- | --- |
| No secrets in the binary | Release configs pass the secret-name check; no credential in source | `check_release_config`, gitleaks job |
| Sensitive data out of logs and reports | Logging is debug-only and redacted; reports are `NetworkFailure`; SDK scrubbing on | code review of §3 interceptors, one forced error inspected in the dashboard per release |
| Dependency and secret scanning | OSV and gitleaks pass; findings triaged within a week | `security` job |
| Session expiry and sign-out | All six interceptor invariants and the `SessionStore` contract hold | §7.5 and §7.7 suites, integration auth flow |

**Reliability**

| Requirement | Acceptance | Evidence |
| --- | --- | --- |
| Offline and poor network defined | Every data screen shows the offline message with retry; no infinite spinner; nothing logs the user out | widget error-state tests; manual run with network link conditioner on a real device |
| Timeout, retry, cancellation documented | Timeouts as in §3 `AppConfig`; replay policy per ADR-005; cancelled requests never toast | interceptor and notifier tests |
| Crash reporting with alerts | Symbols uploaded; alert on any new fatal issue and on crash-free sessions below the threshold in §11.5 | crash dashboard, alert rule screenshot in the release ticket |
| Critical flows on real devices | Sign-in, the critical business flow, and sign-out on the oldest supported Android and iOS devices you target | release ticket checklist |

**UX and accessibility**

| Requirement | Acceptance | Evidence |
| --- | --- | --- |
| Screen readers and semantics | Every interactive element labeled; logical focus order | accessibility guideline tests; one TalkBack and one VoiceOver pass per release on changed screens |
| Text scaling and contrast | No overflow at 2.0x; AA contrast in both themes | §7.8 tests |
| Loading, empty, error, retry consistent | Four states on every data screen, shared components | widget tests |
| Localization readiness | All copy in `Strings`; dates and numbers through `intl` formatters, never string interpolation | code review; ARB migration plan exists if a second locale is on the roadmap |

**Performance and release**

| Requirement | Acceptance | Evidence |
| --- | --- | --- |
| Startup and scrolling measured | Budgets below, measured in profile mode on the low-end reference device | `integration_test` with `traceAction` timeline summaries, compared release over release |
| Large lists and images | `ListView.builder` for anything unbounded; images decoded at display size (`cacheWidth`/`cacheHeight`) | code review; DevTools memory snapshot on the longest list |
| Android and iOS release builds validated | Prod-flavor builds install and pass the real-device checklist from the internal track | release workflow, store internal track |
| Rollback and incidents documented | §11.5 followed; owner named in the release ticket | release ticket |

**Default budgets** (set your own in ADR form if these do not fit): cold start to first frame
under 2 s at p90 on the low-end reference device; no more than 1% of frames over 16 ms during
the scroll test of the longest list; release APK download size tracked per build, with any
increase over 10% explained in the PR.

**Platform coverage: what each check proves, and where it runs.**

| Check | Android | iOS | Where |
| --- | --- | --- | --- |
| Dart logic, interceptor, session contract, widgets, the critical flow headless | covered | covered | `verify` (Linux; platform-independent Dart) |
| Prod release bundle compiles, obfuscates, splits symbols | automated | n/a | `android` job |
| Release build compiles for device, unsigned | n/a | automated | `ios` job |
| Critical flow through real platform channels | emulator, API 34 | simulator | `android` / `ios` jobs |
| Backup exclusion of the secure store | file check in CI | n/a (`this_device` accessibility, pinned by a test) | `verify` |
| Keystore and Keychain behavior: first unlock after reboot, reinstall wipe, upgrade from a 1.x install | real device | real device | release checklist |
| Signing, store upload, crash symbol upload | release workflow | release workflow | tags only |

The emulator and simulator prove the app talks to the platform; they cannot prove Keystore or
Keychain edge cases, which differ by device and OS version. Those rows stay manual, on the
oldest supported device of each platform, and are ticked in the release ticket.

**The physical-device session script.** Run on one Android and one iOS device per release, each
the oldest supported model, against staging. Shorten the staging access-token lifetime to a
minute for the run. Every step has an expected result; anything else is a release blocker.

| # | Do | Expect |
| --- | --- | --- |
| 1 | Fresh install, sign in with "remember me", kill the app, relaunch | Signed in, no login screen |
| 2 | Wait past the access-token lifetime, open a data screen | Data loads; staging logs show one `/auth/refresh` |
| 3 | Same, with three screens' requests firing at once | Still exactly one refresh |
| 4 | Turn on airplane mode, wait past expiry, open a screen | "You appear to be offline." Still signed in when the network returns |
| 5 | Revoke the refresh token on the server, open a screen | Signed out once, one "session expired" message, login screen once |
| 6 | Throttle the network (Network Link Conditioner, or an emulator's slow profile), start a load as user A, sign out and sign in as user B before it returns | B never sees A's data; staging logs show no request from A's session carrying B's token |
| 7 | Throttle, let a refresh start, swipe the app away mid-request, relaunch | Signed in or cleanly signed out; never stuck on a blank screen |
| 8 | Sign in, reboot, let a background launch happen before unlocking (iOS: a push), then unlock | Still signed in |
| 9 | iOS: sign in, uninstall, reinstall | Signed out |
| 10 | Install the previous release, sign in, upgrade to this build | Still signed in (the 1.x key migration on the first upgrade) |
| 11 | Android: transfer to a second device with the platform's device-to-device tool | Opens signed out, no crash, no "failed to unwrap key" in logcat |
| 12 | Place an order on a throttled network and tap twice | One order on the server |

### 11.5 Rollout, rollback, and incidents

**Staged rollout.** Android: staged rollout on Play at 5%, then 20%, 50%, 100%, with at least
24 hours and a dashboard check between steps. iOS: phased release on the App Store.

**Halt criteria.** Pause the rollout when any of these holds for the new version compared with
the previous one: crash-free sessions below 99.5%; a new fatal issue affecting more than a
handful of users; sign-in success rate drops; a spike in `unauthorized` errors (the signature of
a broken refresh). The thresholds are defaults; record yours.

**Rollback.** A mobile release cannot be un-installed, so "rollback" means three things, decided
in advance:

- **Halt** the staged rollout (Play) or pause phased release (App Store). This limits blast radius; it does not fix installed devices.
- **Remote kill switches** for risky features, read from a remote-config provider injected like any other `Provider`, so a feature can be turned off without a release.
- **Minimum supported version.** The app checks a server-provided minimum version at launch and shows a blocking update screen below it. This is the only way to retire a build with a security bug. It ships *before* you need it.

Then ship a fix forward as a new build number. Never re-upload the old artifact as a "rollback".

**Incident procedure.** One named owner per release. When a halt criterion fires: halt the
rollout, open an incident note with the timeline, decide between kill switch, fix-forward, or
forced update, and communicate in the channel the team agreed. Within a week, write a
blameless postmortem; any rule change it produces goes into this playbook through §12.

### 11.6 When the requirement arrives

None of these are mandatory until their trigger is true. When it is, the minimum below becomes a
rule for that app, with the evidence in the last column. Each one bolts onto an existing seam (a
provider, a source behind a repository, a component), never a rewrite.

| Concern | Trigger | Minimum rules | Verified by |
| --- | --- | --- | --- |
| **Pagination** | a list the server pages, or one that can exceed about 100 items | Cursor-based where the API allows it. The notifier state holds items, cursor, `hasMore`, and a separate `DataState` for the next page, so a failed page keeps the items already shown. `ListView.builder`; the next page loads near the end, not at it; one request in flight per list | notifier tests: first page, next page, next-page failure keeps items, refresh resets the cursor |
| **Offline and sync** | users must read or write without a network | `drift` or `isar` behind the repository, which stays the single source of truth and returns local data first. Writes go to an outbox with idempotency keys (§5). A schema migration test per version | repository tests offline and online; migration tests; device script step 4 extended to writes |
| **Caching and invalidation** | data reused across screens or sessions | Every cache has a TTL and an owner (one repository). Cache keys include the user id, or the cache is rebuilt per session (as `profileRepositoryProvider` is). Mutations invalidate what they change; pull-to-refresh passes `forceRefresh`. Stale-while-revalidate only where showing stale data is acceptable | repository tests for hit, bypass, expiry, and invalidation after a mutation |
| **Localization** | a second locale | ARB or `slang`; `Strings` retired. Dates, numbers and currency through `intl`, never interpolation. Plurals through ICU. Right-to-left checked if any locale needs it | a widget test per screen in the longest locale at 2.0x text; a pseudo-locale run |
| **Accessibility audit** | before the first public release, then yearly | Beyond the widget-test guidelines: a full TalkBack and VoiceOver pass of every flow, focus order, announcements for async results, no information carried by color alone | the audit checklist in the release ticket, with issues tracked like bugs |
| **Performance** | before the first release; again when a screen feels slow | Profile mode on the low-end reference device. Budgets from §11.4. Images decoded at display size; no work in `build` that belongs in a notifier; `const` widgets; `RepaintBoundary` only where DevTools shows it helps | `integration_test` timeline summaries compared release over release; a DevTools memory snapshot of the longest list |
| **Crash-free monitoring** | the first production release | Crash reporter with symbols uploaded per build (§11.3). Alerts on any new fatal issue and on crash-free sessions below 99.5% for the current version. Rollout halts on those signals (§11.5) | the alert rules, linked in the release ticket |
| **Analytics and feature flags** | product measurement or gradual rollout | An analytics facade and a remote-config provider, injected like any other `Provider`. No personal data in events; event names in one file | a test that the facade is a no-op in dev and tests |
| **Push and deep links** | re-engagement or URL entry points | FCM/APNs; `go_router` (§4 trigger). A deep link to a protected route goes through the guard | a flow test that opens a deep link signed out and lands on login |
| **State restoration** | users lose work to process death mid-flow | `RestorationMixin` on the few flows that warrant it | an Android "don't keep activities" run of that flow |
| **Background work** | work that continues past app exit | `workmanager` behind a domain-level scheduler contract; tasks idempotent and resumable | unit tests of the task body; a device run with the app killed |

## 12. Governance

A standard that cannot change becomes ignored, and a standard that changes silently becomes
folklore. This section is how the playbook evolves without either.

### 12.1 Architecture decision records

Every significant decision, and every deviation from this document, is an ADR in `docs/adr/`.
Short is fine; unrecorded is not.

```markdown
# ADR-NNN: <decision in one line>

- Status: proposed | accepted | superseded by ADR-MMM
- Date: YYYY-MM-DD
- Revisit when: <the concrete trigger that would reopen this>

## Context
What forces the decision. Facts, not opinions.

## Decision
What we do, stated so a reviewer can check a PR against it.

## Alternatives considered
Each with the reason it lost.

## Consequences
What gets easier, what gets harder, what we now have to maintain.
```

The decisions this playbook already rests on:

| ADR | Decision | Revisit when |
| --- | --- | --- |
| 001 | Feature-first folders over the official layer-first layout | the app shrinks below about four features |
| 002 | Riverpod `Notifier` over `ChangeNotifier` + `provider` | Riverpod's major version changes the notifier model |
| 003 | `Result<T>` in data, `DataState<T>` in UI | never, without a replacement for both |
| 004 | Navigator 1.0 until a `go_router` trigger in §4 holds | any trigger holds |
| 005 | 401 replay only for idempotent methods or requests with an `Idempotency-Key` | the backend documents that auth precedes execution on every route |
| 006 | Session in memory with write-through to secure storage; scoped recovery | the platform plugin changes its failure modes |
| 007 | Native analyzer plugins for architecture rules (supersedes `custom_lint`) | the plugin API reaches 1.0, or `flutter analyze` starts reporting plugin diagnostics |
| 008 | `flutter_screenutil` behind the `Dimensions` facade | tablets, foldables, or web become targets |
| 009 | A feature's public API is its `domain/` plus one root file | a feature must expose more than providers and domain types |

### 12.2 Exception policy

A rule with no way to deviate gets deleted by frustration. A rule with a free way to deviate
gets deleted by attrition. The way to deviate here costs one paragraph:

1. Write the ADR explaining why this case is different and what bounds the exception.
2. Suppress the specific lint on the specific line, with the ADR cited directly above it:

   ```dart
   // ADR-012: the legacy importer reads billing DTOs until the v3 API ships (2027-Q2).
   // ignore: app_lints/no_cross_feature_imports
   import 'package:app/features/billing/data/dto/invoice_dto.dart';
   ```

3. Review: a PR that adds or widens an exception needs a second reviewer's approval.

`tool/check_architecture.dart` fails CI on any suppression of an architecture rule without an
`// ADR-NNN` line directly above it. `ignore_for_file` for these rules is allowed only with the
same citation and should be rare enough that every one is known by name.

### 12.3 Dependencies and compatibility

**Supported baseline.** One Flutter version, pinned in CI (`3.47.8`, Dart `3.13.7`, at the time
of writing), and the Dart SDK it ships. The reference app's lockfile records the rest:
`flutter_riverpod 3.4.3`, `riverpod_lint 3.1.9`, `dio 5.11.1`, `retrofit 4.10.0`,
`flutter_secure_storage 11.2.0`, `freezed 4.0.3`, `analysis_server_plugin 0.3.25`. The Android `minSdk` and iOS deployment target are recorded in the
reference app and raised only by ADR, because raising them drops users.

**Upgrade cadence.**

- **Patch and minor** updates arrive weekly as one grouped Dependabot PR. CI green is sufficient to merge.
- **Major** updates are one dependency per PR, never grouped, with the checklist below in the description.
- **The Flutter SDK** moves in its own PR, after the new stable has had its first hotfix, so a toolchain regression reads as a toolchain regression.

**Watched dependencies.** A major bump of any of these needs the named re-check, not just a
green build:

| Dependency | Re-check |
| --- | --- |
| `flutter_secure_storage` | §3 platform options (`resetOnError` default), all three SharedPreferences file names in `data_extraction_rules.xml` against the plugin's Android source, `SessionStore` suite on a real device |
| `dio` | `AuthInterceptor` suite, `DioException` types in the mapper, `QueuedInterceptor` semantics |
| `flutter_riverpod` / `riverpod_lint` | `ref.mounted`, retry and pause behavior (§6), notifier lifecycle on rebuild |
| Dart SDK / `analysis_server_plugin` / `analyzer` | fixture run and root canary, `dart analyze` at the root still reports plugins, the plugin still loads in the IDE |
| `freezed`, `retrofit`, `build_runner` | regenerate, diff the generated output, `@Extra` and `@Header` still applied, factory `baseUrl` type still matches |
| `flutter_screenutil` | widget tests at 2.0x text scale |

**Lockfile discipline.** `pubspec.lock` is committed for the app, for `tool/lints`, and for `tool/lints/fixture`.
CI runs `pub get --enforce-lockfile`, so a dependency cannot change without a visible diff.

### 12.4 Reference implementation

The playbook is proven by a small app, not by this document. The reference repository
(`reference_app`, shipped with this version) contains:

- `auth` exactly as in §5: login, the guard, session expiry, sign-out with revoke;
- `profile`, a cached read with `forceRefresh`, a 404-as-empty case, and a session-scoped cache;
- `orders`, a non-idempotent write with an idempotency key owned by the notifier's intent;
- every test in §7, the lint plugin and its fixture, the `tool/` scripts, the feature template and its scaffolder, the ADRs, and the CI workflow in §11.2.

It doubles as the starter repository (Quick start): it passes every check in §11.2 as shipped.

**Snippets come from the reference app.** Every non-conceptual code block in this document was
generated from the reference app's files when this version was cut. When the reference app
changes, the document changes in the same PR, and the reviewer checks the two against each
other. A snippet that no longer matches its source is a documentation bug with a deadline: the
next release.

**A version of this playbook is final only when the reference app's CI is green on GitHub at
that version's pins.** That is what removes `-rc` from the version number.

### 12.5 Versioning and changelog

Semantic versioning, applied to rules rather than code:

- **Major**: a rule changes such that compliant code becomes non-compliant (a new lint, a changed contract).
- **Minor**: a new section, pattern, or optional practice; compliant code stays compliant.
- **Patch**: corrections and clarifications that change no rule.

Each entry says what changed and what an existing app must do about it.

#### 2.0.0-rc.3

From a second external review. No existing rule is relaxed; one is added (server copy).

- **Server copy is constrained.** `_extractServerMessage` strips control characters, line separators and bidirectional overrides, and drops empty or over-long messages, so `errorToMessage` falls back to generic copy. *Migrate:* replace `dio_error_mapper.dart`.
- **Threat model and limits (§3).** Each protected scenario mapped to the test that proves it, and what the design does not protect against.
- **Backend idempotency contract (§5).** Five requirements and a staging check; the client cannot verify exactly-once on its own.
- **Quick start and Rule index.** One page to adopt the playbook; every rule labeled mandatory, conditional, or optional, with what enforces it and which rows a reviewer owns.
- **Feature template.** `dart run tool/new_feature.dart <name>` writes ten files that already pass the rules; CI renders and tests the template on every PR.
- **Physical-device session script (§11.4)** and a **run record template (§12.6)**.
- **§11.6 rewritten** as conditional requirements: pagination, offline and sync, caching and invalidation, localization, accessibility audits, performance, crash-free monitoring, and more, each with a trigger and its evidence.
- New tests: server-copy sanitization; the first-launch keychain wipe. 29 sabotages, all killed.

#### 2.0.0-rc.2

Every item below was found by building and running the reference app, not by review. An
existing rc.1 app must migrate where it says so.

Correctness and security:

- **The boundary lints moved to a native analyzer plugin.** `custom_lint` 0.8.1 and `riverpod_lint` 3.x no longer resolve together. *Migrate:* replace `tool/lints` with the native plugin (§8), drop `custom_lint`, add the `plugins:` block, and change suppressions to `app_lints/<rule>`.
- **CI analyzes with `dart analyze` at the root.** On Flutter 3.47.8, `flutter analyze` and path-scoped `dart analyze` report no plugin diagnostics, and a wrong plugin path is silent. *Migrate:* replace the analyze step; add `check_lint_fixture.dart`, which now also plants a canary in the app.
- **The worked example violated its own layer rule.** Repository providers lived in `data/` and every notifier imported them. *Migrate:* move each `<feature>RepositoryProvider` to `features/<f>/<f>.dart` (ADR-009). Cross-feature reads of `authStateProvider` go through `auth.dart`.
- **`layer_boundary` now forbids `data/` importing `presentation/`.** rc.1 promised every rule an enforcer; this one had none.
- **The invariant-2 interceptor test could not fail.** Two mechanisms each covered the other; with its named sabotage applied, the test stayed green. `_failedGeneration` is gone; `_expired` stops refresh after teardown, and the epoch bump quiets siblings, each with its own test. *Migrate:* replace `auth_interceptor.dart`.
- **Android silently overrode the session contract.** `flutter_secure_storage` 11 defaults `resetOnError` to `true`, deleting undecryptable keys and reporting them as absent. *Migrate:* pass `resetOnError: false`.
- **Backup rules missed the key file.** `FlutterSecureKeyStorage.xml` and `FlutterSecureStorageConfiguration.xml` are now excluded from cloud backup and device transfer, and CI checks all six exclusions. *Migrate:* replace `data_extraction_rules.xml`.
- **The session is one write.** Both tokens are stored as one JSON value under one key, so a crash mid-rotation cannot strand a consumed refresh token. *Migrate:* none; `SessionStore.load` migrates the two-key layout on first launch.
- **The storage chain cannot be poisoned.** `_serial` catches and reports what an operation lets escape, so one failure cannot skip every later storage call.
- **Smaller fixes.** 429 maps to `retryable`. `DataState.from` moved onto the class and returns `DataState<T>.idle()`, not an inferred `DataIdle<Never>`. Retrofit factories take `String? baseUrl`. Flavor names use `manifestPlaceholders` (AGP rejects `resValue` by default). `SessionExpiry` replaces the raw `StreamController` provider. OSV is pinned to a real version.

Verification (new):

- `tool/check_sabotage.dart` applies all 27 named sabotages and fails if any test survives, does not compile, or no longer exists. 27 of 27 are killed.
- `tool/check_coverage.dart` replaces `lcov` and `grep -oP`.
- The critical flow runs headless on every PR and on an emulator and a simulator in CI, from one shared function, and asserts the token each request carried.
- New tests: session contract rows for atomic rotation, a hanging keystore, legacy migration and the chain; secure-storage options; logger and reporter redaction; refresher classification; every `ErrorType` mapping; `AppConfig.parse`; idempotency intent; profile cache expiry; accessibility in both themes.
- The `android` and `ios` jobs, and §11.4's platform coverage table.

Governance: ADR-007 superseded, ADR-009 added, mandatory-versus-proportional table in §2, a
debugging procedure for false-positive lints in §8, §12.6 verification record.

#### 2.0.0-rc.1

Correctness and security fixes (existing apps must migrate):

- **Boundary lints now actually run in CI.** `flutter analyze` never reported `custom_lint` rules; CI adds `dart run custom_lint` and a fixture run. *Migrate:* add both steps, then fix whatever they report.
- **Release config can no longer ship a LAN URL.** `AppConfig.fromEnvironment()` throws in every build mode; unknown `APP_ENV` is an error; CI validates release configs. *Migrate:* replace `app_config.dart`, add `appConfigProvider` override in `main()`.
- **Account switch can no longer replay user A's request with user B's token.** Session epochs in `AuthInterceptor`; `startSession()` and `endSession()` replace `onSessionStarted()`. *Migrate:* call `endSession()` in sign-out.
- **A network failure during refresh no longer logs the user out.** `RefreshOutcome` replaces a nullable token; only 400/401/403 from the refresh endpoint end a session.
- **Wrong password no longer says "session expired".** `kSkipAuth` on body-authenticated endpoints; 401 there maps to `rejected`. 403 maps to `forbidden`. *Migrate:* add `@Extra({kSkipAuth: true})` to login, register, refresh, revoke, reset.
- **Non-idempotent requests are no longer replayed blindly.** Replay requires an idempotent method or an `Idempotency-Key`; otherwise `retryable`. *Migrate:* add idempotency keys to writes that must survive a refresh.
- **Secure-storage recovery is scoped.** `SessionStore` replaces `safeRead`; no `deleteAll()` on read failure; reads that fail before first unlock no longer sign the user out. *Migrate:* route all credential access through `SessionStore`.
- **iOS keychain items no longer restore onto other devices.** `first_unlock_this_device` replaces `first_unlock`.
- **Android 12+ device transfer excluded.** `dataExtractionRules` added; CI checks the manifest.
- **Refresh and replay clients share transport policy** with the main client via `buildDio`, so certificate pinning cannot be bypassed.
- **Crash reports carry `NetworkFailure`, not raw `DioException`,** so query strings and ids stay out of the dashboard; 4xx verdicts are no longer reported.
- **Notifier fixes.** `ref.mounted` after every await; Pattern 3 no longer uses `late final` assigned in `build()`; `setAuthenticated` waits for a pending cold-start restore.
- **`common/` no longer depends on a feature.** `AuthGuard` moved to `features/auth/presentation/`; enforced by `shared_layer_independence`. The guard is now the single navigation authority on session end.

Enforcement and verification (new):

- Two new lints, `shared_layer_independence` and `domain_purity`; resolution by `Uri.resolve` instead of the analyzer element API; export and conditional-import coverage.
- `tool/check_architecture.dart` for feature cycles and ADR-backed suppressions; replaces the unverified third-party cycle check.
- Generated-code check uses `git status --porcelain` so untracked output is caught.
- Coverage: 80% overall, 90% on `core/network` and `core/session`, with a helper so untested files count.
- Test matrix (§7.1), test honesty rules (§7.3), session suite, interceptor suite expanded from six to eleven cases, widget tests with accessibility and text-scale checks, integration test shape.
- `security` and `integration` CI jobs; Dependabot for actions and pub; SHA-pinning policy.

Production and governance (new):

- Production readiness gates with evidence (§11.4), performance budgets, rollout, rollback, and incident procedure (§11.5).
- ADRs, exception policy, dependency and compatibility policy, reference implementation, and this changelog (§12).

### 12.6 Verification record

What was executed for `2.0.0-rc.3`, against a clean clone of the reference app, on Linux with
Flutter 3.47.8 / Dart 3.13.7, on 2026-10-28.

| Check | Result |
| --- | --- |
| `pub get --enforce-lockfile` (app, plugin, fixture), `dart format`, `build_runner` then `git status --porcelain` | resolves; formatted; generated code committed and current |
| `dart analyze --fatal-infos` (app, both plugins) | no issues |
| `dart analyze --fatal-infos` (`tool/lints`) | no issues |
| `check_lint_fixture.dart` | 13 expected diagnostics, 4 rules, suppression honored; canary sees both plugins. With the plugin path misspelled: fails, as it should |
| `check_architecture.dart` | 3 features, no cycles. With an `auth -> profile` import planted: fails with `auth -> profile -> auth` |
| `check_release_config.dart` | staging and prod pass. With `APP_ENV: production`, an `http` URL and `API_KEY`: three errors |
| `flutter test --coverage` | 98 tests pass (including the conceptual Pattern 3 compile check) |
| `check_coverage.dart` | 85.5% overall, 93.5% `core/network`, 98.3% `core/session` |
| `check_sabotage.dart` | 29 of 29 killed. A sabotage that does not compile is reported as invalid, not as killed (caught once during this run) |
| `new_feature.dart --check` | 10 files render, analyze, and pass. With a wrong expectation in the template test, or a `presentation -> data` import in the template: fails |
| rc.1 invariant-2 test, reproduced, with its named sabotage | still passes (the defect fixed in rc.2) |
| OSV scan of `pubspec.lock` (v2.7.0) | 118 packages, no issues |
| `flutter build appbundle --flavor prod --release --obfuscate --split-debug-info` | builds (50.1 MB). Merged manifest: `allowBackup=false`, extraction rules present, no cleartext, no `.dev` suffix. Symbols split for arm, arm64, x64 |
| `flutter build apk --flavor dev --debug` (rc.2 code) | builds. Merged manifest: `.dev` suffix, "Reference Dev", cleartext allowed (dev only) |
| Android emulator flow, iOS build and simulator flow, gitleaks | **not run here**: the `android`, `ios` and `security` CI jobs on GitHub |
| Physical-device session script (§11.4) | **not run here**: per app release |

The `-rc` suffix comes off when the emulator, simulator and security jobs are green on GitHub
(§12.4). The device script belongs to each app release, not to the playbook's version.

**Run record template.** Copy this into the release ticket, or into this section for a playbook
version, every time. A result without its commit and versions is not evidence.

```markdown
- Playbook version / app version:
- Commit SHA:
- Flutter / Dart:            (output of `flutter --version`)
- pubspec.lock SHA-256:      (sha256sum pubspec.lock)
- CI run URLs:               verify / security / android / ios
- Devices:                   model, OS version, for each platform
- check_sabotage:            N of N killed
- Coverage:                  overall / core/network / core/session
- Device script (§11.4):     steps 1 to 12, pass or the failing step with a ticket
- Deviations:                any ADR-backed exception added since the last run
```
