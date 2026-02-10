# Riverpod KM - Complete Project Architecture

> **State Management**: Riverpod (AsyncNotifier + ProviderScope)
> **Dependency Injection**: GetIt (Data Layer) + Riverpod (UI Layer)
> **Routing**: GoRouter
> **Architecture**: Clean Architecture with Retrofit & feature-rich Widgets

---

## 1. Project Structure

```
lib/
├── main.dart                    # App entry point (ProviderScope)
├── app/
    ├── data/                    # Models, Repositories, Services (GetIt)
    ├── routes/                  # GoRouter configuration
    ├── providers/               # Riverpod Providers (Logic)
    ├── ui/
        ├── widgets/             # Custom Images, Buttons, Inputs
        ├── pages/               # ConsumerWidgets
```

---

## 2. Main Entry Point

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import 'package:app/app/utils/helpers/exporter.dart';

void main() => configuration(myApp: const ProviderScope(child: MyApp()));

class MyApp extends ConsumerWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      title: 'AppName',
      routerConfig: router,
      builder: EasyLoading.init(
        builder: (context, child) => TextFieldStyleProvider(
            style: AppStyles.of(context).s16w400Black, 
            child: child!),
      ),
    );
  }
}
```

---

## 3. Custom Widgets (Best Features)

### CustomImageView
```dart
class ImageView extends StatelessWidget {
  const ImageView(this.imagePath, {super.key, this.fit = BoxFit.cover, this.radius, this.color});
  final String? imagePath;
  final double? radius;
  final BoxFit fit;
  final Color? color;
  
  @override
  Widget build(BuildContext context) {
    if (imagePath == null) return const SizedBox();
    Widget widget;
    if (imagePath!.endsWith('.svg')) {
        widget = SvgPicture.asset(imagePath!, fit: fit, color: color);
    } else if (imagePath!.startsWith('http')) {
        widget = CachedNetworkImage(imageUrl: imagePath!, fit: fit, errorWidget: (_, __, ___) => const Icon(Icons.error));
    } else if (imagePath!.startsWith('assets')) {
        widget = Image.asset(imagePath!, fit: fit, color: color);
    } else {
        widget = Image.file(File(imagePath!), fit: fit, color: color);
    }

    if (radius != null) return ClipRRect(borderRadius: BorderRadius.circular(radius!), child: widget);
    return widget;
  }
}
```

### TextInputField (Smart Input)
```dart
enum InputType { name, email, password, phoneNumber, digits, multiline }

class TextInputField extends TextFormField {
  TextInputField({
    super.key,
    required InputType type,
    required TextEditingController controller,
    String? hintLabel,
    Widget? prefixIcon,
    bool readOnly = false,
  }) : super(
    controller: controller,
    readOnly: readOnly,
    keyboardType: switch (type) {
      InputType.email => TextInputType.emailAddress,
      InputType.phoneNumber => TextInputType.phone,
      InputType.digits => TextInputType.number,
      InputType.multiline => TextInputType.multiline,
      _ => TextInputType.text,
    },
    inputFormatters: [
      if (type == InputType.digits) FilteringTextInputFormatter.digitsOnly,
      if (type == InputType.phoneNumber) LengthLimitingTextInputFormatter(15),
    ],
    decoration: InputDecoration(
      hintText: hintLabel,
      prefixIcon: prefixIcon,
      border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
    ),
  );
}
```

### AppButton
```dart
enum AppButtonType { primary, secondary, outline }

class AppButton extends StatelessWidget {
  const AppButton({super.key, required this.title, required this.onPressed, this.type = AppButtonType.primary});
  final String title;
  final VoidCallback? onPressed;
  final AppButtonType type;

  @override
  Widget build(BuildContext context) {
    final style = switch (type) {
      AppButtonType.primary => ElevatedButton.styleFrom(backgroundColor: KAppColors.primary, foregroundColor: KAppColors.white),
      AppButtonType.secondary => ElevatedButton.styleFrom(backgroundColor: KAppColors.secondary, foregroundColor: KAppColors.black),
      AppButtonType.outline => OutlinedButton.styleFrom(foregroundColor: KAppColors.primary),
    };
    return ElevatedButton(
      onPressed: onPressed,
      style: style,
      child: Text(title),
    );
  }
}
```

---

## 4. Navigation (GoRouter)

```dart
final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/splash',
    routes: [
      GoRoute(path: '/splash', builder: (_, __) => const SplashPage()),
      GoRoute(path: '/home', builder: (_, __) => const HomePage()),
    ],
  );
});
```

---

## 5. State Management (Riverpod)

### Service Provider Bridge
*Bridge GetIt repos to Riverpod.*
```dart
final authRepoProvider = Provider<AuthRepository>((ref) => getIt<AuthRepository>());
```

### Async Notifier (Recommended)
```dart
class HomeNotifier extends AutoDisposeAsyncNotifier<List<Data>> {
  @override
  FutureOr<List<Data>> build() {
    return ref.read(homeRepoProvider).getData();
  }
  
  Future<void> refresh() {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() => ref.read(homeRepoProvider).getData());
  }
}

final homeProvider = AsyncNotifierProvider.autoDispose<HomeNotifier, List<Data>>(HomeNotifier.new);
```

---

## 6. Guidance & Requirements

> **🤖 AI Agent Instructions**:
>
> 1.  **Reference**: Use THIS file as the single source for functionality and architecture.
> 2.  **Widgets**: YOU MUST implement `CustomImageView`, `TextInputField` (with `InputType`), `AppButton` in `lib/app/ui/widgets/`.
> 3.  **State**: Use `AsyncNotifier` (not `StateNotifier`) for all methods loading data.
> 4.  **Injection**: Inject Repos via `getIt` into Providers, then `ref.watch` in UI.


---

## 7. Localization (Context Extension) 🌍

### Extension Helper
```dart
// lib/app/utils/helpers/extensions/context_ext.dart
extension AppExt on BuildContext {
  AppLocalizations get t => AppLocalizations.of(this)!;
  
  // Bonus helpers
  ThemeData get theme => Theme.of(this);
  TextTheme get textTheme => Theme.of(this).textTheme;
}
```

### Locale Management (Riverpod StateProvider)
```dart
// lib/app/providers/locale_provider.dart
final localeProvider = StateProvider<Locale>((ref) {
  final savedLocale = getIt<SharedPreferences>().getString('locale') ?? 'en';
  return Locale(savedLocale);
});

// Helper to change locale
extension LocaleRef on WidgetRef {
  void changeLocale(String langCode) {
    read(localeProvider.notifier).state = Locale(langCode);
    getIt<SharedPreferences>().setString('locale', langCode);
  }
}
```

### Usage in Pages
```dart
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(title: Text(context.t.homeTitle)),  // 👈 Use context.t
      body: Column(
        children: [
          Text(context.t.welcomeMessage),
          TextInputField(
            type: InputType.email,
            hintLabel: context.t.emailHint,
            controller: emailCtrl,
          ),
          AppButton(
            title: context.t.loginButton,
            onPressed: () => ref.read(authProvider.notifier).login(),
          ),
        ],
      ),
    );
  }
}
```

### Change Language Anywhere
```dart
// In ConsumerWidget
ref.changeLocale('es');  // Spanish
ref.changeLocale('en');  // English
```

**✅ Update main.dart:**
```dart
class MyApp extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final locale = ref.watch(localeProvider);  // 👈 Watch locale changes
    
    return MaterialApp.router(
      locale: locale,
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      routerConfig: ref.watch(routerProvider),
    );
  }
}
```

---

**📌 Updated Guidance:**
> 5. **Localization**: Use `context.t` for all strings in UI. Use `localeProvider` for language switching.
