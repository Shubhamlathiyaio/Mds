# PATTERNS.md — GetX Boilerplate
> How key patterns work. Reference before implementing any of these.

---

## AppScaffold

The outer shell for every page. Handles: AppBar, divider, SystemUiOverlay, SafeArea padding, body expansion.

```dart
AppScaffold(
  title: context.T.pageTitle,         // String title — centers automatically
  titleWidget: MyCustomHeader(),       // OR custom widget — overrides title
  showBackBtn: true,                   // default true; false for root pages
  onTapBackBtn: () => ctrl.onBack(),   // optional override for back action
  action: IconButton(...),             // top-right action (45×45 box)
  showDivider: true,                   // divider below app bar
  showAppBar: false,                   // hide entire app bar area
  bottomNavigationBar: MyNavBar(),
  floatingActionButton: MyFAB(),
  resizeToAvoidBottomInset: true,      // false for pages with bottom sheets
  systemUiOverlayStyle: DarkSystemUiOverlayStyle.style2,  // override if needed
  body: (context) {
    return ListView(...);   // body is a WidgetBuilder — context is available
  },
)
```

Key behaviors:
- `body` is a `WidgetBuilder` (not a `Widget`) — always pass `(context) => YourWidget`
- `paddingTop` param removes default safe area top padding from body when set (use for full-bleed content)
- `showBgLogo: true` replaces `bottomNavigationBar` with the app background logo

---

## SystemUiOverlayStyle

Three named styles. Match scaffold/page background color to avoid nav bar color mismatch.

```dart
// Standard pages with white background
DarkSystemUiOverlayStyle(child: scaffold)
// Status bar icons: dark | Nav bar bg: KColors.bg1 | Nav bar icons: dark

// Pages with dark/colored background (onboarding, full-screen media)
OnboardingSystemUiOverlayStyle(child: scaffold)
// Status bar icons: dark | Nav bar bg: KColors.onboarding

// Splash page only
SplashSystemUiOverlayStyle(child: scaffold)
// Adds systemNavigationBarDividerColor: transparent
```

Custom override (rare):
```dart
AppScaffold(
  systemUiOverlayStyle: const SystemUiOverlayStyle(
    statusBarIconBrightness: Brightness.light,
    systemNavigationBarColor: KColors.primary,
    systemNavigationBarIconBrightness: Brightness.light,
  ),
  body: (context) => ...,
)
```

Rule: the `systemNavigationBarColor` must match the color visible behind the bottom navigation bar — otherwise the nav bar floats over the wrong background color on Android.

---

## ImageView

Handles SVG / asset / network / file — auto-detected from path.

```dart
// Basic
ImageView(Assets.images.logo.path)
ImageView('https://example.com/photo.jpg')

// With size constraints
ImageView(
  Assets.images.hero.path,
  fit: BoxFit.contain,
  inner: const ImageSize(height: 200, width: 200),
)

// Circular network avatar
ImageView(
  user.avatarUrl,
  decoration: const BoxDecoration(shape: BoxShape.circle),
  inner: const ImageSize(dimension: 48, shouldClip: true),
)

// Rounded corners
ImageView(
  Assets.images.card.path,
  decoration: BoxDecoration(borderRadius: BorderRadius.circular(12)),
  inner: ImageSize(shouldClip: true, height: 160, width: double.infinity),
)

// With color tint (for SVG icons)
ImageView(Assets.icons.heart.path, color: colors.primary)

// Custom error widget
ImageView(url, errorWidget: const Icon(Icons.broken_image))

// Custom loader
ImageView(url, loaderBuilder: (progress) => LinearProgressIndicator(value: progress))
```

`inner` = SizedBox wrapping the image itself (clips first).
`outer` = SizedBox wrapping everything after decoration/clipping.

---

## AppFormFocus + AppFocusScope

### AppFocusScope
Wraps any widget tree — dismisses keyboard on background tap.
```dart
AppFocusScope(child: myContent)
// Programmatic dismiss:
AppFocusScope.unfocus(context)
```

### AppFormFocus
Full form: `Form` + `AppFocusScope` + validation helpers.
```dart
AppFormFocus(
  child: Column(
    children: [
      TextInputField(
        type: InputType.email,
        controller: ctrl.emailCtrl,
        validator: (v) => v!.isEmpty ? context.T.fieldRequired : null,
      ),
      AppButton(
        title: context.T.submit,
        onPressed: () {
          // Validates + unfocuses + scrolls to first error
          if (AppFormFocus.validateWithScroll(context)) {
            ctrl.submit();
          }
        },
      ),
    ],
  ),
)

// Simple validate (no scroll)
AppFormFocus.validate(context)
```

---

## BottomNavigationBar

```dart
// BottomBarController — @lazySingleton, autoDispose: false
@lazySingleton
class BottomBarController extends GetxController {
  final selectedIndex = 0.obs;

  void changeIndex(int index) {
    // Add any guard logic here (e.g. block nav during active operation)
    selectedIndex.value = index;
  }
}

// Root page
class RootPage extends GetItHook<BottomBarController> {
  const RootPage({super.key});

  @override
  bool get autoDispose => false;   // Persistent controller

  static const _pages = [HomePage(), SearchPage(), ProfilePage()];

  @override
  Widget build(BuildContext context) {
    return Obx(() => AppScaffold(
      showAppBar: false,
      body: (context) => _pages[controller.selectedIndex.value],
      bottomNavigationBar: SafeArea(
        child: BottomNavigationBar(
          currentIndex: controller.selectedIndex.value,
          onTap: controller.changeIndex,
          items: const [
            BottomNavigationBarItem(icon: Icon(Icons.home), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.search), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.person), label: ''),
          ],
        ),
      ),
    ));
  }
}
```

The `SafeArea` on `bottomNavigationBar` ensures the bar never hides behind the system nav bar.
Customize items, icons, and styling per project — the controller pattern stays the same.

---

## Extensions Reference

```dart
// BuildContext
context.T                    // AppLocalizations
context.colors               // AppColors.of(context)
context.styles               // AppStyles.of(context)
context.screenWidth          // MediaQuery width
context.screenHeight         // MediaQuery height
context.isKeyboardOpen       // MediaQuery.viewInsets.bottom > 0
context.colorScheme          // Theme.of(context).colorScheme

// Color
color.changeOpacity(0.5)     // ALWAYS use instead of withOpacity / withAlpha

// String
string.isValidEmail
string.isValidPhone
string.toCapitalized         // "hello world" → "Hello World"
string.convertMd5

// DateTime
dateTime.toFormattedString('dd MMM yyyy')
dateTime.isToday
dateTime.isYesterday

// num
value.clamp01                // .clamp(0.0, 1.0)
value.toDouble               // .toDouble()
value.isNegative

// List
list.firstOrNull
list.lastOrNull

// Logging (debug only, returns self for chaining)
someValue.log                // dev.log(value.toString())
someValue.logWithName('tag')
```

---

## Theme Extension Pattern

When adding a new color or style — always add to the class, never inline.

```dart
// 1. Add to AppColors
final Color myNewColor;
const AppColors({..., this.myNewColor = KColors.someConstant});

// 2. Use it
AppColors.of(context).myNewColor

// 3. For dark mode: override in darkTheme AppColors instance
```

When adding a new text style:
```dart
// 1. Add to AppStyles
this.s18w500Primary = const Poppins(fontSize: 18, fontWeight: FontWeight.w500, color: KColors.primary),
final TextStyle s18w500Primary;

// 2. Use it
AppStyles.of(context).s18w500Primary
```

Naming convention: `s{fontSize}w{weight}{ColorName}`.

---

## Loading / EasyLoading

```dart
Loading.show()      // show spinner
Loading.dismiss()   // hide spinner
```

`Loading` is configured in `configuration()` via `Loading().configLoading()`.
Never use `showDialog` for loading states — use EasyLoading.

---

## Locale / Language Switching

```dart
// Change locale anywhere
getIt<LocaleController>().changeLocale('ar');  // Arabic
getIt<LocaleController>().changeLocale('en');  // English

// Current locale
getIt<LocaleController>().locale   // Locale object
```

Locale persisted in `GetStorage` automatically. `GetMaterialApp.locale` binds to `LocaleController`.
