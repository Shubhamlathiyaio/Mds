# Bloc KM - Complete Project Architecture

> **State Management**: flutter_bloc (Bloc/Cubit)
> **Dependency Injection**: GetIt + Injectable
> **Routing**: GoRouter
> **Architecture**: Clean Architecture with Retrofit & feature-rich Widgets

---

## 1. Project Structure

```
lib/
├── main.dart                    # App entry point (MultiBlocProvider)
├── app/
    ├── data/                    # Models, Repositories, Services (GetIt)
    ├── routes/                  # GoRouter
    ├── logic/                   # Blocs / Cubits
    ├── ui/
        ├── widgets/             # Custom Images, Buttons, Inputs
        ├── pages/               # Pages (BlocBuilder/BlocListener)
```

---

## 2. Main Entry Point

```dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:app/app/utils/helpers/exporter.dart';

void main() => configuration(myApp: const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(create: (_) => getIt<AuthBloc>()),
      ],
      child: MaterialApp.router(
        title: 'AppName',
        routerConfig: router,
        builder: EasyLoading.init(
            builder: (context, child) => TextFieldStyleProvider(
            style: AppStyles.of(context).s16w400Black, 
            child: child!),
        ),
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
final GoRouter router = GoRouter(
  initialLocation: '/splash',
  routes: [
      GoRoute(path: '/splash', builder: (_, __) => const SplashPage()),
      GoRoute(path: '/home', builder: (_, __) => const HomePage()),
  ],
);
```

---

## 5. Bloc Implementation

### State Classes (Equatable)
```dart
sealed class HomeState extends Equatable { ... }
final class HomeInitial extends HomeState {}
final class HomeLoading extends HomeState {}
final class HomeLoaded extends HomeState { final List<Data> data; ... }
final class HomeError extends HomeState { final String message; ... }
```

### UI Consumption
```dart
BlocBuilder<HomeCubit, HomeState>(
  builder: (context, state) {
    if (state is HomeLoading) return const Center(child: CircularProgressIndicator());
    if (state is HomeLoaded) return ListView(children: state.data.map((e) => Text(e.title)).toList());
    return const SizedBox();
  },
)
```

---

## 6. Guidance & Requirements

> **🤖 AI Agent Instructions**:
>
> 1.  **Reference**: Use THIS file as the single source.
> 2.  **Widgets**: YOU MUST implement `CustomImageView`, `TextInputField`, `AppButton` in `lib/app/ui/widgets/`.
> 3.  **State**: Use `Cubit` for simple state, `Bloc` for complex event-driven logic.
> 4.  **Injection**: Blocs are retrieved via `getIt<MyBloc>()` inside `BlocProvider(create: ...)`.
> 5.  **Ambiguity**: Ask setup questions before assuming logic.
