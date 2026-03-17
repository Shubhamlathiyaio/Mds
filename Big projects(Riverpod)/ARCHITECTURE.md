# ARCHITECTURE.md — Riverpod Boilerplate
> Large / long-lived projects. 5+ features. Expected to grow.
> Stack: Riverpod (code-gen) · GetIt + Injectable · go_router · Retrofit + Dio · Hive · Flutter Gen · EasyLoading

---

## Folder Structure

```
lib/
├── main.dart
├── firebase_options.dart          # Only if Firebase is enabled
├── gen/                           # flutter_gen output (assets, fonts)
├── l10n/                          # ARB files + generated AppLocalizations
└── app/
    ├── providers/                 # @riverpod notifiers — one file per feature
    ├── data/
    │   ├── models/                # Freezed models + JSON serialization
    │   ├── repositories/          # Bridge between providers and services
    │   └── services/              # Retrofit API services (registered via GetIt)
    ├── global/                    # AppConfig, EndPoints
    ├── routes/                    # AppRoutes (constants) + router.dart (GoRouter)
    ├── ui/
    │   ├── pages/                 # One folder per feature
    │   └── widgets/               # Shared widgets
    └── utils/
        ├── helpers/               # exporter.dart, injectable/, extensions/, loading.dart
        └── themes/                # app_theme.dart (parts: app_colors, app_styles, …)
```

---

## Entry Point

```dart
// main.dart
void main() => configuration(myApp: const ProviderScope(child: MyApp()));
```

Wrap `MyApp` in `ProviderScope` — this is the Riverpod root. Never place `ProviderScope` anywhere else.

```dart
class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeNotifierProvider);
    return MaterialApp.router(
      routerConfig: appRouter,
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      locale: locale,
      themeMode: ThemeMode.light,
      theme: AppTheme.lightTheme,
      builder: EasyLoading.init(
        builder: (context, child) {
          return TextFieldStyleProvider(
            key: TextFieldStyleProvider.styleKey,
            style: WidgetStateTextStyle.resolveWith(
              (states) => AppStyles.of(context).s16w400Black,
            ),
            child: ButtonTheme(alignedDropdown: true, child: child!),
          );
        },
      ),
    );
  }
}
```

`configuration()` in `utils/helpers/injectable/injectable.dart`:
1. `WidgetsFlutterBinding.ensureInitialized()`
2. `await getIt.init()` — opens Hive boxes, creates Dio, registers repositories
3. *(Optional)* Firebase Crashlytics — only when project requires it
4. `Loading().configLoading()`
5. `runApp(myApp)`

> **Firebase is optional.** Add only when the project explicitly requires it.

---

## Dependency Injection — GetIt + Injectable

GetIt owns **infrastructure**: Dio, Retrofit services, Hive boxes, repositories.
Riverpod owns **UI state**: notifiers, async data, derived state.

They solve different problems. They do not compete.

```
Widget → ref.watch(someProvider) → Notifier → getIt<Repository>() → getIt<Service>() → Dio
```

```dart
@module
abstract class RegisterModule {
  @singleton
  Dio dio() => Dio(BaseOptions(baseUrl: AppConfig.baseUrl))
    ..interceptors.addAll([AppDioInterceptor(), if (kDebugMode) PrettyDioLogger()]);

  @preResolve
  Future<Box> appBox() async {
    await Hive.initFlutter();
    return Hive.openBox('app');
  }
}
```

Rules:
- API services → `@lazySingleton` with `@factoryMethod`
- Repositories → `@lazySingleton`
- Hive boxes / async one-time setup → `@preResolve`
- Never instantiate services inside providers — always `getIt<T>()`
- Never inject Riverpod `Ref` into a repository — keep repositories Riverpod-agnostic

---

## State — Riverpod Code-Gen

Always use `@riverpod` annotation. Never write providers manually.

```dart
// providers/home_provider.dart
part 'home_provider.g.dart';

@riverpod
class HomeNotifier extends _$HomeNotifier {
  @override
  HomeState build() => const HomeState.initial();

  Future<void> fetchData() async {
    state = const HomeState.loading();
    try {
      final result = await getIt<HomeRepository>().getData();
      state = HomeState.data(result);
    } catch (e, st) {
      (e, st).log;
      state = HomeState.error(e.toString());
    }
  }
}
```

State models use `freezed` unions:
```dart
@freezed
class HomeState with _$HomeState {
  const factory HomeState.initial() = _Initial;
  const factory HomeState.loading() = _Loading;
  const factory HomeState.data(List<Item> items) = _Data;
  const factory HomeState.error(String message) = _Error;
}
```

Provider auto-dispose rules:
- `@riverpod` (default) → auto-disposes when no listeners — use for screen-scoped state
- `@Riverpod(keepAlive: true)` → never disposes — use for global/shared state (locale, auth, bottom nav)

---

## Routing

See `NAVIGATION.md` for full setup. Summary:

```dart
// routes/app_routes.dart
class AppRoutes {
  static const splash   = '/splash';
  static const home     = '/home';
  static const search   = '/search';
  static const profile  = '/profile';
  // Sub-routes: '/home/detail/:id'
}
```

Navigation from anywhere with `BuildContext`:
```dart
context.go(AppRoutes.home)       // replace stack
context.push(AppRoutes.detail)   // push
context.pop()
```

Never use `Navigator.push`. Never hardcode route strings.

---

## Theme System

Identical three-layer system across both boilerplates:

- `KColors` → raw constants, never used directly in widgets
- `AppColors.of(context)` → semantic, context-aware colors
- `AppStyles.of(context)` → typography via `Poppins` class

Dark theme: create `AppTheme.darkTheme` with a second `AppColors` instance that overrides semantic tokens (`bg1`, `fieldColor`, `black`, etc.). `KColors` never changes between themes.

---

## Localization

Access strings **only** via `context.T` inside widgets.

```dart
extension BuildContextX on BuildContext {
  AppLocalizations get T => AppLocalizations.of(this)!;
}
```

> There is no `AppStrings.T` in this boilerplate — `Get.context!` does not exist.
> If a notifier needs a user-facing string, the widget passes it as a parameter.

```dart
// providers/locale_provider.dart
@Riverpod(keepAlive: true)
class LocaleNotifier extends _$LocaleNotifier {
  @override
  Locale build() {
    final saved = getIt<Box>().get('locale') ?? 'en';
    return Locale(saved);
  }

  void changeLocale(String langCode) {
    getIt<Box>().put('locale', langCode);
    state = Locale(langCode);
  }
}
```

Change locale from anywhere with a `WidgetRef`:
```dart
ref.read(localeNotifierProvider.notifier).changeLocale('ar');
```

---

## API Layer

Repositories wrap services. Providers call repositories. Never call services directly from providers.

```dart
// services/auth_service.dart
@lazySingleton
@RestApi()
abstract class AuthService {
  @factoryMethod
  factory AuthService(Dio dio) = _AuthService;

  @POST(EndPoints.login)
  Future<LoginResponse> login(@Body() LoginRequest request);
}

// data/repositories/auth_repository.dart
@lazySingleton
class AuthRepository {
  AuthRepository(this._service);
  final AuthService _service;

  Future<User> login(String email, String password) async {
    final res = await _service.login(LoginRequest(email: email, password: password));
    return res.user;
  }
}

// providers/auth_provider.dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  AuthState build() => const AuthState.initial();

  Future<void> login(String email, String password) async {
    state = const AuthState.loading();
    try {
      final user = await getIt<AuthRepository>().login(email, password);
      state = AuthState.authenticated(user);
    } catch (e, st) {
      (e, st).log;
      state = AuthState.error(e.toString());
    }
  }
}
```

---

## Local Storage — Hive

```dart
getIt<Box>().get('key', defaultValue: fallback)
getIt<Box>().put('key', value)
getIt<Box>().delete('key')
```

For typed entities, register a separate `Box<T>` in `RegisterModule` with `@preResolve`.

---

## Code Generation

```bash
dart run build_runner build --delete-conflicting-outputs
# Watch mode:
dart run build_runner watch --delete-conflicting-outputs
```

Affects: `freezed`, `json_serializable`, `injectable`, `retrofit`, `riverpod_generator`, `flutter_gen`.
Commit all generated files.
