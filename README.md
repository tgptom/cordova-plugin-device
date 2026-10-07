---
title: Device
description: Get device information.
---
<!--
# license: Licensed to the Apache Software Foundation (ASF) under one
#         or more contributor license agreements.  See the NOTICE file
#         distributed with this work for additional information
#         regarding copyright ownership.  The ASF licenses this file
#         to you under the Apache License, Version 2.0 (the
#         "License"); you may not use this file except in compliance
#         with the License.  You may obtain a copy of the License at
#
#           http://www.apache.org/licenses/LICENSE-2.0
#
#         Unless required by applicable law or agreed to in writing,
#         software distributed under the License is distributed on an
#         "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
#         KIND, either express or implied.  See the License for the
#         specific language governing permissions and limitations
#         under the License.
-->

# cordova-plugin-device

[![Android Testsuite](https://github.com/apache/cordova-plugin-device/actions/workflows/android.yml/badge.svg)](https://github.com/apache/cordova-plugin-device/actions/workflows/android.yml) [![Chrome Testsuite](https://github.com/apache/cordova-plugin-device/actions/workflows/chrome.yml/badge.svg)](https://github.com/apache/cordova-plugin-device/actions/workflows/chrome.yml) [![iOS Testsuite](https://github.com/apache/cordova-plugin-device/actions/workflows/ios.yml/badge.svg)](https://github.com/apache/cordova-plugin-device/actions/workflows/ios.yml) [![Lint Test](https://github.com/apache/cordova-plugin-device/actions/workflows/lint.yml/badge.svg)](https://github.com/apache/cordova-plugin-device/actions/workflows/lint.yml)

This plugin defines a global `device` object, which describes the device's hardware and software.
Although the object is in the global scope, it is not available until after the `deviceready` event.

```js
document.addEventListener("deviceready", onDeviceReady, false);
function onDeviceReady() {
    console.log(device.cordova);
}
```

## Installation

    cordova plugin add cordova-plugin-device

## Properties

- device.cordova
- device.model
- device.platform
- device.uuid
- device.version
- device.manufacturer
- device.isVirtual
- device.serial
- device.sdkVersion (Android only)

## Native compatibility validation

The fail-fast `compatibility` jobs in `android.yml` and `ios.yml` build a native
debug app, verify the installed platform package version, and run
`tests/tests.js` through Paramedic on an emulator/simulator. These are separate
from the existing OS-version jobs, which allow failures and use floating
platform selections. Platform pins live in `tests/compatibility/*.config.json`;
**Cordova package versions are not emulator/simulator OS versions**.

| Cordova platform package pin | CI build toolchain | Runtime OS |
| --- | --- | --- |
| `cordova-android@14.0.1` | Ubuntu 24.04, JDK 17, SDK 35 / Build Tools >=35.0.0, Gradle 8.13 / AGP 8.7.3 | Android API 35, Google APIs x86_64 |
| `cordova-android@15.1.0` | Ubuntu 24.04, JDK 17, SDK 36 / Build Tools >=36.0.0, Gradle 8.14.2 / AGP 8.10.1 | Android API 36, Google APIs x86_64 |
| `cordova-ios@7.1.1` | macOS 15, Xcode 16.4, CocoaPods >=1.16.0, ios-deploy 1.12.2 | iPhone 16 simulator, iOS 18.5 |
| `cordova-ios@8.1.1` | macOS 15, Xcode 16.4, CocoaPods >=1.16.0, ios-deploy 1.12.2 | iPhone 16 simulator, iOS 18.5 |

All four jobs use Node **22.23.3**, Cordova CLI **13.0.0**, and Paramedic commit
**`1c3a8075f31a20b77361dfcf0b5a10e9b00fcde1`**. The exact platform releases and
requirements were checked against their npm packages (Android
`framework/cdv-gradle-config-defaults.json`, iOS `lib/check_reqs.js`).
Android's Java source/target level 11 does **not** mean builds can use JDK 11:
keep `JAVA_HOME` (and any `CORDOVA_JAVA_HOME`) on JDK 17.
Both pinned iOS packages export `Cordova/Cordova.h`; UIKit is already explicitly
imported by `CDVDevice.h`. Android's `pluginInitialize()` is also available in
the existing minimum platform 7.0.0.

The selected [Paramedic revision](https://github.com/apache/cordova-paramedic/tree/1c3a8075f31a20b77361dfcf0b5a10e9b00fcde1)
preserves `platform@version` during installation. Later refactoring in Paramedic
strips the version before installation, so do not substitute its current HEAD
or the old npm `cordova-paramedic@0.5.0`. Absolute config paths resolve directly
to these repository-owned files, not Paramedic's external `pr/local` configs.
CLI, harness source and platforms are pinned; Paramedic still fetches
`cordova-plugin-test-framework` from upstream GitHub, and its transitive npm
dependencies are not fully locked.

### Local commands

Run from the plugin checkout with the matching toolchain above. Start an Android
emulator at API 35 or 36 before its runtime command. On macOS, select Xcode 16.4
and ensure `xcrun simctl list devices available` includes iPhone 16 / iOS 18.5;
the CI preflight fails rather than silently choosing a different runtime.

```sh
PLUGIN="$(pwd)"
npm ci
npm test # lint only, not native compatibility evidence
npm install -g cordova@13.0.0 \
  github:apache/cordova-paramedic#1c3a8075f31a20b77361dfcf0b5a10e9b00fcde1
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"
sdkmanager "platforms;android-35" "build-tools;35.0.0" \
  "platforms;android-36" "build-tools;36.0.0"
for PIN in 14.0.1 15.1.0; do
  CONFIG="$PLUGIN/tests/compatibility/cordova-android-$PIN.config.json"
  cordova-paramedic --config "$CONFIG" --plugin "$PLUGIN" --justbuild
  cordova-paramedic --config "$CONFIG" --plugin "$PLUGIN"
done
```

On macOS (instead of the Android SDK/emulator commands):

```sh
PLUGIN="$(pwd)"
sudo xcode-select --switch /Applications/Xcode_16.4.app/Contents/Developer
npm install -g cordova@13.0.0 ios-deploy@1.12.2 \
  github:apache/cordova-paramedic#1c3a8075f31a20b77361dfcf0b5a10e9b00fcde1
pod --version # >=1.16.0
xcodebuild -version
xcrun simctl list devices available
for PIN in 7.1.1 8.1.1; do
  CONFIG="$PLUGIN/tests/compatibility/cordova-ios-$PIN.config.json"
  cordova-paramedic --config "$CONFIG" --plugin "$PLUGIN" --justbuild
  cordova-paramedic --config "$CONFIG" --plugin "$PLUGIN"
done
```

Do not pass `--skipMainTests`. Paramedic's advertised `--timeout` is not honored
by this revision; CI has a 60-minute job timeout, and native test callbacks and
readiness checks have bounded Jasmine timeouts. It retains generated projects
and prints their paths for inspection.

### Evidence and remaining checks

At implementation time, `npm ci`, `npm test`, workflow/config parsing and actual
Paramedic config resolution passed. Local Android platform installation and
toolchain checks passed for both pins, but builds stopped fetching AGP from
`dl.google.com` in the sandbox, before plugin compilation. No native runtime
test or iOS build passed locally; **all four native combinations remain pending
CI**, not certified compatibility. Earlier green OS-named workflows are not
proof of these package pins.

The iOS jobs inspect built simulator apps for
`CDVDevice.bundle/PrivacyInfo.xcprivacy`, including the UserDefaults reason
`CA92.1`. This preserves and checks the plugin resource; it does **not** establish
whole-app App Store compliance. Signed release archives and real-device tests
(including UUID persistence across launches and iOS apps on Mac) remain required.

Android serial may legitimately be `"unknown"` on modern Android; tests do not
require a hardware serial or assert emulator-detection heuristics. Android UUID
uses `ANDROID_ID`, scoped by signing key, user and device, not a permanent global
hardware identifier. iOS retains its existing saved UUID/identifierForVendor
behavior. No engine minimum, plugin API, permission or SDK override is changed.

## device.cordova

Returns the Cordova platform's version that is bundled in the application.

The version information comes from the `cordova.js` file.

This property does not display other installed platforms' version information. Only the respective running platform's version is displayed.

Example:

If Cordova Android 10.1.1 is installed on the Cordova project, the `cordova.js` file, in the Android application, will contain `10.1.1`.

The `device.cordova` property will display `10.1.1`.

### Supported Platforms

- Android
- Browser
- iOS

## device.model

The `device.model` returns the name of the device's model or
product. The value is set by the device manufacturer and may be
different across versions of the same product.

### Supported Platforms

- Android
- Browser
- iOS

### Quick Example

```js
// Android: Pixel 4             returns "Pixel 4"
//          Motorola Moto G3    returns "MotoG3"
// Browser: Google Chrome       returns "Chrome"
//          Safari              returns "Safari"
// iOS:     iPad Mini           returns "iPad2,5"
//          iPhone 5            returns "iPhone5,1"
// See https://www.theiphonewiki.com/wiki/Models
// OS X:                        returns "x86_64"
//
var model = device.model;
```

### Android Quirks

- Gets the [model name](https://developer.android.com/reference/android/os/Build.html#MODEL).

### iOS Quirks

The model value is based on the identifier that Apple supplies.

If you need the exact device name, e.g. iPhone 13 Pro Max, a mapper needs to be created to convert the known identifiers to the associated device name.

Example: The identifier `iPhone14,3` is associated to the device `iPhone 13 Pro Max`.

For the full list of all identifiers to device names, see [here](https://www.theiphonewiki.com/wiki/Models)

## device.platform

Get the device's operating system name.

```js
var string = device.platform;
```
### Supported Platforms

- Android
- Browser
- iOS

### Quick Example

```js
// Depending on the device, a few examples are:
//   - "Android"
//   - "browser"
//   - "iOS"
//
var devicePlatform = device.platform;
```

## device.uuid

Get the device's Universally Unique Identifier ([UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier)).

```js
var string = device.uuid;
```

### Description

The details of how a UUID is generated are determined by the device manufacturer and are specific to the device's platform or model.

### Supported Platforms

- Android
- iOS

### Quick Example

```js
// Android: Returns a random 64-bit integer (as a string, again!)
//
// iOS: (Paraphrased from the UIDevice Class documentation)
//         Returns the [UIDevice identifierForVendor] UUID which is unique and the same for all apps installed by the same vendor. However the UUID can be different if the user deletes all apps from the vendor and then reinstalls it.
//
var deviceID = device.uuid;
```

### Android Quirk

The `uuid` on Android is a 64-bit integer (expressed as a hexadecimal string). The behaviour of this `uuid` is different on two different OS versions-

**For < Android 8.0 (API level 26)**

In versions of the platform lower than Android 8.0, the `uuid` is randomly generated when the user first sets up the device and should remain constant for the lifetime of the user's device.

**For Android 8.0 or higher**

The above behaviour was changed in Android 8.0. Read it in detail [here](https://developer.android.com/about/versions/oreo/android-8.0-changes#privacy-all).

On Android 8.0 and higher versions, the `uuid` will be unique to each combination of app-signing key, user, and device. The value is scoped by signing key and user. The value may change if a factory reset is performed on the device or if an APK signing key changes.

Read more here https://developer.android.com/reference/android/provider/Settings.Secure#ANDROID_ID.

### iOS Quirk

The `uuid` on iOS uses the identifierForVendor property. It is unique to the device across the same vendor, but will be different for different vendors and will change if all apps from the vendor are deleted and then reinstalled.
Refer [here](https://developer.apple.com/documentation/uikit/uidevice/1620059-identifierforvendor) for details.
The UUID will be the same if app is restored from a backup or iCloud as it is saved in preferences. Users using older versions of this plugin will still receive the same previous UUID generated by another means as it will be retrieved from preferences.

### OS X Quirk

The `uuid` on OS X is generated automatically if it does not exist yet and is stored in the `standardUserDefaults` in the `CDVUUID` property.

## device.version

Get the operating system version.

    var string = device.version;

### Supported Platforms

- Android
- Browser
- iOS

### Quick Example

```js
// Android:    Froyo OS would return "2.2"
//             Eclair OS would return "2.1", "2.0.1", or "2.0"
//             Version can also return update level "2.1-update1"
//
// Browser:    Returns version number for the browser
//
// iOS:     iOS 3.2 returns "3.2"
//
var deviceVersion = device.version;
```

## device.manufacturer

Get the device's manufacturer.

    var string = device.manufacturer;

### Supported Platforms

- Android
- iOS

### Quick Example

```js
// Android:    Motorola XT1032 would return "motorola"
// iOS:     returns "Apple"
//
var deviceManufacturer = device.manufacturer;
```

## device.isVirtual

whether the device is running on a simulator.

```js
var isSim = device.isVirtual;
```

## device.sdkVersion (Android only)

Get the Android device's SDK version ([SDK_INT](https://developer.android.com/reference/android/os/Build.VERSION#SDK_INT)).

### Supported Platforms

- Android

### OS X and Browser Quirk

The `isVirtual` property on OS X and Browser always returns false.

## device.serial

Get the device hardware serial number ([SERIAL](https://developer.android.com/reference/android/os/Build.html#SERIAL)).

```js
var string = device.serial;
```

### Supported Platforms

- Android
- OS X

### Android Quirk

As of Android 9, the underlying native API that powered the `uuid` property is deprecated and will always return `UNKNOWN` without proper permissions. Cordova have never implemented handling the required permissions. As of Android 10, **all** non-resettable device identifiers are no longer readable by normal applications and will always return `UNKNOWN`. More information can be [read here](https://developer.android.com/about/versions/10/privacy/changes#non-resettable-device-ids).

## device.isiOSAppOnMac

The iOS app is running on the Mac desktop (Apple Silicon ARM64 processor, M1 or newer). 
This parameter is only returned for iOS V14.0 or later, and is not returned for Android devices.

```js
var boolean = device.isiOSAppOnMac;
```

### Supported Platforms

- iOS

---

## iOS Privacy Manifest

As of May 1, 2024, Apple requires a privacy manifest file to be created for apps and third-party SDKs. The purpose of the privacy manifest file is to explain the data being collected and the reasons for the required APIs it uses. Starting with `cordova-ios@7.1.0`, APIs are available for configuring the privacy manifest file from `config.xml`.

This plugin comes pre-bundled with a `PrivacyInfo.xcprivacy` file that contains the list of APIs it uses and the reasons for using them.

However, as an app developer, it will be your responsibility to identify additional information explaining what your app does with that data.

In this case, you will need to review the "[Describing data use in privacy manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_data_use_in_privacy_manifests)" to understand the list of known `NSPrivacyCollectedDataTypes` and `NSPrivacyCollectedDataTypePurposes`.

For example, if you collected the device ID for app functionality and analytics, you would write the following in `config.xml`:

```xml
<platform name="ios">
    <privacy-manifest>
        <key>NSPrivacyTracking</key>
        <false/>
        <key>NSPrivacyTrackingDomains</key>
        <array/>
        <key>NSPrivacyAccessedAPITypes</key>
        <array/>
        <key>NSPrivacyCollectedDataTypes</key>
        <array>
            <dict>
                <key>NSPrivacyCollectedDataType</key>
                <string>NSPrivacyCollectedDataTypeDeviceID</string>
                <key>NSPrivacyCollectedDataTypeLinked</key>
                <false/>
                <key>NSPrivacyCollectedDataTypeTracking</key>
                <false/>
                <key>NSPrivacyCollectedDataTypePurposes</key>
                <array>
                    <string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
                    <string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
                </array>
            </dict>
        </array>
    </privacy-manifest>
</platform>
```

Also, ensure all four keys—`NSPrivacyTracking`, `NSPrivacyTrackingDomains`, `NSPrivacyAccessedAPITypes`, and `NSPrivacyCollectedDataTypes`—are defined, even if you are not making an addition to the other items. Apple requires all to be defined.
