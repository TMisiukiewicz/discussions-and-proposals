---
title: Implement Expo Config Plugins into React Native CLI
author:
- Tomasz Misiukiewicz
date: today
---

# RFC0000: Supporting Expo Config Plugins in React Native CLI

## Summary

Implementing Expo Config Plugins into React Native CLI to support modifying application's config without having to manually edit the native files.

## Basic example

~~If the proposal involves a new or changed API, include a basic code example. Omit this section if it's not applicable.~~

## Motivation

The main motivation between this proposal is simplifying the process of upgrading apps to the newest React Native version. Very often it requires some additional manual steps on the native side, resolving conflicts etc. With Expo Config Plugins implemented into React Native, it would be possible to store all the native-related config on the JS side and apply it into native files whenever it's needed, e.g while running `react-native upgrade` - without having Expo in the project. On top of that, it would become possible to build config plugins for Windows and Mac.

## Detailed design

The very first step would be upstreaming non-Expo related code of `@expo/config-plugins` and `@expo/config-types` into React Native core. It contains all logic and helpers needed to modify native side of RN project easily from JS side. Expo would be still able to extend it on their side with all the stuff related to Expo, like EAS Builds etc.

Upstreaming it will unlock a few new paths that React Native could follow. First of all, and probably most important, it can change the way of generating native code. We can move away from being dependent on `/android` and `ios` directories, and instead of it, create temporary directory with both of these folders generated in the runtime. Native templates could be moved into separate directory in React Native core and copied into temporary directory (if it  does not exist yet) when running one of the commands:
- `start`
- `run-ios`
- `run-android`
- `build-ios`
- `build-android`
- `upgrade`

All of them would use newly created method that would apply all the changes defined in `app.json` file. This file determines how a project is loaded. Expo is using it to determine how to load the app config in Expo Go and Expo Prebuild. With config plugins implemented, this file could work the same way as in Expo, but, by default, it would not support Expo-related properties (Expo will still be able to easily extend this config to match their needs). You can get more info about configuration with app.json in Expo [here](https://docs.expo.dev/workflow/configuration). After quick investigation, React Native could support the following properties by default:

### General
- `name` - The name of the app.
- `platforms` - Platforms that project supports.
- `orientation` - Locks the app to a specific orientation with portrait or landscape. Defaults to no lock. Valid values: `default`, `portrait`, `landscape`
- `androidStatusBar` - Configuration for the status bar on Android.
- `scheme` - URL scheme to link into the app.
- `splash` - Configuration for loading and splash screen.

### iOS
- `bundleIdentifier` - iOS bundle identifier notation unique name for the app.
- `buildNumber` Build number for your iOS standalone app. Corresponds to `CFBundleVersion`
 and must match Apple's [specified format](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion).
- `icon` Local path or remote URL to an image to use for app's icon on iOS. If specified, this overrides the top-level `icon` key. Use a 1024x1024 icon which follows Apple's interface guidelines for icons, including color profile and transparency.
- `bitcode` Enable iOS Bitcode optimizations in the native build. Accepts the name of an iOS build configuration to enable for a single configuration and disable for all others, e.g. Debug, Release.
- `supportsTablet` Whether standalone iOS app supports tablet screen sizes. Defaults to `false`
- `isTabletOnly` If true, indicates that iOS app does not support handsets, and only supports tablets.
- `userInterfaceStyle` - Configuration to force the app to always use the light or dark user-interface appearance, such as "dark mode", or make it automatically adapt to the system preferences. If not provided, defaults to `light`
- `infoPlist` - Dictionary of arbitrary configuration to add to app's native Info.plist.
- `entitlements` Dictionary of arbitrary configuration to add to app's native *.entitlements (plist).
- `associatedDomains` An array that contains Associated Domains for the app.
- `accessesContactNotes` - A Boolean value that indicates whether the app may access the notes stored in contacts. [Receiving a permission from Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_contacts_notes) is required before submitting an app for review with this capability.
- `splash` Configuration for loading and splash screen for iOS apps.
- `runtimeVersion` The runtime version associated with this manifest for the iOS platform. If provided, this will override the top level runtimeVersion key.

### Android
- `package` The package name for Android app. It needs to be unique on the Play Store
- `versionCode` Version number required by Google Play. Must be a positive integer.
- `backgroundColor` The background color for Android app, behind any of React views. Overrides the top-level `backgroundColor` key if it is present.
- `icon` Local path or remote URL to an image to use for app's icon on Android. If specified, this overrides the top-level `icon` key. 1024x1024 png file is recommended (transparency is recommended for the Google Play Store). This icon will appear on the home screen.
- `adaptiveIcon` Settings for an Adaptive Launcher Icon on Android
- `permissions` List of permissions used by the app.
- `blockedPermissions` List of permissions to block in the final `AndroidManifest.xml`.
- `splash` Configuration for loading and splash screen for Android apps.
- `intentFilters` Configuration for setting an array of custom intent filters in Android manifest.
- `allowBackup` Allows user's app data to be automatically backed up to their Google Drive. If this is set to false, no backup or restore of the application will ever be performed (this is useful if your app deals with sensitive information). Defaults to the Android default, which is `true`
- `softwareKeyboardLayoutMode` Determines how the software keyboard will impact the layout of the application. This maps to the `android:windowSoftInputMode` property. Defaults to `resize`. Valid values: `resize`, `pan`
- `runtimeVersion` The runtime version associated with this manifest for the Android platform. If provided, this will override the top level runtimeVersion key.

The CLI would look for any changes in the `app.json` file and regenerate the temporary folders if changes are applied.

Additionally, it would become possible to create custom plugins within the app. Since the plugins would be available to import directly from `react-native`, it would not need any additional setup on the developer side. E.g. developer can easily create a custom plugin:

```js
./plugins/withStringsXml.js


const { AndroidConfig, withStringsXml } = require('react-native')

module.exports = function (config) {
    return withStringsXml(config, (conf) => {
      conf.modResults = AndroidConfig.Strings.setStringItem(
        [
          {
            _: 'true',
            $: {
              name: 'test_value',
              translatable: 'false'
            }
          }
        ],
        conf.modResults
      )
      return conf
    })
}
```

```js
app.json


"plugins": ["./plugins/withStringsXml.js"],
```

A list of the mods could be used for custom plugins:

### Android
- `withAndroidManifest`
- `withStringsXml`
- `withAndroidColors`
- `withAndroidColorsNight`
- `withAndroidStyles`
- `withMainActivity`
- `withMainApplication`
- `withProjectBuildGradle`
- `withAppBuildGradle`
- `withSettingsGradle`
- `withGradleProperties`

### iOS
- `withAppDelegate`
- `withInfoPlist`
- `withEntitlementsPlist`
- `withExpoPlist`
- `withXcodeProject`
- `withPodfileProperties`

It would also be possible to use all of the helpers currently available in `@expo/config-plugins`, like `AndroidConfig.Strings.setStringItem` in the example above.

These changes would possibly also affect libraries development and allow maintainers to use this API within libraries. Based on the [Expo tutorial](https://docs.expo.dev/modules/config-plugin-and-native-module-tutorial/), we could add support for `app.plugin.js` file for libraries and then declare them under `"plugins"` property in `app.json` the same way Expo is doing that with their libraries:
```
"plugins": [
      [
        "expo-camera",
        {
          "cameraPermission": "Allow $(PRODUCT_NAME) to access your camera."
        }
      ]
    ]
```

![Screenshot 2023-04-04 at 11 06 48](https://user-images.githubusercontent.com/13985840/229743911-38cc52e3-877e-4f01-a912-c78af608f1ac.png)

## Drawbacks

~~Why should we _not_ do this? Please consider:~~

- ~~implementation cost, both in term of code size and complexity~~
- ~~whether the proposed feature can be implemented in user space~~
- ~~the impact on teaching people React Native~~
- ~~integration of this feature with other existing and planned features~~
- ~~cost of migrating existing React Native applications (is it a breaking change?)~~

~~There are tradeoffs to choosing any path. Attempt to identify them here.~~

## Alternatives

~~What other designs have been considered? Why did you select your approach?~~

## Adoption strategy

~~If we implement this proposal, how will existing React Native developers adopt it? Is this a breaking change? Can we write a codemod? Should we coordinate with other projects or libraries?~~

## How we teach this

~~What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing React patterns?~~

~~Would the acceptance of this proposal mean the React Native documentation must be re-organized or altered? Does it change how React Native is taught to new developers at any level?~~

~~How should this feature be taught to existing React Native developers?~~

## Unresolved questions

~~Optional, but suggested for first drafts. What parts of the design are still TBD?~~
