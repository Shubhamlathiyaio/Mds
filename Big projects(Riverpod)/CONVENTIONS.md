# CONVENTIONS.md — Riverpod Boilerplate
> Rules the AI must follow without exception. If a requirement is unclear, ask before writing code.

---

## Pages

Every page MUST extend `ConsumerStatefulWidget`:

```dart
class HomePage extends ConsumerStatefulWidget {
  const HomePage({super.key});

  @override
  ConsumerState<HomePage> createState() => _HomePageState();
}

class _HomePageState extends ConsumerState<HomePage> {
  @override
  Widget build(BuildContext context) {
    final state = ref.watch(homeNotifierProvider);
    return AppScaffold(
      title: context.T.homeTitle,
      body: (context) => ...,
    );
  }
}
```

Use `ConsumerWidget` only for small, stateless shared widgets that read providers.
Use `ConsumerStatefulWidget` for all pages — consistency and future flexibility.

---

## Reading Providers

```dart
// Watch — rebuilds widget when state changes (use inside build)
final state = ref.watch(homeNotifierProvider);

// Read — one-time, no rebuild (use inside callbacks)
ref.read(homeNotifierProvider.notifier).fetchData();

// Listen — side effects on change (use in build, not initState)
ref.listen(homeNotifierProvider, (prev, next) {
  next.whenOrNull(error: (msg) => Loading.dismiss());
});
```

Never call `ref.watch` inside a callback or `initState`.
Never call `ref.read` inside `build()` for display values.

---

## Widgets

| Situation | Use | Never use |
|---|---|---|
| Any image (asset/network/svg/file) | `ImageView(path)` | `Image.asset`, `Image.network`, `CachedNetworkImage`, `SvgPicture` |
| Button | `AppButton(title:, onPressed:, type:)` | Raw `ElevatedButton`, `TextButton`, `OutlinedButton` |
| Text input | `TextInputField(type:, controller:)` | Raw `TextField`, `TextFormField` |
| Dropdown | `CustomDropdown(list:, onSelect:)` | Raw `DropdownButton` |
| Page outer shell | `AppScaffold(body:)` | Raw `Scaffold` in pages |
| Form wrapper | `AppFormFocus(child:)` | Raw `Form` |
| Keyboard dismiss | `AppFocusScope(child:)` | Manual `GestureDetector` + `FocusScope` |

Only use raw Flutter widgets inside custom widget implementations.

---

## Colors

```dart
// CORRECT
final colors = context.colors;   // AppColors.of(context)
Container(color: colors.primary)
Text('x', style: style.s16w400Black.copyWith(color: colors.red))

// WRONG
Container(color: const Color(0xFF00868B))
Container(color: KColors.primary)
Container(color: Colors.red)
```

For opacity — always use `changeOpacity`:
```dart
colors.primary.changeOpacity(0.5)   // CORRECT
colors.primary.withOpacity(0.5)     // WRONG — deprecated
```

---

## Text Styles

```dart
// CORRECT
Text('Hello', style: context.styles.s16w400Black)
Text('Hello', style: context.styles.s16w400Black.copyWith(color: colors.primary))

// WRONG
Text('Hello', style: const TextStyle(fontSize: 16))
Text('Hello', style: const Poppins(fontSize: 16))
```

If a needed style does not exist in `AppStyles`, add it there first.
Naming convention: `s{fontSize}w{weight}{ColorName}` — e.g. `s14w500Primary`.

---

## Strings / Localization

```dart
// CORRECT — inside widgets only
Text(context.T.welcomeMessage)
AppButton(title: context.T.loginButton, ...)

// If notifier needs a string — pass it from the widget
ref.read(authNotifierProvider.notifier)
    .login(email, password, errorLabel: context.T.loginFailed)

// WRONG
Text('Welcome')            // hardcoded
Text('welcome'.tr)         // GetX — not used in this project
AppStrings.T.someKey       // does not exist in Riverpod boilerplate
```

---

## Navigation

```dart
context.go(AppRoutes.home)              // replace stack
context.push(AppRoutes.detail)          // push
context.pop()
context.pop(result)
context.go('/home/detail/$id')          // path params
context.push(AppRoutes.detail, extra: myObject)  // object (not URL-safe)
```

Never use `Navigator.push` or `Navigator.pop` directly.
Never hardcode route strings — always `AppRoutes` constants.
All routes defined in `routes/router.dart`.

---

## State Updates

```dart
// In notifier — always set all states
state = const HomeState.loading();
state = HomeState.data(items);
state = HomeState.error(message);

// In widget — handle every state
state.when(
  initial: () => const SizedBox(),
  loading: () => const AppProgressIndicator(),
  data: (items) => ItemList(items: items),
  error: (msg) => ErrorView(message: msg),
)
```

Never use `setState` for business logic — only for local UI state (animation controllers, etc.).

---

## System UI / Status Bar

```dart
DarkSystemUiOverlayStyle(child: scaffold)         // standard light pages
SplashSystemUiOverlayStyle(child: scaffold)        // splash only
OnboardingSystemUiOverlayStyle(child: scaffold)    // dark background pages
```

`AppScaffold` applies `DarkSystemUiOverlayStyle` by default.
Override via `systemUiOverlayStyle` param when needed.
Never call `SystemChrome.setSystemUIOverlayStyle()` imperatively.

---

## Naming

| Item | Convention | Example |
|---|---|---|
| Files | `snake_case` | `home_provider.dart` |
| Notifier class | `PascalCase` + `Notifier` | `HomeNotifier` |
| Provider variable | `camelCase` + `Provider` | `homeNotifierProvider` |
| State class | `PascalCase` + `State` | `HomeState` |
| Repository class | `PascalCase` + `Repository` | `HomeRepository` |
| Route constants | `camelCase` string | `static const home = '/home'` |
| Page files | `_page.dart` | `home_page.dart` |
| Provider files | `_provider.dart` | `home_provider.dart` |
| Model files | `_model.dart` | `user_model.dart` |

---

## Notifier Rules

- Business logic only in notifiers, never in `build()`
- `TextEditingController` lives in `ConsumerState`, disposed in `dispose()`
- Notifiers access infrastructure only via `getIt<Repository>()`
- Notifiers never import widgets or `BuildContext`
- Global/persistent state → `@Riverpod(keepAlive: true)` (auth, locale, bottom nav)
- Screen-scoped state → `@riverpod` (auto-disposes when page leaves)

```dart
class _HomePageState extends ConsumerState<HomePage> {
  final _emailCtrl = TextEditingController();

  @override
  void dispose() {
    _emailCtrl.dispose();
    super.dispose();
  }
}
```

---

## Forms

```dart
AppFormFocus(
  child: Column(
    children: [
      TextInputField(
        type: InputType.email,
        controller: _emailCtrl,
        validator: (v) => v!.isEmpty ? context.T.fieldRequired : null,
      ),
      AppButton(
        title: context.T.submit,
        onPressed: () {
          if (AppFormFocus.validateWithScroll(context)) {
            ref.read(authNotifierProvider.notifier).login(_emailCtrl.text);
          }
        },
      ),
    ],
  ),
)
```

---

## Assets

Always use the generated `Assets` class:
```dart
ImageView(Assets.images.logo.path)
ImageView(Assets.icons.arrowBack.path)
```

Never hardcode asset path strings.

---

## Code Generation

After any model / provider / service / injectable change:
```bash
dart run build_runner build --delete-conflicting-outputs
```
