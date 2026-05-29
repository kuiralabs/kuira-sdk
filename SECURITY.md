# Security Policy

Kuira is a wallet and identity SDK for the Midnight blockchain on Android.
Because it touches seed material, signing keys, and on-chain authority, we
take security reports seriously. This document covers how to report
vulnerabilities, what's in scope, and — honestly — what isn't yet.

---

## Reporting a vulnerability

**Please report security issues privately.** Public GitHub issues are the
wrong venue for unpatched vulnerabilities.

### How to reach us

- **GitHub Security Advisories** (preferred):
  <https://github.com/nel349/kuira-android-wallet/security/advisories/new>
  — Private channel that lets you and the maintainer collaborate on a fix
  before disclosure. Encrypted on GitHub's side.
- **Email:** `norman.lopez.krypto@gmail.com`
  — For PGP-encrypted reports, the maintainer's signing key (used for
  Maven Central artifacts) doubles as the security contact key. Public key
  available on `keyserver.ubuntu.com` — see *Verifying releases* below for
  the fingerprint.

### What to include

- Affected version(s) — at minimum, the Maven coordinate
  (e.g. `io.github.kuiralabs:midnight-sdk:0.1.0-alpha01`).
- Steps to reproduce, or a proof-of-concept.
- Your assessment of impact and exploitability.
- Whether you want public credit, and how you'd like to be named.

### Response timeline

| Stage | Target |
|---|---|
| Acknowledgement of report | within **72 hours** |
| Initial triage + severity assessment | within **1 week** |
| Fix or mitigation plan | within **30 days** for high/critical |
| Coordinated disclosure | within **90 days** of initial report, or sooner if a fix has shipped |

If 90 days pass without a fix or a mutually agreed extension, you're free
to disclose publicly. We'd rather you do that than sit on an unfixed issue.

---

## Scope

### In scope

Vulnerabilities in code shipped under any of the published Maven artifacts:

```
io.github.kuiralabs:midnight-sdk
io.github.kuiralabs:dapp-ui
io.github.kuiralabs:wallet-seed
io.github.kuiralabs:wallet-runtime
io.github.kuiralabs:identity
io.github.kuiralabs:auth
io.github.kuiralabs:crypto
io.github.kuiralabs:network
io.github.kuiralabs:compact-engine
io.github.kuiralabs:indexer
io.github.kuiralabs:connector
io.github.kuiralabs:ledger
io.github.kuiralabs:designsystem
io.github.kuiralabs:testing
```

Particularly interested in issues affecting:

- **Key derivation and storage** — BIP-39 / BIP-32 / Schnorr / Bech32m
  implementations, Android Keystore integration, EncryptedSharedPreferences
  usage, seed wiping after use.
- **Passkey / Sigil identity** — PRF derivation, DID generation,
  authorization payload signing, Block Store backup encryption.
- **Transaction signing and proving** — anything that could produce an
  unintended on-chain effect.
- **Network surface** — indexer / node client (TLS pinning, replay,
  response validation).
- **Hilt graph misconfigurations** — anything that could silently route
  authority through the wrong domain or wrong key.

### Out of scope

Please report these to the appropriate upstream project, not to us:

- The Midnight protocol itself, the ledger, the node, the indexer, the
  Compact compiler — report to <https://github.com/midnightntwrk>.
- Android OS / Keystore / Credentials Manager vulnerabilities — report to
  Google's Android Security team.
- Issues only reproducible on rooted devices, emulators with debugging
  bridges, or with the user having installed malicious apps — see *Known
  limitations* below.
- The Kuira wallet app (not yet published) and example dApps in
  `examples/` (BBoard, Midnight Kicks) — these are reference
  implementations demonstrating SDK usage, not production products.

---

## Threat model

### What Kuira protects against

- **Cold device theft:** Seed material at rest is encrypted via Android
  Keystore (hardware-backed where available, falling back to TEE). A thief
  with the device but not the user's biometric / PIN cannot extract seeds.
- **Local file inspection:** Both the seed vault and the active-match /
  application-state store use `EncryptedSharedPreferences` with a
  Keystore-bound master key. Files in `/data/data/<pkg>/` are unreadable
  outside a Keystore-authenticated process.
- **Cloud backup interception:** The Block Store backup blob is
  **PRF-encrypted client-side** before Google Block Store touches it. A
  Google account compromise alone doesn't yield wallet access — the
  attacker also needs to forge a passkey assertion on a device the user
  controls.
- **Replay across dApps:** Each consumer dApp supplies its own
  `PasskeyConfig` (rpId + rpName); the SDK does not ship a maintainer
  default. Two dApps using the same SDK get cryptographically distinct
  passkeys and cannot impersonate each other.
- **Accidental seed exposure in logs:** Seed and PRF-derived material is
  never logged. Debug log lines that touch sensitive material are gated
  behind `BuildConfig.DEBUG` so R8 strips them from release builds.

### What Kuira does NOT protect against

We're explicit about this because hiding it isn't honest:

- **Compromised device** (rooted, malicious app with elevated privileges,
  unlocked while the attacker has physical access): once Keystore-bound
  keys are accessible to malicious code in the same process, the seed
  vault is recoverable. No app-level mitigation defeats a compromised
  trusted execution environment.
- **Malicious dApp in the same process:** dApps that integrate the SDK
  share the same Android process. There is no inter-dApp sandboxing
  beyond Android's normal process isolation. Vet the dApps you integrate.
- **User-side passkey loss:** If a user loses their Google Password
  Manager account *and* their Block Store backup is unrecoverable, funds
  are unrecoverable — sigil-derived wallets currently have no sovereign
  exit path. Recovery-phrase export (BIP-39 mnemonic display) is being
  built (`docs/projects/midnightWallet.md` references the wishlist item)
  but **not yet shipped in `0.1.0-alpha01`**. Until then, the recovery
  path rides Google's availability.
- **Session-cache theft after unlock:** Once a sigil session is unlocked,
  the decrypted seed lives in `MidnightSdkProvider`'s cached SDK instance.
  Subsequent value-bearing calls do not re-prompt for biometric.
  Auto-lock (idle timeout, background timeout, screen-lock observer) is
  on the roadmap (`docs/projects/midnightWallet.md` wishlist #14) but
  **not yet shipped**. A borrowed-while-unlocked device is a real risk.
- **BLS proving parameters supply chain:** The SDK fetches Midnight's
  protocol-level proving keys (BLS params + wallet / zswap / dust
  circuits) from `midnight-s3-fileshare-dev-eu-west-1`, a Midnight-team
  S3 bucket labeled "dev." A compromise of that bucket would let an
  attacker substitute proving keys. Documented in
  [`INTEGRATION.md`](INTEGRATION.md) § Known limitations.
  Per-contract proving keys are unaffected — each dApp hosts its own.
- **Network-level interception:** The SDK uses HTTPS for indexer / node
  RPC. We do not currently implement certificate pinning. A
  network-level adversary with a forged certificate (e.g. compromised CA)
  could observe or alter RPC traffic. Considered for post-`1.0`.
- **Side-channel attacks on Keystore / TEE:** Out of our threat model.
  We trust Android's threat model for the underlying hardware-backed key
  store. Issues at that layer should be reported to Android Security.
- **Alpha API stability:** `0.1.x-alpha*` releases may have breaking API
  or behavioral changes. The semver guarantees of a `1.0.0` line do not
  apply. Pin exact versions in production usage.

---

## Supported versions

We backport security fixes only to versions still in active development.
For an alpha SDK, that means: **the most recent published alpha line.**

| Version | Status |
|---|---|
| `0.1.x-alpha*` | ✅ Supported — security fixes land in the next alpha bump |
| Older alphas | ❌ Not backported. Upgrade. |

When `1.0.0` ships, this table will be revised to a proper LTS / minor
policy. Until then, **stay current** — there's no minor-version stability
to rely on during alpha.

---

## Verifying releases

Every Maven Central artifact is signed with the maintainer's PGP key. To
verify a downloaded artifact:

```bash
# Fetch the public key
gpg --keyserver keyserver.ubuntu.com --recv-keys <FILL IN: 40-char fingerprint>

# Download the artifact and its signature
ARTIFACT=midnight-sdk-0.1.0-alpha01.aar
curl -O https://repo1.maven.org/maven2/io/github/kuiralabs/midnight-sdk/0.1.0-alpha01/$ARTIFACT
curl -O https://repo1.maven.org/maven2/io/github/kuiralabs/midnight-sdk/0.1.0-alpha01/$ARTIFACT.asc

# Verify
gpg --verify $ARTIFACT.asc $ARTIFACT
```

A successful verification output ends with
`Good signature from "Norman Lopez <…>"`. Any other outcome means the
artifact does **not** correspond to a release the maintainer published —
do not use it.

The fingerprint is also published in this file and in the project's
GitHub repository description, so an attacker would have to compromise
both this repository and the keyserver to substitute a key undetected.

---

## Disclosure policy

We follow **coordinated disclosure**:

- Report → maintainer acknowledges and triages.
- Maintainer prepares a fix and a draft advisory.
- A target disclosure date is set (within the 90-day window, or sooner if
  the fix is ready).
- On the disclosure date: a patched release ships, the advisory is
  published via GitHub Security Advisories, the CHANGELOG names the
  affected versions, and — if the reporter wishes — credit is given by
  name.

If the issue is being actively exploited in the wild, the timeline
compresses. Tell us if you have reason to believe that's the case.

---

## Acknowledgments

We thank the following researchers for responsibly disclosing
vulnerabilities. (Hall of fame opens with the first report.)

| Researcher | Issue | Disclosed in |
|---|---|---|
| *(none yet)* | | |

---

## Out-of-scope but appreciated

Even if a report doesn't qualify as a security vulnerability (typos in
crypto code, missing input validation that isn't exploitable, hardening
suggestions), open a regular GitHub issue or PR — we still want to know.

---

## See also

- [`INTEGRATION.md`](INTEGRATION.md) — known operational limitations for
  consumers (BLS-params dependency, debug-cleartext requirement, etc.).
- [`README.md`](README.md) — install instructions and SDK overview.
