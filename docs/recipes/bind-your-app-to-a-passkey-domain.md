---
title: Bind your app to a passkey domain
tags:
  - identity
  - passkey
  - signing
  - assetlinks
prerequisites:
  - Kuira added to your project (see "Add Kuira to an Android project")
  - A domain you control and can host static files on over HTTPS
agent_bundle: https://raw.githubusercontent.com/kuiralabs/kuira-sdk-android/main/docs/recipes/bind-your-app-to-a-passkey-domain.md
---

# Bind your app to a passkey domain

**Outcome:** your debug build's signing fingerprint is listed in an
`assetlinks.json` served at your `rpId`, and **Forge** succeeds on
device — no `RP_ID_MISMATCH`, no `PRF authentication failed`, no
silent prompt dismissal.

<div data-copy-prompt="https://raw.githubusercontent.com/kuiralabs/kuira-sdk-android/main/docs/recipes/bind-your-app-to-a-passkey-domain.md"
     data-task="Bind an Android app's debug build to a passkey rpId — get the debug SHA-256 from signingReport, write a single-fingerprint assetlinks.json, host at /.well-known/assetlinks.json, verify with curl + Forge on device. Production release-keystore setup is out of scope."></div>

This is the **development-only** path. You'll be running debug builds
on your emulator or device — Android Studio's auto-generated debug
keystore signs them for you. No keystore generation, no
`keystore.properties`, no CI signing setup. ~5 minutes end to end.

---

## Step 1 — Get the debug fingerprint

```bash
./gradlew :app:signingReport
```

Find the `Variant: debug` block. Copy its `SHA-256:` line (keep the
colons):

```
Variant: debug
…
SHA-256: AB:CD:EF:12:34:…
```

---

## Step 2 — Write `assetlinks.json`

```json title="assetlinks.json"
[
  {
    "relation": [
      "delegate_permission/common.handle_all_urls",
      "delegate_permission/common.get_login_creds"
    ],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.myapp",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:…"
      ]
    }
  }
]
```

Two fields:

- `package_name` — your app's `applicationId` from `app/build.gradle.kts`.
- `sha256_cert_fingerprints[0]` — the SHA-256 from Step 1.

Both relations are required: `get_login_creds` makes passkey ceremonies
work; `handle_all_urls` lets the same file double as App Links binding
if you add deep links later.

---

## Step 3 — Host it on your `rpId` domain

Path: `https://<rpId>/.well-known/assetlinks.json`. Must be served over
HTTPS, with `Content-Type: application/json`, with no redirect and no
auth wall.

**Simplest hosting — GitHub Pages.** Put `assetlinks.json` at
`<your-pages-repo>/.well-known/assetlinks.json` on the served branch
(`main` or `gh-pages`). GitHub Pages serves `.json` as
`application/json` by default — no extra config. If you use a custom
domain, that custom domain is your `rpId`.

Other hosts work too — Vercel, Cloudflare Pages, your own nginx — as
long as you can serve `assetlinks.json` at the right path with the right
content type and no redirects.

---

## Step 4 — Verify

```bash
curl -I https://<your-rpId>/.well-known/assetlinks.json
# Expected:
# HTTP/2 200
# content-type: application/json
```

Then on device:

```bash
./gradlew :app:installDebug
```

Launch the app and tap **Forge**. You should get a biometric prompt →
success → Sigil panel transitions to `Forged`.

---

## Troubleshooting

If Forge fails, almost every symptom maps to the same root cause:

| Symptom | First thing to check |
|---|---|
| `RP_ID_MISMATCH` | `curl -I` returns 200 with `content-type: application/json`? |
| `PRF authentication failed` | Same — the passkey API never reaches PRF if assetlinks doesn't validate |
| Biometric prompt appears, dismisses silently | Same |
| Worked before, broken after `adb install -r` | `adb uninstall <pkg>` then re-install — credential-manager state goes stale |

Common `curl -I` red flags:

- `301` / `302` — passkey API doesn't follow redirects; fix the redirect
- `403` / `401` — allow `/.well-known/*` through your auth
- `content-type: text/html` — force `application/json` via your hosting config
- `404` — file not deployed, or your `rpId` doesn't match the hostname you `curl`ed

If the `curl` looks clean and Forge still fails, paste your hosting URL
+ package name + fingerprint into Google's
[Statement List Tester](https://developers.google.com/digital-asset-links/tools/generator)
— it reports any mismatch with a precise error string.

---

## When you ship

The setup above covers development with debug builds. Shipping to the
Play Store or distributing a signed release build to testers needs a
release keystore, `keystore.properties` wiring, and the release
fingerprint added to the same `assetlinks.json`. A dedicated recipe for
that is coming; for now,
[`./gradlew :app:signingReport`](https://developer.android.com/build/building-cmdline#signing_report)
+ [the AGP signing config docs](https://developer.android.com/build/build-variants#signing)
cover the missing pieces.

---

## What's next

- **[Set up Sigil identity](set-up-sigil-identity.md)** — now that the
  passkey domain is bound, wire `SigilStatusPanel` and forge your
  first sigil.
