# Contributing — branching & merge rules

This product is built with the **Enate SDLC Factory**. Work reaches `main` through one of the
lanes below. Direct commits to `main` are blocked in every lane — `main` only ever moves through
a pull request (and review, where required).

> **Required checks.** No CI runs yet. When CI is wired it is an Azure DevOps pipeline that posts a
> status check back to the PR under the context name **`ci`**; that check is then added to the
> branch ruleset as required. Until then the ruleset enforces the structural rules — PR-only,
> linear history, no force-push, no branch deletion, squash-merge only — without a required check.

## 1. Human-in-the-loop (HITL) work

All human work — planning docs, fixes, skill changes, anything — happens on a **branch**, opened
as a **pull request**. You cannot commit to `main` directly. **Branching follows the same rule as
AFK delivery (§2), regardless of who drives the Story:** work under a Feature branches off that
Feature's `feature/<feature-id>-<slug>` branch and merges back into it — a HITL Story never lands
straight on `main` mid-Feature, since the topology has to be identical either way for `main` to
never hold a half-delivered Feature. Work with **no** parent Feature (a Bug, a standalone doc
change) branches directly off `main` and merges to `main`, exactly like parentless AFK work.

**Close out the board yourself.** The orchestrator only writes native `System.State` for AFK-driven
transitions — it never touches a `HITL`-tagged Story, a Bug fixed by hand, or a wholly-HITL
Feature. When your PR merges and the work is done, set the work item's native `System.State` to
`Resolved`/`Closed` yourself before moving on; nothing else will.

## 2. AFK — the orchestrator delivering work

When the orchestrator delivers work autonomously it owns the whole branch lifecycle; the
agent/skill session works only inside the checkout the orchestrator prepared and performs **no
branch operations**. What the branching looks like depends on whether the work has a parent
Feature.

**Feature-parented Stories — the feature-branch integration buffer.** So `main` never holds a
half-delivered Feature, the orchestrator cuts **one `feature/<feature-id>-<slug>` branch per
Feature** from `main`. Each Story branch (`story/<issue#>-<slug>`) is cut from its **parent
Feature's branch** (not `main`) and **squash-merged back into that feature branch** on `Approved` —
so a feature-parented Story's `Done` means "integrated into its feature branch." Stories under a
Feature are sequential (single Story in flight), and the orchestrator brings the feature branch up
to date with `main` before cutting each Story branch, so the buffer never drifts far from `main`.
Once **every child Story is `Done`**, the orchestrator **automatically** merges the Feature to
`main` — event-driven off the last Story's merge — via **refresh (`main`→feature) → a
feature-level `/factory-check-adr-compliance` gate → a history-preserving merge commit** (never squashed at
this boundary, so each Story's commit survives on `main`). Any conflict on the refresh or a gate
fail **fails closed** to a structured comment on the Feature and stops — never an auto-retry, never
a partial merge. The Feature's native ADO `System.State` is projected on the minimal
**`Active → Closed`** ladder as delivery progresses.

**Parentless work — straight off `main`.** A Bug, a chore, or any work with **no** parent Feature
is a complete deliverable in itself, so it branches **directly off `main`** on a single
`story/<issue#>-<slug>` (or `bug/…`) branch and is **squash-merged back to `main`** on `Approved`
(`Done` = "in `main`", the flat rule). No feature branch is involved.

## 3. Repo settings & protection posture (recorded per repo)

_Filled when the orchestrator's feature-branch buffer is first enabled on this repo (the
capability-enablement checklist). Until then this repo runs the flat off-`main` model and this
section stays a placeholder._

- **`feature/*` protection ruleset:** _not yet decided_ — default at enablement is **no ruleset on
  `feature/*`** (orchestrator-owned buffers; every Story lands through the full per-Story gauntlet
  and the feature→`main` merge is separately gated). Record the decision here so an ungated buffer
  is always a documented choice, never an accident.
- **Protected `main` required status checks:** _to be enumerated at enablement_ — list each required
  check and confirm it reports on an orchestrator-opened feature→`main` PR. An unverified check
  **fails closed** (a check that never reports would dead-end every feature merge).
- **Orchestrator ruleset bypass on `main`:** **none** — the orchestrator is never granted a bypass;
  required checks gate its merges exactly as they gate a human's.
