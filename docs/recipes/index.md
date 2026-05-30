---
title: Cookbook
hide:
  - toc
tags:
  - cookbook
---

# Cookbook

Hand-authored recipes that take you from "I have an Android project" to
"I shipped a Midnight dApp." Each recipe is a step-by-step runbook with
a verification gate after every meaningful step — written so a human or
an LLM-driven agent can follow it.

Click into any recipe and you'll see a **:material-content-copy: Copy
prompt for your agent** button. The button copies a templated prompt
that points your agent (Claude Code, Cursor, Codex, etc.) at the raw
markdown context bundle for that recipe. Paste the prompt into your
agent and let it execute the recipe in your project.

!!! tip "Why agent-friendly?"
    Each recipe's raw `.md` source is the same content the rendered page
    shows — the agent doesn't need to scrape HTML or guess at structure.
    A site-root [`/llms.txt`](../llms.txt) lists every recipe so any
    LLM-aware tool that supports the [llms.txt convention](https://llmstxt.org)
    auto-discovers the full cookbook in one fetch.

---

## Recipes

<div class="grid cards" markdown>

-   :material-package-down: **[Add Kuira to an Android project](add-kuira-to-an-android-project.md)**

    ---

    Install the SDK, set your passkey rpId, configure the Hilt module.
    ~10 minutes.

-   :material-key-chain: **[Bind your app to a passkey domain](bind-your-app-to-a-passkey-domain.md)**

    ---

    Get the debug signing fingerprint, write a single-fingerprint
    `assetlinks.json`, host it on your `rpId`, verify Forge works on
    device. Development-only — release-signing is its own future recipe.

-   :material-fingerprint: **[Set up Sigil identity](set-up-sigil-identity.md)**

    ---

    Bootstrap a passkey-derived sigil session: DID + wallet seed from
    a single biometric. ~15 minutes.

-   :material-file-document-multiple: **[Deploy and call a Compact contract](deploy-and-call-a-compact-contract.md)**

    ---

    Compile a `.compact` contract, sync artifacts into your assets,
    deploy via the SDK, and call a circuit. ~30 minutes.

</div>

---

## Recipe conventions

Every recipe page follows the same shape, which makes them predictable
both for humans and agents:

| Section | Purpose |
|---|---|
| **Frontmatter** | Tags, prerequisites, raw-bundle URL — machine-readable metadata. |
| **Outcome** | One sentence: what the developer has at the end. |
| **Prerequisites** | What must be true before starting. |
| **Steps** | Numbered, each with code + a *Verify* gate. |
| **Troubleshooting** | Common failures + their fixes. |
| **Agent prompt** | The :material-content-copy: button at the top of every recipe. |

When the cookbook framework expands in `alpha03+` to include session
state, live-step gates, and CLI-based verifiers, the existing recipes
keep working — the new machinery layers on top without rewriting the
content.
