# Secrets Plan — Dual Anthropic Auth + BOB_INSTALL_TOKEN

This document defines the secrets required to run the `bob-diff-review` GitHub
Action, their storage scope, minimum permission scopes, rotation policy, and the
injection pattern used inside the action.

The Anthropic credential surface is a **dual-auth contract**: the headless
`claude` subprocess accepts **either** an OAuth token **or** an API key, with
deterministic **OAuth-wins** precedence. This exists specifically to prevent a
stray `ANTHROPIC_API_KEY` from silently shadowing the recommended OAuth path and
exhausting pay-per-use credits.

---

## Secrets Overview

| Secret name             | Storage scope | Type                          | Required               | Injected as (env)         |
|-------------------------|---------------|-------------------------------|------------------------|---------------------------|
| `ANTHROPIC_OAUTH_TOKEN` | Org-level     | Anthropic OAuth token         | Optional (primary)\*   | `CLAUDE_CODE_OAUTH_TOKEN` |
| `ANTHROPIC_API_KEY`     | Org-level     | Anthropic API key (metered)   | Optional (fallback)\*  | `ANTHROPIC_API_KEY`       |
| `BOB_INSTALL_TOKEN`     | Org-level     | GitHub App installation token | Required               | `NODE_AUTH_TOKEN`         |

\* **At least one** of `ANTHROPIC_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` must be
provided. Neither is individually `required: true`. If **neither** is supplied
the action fails fast with a clear error naming both inputs and recommending the
OAuth path.

All three secrets are stored at the **organization level** in GitHub
(`Settings → Secrets and variables → Actions → Organization secrets`) and made
available to repositories in the `bobnetsec` organization that use the
`bob-diff-review` action. They are not duplicated at the repo level; individual
repos inherit them from the org via `secrets: inherit`.

---

## Anthropic dual-auth contract

### Why two credentials

- **`ANTHROPIC_OAUTH_TOKEN` — primary / recommended.** A long-lived token minted
  with `claude setup-token`. It bills against an existing Claude subscription
  rather than per-token API usage, so it is the preferred credential for
  routine, high-volume diff review.
- **`ANTHROPIC_API_KEY` — fallback.** A pay-per-use Anthropic API key. Useful
  when no OAuth token is available (e.g. a metered service account), but it can
  exhaust credits quickly under load.

### Env-var mapping (the names the `claude` CLI reads)

| Action input            | Source secret           | Child-process env var      |
|-------------------------|-------------------------|----------------------------|
| `anthropic-oauth-token` | `ANTHROPIC_OAUTH_TOKEN` | `CLAUDE_CODE_OAUTH_TOKEN`  |
| `anthropic-api-key`     | `ANTHROPIC_API_KEY`     | `ANTHROPIC_API_KEY`        |
| `anthropic-model`       | (input / `ANTHROPIC_MODEL`) | `ANTHROPIC_MODEL` (unchanged passthrough) |

The OAuth token is **not** read by the CLI under its secret name — it must be
mapped to `CLAUDE_CODE_OAUTH_TOKEN`, the variable the `claude` CLI actually
inspects.

### Precedence — OAuth wins, single injection (critical)

In the runner `childEnv` that spawns `claude`:

1. **Strip first.** Any inherited `CLAUDE_CODE_OAUTH_TOKEN` and
   `ANTHROPIC_API_KEY` are deleted from the child env before either is set, so a
   stale value in the runner's ambient environment cannot leak through.
2. **OAuth present →** set `CLAUDE_CODE_OAUTH_TOKEN` and **do not set**
   `ANTHROPIC_API_KEY` at all. The API key is deliberately omitted so it cannot
   silently shadow OAuth and exhaust pay-per-use credits — the exact bug this
   contract fixes.
3. **OAuth absent, API key present →** set `ANTHROPIC_API_KEY` and do not set the
   OAuth var.
4. **Exactly one** credential is injected, never both.
5. **Neither present →** fail fast with an error naming both
   `anthropic-oauth-token` and `anthropic-api-key`, recommending OAuth.

The `ANTHROPIC_MODEL` passthrough is preserved unchanged and is independent of
which credential is chosen.

This diverges intentionally from the `@brutalist` orchestrator, which forwards
both credentials because its brain + inner-critic topology spawns multiple
agents. Our single-invocation runner needs deterministic single-injection, not
forward-both.

### Storage and scope

- **Scope:** Organization-level secrets (`bobnetsec` org).
- **Propagation:** Available to workflows triggered by internal (non-forked)
  PRs only. Do **not** propagate either Anthropic credential to forked-PR
  workflows; treat any `pull_request` event from a fork as untrusted.

### Minimum permissions

Neither an Anthropic OAuth token nor an API key carries a fine-grained
permission model; each grants access up to the associated account's quota and
tier. Both should belong to a dedicated service account (not a personal account)
to avoid entanglement with human billing.

### Injection into the action

The calling workflow passes both as action inputs (one may be empty):

```yaml
- uses: bobnetsec/bob-workflows/packages/bob-diff-review@v1
  with:
    github-token:          ${{ secrets.GITHUB_TOKEN }}
    anthropic-oauth-token: ${{ secrets.ANTHROPIC_OAUTH_TOKEN }}  # recommended
    anthropic-api-key:     ${{ secrets.ANTHROPIC_API_KEY }}      # optional fallback
    bob-install-token:     ${{ secrets.BOB_INSTALL_TOKEN }}
```

Inside the runner, the chosen credential is mapped onto the `claude` child
process env (never via a `run:` echo):

```text
oauth token present  -> childEnv.CLAUDE_CODE_OAUTH_TOKEN = <token>   (ANTHROPIC_API_KEY omitted)
only api key present -> childEnv.ANTHROPIC_API_KEY        = <key>     (CLAUDE_CODE_OAUTH_TOKEN omitted)
neither present      -> fail fast
```

### Rotation policy

**`ANTHROPIC_OAUTH_TOKEN` (primary):**

- **Frequency:** Re-mint every 90 days or immediately on any suspected exposure.
- **Procedure:**
  1. Run `claude setup-token` under the service account to mint a new token.
  2. Update the `ANTHROPIC_OAUTH_TOKEN` org secret in GitHub.
  3. Revoke the previous token from the Claude account session list.
  4. Verify the next scheduled workflow run succeeds before closing the ticket.

**`ANTHROPIC_API_KEY` (fallback):**

- **Frequency:** Rotate every 90 days or immediately on suspected exposure.
- **Procedure:**
  1. Generate a new key in the Anthropic console under the service account.
  2. Update the `ANTHROPIC_API_KEY` org secret in GitHub.
  3. Revoke the old key in the Anthropic console.
  4. Verify the next scheduled workflow run succeeds.
- **Breach response:** Revoke the old key first, then rotate; the old key is
  invalid the instant it is revoked, so there is no window in which both are
  live.

### No-log guarantee

Neither credential value is **ever** echoed in a `run:` step; both flow only via
the child-process env. Any health-check or diagnostic step that needs to confirm
a credential is _present_ (not its value) must use a presence check:

```bash
if [ -n "$CLAUDE_CODE_OAUTH_TOKEN" ]; then echo "OAuth token present"; else echo "OAuth token REDACTED or missing"; fi
if [ -n "$ANTHROPIC_API_KEY" ];     then echo "API key present";     else echo "API key REDACTED or missing"; fi
```

GitHub Actions automatically masks secrets in log output; the pattern above adds
a defense-in-depth layer so the check works even if masking is bypassed by an
indirect expansion.

---

## BOB_INSTALL_TOKEN

### Purpose

Authenticates `npm install` / `npm ci` against the GitHub Packages registry
(`npm.pkg.github.com`) so the runner can download the private `@bobnetsec/*`
packages during the Action run, and checks out the private
`bobnetsec/hacker-bob` repository on cache-miss runs.

### Preferred token type: GitHub App installation token

A **GitHub App installation token** is strongly preferred over a classic PAT:

- Tokens expire after one hour and are refreshed per-run; there is no long-lived
  credential to rotate manually.
- Permissions are scoped to the App installation, not to a human account.
- The token cannot be used interactively, reducing blast radius if intercepted.

If a fine-grained PAT must be used as a bootstrap fallback before the App is
installed:

- Scope it to the `bobnetsec` organization.
- Grant **only** `read:packages` and `contents:read`.
- Set a 90-day expiry and rotate before expiry.

### Storage, scope, and minimum permissions

- **Scope:** Organization-level secret (`bobnetsec` org).
- **Propagation:** Same gate as the Anthropic credentials — internal PRs only;
  never exposed to fork workflows.

| Permission       | Level    | Reason                                             |
|------------------|----------|----------------------------------------------------|
| `read:packages`  | Required | Allows `npm install` from `npm.pkg.github.com`     |
| `contents:read`  | Required | Allows checking out `bobnetsec/hacker-bob` on miss |

No write, admin, or pull-request permissions are needed.

### Injection into the action

```yaml
env:
  NODE_AUTH_TOKEN: ${{ inputs.bob-install-token }}
```

The `actions/setup-node` step must be configured with
`registry-url: https://npm.pkg.github.com` so Node sets `NODE_AUTH_TOKEN`
automatically for `npm ci` calls.

### Rotation policy

- **App installation token (preferred):** No manual rotation. Tokens are issued
  per-run by the App and expire after one hour. Monitor the App installation in
  `bobnetsec` org settings to ensure it is not uninstalled or downgraded.
- **Fine-grained PAT (fallback):** Rotate every 90 days or on suspected
  exposure: create a new PAT with `read:packages` + `contents:read`, update the
  org secret, verify `npm ci` completes without 401, then revoke the old PAT.

### No-log guarantee

The token value is never echoed. Presence checks use:

```bash
if [ -n "$NODE_AUTH_TOKEN" ]; then echo "BOB_INSTALL_TOKEN present"; else echo "BOB_INSTALL_TOKEN REDACTED or missing"; fi
```

---

## Reusable + caller workflow contract

The reusable workflow (`bob-review.yml`) declares all three secrets as optional
on `workflow_call` and forwards them to the action's `with:` block:

```yaml
on:
  workflow_call:
    secrets:
      ANTHROPIC_OAUTH_TOKEN:
        required: false   # recommended; at least one Anthropic credential required
      ANTHROPIC_API_KEY:
        required: false   # fallback; OAuth wins when both are set
      BOB_INSTALL_TOKEN:
        required: false

jobs:
  bob-diff-review:
    steps:
      - uses: bobnetsec/bob-workflows/packages/bob-diff-review@main
        env:
          ANTHROPIC_MODEL: ${{ inputs.anthropic-model }}   # passthrough preserved
        with:
          github-token:          ${{ secrets.GITHUB_TOKEN }}
          anthropic-oauth-token: ${{ secrets.ANTHROPIC_OAUTH_TOKEN }}
          anthropic-api-key:     ${{ secrets.ANTHROPIC_API_KEY }}
          bob-install-token:     ${{ secrets.BOB_INSTALL_TOKEN }}
```

The caller workflow passes both Anthropic secrets through (typically via
`secrets: inherit`), supplying at least one. The at-least-one check is enforced
inside the action, not by `required: true` on either input.

---

## PR / fork gating

All three secrets must be gated so they are never available in fork-triggered
workflows. Add the following guard on every job that reads them:

```yaml
if: >-
  github.event.pull_request.head.repo.full_name == github.repository
```

Forked PRs then skip the secret-consuming job entirely.

---

## Summary checklist

- [ ] `ANTHROPIC_OAUTH_TOKEN` created as org-level secret in `bobnetsec`, minted
      via `claude setup-token` under a service account (recommended primary).
- [ ] `ANTHROPIC_API_KEY` created as org-level secret (optional pay-per-use
      fallback).
- [ ] At least one Anthropic credential is configured; the action fails fast
      with a clear error if neither is present.
- [ ] OAuth-wins single-injection verified: when both are set, only
      `CLAUDE_CODE_OAUTH_TOKEN` reaches the `claude` subprocess and
      `ANTHROPIC_API_KEY` is omitted.
- [ ] Env mapping verified: `ANTHROPIC_OAUTH_TOKEN -> CLAUDE_CODE_OAUTH_TOKEN`,
      `ANTHROPIC_API_KEY -> ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` passthrough
      unchanged.
- [ ] `BOB_INSTALL_TOKEN` created as org-level secret (App token or 90-day
      fine-grained PAT with `read:packages` + `contents:read`).
- [ ] Workflows that consume the secrets gate on
      `head.repo.full_name == github.repository`.
- [ ] No `run:` step echoes any credential value; presence checks use the
      `if [ -n "$SECRET" ]; then echo present; fi` pattern.
- [ ] Rotation reminders set in the team calendar for PAT-based and OAuth tokens.
