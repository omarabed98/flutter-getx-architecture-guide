# Flutter Architecture Guide (GetX-based)

> Throughout this document, replace `App` / `<Brand>` / `app_name` with the actual name of your project. For example, if your project is `Falcon`, use `FalconRoutes`, `FalconNavigator`, `FalconGo`, the `Falcon*` widget prefix, and `falcon_app` as the package name. The architecture itself is brand-agnostic.

## 1. Purpose of This Document

This guide is the internal reference for any developer building or maintaining a Flutter project that uses this architecture. It documents the structure, layers, GetX usage, routing, bindings, theme, localization, networking, and feature creation conventions.

Use this document when:

- You are creating a new screen or feature in an existing project that follows this architecture.
- You are starting a brand new Flutter project and want to reuse the architecture as a baseline.
- You need to understand how an existing piece of the architecture (routing, bindings, network adapter, etc.) is wired so you can follow the same pattern.

This is not a tutorial on Flutter or GetX. It assumes basic familiarity with both. It is also not a business-logic document. It focuses on structure, patterns, and conventions.

## 2. High-Level Architecture

The app is a single-module Flutter project organized by **feature** with a shared **core** module.

The architecture has three layers per feature:

- `presentation/` — screens, controllers, widgets. Built on GetX (`GetxController`, `Obx`, `GetBuilder`, `Bindings`).
- `domain/` — feature-level models, sometimes interfaces or utility types. Pure Dart, no Flutter imports beyond what models need.
- `data/` — `data_source/` (remote, talks to the API) and `repository/` (the abstraction the controller depends on).

Key architectural elements:

- **State management**: GetX (`get: ^4.6.x`). Controllers extend `GetxController`. Reactive state uses `Rx`/`.obs` with `Obx`, plus `GetBuilder` with `update([id])` when targeted, scoped rebuilds are needed.
- **Routing**: Named routes through `GetMaterialApp(getPages: AppNavigator.routesList, initialRoute: AppRoutes.splash, initialBinding: CoreBinding())`. Route names live in `AppRoutes`. Page-to-binding-to-screen wiring lives in `AppNavigator`.
- **Navigation**: Most code uses `Get.toNamed` / `Get.offNamed` / `Get.offAllNamed` / `Get.back`. There is also a thin wrapper, `AppGo`, that adds custom transitions while still respecting the route's bindings.
- **Dependency injection**: `Bindings` per feature register data sources, repositories, and controllers via `Get.lazyPut` / `Get.put`. A global `CoreBinding` registers the network adapter, token manager, session manager and core data source.
- **Networking**: A single `NetworkAdapter` (Dio-based) implements `NetworkAdapterAbstraction`. Data sources build typed `NetworkRequest` objects with a `NetworkRouter` enum and receive a `NetworkResponse`. Errors are mapped to `Failure` subclasses and returned as `Either<Failure, T>` from `dartz`.
- **Models**: Plain Dart classes with `fromJson` factories (and `toJson` where needed). Pagination is wrapped in shared `PaginatedData<T>` / `CursorPaginatedData<T>`.
- **Shared widgets**: Live in `lib/core/presentation/widgets/` and use a project-wide brand prefix (e.g. `App*`).
- **Theme**: Centralized in `lib/core/presentation/theme/` with singletons `ColorManager`, `AppTheme`, `AppTextTheme`, `AppTypography`, `GradientManager`, `SchemeManager`. Light and dark themes are both defined.
- **Localization**: GetX `Translations`. Languages are declared in `LanguageRegistry`, translation maps are in `lib/core/presentation/localization/languages/{<tag>}.dart`, and keys are constants on `LocalizationKeys`. Strings are accessed as `LocalizationKeys.someKey.tr`.
- **Environment / config**: `flutter_flavor` plus `envied`-generated env classes. Environment variables (BASE_URL, Firebase options, env instance) are returned by an `AppEnvironments` enum. (Optional — see section 14.)

## 3. Folder Structure

The top-level `lib/` folder has three children:

```
lib/
├── core/
├── environments/      (only if you use flavors)
├── features/
└── main.dart
```

### `lib/core/`

Anything shared by more than one feature.

```
lib/core/
├── cache/
│   ├── local_storage.dart       # SharedPreferences wrapper (singleton)
│   ├── secure_storage.dart      # FlutterSecureStorage wrapper (singleton)
│   └── storage_keys.dart        # enum StorageKeys
├── data/
│   ├── data_sources/
│   │   ├── core_remote_data_source.dart
│   │   └── local_storage_keys.dart
│   └── networking/
│       ├── network_adapter.dart        # Dio-based NetworkAdapter
│       └── data/
│           ├── network_request.dart    # NetworkRequest, RequestType enum
│           ├── network_response.dart   # NetworkResponse, NetworkResponseStatus
│           └── network_router.dart     # NetworkRouter enum (all endpoints)
├── domain/
│   ├── constants/                # constants files (assets.dart, etc.)
│   ├── errors/failure.dart       # Failure, BusinessFailure, ServerFailure, ...
│   ├── models/                   # cross-feature models (User, UserProfile, ...)
│   ├── routing/
│   │   ├── app_routes.dart       # route name constants
│   │   ├── app_navigator.dart    # GetPage list (routesList)
│   │   ├── app_go.dart           # AppGo navigation wrapper + GoTransitions
│   │   └── core_binding.dart     # CoreBinding (initialBinding)
│   └── utils/                    # alerts mixins, extensions
├── presentation/
│   ├── controllers/              # global controllers (ThemeController, LocalizationController)
│   ├── localization/             # AppLocalization, LanguageRegistry, languages/
│   ├── theme/                    # AppTheme, ColorManager, AppTypography, components/
│   └── widgets/                  # shared App* widgets
├── services/                     # global services (TokenManagerService, AnalyticsService, ...)
└── utils/                        # misc helpers (AppUtils, GetXUtils, MediaUtils)
```

### `lib/features/`

One folder per feature. Each feature follows the same internal layout:

```
lib/features/<feature_name>/
├── binding/
│   └── <feature>_binding.dart
├── data/
│   ├── data_source/
│   │   └── <feature>_remote_data_source.dart
│   └── repository/
│       └── <feature>_repo.dart
├── domain/
│   └── models/
│       └── <model>.dart
└── presentation/
    ├── <feature>_screen.dart        # or a views/ subfolder for multi-screen features
    ├── <feature>_controller.dart    # or a controllers/ subfolder
    └── widgets/
        └── <component>.dart
```

Examples of feature shapes you can use:

- **Small feature** — single screen + controller + binding + data layer.
- **Medium feature** — `presentation/views/`, `presentation/controllers/`, `presentation/widgets/` subfolders.
- **Large feature** — multiple sub-flows under `presentation/login/`, `presentation/register/`, etc., each with its own widgets.
- **Cross-cutting feature** — kept under `features/common/<feature>/` (e.g. `features/common/static_pages/`).

### Where to put new files

| You're creating … | Put it in … |
|---|---|
| A new screen for an existing feature | `features/<feature>/presentation/` (or `presentation/views/` if the feature already uses subfolders) |
| A new feature-only widget | `features/<feature>/presentation/widgets/` |
| A widget reused across features | `lib/core/presentation/widgets/` (with the project's brand prefix) |
| A new model used in one feature | `features/<feature>/domain/models/` |
| A new model shared across features | `lib/core/domain/models/` |
| A new API endpoint | Add a new entry in `lib/core/data/networking/data/network_router.dart` |
| A new global service | `lib/core/services/` |
| A new constant | `lib/core/domain/constants/` |
| A new shared mixin/util | `lib/core/domain/utils/` |

### What does **not** belong where

- No business logic or controllers in `core/presentation/widgets/` — only reusable UI.
- No raw Dio/http calls in repositories or controllers — always go through `NetworkAdapter`.
- No `Get.toNamed` strings with hardcoded paths — always use `AppRoutes.*` constants.
- No translation strings hardcoded in screens — always use `LocalizationKeys.*.tr`.
- No environment variables read directly in features — always go through an environment helper or a service that wraps it.

## 4. Feature Structure

Every feature is a self-contained slice composed of three layers (presentation / domain / data) plus one binding. The standard pieces are:

1. **View / Screen** (`presentation/<feature>_screen.dart`)
   - A `StatelessWidget` (or `HookWidget` if it needs `useState`/`useEffect`).
   - Uses `Get.find<XController>()` or `GetBuilder<XController>()` to read the controller.
   - Uses `Obx` for reactive parts and `GetBuilder` (with an `id`) for targeted rebuilds.
2. **Controller** (`presentation/<feature>_controller.dart`)
   - Extends `GetxController`. Often mixes in `Alerts`, etc.
   - Holds Rx state and exposes async methods. Calls the repository.
3. **Binding** (`binding/<feature>_binding.dart`)
   - Extends `Bindings` and overrides `dependencies()`.
   - Registers data source → repository → controller, in that order.
4. **Repository** (`data/repository/<feature>_repo.dart`)
   - Defines an abstract `XRepositoryAbstraction` and a concrete `XRepository`.
   - Mostly a passthrough to the data source, plus optional local-storage interactions.
5. **Remote Data Source** (`data/data_source/<feature>_remote_data_source.dart`)
   - Defines an abstract `XRemoteDataSourceAbstraction` and a concrete `XRemoteDataSource`.
   - Uses the injected `NetworkAdapterAbstraction` to make `NetworkRequest` calls.
   - Maps responses into models and returns `Either<Failure, T>`.
6. **Models** (`domain/models/`)
   - Plain Dart classes with `fromJson` (and `toJson` if needed).
   - `factory X.empty()` helpers are common for initial Rx state.
7. **Widgets** (`presentation/widgets/`)
   - UI pieces specific to this feature only.
8. **Route constant + GetPage entry** — see section 6.

### Step-by-step: adding a new feature

Assume a new feature called `news`:

1. Create `lib/features/news/` with the standard subfolders.
2. Add the API endpoint to `NetworkRouter`:
   ```dart
   news(path: 'api/v1/news/'),
   ```
3. Create `domain/models/news_model.dart` with a `fromJson` factory.
4. Create `data/data_source/news_remote_data_source.dart`:
   ```dart
   abstract class NewsRemoteDataSourceAbstraction {
     Future<Either<Failure, PaginatedData<NewsModel>>> news({Map<String, dynamic>? parameters});
   }

   class NewsRemoteDataSource implements NewsRemoteDataSourceAbstraction {
     final NetworkAdapterAbstraction _networkAdapter;
     NewsRemoteDataSource(this._networkAdapter);

     @override
     Future<Either<Failure, PaginatedData<NewsModel>>> news({Map<String, dynamic>? parameters}) async {
       final response = await _networkAdapter.request(NetworkRequest(
         route: NetworkRouter.news,
         requestType: RequestType.get,
         isAuthorizationRequired: true,
         parameters: parameters,
       ));
       if (response.status == NetworkResponseStatus.success) {
         final list = (response.data['results'] as List)
             .map((e) => NewsModel.fromJson(e)).toList();
         return Right(PaginatedData.fromJson(json: response.data, dataList: list));
       }
       return Left(response.failure!);
     }
   }
   ```
5. Create `data/repository/news_repo.dart` mirroring the data source's interface.
6. Create `presentation/news_controller.dart`:
   ```dart
   enum NewsState { initial, loading, success, failure }

   class NewsController extends GetxController with Alerts {
     NewsController({required NewsRepositoryAbstraction repository})
         : _repository = repository;
     final NewsRepositoryAbstraction _repository;

     final state = NewsState.initial.obs;
     final data = Rx<PaginatedData<NewsModel>>(PaginatedData.empty());

     @override
     void onInit() { super.onInit(); fetch(); }

     Future<void> fetch() async {
       state.value = NewsState.loading;
       final result = await _repository.news();
       result.fold(
         (f) { state.value = NewsState.failure; showFailSnackbar(text: f.message); },
         (page) { state.value = NewsState.success; data.value = page; },
       );
     }
   }
   ```
7. Create `presentation/news_screen.dart` (a `StatelessWidget` that uses `Obx` or `GetBuilder<NewsController>`).
8. Create `binding/news_binding.dart`:
   ```dart
   class NewsBinding extends Bindings {
     @override
     void dependencies() {
       Get.lazyPut<NewsRemoteDataSourceAbstraction>(() => NewsRemoteDataSource(Get.find()));
       Get.lazyPut<NewsRepositoryAbstraction>(() => NewsRepository(Get.find()));
       Get.put(NewsController(repository: Get.find()));
     }
   }
   ```
9. Add a route constant in `AppRoutes`:
   ```dart
   static const news = '/news';
   ```
10. Register the page in `AppNavigator.routesList`:
    ```dart
    GetPage(
      name: AppRoutes.news,
      page: () => const NewsScreen(),
      binding: NewsBinding(),
    ),
    ```
11. Navigate from anywhere with `Get.toNamed(AppRoutes.news)`.

## 5. GetX State Management Pattern

### Controllers

- Always extend `GetxController`.
- Constructor takes its dependencies (repositories, other controllers) as `required` parameters. They are wired via the binding using `Get.find()`.
- `onInit()` is used to kick off initial loads.
- `onClose()` disposes ScrollControllers, TextControllers, etc.

```dart
class NewsController extends GetxController with Alerts {
  NewsController({required NewsRepositoryAbstraction repository}) : _repository = repository;
  final NewsRepositoryAbstraction _repository;

  final state = NewsState.initial.obs;
  final data = Rx<PaginatedData<NewsModel>>(PaginatedData.empty());

  @override
  void onInit() { super.onInit(); fetchNews(); }
}
```

### Reactive state

Two reactive flavors are used in this architecture, sometimes in the same controller:

- **`Rx` + `Obx`** — for fine-grained reactive UI:
  ```dart
  final state = NewsState.initial.obs;
  // In the screen:
  Obx(() => state.value == NewsState.loading ? const Loader() : ListView(...));
  ```
- **`GetBuilder<T>` + `update([id])`** — for targeted rebuilds of an entire screen scope. The `id` is conventionally the screen `Type` itself:
  ```dart
  GetBuilder<NewsController>(
    id: NewsScreen,
    builder: (controller) { ... },
  );
  // In the controller:
  state.value = NewsState.loading;
  update([NewsScreen]);
  ```

  Use `GetBuilder` with `id` when many fields change at once and you don't want every `Obx` to recompute, or when you need to rebuild a stateful section as a whole.

### Dependency lookup

- Inside controllers, always inject through the constructor.
- Inside screens/widgets, use `Get.find<XController>()` (the binding has already registered it).
- `Get.put(...)` inside a screen's `build()` is a legacy pattern — prefer the binding-based approach.

### Bindings

- One binding per feature, in `binding/<feature>_binding.dart`.
- The binding is responsible for registering the **whole dependency tree** the screen needs (data source → repo → controller).
- Use `Get.lazyPut` for data sources, repositories and most controllers — they are created on first `Get.find`.
- Use `Get.put` (eager) for controllers that must exist as soon as the route is mounted.
- Use `Get.put(..., permanent: true)` for global state (e.g. a profile controller used across many screens).
- Use `fenix: true` (with `lazyPut`) for dependencies that should be re-created if they were disposed.
- Use `Get.isRegistered<T>()` guards before re-registering shared dependencies in another feature's binding.

### Lifecycle

- `onInit()` — first build, kick off async work.
- `onReady()` — after the first frame; rarely used.
- `onClose()` — clean up controllers, listeners, debouncers.

### Naming conventions (state management)

- Controller class: `XController` (e.g. `NewsController`).
- State enum: `XState` with values `{ initial, loading, pageLoading, success, failure }`.
- Rx variables: lowerCamelCase (`state`, `data`, `selectedSort`).
- Data lists are wrapped in `Rx<PaginatedData<T>>` or `Rx<CursorPaginatedData<T>>` for paginated screens.

### Do / Don't (state management)

| Do | Don't |
|---|---|
| Inject dependencies through the controller constructor. | Reach into singletons inside controllers without DI. |
| Use `Obx` for small reactive nodes; `GetBuilder` with `id` for screen-scope rebuilds. | Sprinkle `setState` or `StatefulWidget` for things GetX already manages. |
| Define a `XState` enum and drive UI off `state.value`. | Use ad-hoc booleans (`isLoading`, `isFailure`, `isSuccess`). |
| Dispose ScrollControllers and TextEditingControllers in `onClose()`. | Forget to dispose them — they leak. |
| Guard with `if (!Get.isRegistered<T>())` when a binding might run twice. | Re-register the same singleton blindly. |

## 6. Routing and Navigation

This architecture uses GetX named routing exclusively.

### `GetMaterialApp` setup

```dart
GetMaterialApp(
  title: 'App',
  translations: AppLocalization(),
  locale: localeController.currentLanguage.locale,
  textDirection: localeController.textDirection,
  supportedLocales: LanguageRegistry.supportedLocales(),
  fallbackLocale: const Locale('en'),
  theme: AppTheme().light,
  darkTheme: AppTheme().dark,
  themeMode: themeController.themeMode,
  defaultTransition: GoTransitions.slide,
  transitionDuration: GoTransitions.normalDuration,
  getPages: AppNavigator.routesList,
  initialBinding: CoreBinding(),
  initialRoute: AppRoutes.splash,
  navigatorObservers: [AppNavigationObserver()],
);
```

Important pieces:

- `getPages: AppNavigator.routesList` — the single source of truth for all `GetPage` definitions.
- `initialBinding: CoreBinding()` — runs once at app start, registers the global services.
- `initialRoute: AppRoutes.splash` — every app launch starts on the splash screen.
- `defaultTransition: GoTransitions.slide` and `transitionDuration: GoTransitions.normalDuration` — locale-aware default transitions.

### `AppRoutes` (route names)

`lib/core/domain/routing/app_routes.dart` is an `abstract final class` with `static const String` route names. **Always reference routes through this class — never with a string literal.**

```dart
abstract final class AppRoutes {
  static const splash = '/splash';
  static const dashboard = '/dashboard';
  static const news = '/news';
  static const newsDetails = '/news-details';
  // ...
}
```

### `AppNavigator.routesList` (page registry)

`lib/core/domain/routing/app_navigator.dart` exposes `routesList`, a `List<GetPage>` consumed by `getPages`. Every screen in the app is registered here. Each entry combines:

- `name` — pulled from `AppRoutes`.
- `page` — a builder returning the screen widget. Argument decoding (`Get.arguments`) happens here so the screen widget itself can stay clean.
- `binding` or `bindings` — one or more `Bindings` to set up dependencies for that screen.

Example entries:

```dart
GetPage(
  name: AppRoutes.news,
  page: () => const NewsScreen(),
  binding: NewsBinding(),
),
GetPage(
  name: AppRoutes.newsDetails,
  page: () {
    final args = Get.arguments;
    final int postId = args is Map ? (args['id'] ?? -1) : (args is int ? args : -1);
    return NewsDetailsScreen(postId: postId);
  },
  binding: NewsDetailsBinding(),
),
GetPage(
  name: AppRoutes.dashboard,
  page: () => const DashboardScreen(),
  bindings: [
    ProfileBinding(),
    HomeSectionsBinding(),
  ],
),
```

A few patterns worth noticing:

- Use `binding:` for one binding, `bindings:` for several.
- Pass complex arguments through a `Map`, decode them in the `page:` builder.
- Some pages have no binding because they have no controller of their own.

### `CoreBinding` (initial binding)

`CoreBinding` is the global binding registered on `initialBinding`. It sets up the things every screen needs:

- `SessionManagerService`
- `TokenManagerService`
- `NetworkAdapter` (concrete) registered as `NetworkAdapterAbstraction`
- `CoreRemoteDataSourceAbstraction`
- Any other app-wide service

These are available via `Get.find()` from any subsequent feature binding.

### `AppGo` wrapper

`lib/core/domain/routing/app_go.dart` wraps `Get.toNamed`, `Get.offNamed`, `Get.offAllNamed`, `Get.back`, `Get.to`, `Get.off` and adds:

- Custom transitions (`Transition` / `CustomTransition`) per call without modifying the route definition.
- A `GoTransitions` class that holds preset transitions and durations — including locale-aware `slide` / `slideBack` that automatically swap direction in RTL.
- Optional custom transitions (e.g. a `ZoomInFadeOutTransition`) for special cases.

Use it when you need a one-off transition that differs from the route's default:

```dart
AppGo.toNamed(
  AppRoutes.createUserPost,
  transition: GoTransitions.downToUp,
  duration: GoTransitions.normalDuration,
);
```

For everyday navigation, plain GetX is still the rule:

```dart
Get.toNamed(AppRoutes.dashboard);
Get.offNamed(AppRoutes.welcome);
Get.offAllNamed(AppRoutes.splash);
Get.back();
```

### Step-by-step: adding a new route

1. Add the constant to `AppRoutes`:
   ```dart
   static const carDetails = '/car-details';
   ```
2. Create the screen and (if needed) the binding for the feature.
3. Register the page in `AppNavigator.routesList`:
   ```dart
   GetPage(
     name: AppRoutes.carDetails,
     page: () => const CarDetailsScreen(),
     binding: CarsBinding(),
   ),
   ```
4. Navigate to it: `Get.toNamed(AppRoutes.carDetails, arguments: carId);`.
5. If you need a non-default transition, swap `Get.toNamed` for `AppGo.toNamed` and pass `transition`/`duration`.

## 7. Bindings and Dependency Injection

### `CoreBinding` — global setup

`lib/core/domain/routing/core_binding.dart` registers app-wide singletons. It runs once via `initialBinding`. Anything any feature needs in *every* screen is here:

- `SessionManagerService` (eager, default scope)
- `TokenManagerService` (eager)
- `NetworkAdapterAbstraction` (eager — the network adapter is needed by every data source)
- Any service that should always be available (`permanent: true`)
- `CoreRemoteDataSourceAbstraction` (eager)

### Feature bindings — per-screen setup

Every feature has a `binding/<feature>_binding.dart`. The standard shape is:

```dart
class NewsBinding extends Bindings {
  @override
  void dependencies() {
    Get.lazyPut<NewsRemoteDataSourceAbstraction>(
      () => NewsRemoteDataSource(Get.find()),
    );
    Get.lazyPut<NewsRepositoryAbstraction>(
      () => NewsRepository(Get.find()),
    );
    Get.put(NewsController(repository: Get.find()));
  }
}
```

Order matters: data source is registered first, then the repository (which depends on it), then the controller (which depends on the repo).

### When to use which `Get.put` variant

| Variant | When to use |
|---|---|
| `Get.put(X)` | Eager singleton — created immediately when the binding runs. Use for the controller of the route. |
| `Get.put(X, permanent: true)` | App-lifetime singleton — never removed even when the route is popped. Use for cross-feature controllers (e.g. profile). |
| `Get.lazyPut<T>(() => X)` | Created on first `Get.find<T>()`. Use for data sources and repositories that may not be needed if the controller never asks for them. |
| `Get.lazyPut<T>(() => X, fenix: true)` | Same as `lazyPut`, but recreated if previously disposed. Use for shared feature dependencies that may outlive their first use. |
| `Get.putAsync<T>(() async => X)` | Async initialization. Most async setup is in `main.dart` before `runApp`, so this is rarely needed. |

### Where global vs feature dependencies live

- **Global**: `CoreBinding`. Network adapter, token / session manager, core data source.
- **App-lifetime feature dependency**: register in a feature binding with `permanent: true` (e.g. a profile controller). Safe because that binding is reached very early via the splash binding.
- **Feature-local**: regular `lazyPut` / `put` in the feature binding.
- **Cross-feature reuse**: a binding can call another binding's `dependencies()` directly (e.g. `ProfileBinding` instantiates and runs `ReactionBinding().dependencies()`).

### Guarding double-registration

When more than one feature pulls in the same dependency, wrap the registration:

```dart
if (!Get.isRegistered<UserPostRemoteDataSourceAbstraction>()) {
  Get.lazyPut<UserPostRemoteDataSourceAbstraction>(
    () => UserPostRemoteDataSource(networkInterceptor: Get.find()),
  );
}
```

A small `GetXUtils` helper (in `lib/core/utils/`) can wrap `find`, `put`, `lazyPut`, `delete`, etc., with these guards if you prefer it.

## 8. API, Services, and Data Layer

### `NetworkAdapter`

`lib/core/data/networking/network_adapter.dart` is the single entry point for all HTTP calls. It:

- Wraps `Dio` with debug interceptors (`PrettyDioLogger`, `CurlLoggerDioInterceptor`).
- Builds the URL from `BASE_URL` (per-environment) and the `NetworkRouter` enum.
- Adds standard headers: `Authorization`, `Accept-Language`, `X-Api-Key`, `X-Device-ID`, `X-Device-OS`, `X-Device-OS-Version`, `X-Device-Model`, `X-App-Version`.
- Calls `TokenManagerService.ensureValidToken()` for authorized requests and refreshes the token on 401.
- Maps HTTP status codes to `NetworkResponse(status, data, failure)` using the `Failure` hierarchy.
- Logs network and API errors to analytics.

### `NetworkRequest`

```dart
NetworkRequest(
  route: NetworkRouter.news,
  requestType: RequestType.get,           // get | post | put | patch | delete | download
  isAuthorizationRequired: true,          // adds Bearer token + token validity check
  parameters: {'page': 1, 'page_size': 20},
  data: {'token': accessToken},
  isFormData: false,                      // wraps `data` in FormData if true
  files: [...],                           // optional multipart files
  additionalHeaders: {...},               // extra headers
  urlIdentifier: '$id/',                  // appended to the route path or replaces a `<placeholder>`
);
```

### `NetworkRouter` (endpoint enum)

All endpoints live in `lib/core/data/networking/data/network_router.dart`:

```dart
enum NetworkRouter {
  login(path: 'auth/login/'),
  news(path: 'api/v1/news/'),
  postDetails(path: 'api/posts/'),
  presenterEpisodes(path: 'api/v1/shows/presenters/<presenter_id>/episodes/'),
  // ...
  final String path;
  const NetworkRouter({required this.path});
}
```

When you add an endpoint, add it here — never inline a path string in a data source.

### Data source pattern

Each feature has a `data/data_source/<feature>_remote_data_source.dart` with two pieces:

- An `abstract class XRemoteDataSourceAbstraction` describing the API surface.
- A concrete `XRemoteDataSource implements XRemoteDataSourceAbstraction` that holds a `NetworkAdapterAbstraction` and turns each method into a `NetworkRequest`.

```dart
class NewsRemoteDataSource implements NewsRemoteDataSourceAbstraction {
  final NetworkAdapterAbstraction _networkAdapter;
  NewsRemoteDataSource(this._networkAdapter);

  @override
  Future<Either<Failure, PaginatedData<NewsModel>>> news({Map<String, dynamic>? parameters}) async {
    final response = await _networkAdapter.request(NetworkRequest(
      route: NetworkRouter.news,
      requestType: RequestType.get,
      isAuthorizationRequired: true,
      parameters: parameters,
    ));
    if (response.status == NetworkResponseStatus.success) {
      final list = NewsModel.decodeList(response.data['results']);
      return Right(PaginatedData.fromJson(json: response.data, dataList: list));
    }
    return Left(response.failure!);
  }
}
```

### Repository pattern

Every feature has a `data/repository/<feature>_repo.dart` with:

- `abstract class XRepositoryAbstraction` — the interface controllers depend on.
- `class XRepository implements XRepositoryAbstraction` — almost always a passthrough to the data source. It is the place to add local caching or compose multiple data sources, but most repositories simply forward calls.

The data source and repository layers always coexist, even if the repository is a passthrough. Keep this convention so the dependency graph stays uniform across features.

### Error handling

- All async data methods return `Future<Either<Failure, T>>` from `dartz`.
- The response mapper produces:
  - `UnAuthenticatedFailure` on `401`,
  - `ForbiddenFailure` on `403`,
  - `ServerFailure` on other HTTP errors,
  - `InternetFailure` on `connectionError`/`receiveTimeout`,
  - `ServerFailure` on any other Dio exception,
  - `BusinessFailure` for app-level domain errors,
  - `ThirdPartyFailure` for third-party SDK errors (e.g. social login).
- Controllers consume the `Either` with `.fold(onFailure, onSuccess)` and typically call `showFailSnackbar(text: failure.message)` from the `Alerts` mixin.

### Models and JSON

Models always have `factory X.fromJson(Map<String, dynamic> json)` and frequently a `factory X.empty()` constructor for use in `Rx<X>(X.empty())`. List decoding lives next to the model:

```dart
static List<NewsModel> decodeList(List list) =>
    list.map((item) => NewsModel.fromJson(item)).toList();
```

Pagination decoding always uses `PaginatedData.fromJson` (offset/page based) or `CursorPaginatedData.fromJson` (cursor based). Both live in `lib/core/domain/models/`.

## 9. Models and Data Objects

- **Cross-feature models** live in `lib/core/domain/models/` (`User`, `UserProfile`, pagination wrappers, etc.).
- **Feature-local models** live in `features/<feature>/domain/models/`.
- **Sub-models** of a model (e.g. nested response objects) live in a `submodels/` subfolder when there are several.
- **Request bodies** that need to be modeled separately go in `domain/request/`.

### Naming

- Models are PascalCase: `NewsModel`, `UserProfile`, `RefreshToken`.
- Files are snake_case after the class: `news_model.dart`, `user_profile.dart`.
- Use `Model` suffix when the name would otherwise collide with a domain noun (`NewsModel`, `TribeModel`); otherwise drop the suffix (`User`, `Tier`, `RefreshToken`).

### JSON serialization

JSON serialization is hand-written in this architecture — there is no code generation for models. Every model has a `fromJson` factory written by hand. Keep it that way unless there is a specific reason to switch (it keeps the model file readable and free of generated `.g.dart` files).

```dart
factory User.fromJson(Map<String, dynamic> json) {
  return User(
    email: json['email'],
    username: json['username'],
    firstName: json['first_name'],
    lastName: json['last_name'],
    userId: json['pk'],
    isOnboarded: json['is_onboarded'] == true,
    isGuest: false,
  );
}
```

Add `toJson` only when the model is sent back to the API or persisted.

### Response models vs UI models

The same model class is generally used for the API response and the UI. Wrap collections in `PaginatedData<T>` or `CursorPaginatedData<T>` for paginated lists, and provide a `factory X.empty()` so controllers can initialize Rx state cleanly.

## 10. Shared Widgets and UI Components

- **Reusable widgets** live in `lib/core/presentation/widgets/`. They use the project's brand prefix (e.g. `App*`):
  `AppButton`, `AppListView`, `AppDialog`, `AppLoader`, `AppNetworkImage`, `AppAppbar`, `AppCheckbox`, `AppClickable`, `AppDropdownField`, `AppError`, `AppHtmlView`, `AppCountrycodePicker`, `AppSliverList`, `AppSectionHeader`, `AppWebViewScreen`, etc.
- **Specialized widget groups** are nested folders under `widgets/`:
  - `widgets/fields/` — input fields.
  - `widgets/sheets/` — bottom sheets.
  - `widgets/popup_menu/` — popup menus.
- **Feature-only widgets** stay inside the feature: `features/<feature>/presentation/widgets/`.

### Naming and organization

- Public widgets meant to be reused across features → `lib/core/presentation/widgets/` with the brand prefix.
- Widgets used by exactly one feature → `features/<feature>/presentation/widgets/`.
- Subcomponents of a complex screen → `presentation/views/<screen>/widgets/` or `.../components/`.

### Keeping UI consistent

- Always pull colors from `ColorManager()` (e.g. `ColorManager().textDefault`, `ColorManager().primary`).
- Always pull text styles from `AppTypography` and apply weight extensions (`.bold`, `.semiBold`, `.medium`).
- Use `LocalizationKeys.<key>.tr` for any human-visible string.
- Use the `Alerts` mixin for snackbars (`showSuccessSnackbar`, `showFailSnackbar`, `showInfoSnackbar`).
- Use `AppButton.primary` / `AppButton.secondary` instead of building bare `FilledButton`/`ElevatedButton`.
- Use `AppListView.paginated` for paginated lists with refresh + bottom-load.

## 11. Theme and Design System

The theme system lives in `lib/core/presentation/theme/` and is composed of singletons.

- **`AppTheme`** (`app_theme.dart`) — exposes `light` and `dark` `ThemeData`. Both are wired into `GetMaterialApp(theme:, darkTheme:, themeMode:)`.
- **`ColorManager`** (`color_manager.dart`) — single source of truth for colors. Holds light/dark variants for surfaces, text, icons, borders, errors, and a `primary` color (which can optionally be flavor-aware). Components read colors via `ColorManager().<token>` which resolves to the right light/dark value through `LocalStorageService().isDarkMode`.
- **`AppTypography` + `AppTextTheme`** (`text_manager.dart`, `components/text_theme.dart`) — typography presets (`headingM`, `subheadingM`, `bodyM`, `captionM`, etc.) plus `.bold`, `.semiBold`, `.medium`, `.regular` extensions and a `withColor(...)` helper. Fonts are exposed through an `AppFonts` class (e.g. `cairo`, `inter`, `roboto`).
- **`SchemeManager`** — Material 3 `ColorScheme` for light and dark.
- **`GradientManager`** — named gradients used across screens (e.g. `scaffoldGradient`).
- **Component themes** (`components/`) — themed styles for buttons, inputs, popup menus, date pickers, bottom navigation. Composed inside `AppTheme`.

### Dark / light mode

- The current `ThemeMode` is held in `ThemeController` (`lib/core/presentation/controllers/theme_controller.dart`) and persisted in `LocalStorageService.themeMode`.
- `ThemeController` listens to system brightness via `LocalStorageService.systemBrightnessNotifier` and calls `Get.forceAppUpdate()` when dark mode toggles, so anything that reads `ColorManager()` re-resolves to the correct shade.
- `ColorManager().isDarkMode` (via `LocalStorageService().isDarkMode`) is the single check the UI uses when it can't rely on `Theme.of(context)`.

### Spacing / sizing

This architecture does not include a central spacing token file. Use literal `EdgeInsets`/`SizedBox` values in widgets, but try to align with a common rhythm (e.g. 8 / 12 / 14 / 16 / 24 / 28). If you want to enforce one, add a small `AppSpacing` class to `lib/core/presentation/theme/`.

### How new UI should use the design system

- Pull colors only from `ColorManager()`.
- Pull text styles only from `AppTypography` / `AppTextTheme`.
- Pull font families only from `AppFonts`.
- Use `Theme.of(context).colorScheme.primary` when you need the theme-derived color (e.g. inside `RichText`).
- Add new tokens to the relevant manager rather than introducing constants in feature code.

## 12. Localization

Localization is wired through GetX `Translations`. Multiple languages are supported, with one set as the default. RTL is supported via the language registry.

### Where translations live

- `lib/core/presentation/localization/localization_keys.dart` — the `LocalizationKeys` class, a flat list of `static const String` keys. Every translatable string in the app has a key here.
- `lib/core/presentation/localization/languages/<tag>.dart` — translation map per language (e.g. `en.dart`, `ar.dart`), keyed by `LocalizationKeys`.
- `lib/core/presentation/localization/language_registry.dart` — `LanguageRegistry.supported` lists all `AppLanguage` entries (tag, RTL flag, display name, bundle builder).
- `lib/core/presentation/localization/app_localization.dart` — `AppLocalization extends Translations` exposes `LanguageRegistry.asGetXKeys()` to GetX.

### How locale switching works

- `LocalizationController` (`lib/core/presentation/controllers/locale_controller.dart`) holds `Rx<AppLanguage> currentLanguage` and exposes `changeLocale(...)` and `toggleLanguage()`.
- It is registered globally in the entry point (`Get.put(LocalizationController())`).
- Locale persistence is handled via `LocalStorageService.locale`.
- Switching the language calls `Get.updateLocale(lang.locale)` and pops the navigation stack to the splash route so data is refetched in the new language.
- `GetMaterialApp` is keyed by the current language tag (`ValueKey(localeController.currentLanguage.tag)`) so the whole tree rebuilds on locale change.

### Using strings in code

```dart
Text(LocalizationKeys.signInToStartTheRide.tr);
showFailSnackbar(text: LocalizationKeys.somethingWentWrong.tr);
```

`.tr` is a GetX extension that looks up the key in the active translation map.

### Adding a new translatable string

1. Add a key to `LocalizationKeys`:
   ```dart
   static const myNewKey = 'myNewKey';
   ```
2. Add the value to **every** language map:
   ```dart
   LocalizationKeys.myNewKey: 'My new string',
   ```
3. Use it: `Text(LocalizationKeys.myNewKey.tr)`.

### Adding a new language

1. Create `lib/core/presentation/localization/languages/<tag>.dart` with a `Map<String, String> map` covering every key.
2. Add an `AppLanguage` entry to `LanguageRegistry.supported` with the tag, RTL flag, display name, and the bundle builder.
3. Locale support, `supportedLocales`, and translation registration happen automatically via `LanguageRegistry`.

## 13. Environment and Config

### Environment files

- `lib/environments/env.dart` — abstract `Env` interface (e.g. `sentryDsn`, `oneSignalAppId`, `appsFlyerDevKey`, `apiKey`).
- `lib/environments/<flavor>/env_<flavor>.dart` — concrete `EnvDev`, `EnvQa`, `EnvStaging`, `EnvProd` classes, generated by [`envied`](https://pub.dev/packages/envied) from `env/.env.<flavor>` files (the `.env.*` files are git-ignored).
- `lib/environments/<flavor>/env_<flavor>.g.dart` — generated obfuscated values (build via `dart run build_runner build`).
- `lib/environments/<flavor>/firebase_options_<flavor>.dart` — Firebase options per flavor.
- `lib/environments/<flavor>/main_<flavor>.dart` — flavor-specific entry point.
- `lib/environments/_main.dart` — shared `mainApp()` that all entry points call after registering their flavor.
- `lib/environments/app_environments.dart` — `AppEnvironments` enum and an `EnvironmentHelper`.

### Base URL and per-environment variables

`AppEnvironments.variables` returns a `Map<String, dynamic>` per flavor with at least:

- `BASE_URL` — used by `NetworkAdapter._urlBuilder`.
- `env` — the concrete `Env` instance (used for Sentry DSN, OneSignal app id, AppsFlyer dev key, API key).
- `firebase_options` and `firebase_name` — Firebase configuration.

`EnvironmentHelper().getEnvironmentVariable('BASE_URL')` returns the value for the active flavor (`FlavorConfig.instance.name`).

### App config

- App name, app store id, transitions, etc., live as `const` values in the entry point or in `lib/core/domain/constants/`.
- App version and build number are read from `package_info_plus` and persisted to `LocalStorageService` by `AppUtils.setAppVersion()`.
- Locale, theme mode, font size variant, and similar user preferences live in `LocalStorageService` and `SecureStorageService`.

### Constants

`lib/core/domain/constants/` is the single home for compile-time constants used across features (asset paths, color collections, etc.). Add a new file rather than dumping more constants into existing ones.

### Secrets handling

- Secrets are kept in `env/.env.<flavor>` files and consumed by `envied` with `obfuscate: true`. The plain `.env.*` files are git-ignored.
- The `Env` interface only exposes the values that need to be read at runtime.
- Tokens are stored at runtime through `LocalStorageService` (non-sensitive copy) and `SecureStorageService` (canonical copy via `flutter_secure_storage`).
- Any other secrets the project uses (e.g. service-account JSON for build tooling) must be git-ignored.

### What should and should not be committed

- **Commit**: every file under `lib/`, `pubspec.yaml`, generated `.g.dart` files for envied (so CI can build without secrets re-encoding), `firebase_options_*.dart`, `pubspec.lock`.
- **Do not commit**: `env/.env.*` source files, real secrets, debug builds, `*.iml`, IDE-specific files (`.idea/`, `.vscode/`).

## 14. Optional Module: Flavors

> **Optional section.** Flavors are a deployment / build-config feature, not a requirement of the core architecture. If your project does not need flavors, you can remove this section, replace `lib/environments/` with a single `main.dart` and a single `AppConfig` class, and the rest of the guide still applies.

### How flavors are structured

Typical flavors are: `local`, `dev`, `qa`, `staging`, `production`. Each one is just:

1. An entry in the `AppEnvironments` enum with its `variables` map (BASE_URL, Env, Firebase options).
2. An `Env<Flavor>` class generated from `env/.env.<flavor>` via `envied`.
3. A `firebase_options_<flavor>.dart`.
4. A `main_<flavor>.dart` that:
   ```dart
   void main() {
     FlavorConfig(
       name: AppEnvironments.dev.name,
       variables: AppEnvironments.dev.variables,
       location: BannerLocation.bottomEnd,
     );
     mainApp();
   }
   ```

`mainApp()` (in `_main.dart`) is shared and reads the active flavor through `FlavorConfig.instance`.

### Where flavor config lives

- Flavor variables: `lib/environments/app_environments.dart`.
- Flavor entry points: `lib/environments/<flavor>/main_<flavor>.dart`.
- Flavor secrets: `env/.env.<flavor>` (git-ignored).
- Flavor Firebase options: `lib/environments/<flavor>/firebase_options_<flavor>.dart`.

### Running each flavor

```
flutter run --flavor dev      -t lib/environments/dev/main_dev.dart
flutter run --flavor qa       -t lib/environments/qa/main_qa.dart
flutter run --flavor staging  -t lib/environments/stag/main_staging.dart
flutter run --flavor prod     -t lib/main.dart
flutter run --flavor local    -t lib/environments/local/main_local.dart
```

(Adjust based on your Android/iOS schemes — the iOS schemes mirror the flavor names.)

### What changes between flavors

- Backend `BASE_URL`.
- Sentry DSN, OneSignal app id, AppsFlyer dev key, server `apiKey` (via `Env*`).
- Firebase project (`firebase_options_*.dart`, `firebase_name`).
- Optional flavor-specific tints — `ColorManager.primary` can be flavor-aware if you choose.

### How to remove flavors from a derived project

1. Delete `lib/environments/` and `env/`.
2. Move the body of `mainApp()` directly into `main()` in `lib/main.dart`.
3. Replace `EnvironmentHelper().getEnvironmentVariable('BASE_URL')` with a single constant (or an `AppConfig` class).
4. Replace any per-flavor primary color in `ColorManager.primary` with one constant.
5. Drop `flutter_flavor` and `envied` from `pubspec.yaml`.

## 15. Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Files | `snake_case.dart` | `news_controller.dart`, `news_remote_data_source.dart` |
| Folders | `snake_case` | `data_source`, `presentation` |
| Classes | `PascalCase` | `NewsController`, `NewsBinding`, `NewsModel` |
| Controllers | `<Feature>Controller` extends `GetxController` | `NewsController`, `AuthController` |
| Bindings | `<Feature>Binding` extends `Bindings` | `NewsBinding`, `CoreBinding` |
| Routes | static const on `AppRoutes`, lowerCamelCase | `news`, `newsDetails`, `showDetails` |
| Route paths | `'/kebab-case'` | `'/news-details'`, `'/show-details'` |
| Repositories | `<Feature>RepositoryAbstraction` + `<Feature>Repository` | `NewsRepositoryAbstraction`, `NewsRepository` |
| Remote data sources | `<Feature>RemoteDataSourceAbstraction` + `<Feature>RemoteDataSource` | `NewsRemoteDataSourceAbstraction` |
| Network endpoints | enum value on `NetworkRouter`, lowerCamelCase | `news`, `userBadges`, `groupCreate` |
| Models | `PascalCase`, optional `Model` suffix when needed for clarity | `User`, `NewsModel`, `RefreshToken` |
| Shared widgets | brand prefix (e.g. `App*`) | `AppButton`, `AppDialog`, `AppListView` |
| Feature widgets | `PascalCase`, no brand prefix | `NewsItem`, `GarageCarCard` |
| State enums | `<Feature>State { initial, loading, pageLoading, success, failure }` | `NewsState`, `ProfileState`, `AuthState` |
| Localization keys | `lowerCamelCase` static const on `LocalizationKeys` | `signUpTitle`, `noNewsFound` |
| Storage keys | enum value on `StorageKeys` (lowerCamelCase) with backing constant in SCREAMING_SNAKE | `accessToken('ACCESS_TOKEN')` |

## 16. Coding Standards

### Organization

- Group related logic in the matching layer: data → repo → controller → view.
- One feature folder = one binding. If a feature has multiple major sub-areas, create one binding per sub-area in the same folder.
- Keep widget files small. If a screen file grows past ~300 lines, extract widgets into `presentation/widgets/`.

### Patterns developers must follow

- All HTTP calls go through `NetworkAdapter` via a `NetworkRequest`.
- All paths come from `NetworkRouter` enum values, never inline.
- All routes come from `AppRoutes` constants, never inline.
- All translatable text comes from `LocalizationKeys` + `.tr`.
- All colors come from `ColorManager()`.
- All typography comes from `AppTypography` / `AppTextTheme`.
- Async data methods return `Future<Either<Failure, T>>`.
- Controllers expose state through `Rx`/`.obs`; views read it through `Obx` or `GetBuilder`.
- Bindings register `data source → repository → controller`.
- Snackbars and dialogs use the `Alerts` mixin or `AppDialog`.

### Patterns that are forbidden

- Calling Dio or `http` directly from a feature.
- Hardcoding endpoint paths or route names.
- Hardcoding text or colors in widgets.
- Putting business logic inside widgets.
- Using `Navigator.push(context, MaterialPageRoute(...))` — always use GetX named routes.
- Using `setState` inside a widget that already has a controller.
- Calling `Get.put` inside `build()` for anything other than legacy code.
- Mutating Rx values from the view layer.

### Keeping files clean

- Imports ordered: dart → flutter → external packages → project imports. Keep them grouped.
- Prefer composition over deep widget trees; lift complex blocks into named widgets.
- Avoid `late` for things that can be `final` and initialized in the constructor or `onInit`.

### Avoiding duplicated logic

- If two features need the same UI piece, move it to `lib/core/presentation/widgets/` with the brand prefix.
- If two features need the same model, move it to `lib/core/domain/models/`.
- If two features need the same service or helper, move it to `lib/core/services/` or `lib/core/utils/`.
- Cross-feature mixins (e.g. `ReactionMixin`, `PostActionsMixin`, `PollVoteMixin`) live next to the feature that owns them and are mixed into other feature controllers.

### Keeping business logic out of widgets

- Data fetching, mutation, validation, and navigation decisions live in the controller.
- The widget reads `controller.<rx>.value` and calls `controller.<method>()`.
- Use `Obx` / `GetBuilder` to react to state, not `StatefulWidget` + `setState`.

## 17. Adding a New Feature: Checklist

Use this for every new feature.

- [ ] Create `lib/features/<feature>/` with `binding/`, `data/data_source/`, `data/repository/`, `domain/models/`, `presentation/`, `presentation/widgets/`.
- [ ] Add new endpoint(s) to `NetworkRouter`.
- [ ] Create the model(s) under `domain/models/` with `fromJson`, `empty()` if needed.
- [ ] Create `data/data_source/<feature>_remote_data_source.dart`:
  - [ ] Abstract class describing the API surface.
  - [ ] Concrete class taking a `NetworkAdapterAbstraction`.
  - [ ] Each method returns `Future<Either<Failure, T>>`.
- [ ] Create `data/repository/<feature>_repo.dart` mirroring the data source's interface.
- [ ] Create `presentation/<feature>_controller.dart`:
  - [ ] Extends `GetxController`, mixes in `Alerts` if it shows snackbars.
  - [ ] Constructor takes `<Feature>RepositoryAbstraction`.
  - [ ] Defines a `XState` enum and the state Rx.
  - [ ] Initializes work in `onInit`, cleans up in `onClose`.
- [ ] Create `presentation/<feature>_screen.dart`:
  - [ ] `StatelessWidget` (or `HookWidget`).
  - [ ] Uses `Get.find<XController>()` or `GetBuilder<XController>()`.
  - [ ] Uses `LocalizationKeys.*.tr`, `ColorManager()`, `AppTypography`, `App*` widgets.
- [ ] Create `binding/<feature>_binding.dart`:
  - [ ] `Get.lazyPut<DataSourceAbstraction>(...)`.
  - [ ] `Get.lazyPut<RepositoryAbstraction>(...)`.
  - [ ] `Get.put(Controller(repository: Get.find()))`.
- [ ] Add a route constant to `AppRoutes`.
- [ ] Register a `GetPage` entry in `AppNavigator.routesList` with the binding.
- [ ] Add any new `LocalizationKeys` and translations in every language file.
- [ ] Test navigation: `Get.toNamed(AppRoutes.<route>)`.
- [ ] Test the loading/success/failure states of the controller.
- [ ] Verify dark mode and RTL look right.

## 18. Do and Don't

### Do

- Do put every API path in `NetworkRouter`.
- Do put every route name in `AppRoutes`.
- Do put every UI string in `LocalizationKeys` with translations in every language file.
- Do return `Either<Failure, T>` from data methods.
- Do drive the UI off a `XState` enum with `Obx`/`GetBuilder`.
- Do use `Get.lazyPut` for data sources and repositories.
- Do use `permanent: true` for controllers that must survive route changes.
- Do guard cross-binding registration with `if (!Get.isRegistered<T>())`.
- Do use `App*` shared widgets and the `ColorManager()` / `AppTypography` design system.
- Do use `AppGo` when you need a custom transition; otherwise plain `Get.toNamed`.

### Don't

- Don't bypass `NetworkAdapter` to hit Dio or `http` directly.
- Don't inline route strings or path strings.
- Don't hardcode UI text — even temporary placeholders should go through `LocalizationKeys`.
- Don't mutate Rx values from inside widgets.
- Don't use `Navigator.push` / `MaterialPageRoute` — always GetX.
- Don't perform business logic inside widgets.
- Don't call `Get.put` inside a `build()` method except in clearly-marked legacy spots.
- Don't add new translations to only one language file.
- Don't read environment variables outside of `EnvironmentHelper`.
- Don't skip the binding for a screen that has a controller.

## 19. New Project Bootstrap Guide

To start a new Flutter project using this architecture from scratch:

1. **Create the project**
   - `flutter create my_app` (org and platform options as needed).
   - Set Dart SDK constraint to `>=3.3.4 <4.0.0` in `pubspec.yaml`.

2. **Add core dependencies**:
   - State / routing: `get`.
   - Networking: `dio`, `pretty_dio_logger`, `curl_logger_dio_interceptor`.
   - Storage: `shared_preferences`, `flutter_secure_storage`.
   - Functional types: `dartz`.
   - Optional: `flutter_hooks`, `flutter_flavor`, `envied` + `envied_generator` (if you need flavors), Sentry, Firebase, etc.

3. **Set up the core folders**
   ```
   lib/
   ├── core/
   │   ├── cache/             # local_storage.dart, secure_storage.dart, storage_keys.dart
   │   ├── data/
   │   │   ├── data_sources/
   │   │   └── networking/
   │   │       ├── network_adapter.dart
   │   │       └── data/
   │   │           ├── network_request.dart
   │   │           ├── network_response.dart
   │   │           └── network_router.dart
   │   ├── domain/
   │   │   ├── errors/failure.dart
   │   │   ├── models/
   │   │   ├── routing/
   │   │   │   ├── app_routes.dart
   │   │   │   ├── app_navigator.dart
   │   │   │   ├── app_go.dart
   │   │   │   └── core_binding.dart
   │   │   └── utils/
   │   ├── presentation/
   │   │   ├── controllers/   # ThemeController, LocalizationController
   │   │   ├── localization/  # localization_keys.dart, language_registry.dart, languages/
   │   │   ├── theme/         # app_theme.dart, color_manager.dart, text_manager.dart, components/
   │   │   └── widgets/       # App* shared widgets
   │   ├── services/          # token_manager_service.dart, session_manager_service.dart
   │   └── utils/
   ├── features/
   └── main.dart
   ```

4. **Routing setup**
   - Create the routes constants class (`AppRoutes`).
   - Create the navigator class with a `static List<GetPage> get routesList`.
   - Create `CoreBinding` registering `LocalStorageService`, `SecureStorageService`, `NetworkAdapter`, `TokenManagerService`, `SessionManagerService`, and any project-wide data source.

5. **GetX setup in `main.dart`**
   ```dart
   void main() async {
     final widgetsBinding = WidgetsFlutterBinding.ensureInitialized();
     await LocalStorageService().init();
     await SecureStorageService().init();
     Get.put(LocalizationController());
     Get.put(ThemeController());
     runApp(const MyApp());
   }

   class MyApp extends StatelessWidget {
     @override
     Widget build(BuildContext context) {
       final loc = Get.find<LocalizationController>();
       final theme = Get.find<ThemeController>();
       return Obx(() => GetMaterialApp(
         translations: AppLocalization(),
         locale: loc.currentLanguage.locale,
         supportedLocales: LanguageRegistry.supportedLocales(),
         fallbackLocale: const Locale('en'),
         theme: AppTheme().light,
         darkTheme: AppTheme().dark,
         themeMode: theme.themeMode,
         defaultTransition: GoTransitions.slide,
         transitionDuration: GoTransitions.normalDuration,
         getPages: AppNavigator.routesList,
         initialBinding: CoreBinding(),
         initialRoute: AppRoutes.splash,
       ));
     }
   }
   ```

6. **Theme setup**
   - Add `ColorManager`, `AppTextTheme`, `AppTypography`, `SchemeManager`, `GradientManager`.
   - Compose them in `AppTheme.light` / `AppTheme.dark`.
   - Wire `ThemeController` to persist `ThemeMode` in `LocalStorageService`.

7. **Localization setup**
   - Define `LocalizationKeys`.
   - Add `languages/en.dart` and any other languages.
   - Register them in `LanguageRegistry.supported`.
   - Wire `LocalizationController` to update the locale and persist it.

8. **Env setup (optional, only if you need flavors)**
   - Add `flutter_flavor` and `envied`.
   - Mirror the `lib/environments/` folder layout from section 13.
   - Otherwise: replace with a single `AppConfig` class containing `BASE_URL` and any keys.

9. **First feature setup**
   - Build the splash feature first: splash binding, splash controller, splash screen, splash repo, splash data source. Use it as the integration test of the whole stack.
   - Then add auth or whatever is the first real flow.

## 20. Final Developer Notes

These are the rules that matter most. If you only remember a handful of things from this document:

- **One feature = one folder** with `binding/`, `data/data_source/`, `data/repository/`, `domain/models/`, `presentation/`. Always.
- **One binding per route**, registered in `AppNavigator.routesList`. Bindings register data source → repository → controller.
- **All HTTP goes through `NetworkAdapter`** with a `NetworkRequest` whose route is a `NetworkRouter` enum value. Data methods return `Either<Failure, T>`.
- **All routes are `AppRoutes` constants**. Navigate with `Get.toNamed`/`Get.offNamed`/`Get.offAllNamed`/`Get.back`, or `AppGo.*` if you need a custom transition.
- **All UI text is `LocalizationKeys.*.tr`**, with values in every language file.
- **All colors come from `ColorManager()`**, all typography from `AppTypography` / `AppTextTheme`, all reusable UI from `App*` widgets.
- **Controllers extend `GetxController`** and expose state via Rx. Views read it via `Obx` or `GetBuilder`. No business logic in widgets.
- **Global state** (token manager, session manager, network adapter, profile controller, theme controller, localization controller) is registered once and reused everywhere.
- **Flavors are optional infrastructure**, not part of the core architecture. Strip them out cleanly when a project doesn't need them — the rest of the architecture stands on its own.
