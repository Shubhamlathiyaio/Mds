# CONVENTIONS.md — GetX Boilerplate
> Rules the AI must follow without exception. If a requirement is unclear, ask before writing code.

---

## Pages

Every page MUST extend `GetItHook<ControllerType>`:
```dart
class MyPage extends GetItHook<MyController> {
  const MyPage({super.key});

  @override
  bool get autoDispose => true;

  @override
  Widget build(BuildContext context) { ... }
}
```

Never extend `StatelessWidget`, `StatefulWidget`, or `GetView` for pages.

---

## Widgets

| Situation | Use | Never use |
|---|---|---|
| Any image (asset/network/svg/file) | `ImageView(path)` | `Image.asset`, `Image.network`, `CachedNetworkImage`, `SvgPicture` |
| Button | `AppButton(title:, onPressed:, type:)` | `ElevatedButton`, `TextButton`, `OutlinedButton` raw |
| Text input | `TextInputField(type:, controller:)` | `TextField`, `TextFormField` raw |
| Dropdown | `CustomDropdown(list:, onSelect:)` | `DropdownButton` raw |
| Page outer shell | `AppScaffold(body:)` | `Scaffold` raw in pages |
| Form wrapper | `AppFormFocus(child:)` | `Form` raw |
| Keyboard dismiss | `AppFocusScope(child:)` | `GestureDetector` + `FocusScope` manual |

Only use raw Flutter widgets inside the custom widget implementations themselves.

---

## Colors

```dart
// CORRECT
final colors = AppColors.of(context);
Container(color: colors.primary)
Text('...', style: style.s16w400Black.copyWith(color: colors.red))

// WRONG — never do these
Container(color: const Color(0xFF00868B))   // hardcoded
Container(color: KColors.primary)           // raw constant in widget
Container(color: Colors.red)                // Material color
```

For opacity — always use `ColorX.changeOpacity`:
```dart
colors.primary.changeOpacity(0.5)   // CORRECT
colors.primary.withOpacity(0.5)     // WRONG — deprecated
colors.primary.withAlpha(128)       // WRONG — use the extension
```

---

## Text Styles

```dart
// CORRECT
Text('Hello', style: AppStyles.of(context).s16w400Black)

// Add a one-off override with copyWith
Text('Hello', style: AppStyles.of(context).s16w400Black.copyWith(color: colors.primary))

// WRONG
Text('Hello', style: TextStyle(fontSize: 16, color: Colors.black))  // hardcoded
Text('Hello', style: const Poppins(fontSize: 16))                   // bypasses theme
```

Never hardcode `FontWeight`, `fontSize`, or `Color` in widgets. If a style doesn't exist in `AppStyles`, add it there first.

---

## Strings / Localization

```dart
// Inside widgets
Text(context.T.welcomeMessage)
AppButton(title: context.T.loginButton, ...)

// Inside controllers (no BuildContext)
AppStrings.T.errorMessage

// WRONG
Text('Welcome')         // hardcoded
Text('welcome'.tr)      // GetX translation system — not used
```

---

## Navigation

```dart
Get.toNamed(AppRoutes.home)
Get.offAllNamed(AppRoutes.splash)
Get.back()
Get.back(result: someValue)

// With arguments
Get.toNamed(AppRoutes.detail, arguments: {'id': 42})
// Receive
final args = Get.arguments as Map;
```

Never use `Navigator.push`, `Navigator.pop`, or `context.push` directly.
Never hardcode route strings — always use `AppRoutes` constants.

---

## State (Reactive)

```dart
// In controller
final count = 0.obs;
final user = Rxn<UserModel>();   // nullable reactive

// In widget
Obx(() => Text('${controller.count}'))
ObxAny(builder: (context, child) => ..., child: someWidget)

// Update
controller.count.value++;
controller.user.value = fetchedUser;
```

Do not use `setState` in pages. Do not use `StreamBuilder` unless wrapping a non-GetX stream.

---

## System UI / Status Bar

Every page must be wrapped in the correct `SystemUiOverlayStyle`:

```dart
// Standard page (light background)
DarkSystemUiOverlayStyle(child: scaffold)

// Splash page
SplashSystemUiOverlayStyle(child: scaffold)

// Onboarding / dark background pages
OnboardingSystemUiOverlayStyle(child: scaffold)
```

`AppScaffold` uses `DarkSystemUiOverlayStyle` by default.
Pass `systemUiOverlayStyle` param to override per-page if needed.
Never set `SystemChrome.setSystemUIOverlayStyle()` imperatively in pages.

---

## Naming

| Item | Convention | Example |
|---|---|---|
| Files | `snake_case` | `home_controller.dart` |
| Classes | `PascalCase` | `HomeController` |
| Variables / methods | `camelCase` | `isLoading`, `fetchUser()` |
| Route constants | `camelCase` string | `static const home = '/home'` |
| Reactive variables | suffix `.obs` | `final items = <Item>[].obs` |
| Private fields | leading `_` | `final _dio = ...` |
| Model files | `_model.dart` suffix | `user_model.dart` |
| Page files | `_page.dart` suffix | `home_page.dart` |
| Controller files | `_controller.dart` suffix | `home_controller.dart` |

---

## Controller Rules

- All controllers annotated `@lazySingleton`
- All controllers extend `GetxController`
- Business logic lives in controllers, never in page `build()`
- `TextEditingController` instances live in the feature controller, disposed in `onClose()`
- Never call `Get.context!` inside a controller — use `AppStrings.T` for strings

```dart
@lazySingleton
class HomeController extends GetxController {
  final emailCtrl = TextEditingController();

  @override
  void onClose() {
    emailCtrl.dispose();
    super.onClose();
  }
}
```

---

## Forms

```dart
// Always wrap form pages with AppFormFocus
AppFormFocus(
  child: Column(
    children: [
      TextInputField(type: InputType.email, controller: ctrl.emailCtrl),
      AppButton(
        title: context.T.submit,
        onPressed: () {
          if (AppFormFocus.validateWithScroll(context)) {
            ctrl.submit();
          }
        },
      ),
    ],
  ),
)
```

Validation scrolls to the first error field automatically.
Keyboard dismisses on background tap automatically.

---

## Assets

Always access assets via generated `Assets` class (flutter_gen):
```dart
ImageView(Assets.images.logo.path)
ImageView(Assets.icons.arrowBack.path)
```

Never hardcode asset paths as strings directly.

---

## Code Generation

After adding/modifying a model, service, or injectable:
```bash
dart run build_runner build --delete-conflicting-outputs
```

Commit generated files. Do not `.gitignore` them.
