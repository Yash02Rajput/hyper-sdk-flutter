# Visa Infinite Access SDK for Flutter

## Requirements

- Android minimum SDK 21.
- iOS 13 or later.
- A VIA client ID with its Airborne configuration provisioned by the VIA team.

## Install

```sh
flutter pub add visa_infinite_access_sdk_flutter
flutter pub get
```

## Android setup

The application owns the VIA Assets Plugin setup. The Flutter SDK does not apply this plugin; it supplies only the HyperSDK dependencies.

Add the plugin repository and classpath, plus the VIA Airborne asset repository, to the root `android/build.gradle` or `android/build.gradle.kts`.

Groovy:

```groovy
buildscript {
    repositories {
        google()
        mavenCentral()
        maven { url "https://maven.juspay.in/jp-build-packages/hyper-sdk/" }
    }
    dependencies {
        classpath "com.visa:visainfiniteaccess.plugin:0.0.1"
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url "https://visabenefits.me1.juspay.io/hyper-sdk/" }
    }
}
```

Kotlin DSL:

```kotlin
buildscript {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://maven.juspay.in/jp-build-packages/hyper-sdk/") }
    }
    dependencies {
        classpath("com.visa:visainfiniteaccess.plugin:0.0.1")
    }
}

allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://visabenefits.me1.juspay.io/hyper-sdk/") }
    }
}
```

Apply and configure the plugin in the application's `android/app/build.gradle` or `android/app/build.gradle.kts`.

Groovy:

```groovy
apply plugin: "visainfiniteaccess.plugin"

visaInfiniteAccessPlugin {
    clientId = "viaglobal"
}
```

Kotlin DSL:

```kotlin
apply(plugin = "visainfiniteaccess.plugin")

configure<com.visa.visainfiniteaccess.VisaInfiniteAccessPluginExtension> {
    clientId.set("viaglobal")
}
```

The app-level plugin downloads only the VIA Airborne asset bundle. Do not add the VIA plugin to the Flutter SDK or add consumer HyperSDK version overrides.

The application's `MainActivity` must extend `FlutterFragmentActivity`:

```kotlin
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity : FlutterFragmentActivity()
```

## iOS setup

Create `ios/VisaInfiniteAccessConfig.json`. This file does not need to be added to the Xcode target:

```json
{
  "clientConfigs": {
    "viaglobal": {}
  }
}
```

The SDK-owned `Fuse.rb` reads this file to download the VIA assets for the configured clients.

### CocoaPods

To use CocoaPods when Flutter's Swift Package Manager support is enabled, disable SPM in the application's `pubspec.yaml`:

```yaml
flutter:
  config:
    enable-swift-package-manager: false
```

In `ios/Podfile`, retain Flutter's normal `post_install` build settings and add the SDK-owned Fuse invocation:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
  end

  fuse_path = "./.symlinks/plugins/visa_infinite_access_sdk_flutter/ios/Fuse.rb"
  abort("Visa Infinite Access Fuse.rb not found at #{fuse_path}") unless File.exist?(fuse_path)
  abort("Visa Infinite Access asset generation failed") unless system("ruby", fuse_path)
end
```

Do not invoke `./Pods/HyperSDK/Fuse.rb` directly: the Flutter wrapper supplies the VIA tenant parameters and configuration filename.

Run from the Flutter application's root directory:

```sh
flutter pub get
cd ios
pod install
```

### Swift Package Manager

Keep Flutter's Swift Package Manager support enabled. In Xcode, open `ios/Runner.xcworkspace` and select **Product → Scheme → Edit Scheme**. Under **Build → Pre-actions**, add a **New Run Script Action** after Flutter's **Run Prepare Flutter Framework Script**. Select the application target for **Provide build settings from**, then add:

```sh
HYPER_SDK_DIR="${BUILD_DIR%Build/*}SourcePackages/artifacts/hypersdk-ios/HyperSDK"
[ -d "$HYPER_SDK_DIR" ] || HYPER_SDK_DIR="${BUILD_DIR}/SourcePackages/artifacts/hypersdk-ios/HyperSDK"
XCFRAMEWORK_DIR="${HYPER_SDK_DIR}/HyperSDK.xcframework"
FUSE_MARKER="${XCFRAMEWORK_DIR}/.fuse_completed"
VALIDATION_SCRIPT="${HYPER_SDK_DIR}/ValidateHyperSDK.rb"

# Resolve the Flutter package location for local, Git, and pub.dev dependencies.
FUSE_SCRIPT="$(ruby -rjson -ruri -e '
  config_path = File.expand_path(ARGV[0])
  config = JSON.parse(File.read(config_path))
  package = config["packages"].find { |item| item["name"] == "visa_infinite_access_sdk_flutter" }
  abort "Visa Infinite Access package not found in package_config.json" unless package
  root_uri = package["rootUri"]
  root_path = root_uri.start_with?("file:") ? URI.parse(root_uri).path : File.expand_path(root_uri, File.dirname(config_path))
  puts File.join(URI::DEFAULT_PARSER.unescape(root_path), "ios", "Fuse.rb")
' "${PROJECT_DIR}/../.dart_tool/package_config.json")" || exit 1

if [ ! -f "$FUSE_MARKER" ] || [ "${ACTION}" = "clean" ]; then
  cd "${PROJECT_DIR}" || exit 1
  echo "Running Visa Infinite Access Fuse.rb..."
  ruby "$FUSE_SCRIPT" || exit 1
fi

ruby "$VALIDATION_SCRIPT" && touch "$FUSE_MARKER" || exit 1
```

The script resolves the package's `ios/Fuse.rb`, downloads assets on the first build and after a clean, and validates the generated HyperSDK bundle.

## Invoke the SDK

The inner VIA payload fields match the native integration. Unlike the native wrapper APIs, the Flutter API takes the final HyperSDK envelope directly. Payloads carry identifiers only; do not pass cardholder PII or card data as SDK parameters.

Every `initiate` and `process` request has this structure:

| Field | Value |
| :---- | :---- |
| `requestId` | A unique UUID for the request. It must exactly match `payload.requestId`. |
| `service` | Always `com.visa.visainfiniteaccess`. |
| `payload` | The VIA request, including its `action`, `clientId`, matching `requestId`, and action-specific fields. |

Replace `viaglobal` in the examples with the client ID assigned by the VIA team.

### Import and create the SDK instance

```dart
import 'dart:convert';

import 'package:flutter/services.dart';
import 'package:visa_infinite_access_sdk_flutter/visa_infinite_access_sdk_flutter.dart';

final viaSdk = VisaInfiniteAccessSdk();
```

### Handle SDK events

Pass the callback to `initiate` and every `process` call. `hide_loader` indicates that the SDK has rendered its screen. `result` contains the final status; on failure, `error` is true and `errorMessage` explains the failure.

```dart
void viaCallbackHandler(MethodCall call) {
  switch (call.method) {
    case 'hide_loader':
      // Hide the host app's loader.
      break;
    case 'result':
      final response = jsonDecode(call.arguments as String)
          as Map<String, dynamic>;
      final error = response['error'] == true;
      final errorMessage = response['errorMessage']?.toString() ?? '';

      if (error) {
        // Show errorMessage or apply the host app's error handling.
      }
      break;
  }
}
```

### Initiate at app launch

Call `initiate` once when the VIA app starts. It downloads assets and partner configuration and does not require authentication.

```dart
const initiateRequestId = '0193f2a3-1d4b-7e21-a6c8-3b9e0f4d2c17';
final initiatePayload = <String, dynamic>{
  'requestId': initiateRequestId,
  'service': 'com.visa.visainfiniteaccess',
  'payload': <String, dynamic>{
    'requestId': initiateRequestId,
    'action': 'initiate',
    'clientId': 'viaglobal',
    'environment': 'sandbox',
    'language': 'en',
    'customerId': '<VIA customer id>',
    'deviceId': '<VIA device id>',
  },
};

await viaSdk.initiate(initiatePayload, viaCallbackHandler);
```

| Field | Description | Required |
| :---- | :---- | :---- |
| `requestId` | Unique UUID for each request | Yes |
| `environment` | `sandbox` or `production` | Yes |
| `language` | ISO 639-1 language code, such as `en`, `ar`, `fr`, `pt`, `ru`, or `uk` | Yes |
| `customerId` | VIA opaque cardholder ID; the SDK caches at this level | Yes |
| `deviceId` | VIA device ID for device and risk context | No |

### Log in after cardholder authentication

After the cardholder logs in to VIA, pass the auth code from Visa B2C IAM. Juspay's backend exchanges it for the SDK session.

```dart
const loginRequestId = '2941581f-b487-4ecf-8428-61d9696313a9';
final loginPayload = <String, dynamic>{
  'requestId': loginRequestId,
  'service': 'com.visa.visainfiniteaccess',
  'payload': <String, dynamic>{
    'requestId': loginRequestId,
    'action': 'login',
    'clientId': 'viaglobal',
    'authCode': '<B2C IAM auth code or Visa-sealed JWE>',
  },
};

await viaSdk.process(loginPayload, viaCallbackHandler);
```

### Open a benefit journey

Call `redeemBenefit` once for each benefit the cardholder opens.

```dart
const redeemRequestId = '0193f2a4-7c1e-7b52-9d3a-5e8f1c2b4a60';
final redeemBenefitPayload = <String, dynamic>{
  'requestId': redeemRequestId,
  'service': 'com.visa.visainfiniteaccess',
  'payload': <String, dynamic>{
    'requestId': redeemRequestId,
    'action': 'redeemBenefit',
    'clientId': 'viaglobal',
    'benefitId': '<VDBP benefit id>',
    'benefitPartnerId': '<VIA partner id>',
    'cardAlias': '<card alias>',
  },
};

await viaSdk.process(redeemBenefitPayload, viaCallbackHandler);
```

| Field | Description | Required |
| :---- | :---- | :---- |
| `requestId` | Unique UUID for each request | Yes |
| `action` | `redeemBenefit` | Yes |
| `benefitId` | VDBP benefit or offer ID being opened | Yes |
| `benefitPartnerId` | VIA partner ID used to select the Juspay partner adapter | No |
| `cardAlias` | Card on which the benefit is used | Required when the cardholder has more than one card |

### Open benefit details

When the cardholder opens a benefit's detail page, use the same fields with `viewBenefitDetail`.

```dart
const detailRequestId = '0193f2a5-2b6d-7c84-8e1f-4a7c9d3b5e21';
final viewBenefitDetailPayload = <String, dynamic>{
  'requestId': detailRequestId,
  'service': 'com.visa.visainfiniteaccess',
  'payload': <String, dynamic>{
    'requestId': detailRequestId,
    'action': 'viewBenefitDetail',
    'clientId': 'viaglobal',
    'benefitId': '<VDBP benefit id>',
    'benefitPartnerId': '<VIA partner id>',
    'cardAlias': '<card alias>',
  },
};

await viaSdk.process(viewBenefitDetailPayload, viaCallbackHandler);
```

### Log out of VIA

Call this action when the cardholder logs out of VIA. It revokes the Visa access and refresh tokens and invalidates the Juspay JWT. The next session requires a fresh `login` action.

```dart
const logoutRequestId = '0193f2a6-4e8a-7d13-b5c2-9f1e6a7c3d48';
final logoutPayload = <String, dynamic>{
  'requestId': logoutRequestId,
  'service': 'com.visa.visainfiniteaccess',
  'payload': <String, dynamic>{
    'requestId': logoutRequestId,
    'action': 'logout',
    'clientId': 'viaglobal',
  },
};

await viaSdk.process(logoutPayload, viaCallbackHandler);
```

Call `logout` before `terminate` when the cardholder is logging out.

### Release the SDK instance

Call `terminate` when the SDK screen closes. It releases the on-device SDK instance only; it makes no backend call and does not log the cardholder out.

```dart
await viaSdk.terminate();
```

### Android back handling

Forward Android hardware back presses while the SDK screen is active:

```dart
final backPressResult = await viaSdk.onBackPress();
final consumedBySdk = backPressResult.toLowerCase() == 'true';
```

When `consumedBySdk` is false, the host application handles the back navigation.

## License

`visa_infinite_access_sdk_flutter` is distributed under the [AGPL-3.0-only](https://pub.dev/packages/visa_infinite_access_sdk_flutter/license) license.

### Attribution

This project is based on the OSS project juspay_flutter by [Deep Rooted.co](https://deep-rooted.co), published under the MIT License. The original repository is available [here](https://github.com/deep-rooted-co/juspay_flutter).
