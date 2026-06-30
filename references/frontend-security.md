# Frontend Security Reference

## React + TypeScript

### XSS Prevention
```tsx
// ✅ React escapes by default — never use dangerouslySetInnerHTML with user content
<p>{userContent}</p>  // safe — escaped

// ❌ CRITICAL
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// If rich text is needed: use DOMPurify
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

### Content Security Policy (CSP)
```html
<!-- index.html or HTTP header -->
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self';
           script-src 'self';
           style-src 'self' 'unsafe-inline';
           img-src 'self' data: https:;
           connect-src 'self' https://api.example.com;
           frame-ancestors 'none';" />
```

### Secure Token Storage
```ts
// ✅ Store JWT in memory (React state/context) — not localStorage
// Refresh token in httpOnly cookie only

// ❌ CRITICAL — localStorage is accessible to XSS
localStorage.setItem('token', jwt)

// ✅ Use httpOnly cookie for refresh token (set by server)
// Keep access token in memory only
const [accessToken, setAccessToken] = useState<string | null>(null)
```

### CSRF Protection
- SPA + JWT: CSRF is not needed for Bearer auth (no cookies carry auth)
- If using session cookies: require `SameSite=Strict` + CSRF token header

### Dependency Security
```bash
# Run before every release
npm audit --audit-level=high
npx better-npm-audit audit
```

---

## Blazor (.NET)

### Anti-Forgery (Blazor Server)
```csharp
// Program.cs — enable antiforgery
builder.Services.AddAntiforgery();
app.UseAntiforgery();

// In Razor component forms
<EditForm Model="model" OnValidSubmit="HandleSubmit">
    <AntiforgeryToken />
    ...
</EditForm>
```

### Blazor WASM — Secure API Calls
```csharp
// Never store sensitive tokens in localStorage in WASM
// Use in-memory token storage via DI-registered AuthenticationStateProvider

// ✅ HttpClient with Bearer — token from memory service
http.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", await tokenService.GetTokenAsync());
```

### Output Encoding
```razor
@* Blazor auto-encodes by default *@
<p>@userContent</p>  @* safe *@

@* MarkupString bypasses encoding — only use with sanitised content *@
@((MarkupString)DomPurify.Sanitize(userContent))
```

---

## Flutter (Mobile + Desktop)

### Secure Storage (API Keys, Tokens)
```dart
// ✅ Use flutter_secure_storage — backed by Keychain (iOS) / Keystore (Android)
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

const storage = FlutterSecureStorage();
await storage.write(key: 'access_token', value: token);
final token = await storage.read(key: 'access_token');

// ❌ Never use SharedPreferences for tokens — plaintext on disk
```

### Certificate Pinning
```dart
// Prevent MITM attacks
import 'package:http/io_client.dart';
import 'dart:io';

HttpClient getSecureClient() {
  final client = HttpClient()
    ..badCertificateCallback = (cert, host, port) {
      // Validate against pinned certificate hash
      final expectedHash = 'sha256//YOUR_CERT_HASH_HERE=';
      return cert.sha256.toString() == expectedHash;
    };
  return client;
}
```

### Deep Link Security
```dart
// Validate deep link parameters before trusting them
// Never pass deep link data directly to API calls without validation
GoRouter(
  routes: [
    GoRoute(
      path: '/reset-password',
      builder: (context, state) {
        final token = state.uri.queryParameters['token'];
        if (token == null || token.length < 32) {
          return const ErrorPage(message: 'Invalid reset link');
        }
        return ResetPasswordPage(token: token);
      },
    ),
  ],
)
```

### Android Security Config
```xml
<!-- android/app/src/main/res/xml/network_security_config.xml -->
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
  <domain-config cleartextTrafficPermitted="false">
    <domain includeSubdomains="true">api.example.com</domain>
  </domain-config>
  <!-- No cleartext traffic allowed -->
  <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

### ProGuard (Android Release)
```
# Keep model classes from being obfuscated
-keep class com.example.app.models.** { *; }
# Remove logging in release
-assumenosideeffects class android.util.Log { *; }
```

---

## WinUI 3 (Windows App SDK)

### Credential Storage (Windows Credential Manager)
```csharp
// ✅ Use Windows.Security.Credentials.PasswordVault
using Windows.Security.Credentials;

void StoreToken(string token)
{
    var vault = new PasswordVault();
    vault.Add(new PasswordCredential("MyApp", "access_token", token));
}

string? GetToken()
{
    var vault = new PasswordVault();
    try
    {
        var cred = vault.Retrieve("MyApp", "access_token");
        cred.RetrievePassword();
        return cred.Password;
    }
    catch { return null; }
}
```

### Data Protection API (DPAPI)
```csharp
using System.Security.Cryptography;

// Encrypt sensitive data at rest
byte[] Protect(byte[] data) =>
    ProtectedData.Protect(data, null, DataProtectionScope.CurrentUser);

byte[] Unprotect(byte[] encrypted) =>
    ProtectedData.Unprotect(encrypted, null, DataProtectionScope.CurrentUser);
```

### WebView2 Security
```csharp
// Restrict navigation to your own domains
webView.CoreWebView2.NavigationStarting += (s, e) =>
{
    var uri = new Uri(e.Uri);
    if (uri.Host != "app.example.com")
    {
        e.Cancel = true;
        // Open in default browser instead
        Windows.System.Launcher.LaunchUriAsync(uri);
    }
};
```

---

## Universal Frontend Rules (All Stacks)

1. **Never log tokens or PII** to the console in production builds
2. **Validate on server** — client-side validation is UX only, not security
3. **Secure all redirects** — whitelist allowed redirect URLs (OAuth flows)
4. **Dependency audit** — run `npm audit` / `flutter pub audit` / `dotnet list package --vulnerable` in CI
5. **Clickjacking** — set `X-Frame-Options: DENY` or `frame-ancestors 'none'` in CSP
6. **Referrer** — set `Referrer-Policy: strict-origin-when-cross-origin`
