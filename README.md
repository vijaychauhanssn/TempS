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

Yes. You can verify many important things before installing the APK on your Android phone. In fact, I recommend adding an APK pre-installation verification process to your React Native build workflow.

However, there is one limitation: static checks can verify that the JavaScript bundle is packaged, but they cannot guarantee that the app will successfully execute on every Android device. For that, an emulator test is needed.

## 1. Pre-installation verification checklist

You can perform these five tests on Windows.

| Test                 | What it verifies                         |
| -------------------- | ---------------------------------------- |
| APK integrity        | APK is not corrupted                     |
| APK contents         | JavaScript bundle is included            |
| Expo bundling        | Metro can generate production JavaScript |
| Gradle configuration | Release bundling is enabled              |
| APK signing          | APK signature is valid                   |

## 2. Test 1 — Check APK integrity

Open PowerShell:

```
cd D:\USGI_APP\artifacts\sompo-insurance-mobile\android

$apk = "app\build\outputs\apk\release\app-release.apk"

jar tf $apk > $null

if ($LASTEXITCODE -eq 0) {
    Write-Host "APK ZIP integrity: PASS"
} else {
    Write-Host "APK ZIP integrity: FAIL"
}
```

This checks whether the APK is a readable ZIP archive.

## 3. Test 2 — Check JavaScript bundle inside APK

```
jar tf $apk | Select-String "index.android.bundle"
```

Expected entry:

```
assets/index.android.bundle
```

If no output appears, investigate whether the release JavaScript bundle was packaged.

Also check the Gradle-generated bundle:

```
Test-Path "app\build\generated\assets\createBundleReleaseJsAndAssets\index.android.bundle"
```

If it returns `True`, the expected generated bundle exists at that location.

Note: Build variants and Gradle versions can use different output paths.

## 4. Test 3 — Check APK signature

Find your Android SDK Build Tools:

```
Get-ChildItem "$env:ANDROID_HOME\build-tools"
```

Then use the installed `apksigner.bat`:

```
& "$env:ANDROID_HOME\build-tools\35.0.0\apksigner.bat" verify --verbose $apk
```

Replace `35.0.0` with your installed Build Tools version.

Expected:

```
Verifies
```

This checks the APK signature, not whether React Native JavaScript executes correctly.

## 5. Test 4 — Check using Android Studio

Android Studio has a useful feature:

Build → Analyze APK

[Cómo analizar tu compilación con el Analizador de APK  |  Android Studio  |  Android Developers](https://images.openai.com/static-rsc-4/IY_R1HqBvZ4miBSwzq6UZzV1Xoiw-uAgtC9dEOWoW9E9k7x0ACFsKNDZhRVO6iAgPRJka1gJUADDbyK-dd2JpoSAyLAiM0s6lQEak3rJXShq4Jr8MCzGBuRugbu_rjswWLKZj-CSfEdekdPd_qKDumb-rtAivItnaBkbnEItkbA?purpose=inline)

[developer.android.com](https://developer.android.com/studio/debug/apk-analyzer?hl=es-419\&utm_source=chatgpt.com)

Select your generated APK.

Inspect:

- `assets/`
- `assets/index.android.bundle`
- `lib/`
- `classes.dex`
- `AndroidManifest.xml`

Check that the JavaScript bundle is present and that the native libraries are packaged.

## 6. Test 5 — Most reliable: Android Emulator

Recommended runtime verification

Static checks cannot guarantee that the application launches successfully.

Use Android Studio's emulator:

1. Open Device Manager.
2. Start an Android emulator.
3. Install your release APK on the emulator.
4. Disable Wi-Fi and ensure Metro is stopped.
5. Launch your application.

If the home screen loads, the APK has passed an important standalone runtime test.

## 7. My recommendation: Create a pre-build verification script

You can use this PowerShell script after every release build.

```

$apk = "app\build\outputs\apk\release\app-release.apk"

Write-Host "`n===== APK PRE-FLIGHT CHECK ====="

# 1. APK exists
if (!(Test-Path $apk)) {
    Write-Host "FAIL: APK not found"
    exit 1
}
Write-Host "PASS: APK exists"

# 2. ZIP integrity
jar tf $apk > $null
if ($LASTEXITCODE -ne 0) {
    Write-Host "FAIL: APK integrity"
    exit 1
}
Write-Host "PASS: APK integrity"

# 3. Bundle presence
$entries = jar tf $apk
$bundle = $entries | Where-Object {
    $_ -match 'assets/index\.android\.bundle$'
}

if ($bundle) {
    Write-Host "PASS: JavaScript bundle found"
} else {
    Write-Host "WARNING: Expected bundle not found"
}

# 4. APK size
$size = (Get-Item $apk).Length / 1MB
Write-Host ("APK Size: {0:N2} MB" -f $size)

Write-Host "`n===== CHECK COMPLETED ====="

```

This is a basic preflight check; add signature verification and emulator testing for stronger assurance.

## My final recommendation for your project

Ideal React Native release workflow

1. pnpm install — dependencies installed successfully.
2. Expo export — JavaScript bundling succeeds.
3. Gradle assembleRelease — APK generated.
4. APK Analyze — bundle and native assets verified.
5. APK signature and integrity — verified.
6. Emulator test — app launches without Metro.
7. Physical Android installation — final verification.

The most important check for your original error is Step 2 and Step 4: verify that the bundle was generated and that the exact bundle is inside the APK.

This will help catch the missing-bundle problem before you transfer the APK to your phone.
