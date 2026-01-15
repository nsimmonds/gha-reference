# Flutter CI/CD Reference

A reference implementation for Flutter CI/CD pipelines using GitHub Actions. This repo demonstrates a branch-based deployment strategy for multi-platform Flutter apps.

## Branch Strategy

| Branch | Trigger | Purpose |
|--------|---------|---------|
| `main` | Push, PR | Run tests and static analysis |
| `build` | Push | Build signed artifacts for all platforms |
| `deploy` | Push | Build and deploy to stores (Play Store, TestFlight) |

### Tags

| Tag | Purpose |
|-----|---------|
| `build-android` | Trigger Android build only (useful for quick APK) |
| `build-ios` | Trigger iOS build and TestFlight upload |

## Workflows

### `test.yml`
Runs on every push to `main` and on PRs:
- Flutter analyze (static analysis)
- Flutter test (unit tests)

### `build.yml`
Runs on push to `build` branch or `build-android` tag:
- **Android**: Builds signed AAB and APK, uploads as artifacts
- **iOS**: Builds signed IPA, uploads as artifact

### `deploy.yml`
Runs on push to `deploy` branch:
- **Android**: Builds AAB, uploads to Google Play internal testing track
- **iOS**: Builds IPA, uploads to TestFlight
- **macOS**: Builds signed app, creates DMG, uploads as artifact
- **Windows**: Builds release, creates ZIP, uploads as artifact

## Required Secrets

### Android

| Secret | Description |
|--------|-------------|
| `KEYSTORE_BASE64` | Base64-encoded upload keystore (.jks file) |
| `KEY_ALIAS` | Keystore key alias |
| `KEY_PASSWORD` | Key password |
| `STORE_PASSWORD` | Keystore password |
| `PLAY_STORE_SERVICE_ACCOUNT_JSON` | Google Play API service account JSON (for deploy) |

#### Creating Android Secrets

1. **Generate keystore** (if you don't have one):
   ```bash
   keytool -genkey -v -keystore upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
   ```

2. **Encode keystore to base64**:
   ```bash
   base64 -i upload-keystore.jks | pbcopy  # macOS
   base64 upload-keystore.jks | xclip      # Linux
   ```

3. **Create Play Store service account**:
   - Go to Google Play Console → Setup → API access
   - Create or link a Google Cloud project
   - Create a service account with "Release manager" role
   - Download the JSON key file
   - Copy the entire JSON content as the secret value

### iOS

| Secret | Description |
|--------|-------------|
| `IOS_P12_BASE64` | Base64-encoded distribution certificate (.p12) |
| `IOS_P12_PASSWORD` | Password for the .p12 file |
| `IOS_PROVISIONING_PROFILE_BASE64` | Base64-encoded provisioning profile |
| `IOS_KEYCHAIN_PASSWORD` | Any password (used for temp keychain) |
| `APP_STORE_CONNECT_API_KEY_BASE64` | Base64-encoded App Store Connect API key (.p8) |
| `APP_STORE_CONNECT_API_KEY_ID` | API key ID (e.g., `ABC123XYZ`) |
| `APP_STORE_CONNECT_ISSUER_ID` | Issuer ID from App Store Connect |

#### Creating iOS Secrets

1. **Export certificate from Keychain Access**:
   - Open Keychain Access
   - Find your "Apple Distribution" certificate
   - Right-click → Export → Save as .p12
   - Base64 encode: `base64 -i Certificates.p12 | pbcopy`

2. **Download provisioning profile**:
   - Go to Apple Developer → Certificates, Identifiers & Profiles
   - Download your App Store distribution profile
   - Base64 encode: `base64 -i profile.mobileprovision | pbcopy`

3. **Create App Store Connect API key**:
   - Go to App Store Connect → Users and Access → Keys
   - Generate a new key with "App Manager" access
   - Download the .p8 file (only available once!)
   - Base64 encode: `base64 -i AuthKey_XXXXX.p8 | pbcopy`
   - Note the Key ID and Issuer ID

### macOS

| Secret | Description |
|--------|-------------|
| `MACOS_CERTIFICATE_BASE64` | Base64-encoded Developer ID certificate (.p12) |
| `MACOS_CERTIFICATE_PASSWORD` | Password for the .p12 file |
| `MACOS_TEAM_ID` | Apple Developer Team ID |

### Required Files

#### `ios/ExportOptions.plist`

Create this file for iOS builds:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

## Customization

Before using these workflows, update the following:

### `deploy.yml`

1. **Android package name** (line 61):
   ```yaml
   packageName: com.example.your_app  # Change to your package name
   ```

2. **macOS developer name** (line 180):
   ```yaml
   DEVELOPER_NAME: Your Name  # Change to your developer name
   ```

3. **macOS/Windows app name** (lines 188, 228):
   ```yaml
   APP_NAME: YourApp  # Change to your app name
   ```

## Usage

### Running Tests
```bash
git push origin main
```

### Building Artifacts
```bash
git push origin main:build
# Or for Android only:
git tag build-android && git push origin build-android
```

### Deploying to Stores
```bash
git push origin main:deploy
# Or for iOS TestFlight only:
git tag build-ios && git push origin build-ios
```

## Build Numbers

Deploy builds automatically increment the build number using:
```
$(( github.run_number + 100 ))
```

This ensures build numbers always increase and start above any manual builds you may have done.

## License

MIT - Use freely for your own projects.
