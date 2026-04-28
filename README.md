# Release-bomb demo — GITHUB_TOKEN dormant-bomb chain


This public repo demonstrates that a workflow run authenticated only by the
auto-issued `GITHUB_TOKEN` (no PAT, no GitHub App) can plant a release that
DETONATES when any non-bot actor (typically a human maintainer) interacts
with it.

## Setup

The planter workflow at `.github/workflows/planter.yml` is dispatch-triggered.
Its only credential is `${{ secrets.GITHUB_TOKEN }}` with `contents: write`.

When run, the planter:

1. Builds an orphan commit via Git Data API. The orphan tree contains a
   smuggled `.github/workflows/<random>.yml` planted via tree-splicing.
2. Creates a draft release with `target_commitish=main` (gate clean).
3. PATCHes `target_commitish` to the orphan SHA (PATCH bypasses the
   2023 release-creation workflow-scope check).
4. PATCHes `draft=false` to publish.

GitHub's loop-prevention rule suppresses the immediate workflow dispatch
because the publishing actor is `GITHUB_TOKEN`. The release sits dormant
in the Releases tab as `untagged-XXXXXXXX`.

## Detonation

The bomb fires when ANY non-bot actor interacts with the release:

- Maintainer renames the release via the web UI
- Maintainer uploads or deletes an asset
- Maintainer toggles draft / prerelease
- A release-management bot (release-please etc.) using a PAT processes it

Each interaction fires `release: edited`. Loop prevention does NOT apply
because the actor is not GITHUB_TOKEN. The planted workflow at the orphan
SHA executes, with `permissions: id-token: write` (or whatever the orphan
declared) and a fresh GITHUB_TOKEN.

## Threat model

A malicious dependency in any CI step that has `permissions: contents: write`
declared (typical for: release-please, semantic-release, automerge,
auto-tag bots, deployment workflows that commit back, etc.) reads the
runner's GITHUB_TOKEN and runs the planter chain.

The attacker doesn't need persistent access. Plant once, wait for any
maintainer or bot interaction.

## Reproduction

1. Trigger `Run workflow` on the **planter** workflow above.
2. Observe: planter run completes successfully.
3. Check Releases tab: a new `untagged-XXX` release appears at an orphan SHA.
4. Verify: zero new Actions runs from the planted workflow.
5. Click "Edit" on the dormant release, change anything, save.
6. Observe: planted workflow runs immediately with sentinel in log.
