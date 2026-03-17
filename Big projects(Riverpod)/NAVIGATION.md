# NAVIGATION.md — Riverpod Boilerplate
> go_router + ShellRoute setup. AI gets this wrong without explicit guidance.

---

## Core Concept

go_router owns navigation entirely. `BottomNavNotifier` only tracks which tab index is visually selected — it does not drive navigation. Navigation drives the index.

```
context.go(route) → go_router → ShellRoute → RootPage(child: ActiveTabPage)
                                                      ↓
                                            BottomNavNotifier syncs index
```

---

## Router Setup

```dart
// routes/router.dart
final appRouter = GoRouter(
  initialLocation: AppRoutes.splash,
  debugLogDiagnostics: kDebugMode,
  redirect: _redirect,
  routes: [
    // Pages outside the bottom nav shell
    GoRoute(
      path: AppRoutes.splash,
      builder: (context, state) => const SplashPage(),
    ),
    GoRoute(
      path: AppRoutes.login,
      builder: (context, state) => const LoginPage(),
    ),

    // Shell — wraps all bottom nav tabs
    ShellRoute(
      builder: (context, state, child) => RootPage(child: child),
      routes: [
        GoRoute(
          path: AppRoutes.home,
          builder: (context, state) => const HomePage(),
          routes: [
            // Nested routes stay inside shell (bottom nav stays visible)
            GoRoute(
              path: 'detail/:id',
              builder: (context, state) {
                final id = state.pathParameters['id']!;
                return DetailPage(id: id);
              },
            ),
          ],
        ),
        GoRoute(
          path: AppRoutes.search,
          builder: (context, state) => const SearchPage(),
        ),
        GoRoute(
          path: AppRoutes.profile,
          builder: (context, state) => const ProfilePage(),
        ),
      ],
    ),
  ],
);
```

---

## AppRoutes

```dart
class AppRoutes {
  static const splash   = '/splash';
  static const login    = '/login';
  static const home     = '/home';
  static const search   = '/search';
  static const profile  = '/profile';

  // Nested — full path used in context.go(), relative path in GoRoute definition
  static const homeDetail = '/home/detail';  // use as '/home/detail/$id'
}
```

---

## Redirect / Auth Guard

```dart
String? _redirect(BuildContext context, GoRouterState state) {
  final isLoggedIn = getIt<Box>().get('token') != null;
  final isOnAuth = state.matchedLocation == AppRoutes.login
      || state.matchedLocation == AppRoutes.splash;

  if (!isLoggedIn && !isOnAuth) return AppRoutes.login;
  if (isLoggedIn && isOnAuth) return AppRoutes.home;
  return null;
}
```

For reactive redirects (reacts to auth state changes):
```dart
final appRouter = GoRouter(
  refreshListenable: RouterNotifier(ref),
  redirect: _redirect,
  routes: [...],
);

class RouterNotifier extends ChangeNotifier {
  RouterNotifier(this._ref) {
    _ref.listen(authNotifierProvider, (_, __) => notifyListeners());
  }
  final Ref _ref;
}
```

---

## RootPage (Shell)

```dart
class RootPage extends ConsumerWidget {
  const RootPage({super.key, required this.child});
  final Widget child;

  // Order must match ShellRoute's routes list exactly
  static const _tabRoutes = [
    AppRoutes.home,
    AppRoutes.search,
    AppRoutes.profile,
  ];

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final index = ref.watch(bottomNavNotifierProvider);

    // Sync index from actual route (handles deep links + back button)
    final location = GoRouterState.of(context).matchedLocation;
    final syncedIndex = _tabRoutes.indexWhere((r) => location.startsWith(r));
    if (syncedIndex != -1 && syncedIndex != index) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        ref.read(bottomNavNotifierProvider.notifier).changeIndex(syncedIndex);
      });
    }

    return AppScaffold(
      showAppBar: false,
      body: (context) => child,
      bottomNavigationBar: SafeArea(
        child: BottomNavigationBar(
          currentIndex: index,
          onTap: (i) {
            ref.read(bottomNavNotifierProvider.notifier).changeIndex(i);
            context.go(_tabRoutes[i]);
          },
          items: const [
            BottomNavigationBarItem(icon: Icon(Icons.home_outlined), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.search), label: ''),
            BottomNavigationBarItem(icon: Icon(Icons.person_outline), label: ''),
          ],
        ),
      ),
    );
  }
}
```

---

## BottomNavNotifier

```dart
// providers/bottom_nav_provider.dart
part 'bottom_nav_provider.g.dart';

@Riverpod(keepAlive: true)
class BottomNavNotifier extends _$BottomNavNotifier {
  @override
  int build() => 0;

  void changeIndex(int index) => state = index;
}
```

---

## Navigation Reference

| Action | Code |
|---|---|
| Go to tab | `context.go(AppRoutes.home)` |
| Push detail inside shell | `context.push('/home/detail/$id')` |
| Push full screen outside shell | `context.push(AppRoutes.settings)` — add as top-level GoRoute |
| Replace current | `context.replace(AppRoutes.login)` |
| Pop | `context.pop()` |
| Pop with result | `context.pop(myResult)` |
| Check can pop | `context.canPop()` |

---

## Common Mistakes — AI Must Avoid

- **Using `Navigator.push` alongside go_router** — causes route stack conflicts
- **Putting shell tab pages as top-level GoRoute** — bottom nav disappears on those pages
- **Driving navigation from `BottomNavNotifier`** — notifier tracks index only, `context.go` drives navigation
- **Using `extra` for data that must survive deep link / hot restart** — use path or query params instead
- **Not wrapping `bottomNavigationBar` in `SafeArea`** — bar hidden behind system nav on Android
- **`systemNavigationBarColor` not matching bottom nav bar color** — visual mismatch on Android
