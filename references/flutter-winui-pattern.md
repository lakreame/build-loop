# Flutter & WinUI 3 Pattern Reference

---

## Flutter (Dart — Cross-Platform Mobile + Desktop)

### Project Structure

```
lib/
├── main.dart
├── app/
│   ├── app.dart            # MaterialApp / CupertinoApp root
│   └── router.dart         # GoRouter routes
├── features/
│   └── [feature]/
│       ├── data/           # Repositories + API clients
│       ├── domain/         # Entities + use cases
│       └── presentation/
│           ├── screens/    # Full page widgets
│           ├── widgets/    # Reusable components
│           └── bloc/       # BLoC / Cubit state
├── core/
│   ├── theme/              # ThemeData, tokens, typography
│   ├── network/            # Dio client + interceptors
│   └── storage/            # Hive / SharedPreferences
└── gen/                    # auto-generated (assets, localization)
```

### State Management — BLoC Pattern

```dart
// features/auth/presentation/bloc/auth_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';

part 'auth_event.dart';
part 'auth_state.dart';

class AuthBloc extends Bloc<AuthEvent, AuthState> {
  final AuthRepository _repo;

  AuthBloc(this._repo) : super(AuthInitial()) {
    on<LoginRequested>(_onLogin);
    on<LogoutRequested>(_onLogout);
  }

  Future<void> _onLogin(LoginRequested event, Emitter<AuthState> emit) async {
    emit(AuthLoading());
    try {
      final user = await _repo.login(event.email, event.password);
      emit(AuthAuthenticated(user));
    } catch (e) {
      emit(AuthError(e.toString()));
    }
  }

  Future<void> _onLogout(LogoutRequested event, Emitter<AuthState> emit) async {
    await _repo.logout();
    emit(AuthInitial());
  }
}
```

### API Client (Dio + Retrofit)

```dart
// core/network/api_client.dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';

part 'api_client.g.dart';

@RestApi(baseUrl: "https://api.example.com")
abstract class ApiClient {
  factory ApiClient(Dio dio, {String baseUrl}) = _ApiClient;

  @GET("/users/{id}")
  Future<UserDto> getUser(@Path("id") String id);

  @POST("/auth/login")
  Future<TokenDto> login(@Body() LoginRequest request);
}
```

### Theme / Design Tokens

```dart
// core/theme/app_theme.dart
import 'package:flutter/material.dart';

class AppTheme {
  static const _primary = Color(0xFF5B5BD6);  // set from frontend-design token
  static const _surface = Color(0xFF0F0F10);

  static ThemeData get dark => ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: _primary,
      brightness: Brightness.dark,
      surface: _surface,
    ),
    textTheme: _textTheme,
    useMaterial3: true,
  );

  static const _textTheme = TextTheme(
    displayLarge: TextStyle(fontFamily: 'InterDisplay', fontWeight: FontWeight.w700),
    bodyMedium:   TextStyle(fontFamily: 'Inter', fontWeight: FontWeight.w400),
  );
}
```

### Navigation (GoRouter)

```dart
// app/router.dart
import 'package:go_router/go_router.dart';

final router = GoRouter(
  initialLocation: '/home',
  routes: [
    GoRoute(path: '/home',    builder: (_, __) => const HomeScreen()),
    GoRoute(path: '/login',   builder: (_, __) => const LoginScreen()),
    GoRoute(
      path: '/dashboard',
      builder: (_, __) => const DashboardScreen(),
      redirect: (context, state) => _authGuard(context, state),
    ),
  ],
);

String? _authGuard(BuildContext context, GoRouterState state) {
  final auth = context.read<AuthBloc>().state;
  return auth is AuthAuthenticated ? null : '/login';
}
```

### pubspec.yaml (Core Dependencies)

```yaml
dependencies:
  flutter_bloc: ^8.1.6
  go_router: ^14.2.0
  dio: ^5.7.0
  retrofit: ^4.4.1
  hive_flutter: ^1.1.0
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0

dev_dependencies:
  build_runner: ^2.4.12
  freezed: ^2.5.7
  json_serializable: ^6.8.0
  retrofit_generator: ^9.1.4
  flutter_test:
    sdk: flutter
  bloc_test: ^9.1.7
```

### Stripe (Flutter)

```dart
import 'package:flutter_stripe/flutter_stripe.dart';

Future<void> initStripe() async {
  Stripe.publishableKey = const String.fromEnvironment('STRIPE_PK');
  await Stripe.instance.applySettings();
}

Future<void> presentPaymentSheet(String clientSecret) async {
  await Stripe.instance.initPaymentSheet(
    paymentSheetParameters: SetupPaymentSheetParameters(
      paymentIntentClientSecret: clientSecret,
      merchantDisplayName: 'YourApp',
    ),
  );
  await Stripe.instance.presentPaymentSheet();
}
```

### Playwright Testing (Flutter Web)

```ts
// e2e/flutter-login.spec.ts
import { test, expect } from '@playwright/test'

test('Flutter Web login flow', async ({ page }) => {
  await page.goto('http://localhost:8080')
  await page.getByLabel('Email').fill('user@example.com')
  await page.getByLabel('Password').fill('secret')
  await page.getByRole('button', { name: 'Login' }).click()
  await expect(page.getByText('Dashboard')).toBeVisible()
})
```

---

## WinUI 3 (C# — Windows App SDK)

### Project Structure

```
MyApp/
├── MyApp.csproj
├── App.xaml / App.xaml.cs
├── MainWindow.xaml / MainWindow.xaml.cs
├── Views/
│   ├── HomePage.xaml
│   └── SettingsPage.xaml
├── ViewModels/
│   ├── HomeViewModel.cs
│   └── SettingsViewModel.cs
├── Models/
│   └── UserModel.cs
├── Services/
│   ├── ApiService.cs
│   └── AuthService.cs
└── Assets/
```

### MVVM — ViewModel Pattern (CommunityToolkit.Mvvm)

```csharp
// ViewModels/HomeViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class HomeViewModel : ObservableObject
{
    private readonly IApiService _api;

    [ObservableProperty]
    private string _greeting = string.Empty;

    [ObservableProperty]
    private bool _isLoading;

    public HomeViewModel(IApiService api) => _api = api;

    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            var data = await _api.GetGreetingAsync();
            Greeting = data.Message;
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### XAML Page with WinUI Controls

```xml
<!-- Views/HomePage.xaml -->
<Page x:Class="MyApp.Views.HomePage"
      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">

  <Grid Padding="32">
    <StackPanel Spacing="16">
      <TextBlock Text="Dashboard"
                 Style="{StaticResource TitleLargeTextBlockStyle}" />

      <ProgressRing IsActive="{x:Bind ViewModel.IsLoading, Mode=OneWay}" />

      <TextBlock Text="{x:Bind ViewModel.Greeting, Mode=OneWay}"
                 Style="{StaticResource BodyTextBlockStyle}" />

      <Button Command="{x:Bind ViewModel.LoadCommand}"
              Content="Refresh"
              Style="{StaticResource AccentButtonStyle}" />
    </StackPanel>
  </Grid>
</Page>
```

### Design Tokens (WinUI ResourceDictionary)

```xml
<!-- App.xaml — map frontend-design tokens into WinUI brushes -->
<Application.Resources>
  <ResourceDictionary>
    <ResourceDictionary.MergedDictionaries>
      <XamlControlsResources xmlns="using:Microsoft.UI.Xaml.Controls" />
    </ResourceDictionary.MergedDictionaries>

    <!-- Primary palette from frontend-design token system -->
    <Color x:Key="AppPrimaryColor">#5B5BD6</Color>
    <SolidColorBrush x:Key="AppPrimaryBrush" Color="{StaticResource AppPrimaryColor}" />

    <!-- Typography -->
    <FontFamily x:Key="DisplayFont">InterDisplay</FontFamily>
    <FontFamily x:Key="BodyFont">Inter</FontFamily>
  </ResourceDictionary>
</Application.Resources>
```

### DI Setup (Microsoft.Extensions.DependencyInjection)

```csharp
// App.xaml.cs
public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = null!;

    public App()
    {
        var services = new ServiceCollection();
        services.AddHttpClient<IApiService, ApiService>(c =>
            c.BaseAddress = new Uri("https://api.example.com"));
        services.AddTransient<HomeViewModel>();
        services.AddTransient<HomePage>();
        Services = services.BuildServiceProvider();
        InitializeComponent();
    }
}
```

### .csproj (Core Packages)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
    <RuntimeIdentifiers>win-x86;win-x64;win-arm64</RuntimeIdentifiers>
    <UseWinUI>true</UseWinUI>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.WindowsAppSDK" Version="1.6.*" />
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.3.*" />
    <PackageReference Include="CommunityToolkit.WinUI.Controls.Segmented" Version="8.1.*" />
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.*" />
  </ItemGroup>
</Project>
```

### GitHub Actions CI (WinUI)

```yaml
name: WinUI CI
on: [push, pull_request]
jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.x' }
      - run: dotnet restore
      - run: dotnet build --configuration Release
      - run: dotnet test --no-build
```

---

## Frontend-Design Integration

When generating UI for Flutter or WinUI, always apply the `frontend-design` skill:

1. Name the subject, audience, page job
2. Derive a token system (palette, typefaces, layout, signature element)
3. Map tokens → Flutter `ThemeData` / WinUI `ResourceDictionary`
4. Validate: does the result look templated? If yes, revise before writing code
5. Then generate the component/screen following the confirmed design plan

Flutter design constraint: Material 3 is the baseline; override aggressively using `ThemeData`
and `ColorScheme.fromSeed` — do not accept M3 defaults as your design.

WinUI design constraint: Fluent Design is the baseline; override accent colors, typography,
and spacing through `ResourceDictionary` — do not ship the default WinUI blue.

---

## Flutter — Testing Tools

### Unit + BLoC Testing
```dart
// test/features/auth/bloc/auth_bloc_test.dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late MockAuthRepository mockRepo;

  setUp(() => mockRepo = MockAuthRepository());

  blocTest<AuthBloc, AuthState>(
    'emits [AuthLoading, AuthAuthenticated] on successful login',
    build: () {
      when(() => mockRepo.login(any(), any()))
          .thenAnswer((_) async => User(id: '1', email: 'test@test.com'));
      return AuthBloc(mockRepo);
    },
    act: (bloc) => bloc.add(LoginRequested('test@test.com', 'password')),
    expect: () => [AuthLoading(), isA<AuthAuthenticated>()],
  );
}
```

### Widget Testing
```dart
// test/features/auth/presentation/login_screen_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

void main() {
  testWidgets('Login screen shows email and password fields', (tester) async {
    await tester.pumpWidget(
      BlocProvider<AuthBloc>(
        create: (_) => AuthBloc(MockAuthRepository()),
        child: const MaterialApp(home: LoginScreen()),
      ),
    );
    expect(find.byKey(const Key('emailField')), findsOneWidget);
    expect(find.byKey(const Key('passwordField')), findsOneWidget);
    expect(find.byType(ElevatedButton), findsOneWidget);
  });
}
```

### Integration Testing (flutter_driver / integration_test)
```dart
// integration_test/app_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:myapp/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('Full login flow', (tester) async {
    app.main();
    await tester.pumpAndSettle();
    await tester.enterText(find.byKey(const Key('emailField')), 'user@example.com');
    await tester.enterText(find.byKey(const Key('passwordField')), 'password');
    await tester.tap(find.byType(ElevatedButton));
    await tester.pumpAndSettle();
    expect(find.text('Dashboard'), findsOneWidget);
  });
}
```

### dev_dependencies for Testing
```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  bloc_test: ^9.1.7
  mocktail: ^1.0.4
  integration_test:
    sdk: flutter
```

---

## Flutter — GitHub Actions CI

```yaml
# .github/workflows/flutter-ci.yml
name: Flutter CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.x'
          channel: 'stable'
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v4

  build-android:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { flutter-version: '3.24.x' }
      - run: flutter pub get
      - run: flutter build apk --release
      - uses: actions/upload-artifact@v4
        with:
          name: android-release
          path: build/app/outputs/flutter-apk/app-release.apk

  build-web:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { flutter-version: '3.24.x' }
      - run: flutter pub get
      - run: flutter build web --release
      - uses: actions/upload-artifact@v4
        with:
          name: web-release
          path: build/web/
```

---

## Flutter — Build & Deploy

```bash
# Android
flutter build apk --release                    # APK
flutter build appbundle --release              # AAB (Play Store)

# iOS (macOS only)
flutter build ios --release --no-codesign

# Web
flutter build web --release --base-href /app/

# Desktop
flutter build linux --release
flutter build macos --release
flutter build windows --release
```

---

## WinUI 3 — Testing Tools

### Unit Testing (xUnit + Moq)
```csharp
// tests/ViewModels/HomeViewModelTests.cs
using Moq;
using Xunit;

public class HomeViewModelTests
{
    [Fact]
    public async Task LoadAsync_SetsGreeting_WhenApiSucceeds()
    {
        var mockApi = new Mock<IApiService>();
        mockApi.Setup(x => x.GetGreetingAsync())
               .ReturnsAsync(new GreetingDto { Message = "Hello" });

        var vm = new HomeViewModel(mockApi.Object);
        await vm.LoadCommand.ExecuteAsync(null);

        Assert.Equal("Hello", vm.Greeting);
        Assert.False(vm.IsLoading);
    }

    [Fact]
    public async Task LoadAsync_SetsIsLoading_DuringExecution()
    {
        var mockApi = new Mock<IApiService>();
        mockApi.Setup(x => x.GetGreetingAsync())
               .ReturnsAsync(new GreetingDto { Message = "Hi" });

        var vm = new HomeViewModel(mockApi.Object);
        Assert.False(vm.IsLoading);
    }
}
```

### UI Automation (WinAppDriver / Playwright)
```csharp
// For Playwright-based WinUI web testing (WebView2 surface)
// Use standard Playwright setup; target WebView2 embedded content

// For native WinUI UI automation:
// Microsoft.Windows.Apps.Test (WinAppDriver successor)
// https://github.com/microsoft/WinAppDriver
```

### Test project .csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.*" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.*" />
    <PackageReference Include="Moq" Version="4.20.*" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.11.*" />
    <ProjectReference Include="..\MyApp\MyApp.csproj" />
  </ItemGroup>
</Project>
```

---

## WinUI 3 — MSIX Packaging & Deploy

### Package.appxmanifest (key fields)
```xml
<Identity Name="com.example.MyApp"
          Publisher="CN=YourName"
          Version="1.0.0.0" />
<Properties>
  <DisplayName>MyApp</DisplayName>
  <PublisherDisplayName>Your Company</PublisherDisplayName>
  <Logo>Assets\StoreLogo.png</Logo>
</Properties>
<Capabilities>
  <rescap:Capability Name="runFullTrust" />
</Capabilities>
```

### Build MSIX
```bash
dotnet publish -c Release -f net8.0-windows10.0.19041.0 \
  -p:RuntimeIdentifierOverride=win10-x64 \
  -p:WindowsPackageType=MSIX
```

### GitHub Actions CI (WinUI + MSIX)
```yaml
name: WinUI CI
on: [push, pull_request]

jobs:
  build-and-test:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.x' }
      - run: dotnet restore
      - run: dotnet build --configuration Release --no-restore
      - run: dotnet test --no-build --verbosity normal
      - name: Publish MSIX
        run: |
          dotnet publish src/MyApp/MyApp.csproj `
            -c Release `
            -f net8.0-windows10.0.19041.0 `
            -p:RuntimeIdentifierOverride=win10-x64 `
            -p:WindowsPackageType=MSIX
      - uses: actions/upload-artifact@v4
        with:
          name: msix-package
          path: '**/*.msix'
```
