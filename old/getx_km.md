# GetX KM - Complete Project Architecture

> **State Management**: GetX (Controllers + Navigation)
> **Dependency Injection**: GetIt + Injectable
> **Architecture**: Clean Architecture with Retrofit & feature-rich Widgets

---

## 1. Project Structure

```
lib/
├── main.dart                    # App entry point
├── firebase_options.dart        # Firebase configuration
├── gen/                         # Generated assets (flutter_gen)
├── l10n/                        # Localization files
└── app/
    ├── controllers/             # Controllers with @lazySingleton
    ├── customs/                 # Custom packages
    ├── data/
    │   ├── models/              # Data models + JSON serialization
    │   └── services/            # Retrofit API services
    ├── global/                  # Global Config (AppConfig, EndPoints)
    ├── routes/                  # GetX routing (AppRoutes, AppPages)
    ├── ui/
    │   ├── pages/               # Screen pages
    │   └── widgets/             # Reusable widgets (CustomImageView, etc.)
    └── utils/
        ├── constants/           # Constants (AppEdgeInsets, AppStrings)
        ├── helpers/             # Exporter, Injectable, Extensions
        └── themes/              # Theme configuration
```

---

## 2. Main Entry Point

```dart
import 'package:app/app/utils/helpers/exporter.dart';

void main() => configuration(myApp: const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return GetMaterialApp(
      title: 'AppName',
      getPages: AppPages.routes,
      initialRoute: AppRoutes.splash,
      localizationsDelegates: AppLocalizations.localizationsDelegates,
      supportedLocales: AppLocalizations.supportedLocales,
      locale: Locale(getIt<GetStorage>().getAppLocal ?? 'en'),
      themeMode: ThemeMode.light,
      theme: AppTheme.lightTheme,
      builder: EasyLoading.init(
        builder: (context, child) {
          return TextFieldStyleProvider(
            key: TextFieldStyleProvider.styleKey,
            style: WidgetStateTextStyle.resolveWith((states) => AppStyles.of(context).s16w400Black),
            child: ButtonTheme(alignedDropdown: true, child: child!),
          );
        },
      ),
    );
  }
}
```

---

## 3. Dependency Injection (GetIt + Injectable)

### `injectable_properties.dart` (RegisterModule)
```dart
@module
abstract class RegisterModule {
  @singleton
  Dio dio({@Named('versioncode') required String versionCode}) => Dio(...)
      ..interceptors.addAll([AppDioInterceptor(), if (kDebugMode) PrettyDioLogger()]);

  @preResolve
  Future<GetStorage> storage() async {
    await GetStorage.init();
    return GetStorage(); 
  }
}
```

---

## 4. Centralized Exporter

### `utils/helpers/exporter.dart`
```dart
export 'dart:async';
export 'package:dio/dio.dart';
export 'package:flutter/material.dart';
export 'package:flutter/services.dart';
export 'package:flutter_easyloading/flutter_easyloading.dart';
export 'package:flutter_svg/flutter_svg.dart';
export 'package:gap/gap.dart';
export 'package:get/get.dart';
export 'package:get_storage/get_storage.dart';
export 'package:injectable/injectable.dart';

// App modules
export '../../global/app_config.dart';
export '../../routes/app_pages.dart';
export '../../routes/app_routes.dart';

// Widgets
export '../../ui/widgets/custom_buttons.dart';
export '../../ui/widgets/custom_dropdown.dart';
export '../../ui/widgets/custom_date_picker.dart';
export '../../ui/widgets/custom_input_field.dart';
export '../../ui/widgets/custom_image_view.dart';
// ...more widgets

// Helpers
export '../../utils/helpers/extensions/extensions.dart';
export '../../utils/helpers/getItHook/getit_hook.dart';
export '../../utils/helpers/injectable/injectable.dart';
export '../../utils/helpers/loading.dart';
export '../../utils/themes/app_theme.dart';
```

---

## 5. Custom Widgets (Best Features)

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

### CustomDropdown
```dart
class CustomDropdown extends StatefulWidget {
  const CustomDropdown({
    super.key,
    required this.list,
    required this.onSelect,
    this.initialValue,
    this.title,
    this.isOneSelection = true,
  });

  final List<String> list;
  final Function(List<String> value) onSelect;
  final List<String>? initialValue;
  final String? title;
  final bool isOneSelection;

  @override
  State<CustomDropdown> createState() => _CustomDropdownState();
}

class _CustomDropdownState extends State<CustomDropdown> {
  List<String> selectedValues = [];

  @override
  void initState() {
    super.initState();
    if (widget.initialValue != null) selectedValues = List.from(widget.initialValue!);
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _showSelectionSheet,
      child: Container(
        height: 56,
        padding: const EdgeInsets.symmetric(horizontal: 14),
        decoration: BoxDecoration(
          border: Border.all(color: Colors.grey),
          borderRadius: BorderRadius.circular(12),
        ),
        child: Row(
          children: [
            Expanded(
              child: Text(
                selectedValues.isEmpty ? (widget.title ?? 'Select') : selectedValues.join(', '),
                maxLines: 1,
                overflow: TextOverflow.ellipsis,
              ),
            ),
            const Icon(Icons.keyboard_arrow_down),
          ],
        ),
      ),
    );
  }

  void _showSelectionSheet() {
    showModalBottomSheet(
      context: context,
      builder: (ctx) => ListView.builder(
        itemCount: widget.list.length,
        itemBuilder: (ctx, index) {
          final item = widget.list[index];
          final isSelected = selectedValues.contains(item);
          return ListTile(
            title: Text(item),
            trailing: isSelected ? const Icon(Icons.check, color: KAppColors.primary) : null,
            onTap: () {
              setState(() {
                if (widget.isOneSelection) {
                  selectedValues = [item];
                  Navigator.pop(ctx);
                } else {
                  if (isSelected) selectedValues.remove(item);
                  else selectedValues.add(item);
                }
                widget.onSelect(selectedValues);
              });
            },
          );
        },
      ),
    );
  }
}
```

---

## 6. Routing (GetX)

```dart
class AppRoutes {
  static const String splash = '/splash';
  static const String home = '/home';
  // Multi-Flow Naming
  static const String cHome = '/cHome';
}

class AppPages {
  static final routes = [
    GetPage(name: AppRoutes.splash, page: () => const SplashPage()),
    GetPage(name: AppRoutes.home, page: () => const HomePage()),
  ];
}
```

---

## 7. GetItHook (Controller Lifecycle)

```dart
abstract class GetItHook<T extends GetxController> extends StatefulWidget {
  const GetItHook({super.key, T? controller}) : _controller = controller;
  final T? _controller;
  bool get autoDispose;
  Widget build(BuildContext context);
  T get controller => _controller ?? getIt<T>();
  
  @override
  State<GetItHook> createState() => _GetItHookState<T>();
}

class _GetItHookState<T extends GetxController> extends State<GetItHook> {
  @override
  Widget build(BuildContext context) => widget.build(context);
  @override
  void dispose() {
    super.dispose();
    if (widget.autoDispose && getIt.isRegistered<T>()) {
      getIt.resetLazySingleton<T>();
    }
  }
}
```

---

## 8. API Layer

```dart
@lazySingleton
@RestApi(baseUrl: AppConfig.baseUrl)
abstract class AuthService {
  @factoryMethod
  factory AuthService(Dio dio) = _AuthService;

  @POST(EndPoints.userLogin)
  Future<LoginResponse> login(@Body() LoginRequest request);
}
```

---

## 9. Guidance & Requirements

> **🤖 AI Agent Instructions**:
>
> 1.  **Reference**: Use THIS file as the single source for functionality and architecture.
> 2.  **Widgets**: YOU MUST implement `CustomImageView`, `TextInputField`, `AppButton`, `CustomDropdown` EXACTLY as shown in `lib/app/ui/widgets/`. Do NOT use raw Flutter widgets in pages.
> 3.  **Navigation**: Use `Get.toNamed` and `AppRoutes`.
> 4.  **State**: Use `GetItHook` for page controllers to manage lifecycle.
> 5.  **Unclear Requirements**: Ask the user before hallucinating business logic.

