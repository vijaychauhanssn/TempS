# TempS

Unable to load script.

Make sure you're running Metro or that your
bundle 'index.android.bundle' is packaged
correctly for release.

The device must either be USB connected (with
bundler set to "localhost:8081") or be on the
same Wi-Fi network as your computer (with
bundler set to your computer IP) to connect to
Metro.

If you're using USB on a physical device, make
sure you also run this command:

adb reverse tcp:8081 tcp:8081

Your thinking is partially correct. There is a configuration in React Native that determines whether the application loads JavaScript from Metro (`localhost:8081`) or from the APK's internal assets.

But `localhost:8081` is not necessarily hardcoded into your application source code.

Let's understand where it comes from.

## 1. Why does React Native use localhost:8081?

React Native has two JavaScript loading modes.

Debug mode

Requires Metro

Computer running Metro

`localhost:8081`

Android application

Loads JavaScript from Metro

Release mode

Standalone APK

Android APK

`assets/index.android.bundle` (or generated Hermes bundle)

React Native runtime

Loads packaged JavaScript

## 2. Which code controls this?

There are three important places to investigate.

A. `android/app/build.gradle`

Look for:

```
react {
    debuggableVariants = ["debug"]
}
```

This tells the React Native Gradle plugin which variants should load JavaScript from Metro rather than generate a packaged bundle.

If you have accidentally included `release`:

```
debuggableVariants = ["debug", "release"]
```

Remove `release`.

B. Android development server configuration

Look for this file:

```
android/app/src/debug/res/xml/react_native_dev_server.xml
```

Also check whether your Android application has custom native configuration for the Metro development server.

C. Android Dev Settings

In debug builds, React Native can use a configured development server host. This is normally set through developer settings or Android debugging configuration.

## 3. Why does localhost fail on your phone?

This is important:

`localhost` on your Windows computer and `localhost` on your Android phone are different addresses.

| Address                         | Meaning                                           |
| ------------------------------- | ------------------------------------------------- |
| Windows `localhost:8081`        | Your computer's Metro server                      |
| Android `localhost:8081`        | Your Android phone itself                         |
| `adb reverse tcp:8081 tcp:8081` | Forwards phone localhost traffic to your computer |

That's why React Native shows the USB and Wi-Fi instructions.

## 4. What should your permanent configuration be?

For your standalone APK:

```
react {
    debuggableVariants = ["debug"]

    bundleCommand = "export:embed"
}
```

Keep the Expo CLI and entry-point configuration we discussed earlier.

Then generate:

```
cd android

.\gradlew.bat clean
.\gradlew.bat assembleRelease
```

The release APK should load its packaged JavaScript without requiring Metro.

## 5. One important diagnostic question

I want to distinguish between two situations:

When you open your generated APK, what happens?

I install the APK and open it directly. The localhost error appears.

The application works only when Metro is running.

Both debug and release APKs show the same error.

I am not sure which build variant I generated.

Submit

My suspicion: Your Android application is trying to use the debug JavaScript loading mechanism, or the release bundle isn't being generated or loaded correctly. The localhost message alone cannot distinguish these two causes.
