---
title: Add Kuira to an Android project
tags:
  - setup
  - integration
  - hilt
prerequisites:
  - Android Studio Ladybug or newer
  - JDK 17
  - minSdk 30+
  - Hilt + KSP already wired in the project
agent_bundle: https://raw.githubusercontent.com/kuiralabs/kuira-sdk-android/main/docs/recipes/add-kuira-to-an-android-project.md
---

# Add Kuira to an Android project

**Outcome:** your existing Android app builds against the Kuira SDK,
recognises your passkey domain, and has the Hilt graph wired so any
`@HiltViewModel` consuming SDK panels resolves cleanly.

<div data-copy-prompt="https://raw.githubusercontent.com/kuiralabs/kuira-sdk-android/main/docs/recipes/add-kuira-to-an-android-project.md"
     data-task="Integrate Kuira SDK into this Android project — install the dependency, provide a PasskeyConfig Hilt module for the project's own domain, and enable debug cleartext for localnet (assetlinks.json hosting is left to the developer)."></div>

---

## Prerequisites

- An Android project with `minSdk ≥ 30`.
- **A domain you control** that will serve as your passkey relying-party
  identifier (e.g. `yourapp.example.com`). You need write access to host
  `.well-known/assetlinks.json` on it.

### Version pin matrix

The SDK is built against the toolchain below; using a newer Kotlin without
a matching KSP version is the #1 cause of a build error. Pin to these
exact versions until `alpha02` is published, then bump together.

| Tool | Version | Notes |
|---|---|---|
| **AGP** | `8.13.2` | `8.13.x` minimum |
| **Kotlin** | `2.3.20` | Matched KSP below |
| **KSP** | `2.3.6` | Plugin id `com.google.devtools.ksp` |
| **Hilt** | `2.58` | Both `hilt-android` and `hilt-compiler` |
| **Compose BOM** | `2026.03.01` | If you use Compose (Recipe 2 does) |
| **JDK** | `17` | `sourceCompatibility` / `targetCompatibility` / `jvmTarget` |
| **`compileSdk`** | `36` | `35` works; the SDK was built against 36 |
| **`minSdk`** | `30` | Mandatory — biometric Keystore APIs |

---

## Step 1 — Add the dependency

First make sure your root `settings.gradle.kts` declares `mavenCentral()`
in both `pluginManagement` (so the Contract plugin from Recipe 3 can
resolve) and `dependencyResolutionManagement` (so the runtime
artifacts resolve):

```kotlin title="settings.gradle.kts"
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "your-app"
include(":app")
```

Then add the SDK to your app module's dependencies:

=== "Gradle (Kotlin DSL)"

    ```kotlin title="app/build.gradle.kts"
    dependencies {
        // One line; pulls in midnight-sdk, wallet-runtime, identity,
        // compact-engine, designsystem, … as transitive api dependencies.
        implementation("io.github.kuiralabs:dapp-ui:{{ kuira_version }}")

        // Hilt — required (the SDK is Hilt-first).
        implementation("com.google.dagger:hilt-android:2.58")
        ksp("com.google.dagger:hilt-compiler:2.58")
        // hilt-navigation-compose for `hiltViewModel()` — needed by
        // Recipe 2 to obtain the SigilStatusPanel's ViewModel.
        implementation("androidx.hilt:hilt-navigation-compose:1.1.0")

        // Compose — required if you'll render the SDK's panels.
        implementation(platform("androidx.compose:compose-bom:2026.03.01"))
        implementation("androidx.compose.ui:ui")
        implementation("androidx.compose.material3:material3")
        implementation("androidx.activity:activity-compose:1.12.2")

        // FragmentActivity — the SDK's biometric prompts need an
        // Activity that subclasses FragmentActivity (AppCompatActivity
        // qualifies; ComponentActivity does not).
        implementation("androidx.fragment:fragment-ktx:1.8.4")
        implementation("androidx.appcompat:appcompat:1.6.1")
    }
    ```

=== "Gradle (Groovy)"

    ```groovy title="app/build.gradle"
    dependencies {
        implementation 'io.github.kuiralabs:dapp-ui:{{ kuira_version }}'

        implementation 'com.google.dagger:hilt-android:2.58'
        ksp 'com.google.dagger:hilt-compiler:2.58'
        implementation 'androidx.hilt:hilt-navigation-compose:1.1.0'

        implementation platform('androidx.compose:compose-bom:2026.03.01')
        implementation 'androidx.compose.ui:ui'
        implementation 'androidx.compose.material3:material3'
        implementation 'androidx.activity:activity-compose:1.12.2'

        implementation 'androidx.fragment:fragment-ktx:1.8.4'
        implementation 'androidx.appcompat:appcompat:1.6.1'
    }
    ```

And declare the plugins at the root `build.gradle.kts`:

```kotlin title="build.gradle.kts (root)"
plugins {
    id("com.android.application") version "8.13.2" apply false
    id("org.jetbrains.kotlin.android") version "2.3.20" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.3.20" apply false
    id("com.google.devtools.ksp") version "2.3.6" apply false
    id("com.google.dagger.hilt.android") version "2.58" apply false
}
```

Apply them in the app module:

```kotlin title="app/build.gradle.kts"
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
    id("com.google.devtools.ksp")
    id("com.google.dagger.hilt.android")
}
```

`dapp-ui` `api`-exposes the rest of the consumer SDK surface
(`midnight-sdk`, `wallet-runtime`, `identity`, `compact-engine`, …),
so you only need that one Kuira line. Hilt, Compose, and
hilt-navigation-compose are platform deps your app would need anyway.

**Verify:** run `./gradlew :app:dependencies | grep kuiralabs` — you
should see roughly a dozen `io.github.kuiralabs:*` entries pulled in
transitively, all at the same version.

---

## Step 2 — Provide your `PasskeyConfig`

The SDK requires every consumer dApp to declare its own passkey
relying-party identifier (RP ID). Create a Hilt module:

```kotlin title="app/src/main/.../di/IdentityConfigModule.kt"
package com.example.myapp.di

import com.midnight.kuira.core.identity.passkey.PasskeyConfig
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object IdentityConfigModule {

    @Provides
    @Singleton
    fun providePasskeyConfig(): PasskeyConfig = PasskeyConfig(
        rpId = "yourapp.example.com",     // (1)
        rpName = "Your App",               // (2)
    )
}
```

1. **Use a domain you control.** Passkey assertions are bound to this
   value; consumers using `nel349.github.io` or another maintainer
   domain will silently fail PRF derivation.
2. **User-facing label** shown in the passkey prompt
   ("Create a passkey for *Your App*").

**Verify:** `./gradlew :app:assembleDebug` should compile. Hilt's KSP
generation runs at compile time — if `PasskeyConfig` isn't provided,
you'll see a "missing binding for `PasskeyConfig`" error.

---

## Step 3 — Host `assetlinks.json` on your `rpId` domain

Digital Asset Links bind your app's package name + signing fingerprint
to your passkey domain. The passkey API refuses any ceremony until
this is in place — symptoms include `RP_ID_MISMATCH`, `PRF
authentication failed`, and silent biometric-prompt dismissal.

This is its own walkthrough — picking the right repo to host the file
in is the part most people get stuck on, and it's not obvious from
the GitHub Pages URL alone. The dedicated recipe covers fingerprint
extraction, which GitHub repo backs which `rpId`, hosting, and
verification.

[Open the full recipe :material-arrow-right:](bind-your-app-to-a-passkey-domain.md){ .md-button .md-button--primary }

You can skip this step temporarily — your app will compile and the
SDK won't crash, but **Forge will fail** until `assetlinks.json` is
hosted and the rpId matches.

---

## Step 4 — Enable debug cleartext for localnet (optional)

If you'll target a Midnight localnet during development, the indexer +
node speak over plain HTTP on `10.0.2.2` (emulator) or `127.0.0.1`
(physical device with `adb reverse`). Add a debug-only manifest
override so Android's network security policy doesn't block them:

```xml title="app/src/debug/AndroidManifest.xml"
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:usesCleartextTraffic="true" />
</manifest>
```

The release build is unaffected — PREPROD and mainnet use HTTPS.

**Verify:** running against localnet, `adb logcat | grep -i "kuira\|midnight"`
should show indexer connection success, not `CLEARTEXT communication
not permitted`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Could not resolve io.github.kuiralabs:dapp-ui` | Missing `mavenCentral()` in `settings.gradle.kts` `dependencyResolutionManagement.repositories`. | Add it. |
| `Missing binding for PasskeyConfig` at Hilt KSP time | Step 2 not done. | Create `IdentityConfigModule.kt`. |
| `Passkey not supported on this device` at runtime | Device lacks Google Password Manager (Android 13 / GMS Core 23.40.13+ required) or screen-lock is disabled. | Enable a device PIN/biometric. |
| `RP_ID_MISMATCH` at passkey creation | `assetlinks.json` is unreachable, returns wrong content-type, or contains the wrong SHA-256. | Verify `curl -I`, double-check the fingerprint. |
| `CLEARTEXT communication not permitted` | Step 4 not done and you're targeting localnet. | Add `src/debug/AndroidManifest.xml`. |

---

## What's next

- **[Set up Sigil identity](set-up-sigil-identity.md)** — actually use
  the SDK to bootstrap a passkey-derived sigil session in your app.
- **[Deploy and call a Compact contract](deploy-and-call-a-compact-contract.md)**
  — go end-to-end from a compiled `.compact` to an on-chain transaction.
