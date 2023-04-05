---
title: Support Expo Config Plugins in React Native CLI
author:
- Tomasz Misiukiewicz
date: today
---

# RFC0000: Support Expo Config Plugins in React Native CLI

## Summary

Implementing Expo Config Plugins into React Native CLI to support modifying application's config without having to manually edit the native files.

## Basic example

https://user-images.githubusercontent.com/13985840/229797752-81009e28-3153-4702-b52f-14602789a87b.mov

In this example, when running `start` command, Android permissions defined in `app.json` are added to `AndroidManifest.xml` file, and a new key is added into `Info.plist` file based on a custom plugin:

```js
module.exports = function (config, { apiKey }) {
    // Ensure the objects exist
    if (!config.ios) {
      config.ios = {};
    }
    if (!config.ios.infoPlist) {
      config.ios.infoPlist = {};
    }
  
    // Append the apiKey
    config.ios.infoPlist['MY_CUSTOM_NATIVE_IOS_API_KEY'] = apiKey;
  
    return config;
  };
```
## Motivation

The main motivation between this proposal is simplifying the process of upgrading apps to the newest React Native version. Very often it requires some additional manual steps on the native side, resolving conflicts etc. With Expo Config Plugins implemented into React Native, it would be possible to store all the native-related config on the JS side and apply it into native files whenever it's needed, e.g while running `react-native upgrade` - without having Expo in the project. 
On top of that, it will unlock new possibilities to create config plugins for Out-of-Tree platforms, like Windows and MacOS.

## Detailed design

![Screenshot 2023-04-05 at 11 01 31](https://user-images.githubusercontent.com/13985840/230034001-4e40b04a-f474-47a9-98f7-15fdfa1bf7e8.png)

### What are config plugins?
According to the [Expo docs](https://docs.expo.dev/guides/config-plugins/), config plugins are a system for extending Expo config and customizing the prebuild phase of managed builds. Expo CLI is using it internally to generate and configure all the native code for managed projects.

Based on that, for the purpose of this RFC, I'd define "Config Plugins" as a set of configuration options for React Native app, providing a standarized way of generating and configuring native code on all platforms.

### How would it work?

The very first step would be upstreaming non-Expo related code of `@expo/config-plugins` and `@expo/config-types` into React Native core. It contains all logic and helpers needed to modify native side of RN project easily from JS side. We'll handle discussions with Expo to coordinate what parts of config plugins can be safely upstreamed into core. It has to be done in a way where Expo would still able to extend it on their side with all the stuff related to Expo, like EAS Builds etc.

Upstreaming it will unlock a few new paths that React Native could follow. First of all, and probably most important, it can change the way of generating native code. It would become possible to add platform-specific folders like `android` and `ios` to `.gitignore` by default and keep them out of the repository unless it's needed (e.g. when it's not possible to do some native-side changes with using config plugins). These folders would be automatically (re)generated when running one of the following commands:
- `start`
- `run-ios`
- `run-android`
- `build-ios`
- `build-android`
- `upgrade`

All of them would use newly created method that would apply all the changes defined in `app.json` file. Expo is using this file to determine how to load the app config in Expo Go and Expo Prebuild. With config plugins implemented, this file could work the same way as in Expo, but, by default, it would not support Expo-related properties (Expo will still be able to easily extend this config to match their needs). You can get more info about configuration with app.json in Expo [here](https://docs.expo.dev/workflow/configuration). It will also become an opportunity to create configurations for Out-of-Tree Platforms. Here's the list of Expo Config properties, divided by those that React Native could possibly handle out of the box, and the Expo-specific ones: 

#### General
##### Possible to adopt by RN
- `name` - The name of the app.
- `platforms` - Platforms that project supports.
- `orientation` - Locks the app to a specific orientation with portrait or landscape. Defaults to no lock. Valid values: `default`, `portrait`, `landscape`
- `scheme` - URL scheme to link into the app.
- `primaryColor` - determines the color of the app in the multitasker.
- `icon` - local path or remote url to use for app's icon
- `locales` - overrides by locale for System Dialog prompts like permission boxes
- `plugins` - config plugins for adding extra functionality to the project
##### Expo-specific
- `slug` - friendly URL name for publishing
- `owner` - name of the Expo account that owns the project
- `currentFullName` - auto generated Expo account name and slug used for display purposes
- `originalFullName` - auto generated Expo account name and slug used for services like Notifications and AuthSession proxy
- `privacy` - project page access restrictions
- `sdkVersion`- Expo sdkVersion to run the project on.
- `splash` - Configuration for loading and splash screen.
- `githubUrl` - linking GH project with Expo project page
- `userInterfaceStyle` - forces app to always use the light/dark/automatic mode. Requires `expo-system-ui` to be installed
- `backgroundColor` - background color for the app, behind all React views. Requires `expo-system-ui` to be installed
- `androidStatusBar` - configuration of status bar on Android, requires `expo-status-bar`
- `androidNavigationBar` - config for bottom navigator bar. Requires `expo-navigation-bar`
- `developmentClient` - settings specific to running app in dev client
- `extra` - extra fields to pass, accessible via `Expo.Constants.manifest.extra`
- `updates` - configuration for how and when the app should request OTA updates
- `isDetached` - is app detached
- `detach` - extra fields needed by detached apps
- `assetBundlePatterns` - array of file glob strings which point to assets bundled within app binary
- `notification` - configure for remote push notifications
- `jsEngine` - specifies the JS engine for apps, supported only on EAS Build
- `hooks` - configuration for scripts to run to hook into the publish process
- `experiments` - experimental features related to Expo SDK

#### iOS
##### Possible to adopt by RN
- `bundleIdentifier` - iOS bundle identifier notation unique name for the app.
- `buildNumber` - Build number for your iOS standalone app. Corresponds to `CFBundleVersion`
 and must match Apple's [specified format](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion).
- `icon` - Local path or remote URL to an image to use for app's icon on iOS. If specified, this overrides the top-level `icon` key. Use a 1024x1024 icon which follows Apple's interface guidelines for icons, including color profile and transparency.
- `bitcode` - Enable iOS Bitcode optimizations in the native build. Accepts the name of an iOS build configuration to enable for a single configuration and disable for all others, e.g. Debug, Release.
- `supportsTablet` - Whether standalone iOS app supports tablet screen sizes. Defaults to `false`
- `isTabletOnly` - If true, indicates that iOS app does not support handsets, and only supports tablets.
- `userInterfaceStyle` - Configuration to force the app to always use the light or dark user-interface appearance, such as "dark mode", or make it automatically adapt to the system preferences. If not provided, defaults to `light`
- `infoPlist` - Dictionary of arbitrary configuration to add to app's native Info.plist.
- `entitlements` - Dictionary of arbitrary configuration to add to app's native *.entitlements (plist).
- `associatedDomains` - An array that contains Associated Domains for the app.
- `accessesContactNotes` - A Boolean value that indicates whether the app may access the notes stored in contacts. [Receiving a permission from Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_contacts_notes) is required before submitting an app for review with this capability.
- `runtimeVersion` - The runtime version associated with this manifest for the iOS platform. If provided, this will override the top level runtimeVersion key.
- `googleServicesFile` - location of GoogleService-Info.plist` file for configuring Firebase
- `requireFullScreen` - indicates that iOS app does not support Slide Over and Split View on iPad

##### Expo-specific
- `publishManifestPath` - manifest for the iOS version written to this path during publish
- `publishBundlePath` - bundle for the iOS version written to this path during publish
- `splash` - Configuration for loading and splash screen for iOS apps.
- `backgroundColor` -  background color for the app, behind all React views. Requires `expo-system-ui` to be installed
- `appStoreUrl` - link to your store page from your Expo project page if app is public
- `usesIcloudStorage` - indicating if the app uses iCloud Storage for Expo's DocumentPicker
- `usesAppleSignIn` - indicating if the app uses Apple Sign-In with Expo's Apple Authentication
- `jsEngine` - Specifies the JavaScript engine for iOS apps. Supported only on EAS Build
- `config` - config for e.g. Google Maps API, Mobile Ads API etc

#### Android
##### Possible to adopt by RN
- `package` The package name for Android app. It needs to be unique on the Play Store
- `versionCode` Version number required by Google Play. Must be a positive integer.
- `backgroundColor` The background color for Android app, behind any of React views. Overrides the top-level `backgroundColor` key if it is present.
- `icon` Local path or remote URL to an image to use for app's icon on Android. If specified, this overrides the top-level `icon` key. 1024x1024 png file is recommended (transparency is recommended for the Google Play Store). This icon will appear on the home screen.
- `adaptiveIcon` Settings for an Adaptive Launcher Icon on Android
- `permissions` List of permissions used by the app.
- `blockedPermissions` List of permissions to block in the final `AndroidManifest.xml`.
- `intentFilters` Configuration for setting an array of custom intent filters in Android manifest.
- `allowBackup` Allows user's app data to be automatically backed up to their Google Drive. If this is set to false, no backup or restore of the application will ever be performed (this is useful if your app deals with sensitive information). Defaults to the Android default, which is `true`
- `softwareKeyboardLayoutMode` Determines how the software keyboard will impact the layout of the application. This maps to the `android:windowSoftInputMode` property. Defaults to `resize`. Valid values: `resize`, `pan`
- `runtimeVersion` The runtime version associated with this manifest for the Android platform. If provided, this will override the top level runtimeVersion key.
- `googleServicesFile` - Location of the `GoogleService-Info.plist` file for configuring Firebase. Including this key automatically enables FCM in the app.

##### Expo-specific
- `publishManifestPath` - manifest for the iOS version written to this path during publish
- `publishBundlePath` - bundle for the iOS version written to this path during publish
- `splash` Configuration for loading and splash screen for Android apps.
- `userInterfaceStyle` - forces app to always use the light/dark/automatic mode. Requires `expo-system-ui` to be installed
- `playStoreUrl` - URL to your app on the Google Play Store, used to link Expo project with store page
- `jsEngine` - Specifies the JavaScript engine for iOS apps. Supported only on EAS Build
- `config` - config for e.g. Google Maps API, Mobile Ads API etc

The CLI would look for any changes in the `app.json` file and regenerate the platform-specific folders when running one of the React Native commands.

Additionally, it would become possible to create custom plugins within the app. Since the plugins would be available to import directly from the core, it would not need any additional setup on the developer side. E.g. developer can easily create a custom plugin:

`./plugins/withStringsXml.js`:
```js
const { AndroidConfig, withStringsXml } = require('@react-native/config-plugins')

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
`app.json`:
```diff
{
  "name": "MyTestApp",
  "displayName": "MyTestApp",
+  "plugins": ["./plugins/withNewString.js"]
}
```
In the example above, custom plugin is using `withStringsXml` mod. Mods are async functions that modify native files. A list of the mods available for custom plugins:

#### Android
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

#### iOS
- `withAppDelegate`
- `withInfoPlist`
- `withEntitlementsPlist`
- `withExpoPlist`
- `withXcodeProject`
- `withPodfileProperties`

It would also be possible to use all of the helpers currently available in `@expo/config-plugins`, like `AndroidConfig.Strings.setStringItem` in the example above.

The mods can be also are appended to the mods object of the configuration. This section varies from the other parts of the configuration as it is not serialized after the initial reading. It's possible to utilize it to execute operations while generating code. In general, creating custom plugins with mod functions are simpler to work with, so it's recommended to use it instead of using `mods` object in `app.json`.

### `prebuild` command
Additionally, React Native CLI would have a couple of new commands, similar to `prebuild` known from Expo CLI. The method applying config plugins into native code would run with the commands mentioned above, but, `prebuild` command would additionally add the native files into repository to make sure developers can safely make any manual changes in native folders. If there is a need of prebuilding only one of the platforms, it would be possible to use `prebuild-android` and `prebuild-ios` commands. Prebuild would run all config plugins to apply all of the necessary changes. There should also be a possibility of using `--clean` flag with prebuild to install fresh copy of native directories and apply all config plugins. It should be used only if there are no manual changes made to native files.

### Library development

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

### Improved updates
With this solution it seems like upgrading React Native would become much easier. Each time React Native gets upgraded, we can just copy-paste fresh iOS and Android templates from RN core and apply all the changes using config plugins. We could track if any manual changes were made in the native folders, and if yes, warn user about it and allow to upgrade only in an old way.

### Support for other platforms
This approach would also make it possible to extend config plugins with other platforms, e.g. `react-native-windows`. Following the same pattern, a support for `windows` and `macos` properties could be added to `app.json` together with some custom mods for native files.

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
- How it would affect autolinking of the libraries?
- How to correctly handle `pod install`?
- What changes have to be done to point the correct path for temporary directory without exposing this information for developers?
- Would it be possible for library maintainers to [create a native module using config plugins](https://docs.expo.dev/modules/config-plugin-and-native-module-tutorial/) without Expo Modules in the project?
