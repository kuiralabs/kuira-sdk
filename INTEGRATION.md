# Kuira SDK — Integration Guide

**Alpha — `0.1.0-alpha01`**

Build a Midnight zero-knowledge dApp on Android by adding **one dependency**, declaring your **own passkey domain**, and hosting **one tiny JSON file** on it. That's the whole on-ramp.

This guide walks the recipe end-to-end. Two reference dApps — **Midnight
Kicks** (a PvP penalty shoot-out demonstrating commit-reveal + ZK proving)
and **BBoard** (a simpler community-board app) — consume the SDK exactly
as described below. Their sources are currently in a private repository
while the SDK is published here; the code patterns below are extracted
from those apps, so prefer the description here as the canonical guide.

---

## What the SDK gives you

Adding the one line below brings:

- **Sigil identity** — a passkey-derived DID (`did:key` + Ed25519). One biometric, no recovery phrase, no maintainer dependency. Backed up to Google Block Store, encrypted with a key you can't read.
- **Wallet** — Midnight HD wallet (unshielded + shielded NIGHT, DUST gas), with live balance, send, receive, and a drop-in Compose **wallet panel** if you want it.
- **Contract surface** — deploy / call / read state on any `.compact` contract; ZK proof generation runs on-device (no proof-server hop required).
- **Indexer + chain client** — block / state / event subscription, backed by Midnight's official indexer.
- **App-state cloud backup** — your dApp's per-user data rides the sigil's Block Store backup automatically. The backup blob is PRF-encrypted client-side before Google's Block Store touches it.

You can use it headless or pull in the panel UI — see the dependency choice below.

---

## 1. Prerequisites

| | |
|---|---|
| Android Gradle Plugin | **≥ 8.13** (matches the SDK build) |
| Kotlin | **≥ 2.3** |
| `compileSdk` | 35+ |
| `minSdk` | **30** — required (`core:auth` uses keystore APIs added in API 30: biometric ⊕ device-credential) |
| Hilt + KSP | Required — the SDK is Hilt-first; your app applies both |
| A web domain you control | Required for the passkey relying party (GitHub Pages is fine for dev) |
| Your app's signing-cert SHA-256 | Needed for `assetlinks.json` (step 4) |

---

## 2. Add the dependency — *one line*

Maven Central is already in every Android project's defaults, so there's no repo
config to add. In `app/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")          // only if your own UI is Compose
    id("com.google.devtools.ksp")
    id("com.google.dagger.hilt.android")
}

android {
    namespace = "com.example.mydapp"
    compileSdk = 35
    defaultConfig {
        applicationId = "com.example.mydapp"
        minSdk = 30
        targetSdk = 35
    }
}

dependencies {
    // ── Pick ONE Kuira entry ──
    //
    // With the wallet + sigil PANEL (Compose). Use this for Kicks / BBoard-style
    // apps that want the wallet UI dropped in.
    implementation("io.github.kuiralabs:dapp-ui:0.1.0-alpha01")
    //
    // OR — headless (no Compose pulled by the SDK). For dApps building their
    // own UI top-to-bottom.
    // implementation("io.github.kuiralabs:midnight-sdk:0.1.0-alpha01")

    // Hilt processor — required, the SDK is Hilt-first
    ksp("com.google.dagger:hilt-android-compiler:2.58")

    // your own app code, your own Compose deps, etc.
}
```

That single Kuira line brings the whole consumer surface — passkey, sigil,
wallet, ZK proving, contract deploy/call, indexer, panel UI (if you picked
`dapp-ui`) — onto your compile classpath transitively. No per-module
redeclaration.

---

## 3. Declare YOUR passkey domain

The SDK provides **no default `PasskeyConfig`** — that's deliberate. A passkey
is bound to a domain, and that domain must be **yours**. If the SDK baked in a
default, every consumer would silently route through the maintainer's domain
and PRF would only work after the maintainer added them to a maintainer-hosted
`assetlinks.json` — i.e. the "open" SDK would actually be permissioned.

Add a tiny Hilt module (anywhere in your `di/`):

```kotlin
package com.example.mydapp.di

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
    fun providePasskeyConfig() = PasskeyConfig(
        rpId   = "mydapp.example.com",   // YOUR domain (matches assetlinks.json below)
        rpName = "My dApp",              // shown in the biometric prompt
    )
}
```

> If you forget this, you'll get a clear Dagger missing-binding error at build
> time. That's intentional — it forces you to declare your domain before
> shipping.

---

## 4. Host `assetlinks.json` on YOUR domain

Android's `CredentialManager` checks Digital Asset Links to verify *your* app
may use passkeys for *your* domain. Place this file at:

```
https://<your-domain>/.well-known/assetlinks.json
```

```json
[{
  "relation": [
    "delegate_permission/common.get_login_creds",
    "delegate_permission/common.handle_all_urls"
  ],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.mydapp",
    "sha256_cert_fingerprints": ["AB:CD:EF:01:23:45:67:89:..."]
  }
}]
```

**Get your cert SHA-256**:
```bash
keytool -list -v -keystore <your-keystore> -alias <your-alias> | grep SHA256
```
For debug builds, the default keystore is `~/.android/debug.keystore` (alias
`androiddebugkey`, password `android`).

> Without this file (or with a wrong package name / SHA-256), the biometric
> prompt fails and PRF can't derive your sigil. This is a one-time host;
> updates are needed only when you change signing certs.

---

## 5. *(Localnet dev only)* Allow cleartext to `10.0.2.2`

Production hits HTTPS. For local testing against a localnet node + indexer at
`http://10.0.2.2:9944` / `:8088`, declare cleartext **in a debug-only manifest**
so release builds stay HTTPS-clean:

```xml
<!-- app/src/debug/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:usesCleartextTraffic="true" />
</manifest>
```

> Why debug-only: previously this was inherited transitively from a `core:`
> module's debug AAR; once the SDK started publishing release variants for the
> alpha that allowance no longer reaches consumers. So each app declares its
> own — release stays cleartext-free.

---

## 6. Minimal "Hello World" — deploy + call

Assuming you picked `dapp-ui` (panel) and your app's main activity is a
Hilt'd `FragmentActivity`:

```kotlin
@HiltAndroidApp
class MyApp : Application()

@AndroidEntryPoint
class MainActivity : FragmentActivity() {
    @Inject lateinit var sdkProvider: MidnightSdkProvider

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            // Drop-in wallet + sigil panel. Renders status, exposes Backup
            // / Restore / Send / Receive without any further plumbing.
            WalletPanel(activity = this@MainActivity)
        }
    }

    // Your dApp logic, e.g. deploy + call a contract:
    suspend fun deployAndCall() {
        val sdk = sdkProvider.ensureSdk(
            activity = this,
            config = WalletConfig(network = MidnightNetwork.UNDEPLOYED),
        )

        val contract = MidnightContract.create {
            // your compiled `.compact` JS + proving keys, dropped in assets/
            contractJs = "my-contract.js"
            wallet = sdk.wallet
            coinPublicKey = sdk.coinPublicKey
        }
        val address = contract.deploy()
        val result = contract.call("myCircuit", args = …)
    }
}
```

For a real, multi-step contract (commit / reveal / ZK proofs / sudden
death), the reference Midnight Kicks dApp implements a ~600-line
state-machine that exercises every SDK surface — protocol orchestration,
witness packing, indexer-state polling, transaction retry, force-resync,
deadline computation. The patterns described throughout this guide come
from that code path; once the reference repository opens to the public,
this section will link directly to it.

---

## 7. Going headless (no panel)

If you don't want the Compose wallet panel, swap the dep:

```kotlin
implementation("io.github.kuiralabs:midnight-sdk:0.1.0-alpha01")  // no dapp-ui
```

You still own bootstrap + sigil forging — `MidnightSdkProvider.ensureSdk(...)`
throws `SigilRequiredException` until a sigil exists. Forge it through
`PasskeyManager` directly, or use the `SigilSession` helper.

`midnight-sdk` does not pull Compose — the headless entry stays small.

---

## Common pitfalls

| Symptom | Fix |
|---|---|
| Dagger: *"PasskeyConfig cannot be provided without an @Provides-annotated method"* | Step 3 — declare your own `PasskeyConfig` module. |
| Runtime: *"CLEARTEXT communication to 10.0.2.2 not permitted by network security policy"* | Step 5 — add the debug manifest. |
| Biometric prompt fails / PRF returns null | Step 4 — `assetlinks.json` missing, wrong `package_name`, or wrong cert SHA-256. |
| App balance stays at 0 after an airdrop | The wallet's background subscription is live; check `adb logcat` for indexer connectivity. On localnet, state is ephemeral — restarting the localnet wipes funds. |
| `IllegalArgumentException: Could not find io.github.kuiralabs:…` | Check the alpha version is current; the SDK is `0.1.0-alpha01` at the time of writing. |

---

## Known limitations (alpha)

The alpha ships with one known dependency you should be aware of. It isn't a
blocker for building, but it shapes what "alpha" means in practice and will
evolve as Midnight matures.

### Proving infrastructure downloads from a Midnight *dev* URL

On first launch the SDK fetches Midnight's **protocol-level** proving assets
— the universal BLS parameters and the wallet's shielded-spend / dust circuit
keys — from
`https://midnight-s3-fileshare-dev-eu-west-1.s3.eu-west-1.amazonaws.com`.
This is the same bucket Midnight's own tooling uses; it's **Midnight's
bucket, not the SDK maintainer's**, and the URL is labeled `-dev-`. There is
no production SLA on it.

**What this means in practice:**

- If Midnight retires, renames, or restricts that bucket, every alpha dApp
  briefly can't generate proofs until the SDK is updated.
- The right party to publish a production URL is Midnight (the asset owner),
  not the SDK maintainer. So the alpha *documents* this dependency rather
  than self-hosting.

**Your own `.compact` contract's proving keys are NOT affected by this.** Those
download from a URL **you** supply when you call
`ProvingKeyManager.downloadCircuitKeys(baseUrl = …, …)` — Kicks hosts Kicks's
keys, BBoard hosts BBoard's. You always own your contract's keys. This
limitation is only about the *protocol-wide* keys that every Midnight dApp
shares.

**How it will evolve.** When Midnight publishes a production URL — or, if
that's not in their immediate roadmap, when the SDK mirrors these files under
a `kuira-labs`-controlled URL with a version pin (`ProvingKeyManager.CURRENT_VERSION`
tracks the protocol version, currently 9) — the SDK swaps the constant and
re-publishes. The expected migration cost for a consumer is a single SDK
version bump; no app-side code change.

---

## See also

- [`README.md`](README.md) — install instructions and the full module list.
- [`SECURITY.md`](SECURITY.md) — threat model, vulnerability reporting, signature verification.
- [Maven Central — `io.github.kuiralabs`](https://central.sonatype.com/namespace/io.github.kuiralabs) — every published artifact (binary AAR, sources jar, javadoc jar, POM, PGP signature).
