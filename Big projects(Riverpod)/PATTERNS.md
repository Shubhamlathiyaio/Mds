# PATTERNS.md — Riverpod Boilerplate
> How key patterns work. Reference before implementing any of these.

---

## AppScaffold

Identical to GetX boilerplate. See GetX `PATTERNS.md` for full parameter reference.

```dart
AppScaffold(
  title: context.T.pageTitle,
  showBackBtn: true,
  onTapBackBtn: () => context.pop(),   // use context.pop() — not Get.back()
  action: IconButton(...),
  bottomNavigationBar: MyNavBar(),
  resizeToAvoidBottomInset: true,
  body: (context) {
    return ListView(...);
  },
)
```

`body` is always a `WidgetBuilder` — pass `(context) => YourWidget`, never a plain `Widget`.

---

## SystemUiOverlayStyle

```dart
DarkSystemUiOverlayStyle(child: scaffold)       // standard light pages
SplashSystemUiOverlayStyle(child: scaffold)      // splash only
OnboardingSystemUiOverlayStyle(child: scaffold)  // dark/colored background
```

`systemNavigationBarColor` must match the visible color behind the bottom nav bar on Android — otherwise the bar sits on the wrong background.

---

## ImageView

Same API as GetX boilerplate:

```dart
ImageView(Assets.images.logo.path)
ImageView('https://cdn.example.com/photo.jpg')

// Circular avatar
ImageView(
  user.avatarUrl,
  decoration: const BoxDecoration(shape: BoxShape.circle),
  inner: const ImageSize(dimension: 48, shouldClip: true),
)

// Rounded corners
ImageView(
  Assets.images.card.path,
  decoration: BoxDecoration(borderRadius: BorderRadius.circular(12)),
  inner: const ImageSize(shouldClip: true, height: 160),
)

// SVG with color tint
ImageView(Assets.icons.heart.path, color: context.colors.primary)
```

---

## AppFormFocus + AppFocusScope

No GetX dependency — these work identically in both boilerplates.

```dart
// Dismiss keyboard on background tap
AppFocusScope(child: myContent)
AppFocusScope.unfocus(context)   // programmatic

// Full form with validation
AppFormFocus(
  child: Column(
    children: [
      TextInputField(
        type: InputType.email,
        controller: _emailCtrl,
        validator: (v) => v!.isEmpty ? context.T.required : null,
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

`validateWithScroll` — validates, unfocuses, scrolls to first error field automatically.

---

## Bottom Navigation + ShellRoute

See `NAVIGATION.md` for full router setup. The page pattern:

```dart
// Bottom nav index managed by a keepAlive provider
@Riverpod(keepAlive: true)
class BottomNavNotifier extends _$BottomNavNotifier {
  @override
  int build() => 0;

  void changeIndex(int index) => state = index;
}

// Root shell page — receives the active tab as child from ShellRoute
class RootPage extends ConsumerWidget {
  const RootPage({super.key, required this.child});
  final Widget child;

  static const _routes = [
    AppRoutes.home,
    AppRoutes.search,
    AppRoutes.profile,
  ];

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final index = ref.watch(bottomNavNotifierProvider);
    return AppScaffold(
      showAppBar: false,
      body: (context) => child,   // ShellRoute provides the active tab
      bottomNavigationBar: SafeArea(
        child: BottomNavigationBar(
          currentIndex: index,
          onTap: (i) {
            ref.read(bottomNavNotifierProvider.notifier).changeIndex(i);
            context.go(_routes[i]);   // go_router drives actual navigation
          },
          items: const [
            BottomNavigationBarItem(icon: Icon(Icons.home), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.search), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.person), label: ''),
          ],
        ),
      ),
    );
  }
}
```

Key: `SafeArea` on `bottomNavigationBar` prevents the bar from being hidden behind the system nav bar.

---

## AsyncValue Pattern

For data fetching with no custom state class — use `AsyncValue` directly:

```dart
@riverpod
class ItemsNotifier extends _$ItemsNotifier {
  @override
  Future<List<Item>> build() async {
    return getIt<ItemRepository>().getAll();
  }

  Future<void> refresh() async {
    ref.invalidateSelf();
    await future;
  }
}

// In widget
final asyncItems = ref.watch(itemsNotifierProvider);
asyncItems.when(
  loading: () => const AppProgressIndicator(),
  error: (e, st) => ErrorView(message: e.toString()),
  data: (items) => ItemList(items: items),
)
```

Use `AsyncValue` for simple fetch-and-display. Use a custom `freezed` state union when you need multiple named states or local UI logic beyond loading/error/data.

---

## Listening for Side Effects

```dart
// Show snackbar / navigate after state change — use ref.listen in build()
@override
Widget build(BuildContext context, WidgetRef ref) {
  ref.listen(authNotifierProvider, (prev, next) {
    next.whenOrNull(
      authenticated: (_) => context.go(AppRoutes.home),
      error: (msg) {
        Loading.dismiss();
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(msg)));
      },
    );
  });

  final state = ref.watch(authNotifierProvider);
  return AppScaffold(...);
}
```

Never navigate or show dialogs from inside a notifier — do it in `ref.listen`.

---

## Extensions Reference

```dart
// BuildContext
context.T                      // AppLocalizations (strings)
context.colors                 // AppColors.of(context)
context.styles                 // AppStyles.of(context)
context.screenWidth
context.screenHeight
context.isKeyboardOpen         // viewInsets.bottom > 0
context.colorScheme

// Color
color.changeOpacity(0.5)       // always use instead of withOpacity

// String
string.isValidEmail
string.isValidPhone
string.toCapitalized
string.convertMd5

// DateTime
dateTime.toFormattedString('dd MMM yyyy')
dateTime.isToday
dateTime.isYesterday

// num
value.clamp01
value.toDouble
value.isNegative

// List
list.firstOrNull
list.lastOrNull

// Logging (debug — returns self for chaining)
someValue.log
someValue.logWithName('tag')
(error, stackTrace).log        // logs with stack trace
```

---

## Theme Extension Pattern

Adding a new color:
```dart
// 1. Add constant to KColors
static const myColor = Color(0xFF...);

// 2. Add semantic token to AppColors
final Color cardBg;
const AppColors({..., this.cardBg = KColors.myColor});

// 3. Use it
context.colors.cardBg

// 4. For dark mode — override in darkTheme AppColors instance
```

Adding a new text style:
```dart
// 1. Add to AppStyles
this.s18w500Primary = const Poppins(fontSize: 18, fontWeight: FontWeight.w500, color: KColors.primary),
final TextStyle s18w500Primary;

// 2. Use it
context.styles.s18w500Primary
```

---

## Loading

```dart
Loading.show()
Loading.dismiss()
```

Never use `showDialog` for loading states.
