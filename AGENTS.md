<!-- twm-codex-basekit: START -->
## Travel With Me workspace delivery rules

The Travel With Me product is split across independent repositories:

- `TravelWithMe/`: Backend APIs, agents, schemas, prompts, workflows, and server-side business logic.
- `TWM-UI/`: Frontend behavior, UI state, persistence, and Backend integration.
- `TWM_Docs/`: Canonical product behavior and shared-contract documentation.

### Repository boundaries

- Preserve every repository's independent Git history.
- Keep branches, commits, verification results, and pull requests separate by repository.
- Modify only repositories with a proven implementation or documentation delta.
- Avoid unrelated refactors, formatting churn, compatibility layers, and migration paths.
- Treat the product as pre-MVP unless the user explicitly decides otherwise.

### Product intent

- Treat the user's confirmed decisions in active discovery as the authority for intended behavior.
- Use code, prompts, tests, workflows, and documentation as evidence of current behavior, not as authority over a conflicting confirmed decision.
- Surface conflicts or material ambiguity before finalizing scope.
- Record confirmed decisions in the proposed work breakdown and approved Linear issues.

### Mandatory delivery workflow

1. Begin with read-only discovery across every potentially affected repository.
2. Assess Backend, UI, product documentation, and n8n as distinct delivery surfaces. Mark each in scope, out of scope, no change required, or requiring further investigation.
3. Prove a current-versus-required delta before proposing an implementation child story.
4. Present a consolidated work breakdown before creating or updating Linear issues.
5. Wait for explicit approval before writing Linear issues.
6. Wait for explicit selection or approval of a Linear implementation story before editing files.
7. Before editing, confirm the story, repositories, acceptance criteria, and branch plan.
8. Before the first commit, create and switch to a non-default delivery branch in each affected repository. Never commit or push directly to a default branch unless the user explicitly authorizes that exact exception; general approval to "commit and push" does not authorize default-branch delivery.
9. Implement only approved scope. Return to planning for material expansion or an unapproved contract change.
10. Run repository-specific verification and present diffs, results, limitations, and rollback instructions.
11. Wait for separate explicit approvals for commit, push, pull-request creation, and merge.

### Linear structure

- Apply the `Feature` label to a parent capability or delivery container.
- Structure implementation children around dependency-ordered delivery increments, not around repository count alone. Start with prerequisite contract or foundation work, followed by coordinated implementation, end-to-end integration, and documentation where each increment has a proven delta.
- Use separate repository-prefixed children when work is independently implementable or reviewable, or when one piece must block another.
- When Backend and UI tasks are inseparable parts of one useful increment, keep them in one cross-repository story with separate repository-specific task checklists and a combined `[BE][UI]` title prefix. Retain separate branches, commits, verification results, and pull requests per repository.
- Do not create child stories merely to mirror repositories or task groups. Keep checklist tasks inside the story that owns the delivery increment and order them by prerequisite.
- Encode the approved delivery sequence with Linear blocker relationships.
- Prefix implementation-story titles with `[BE]`, `[UI]`, `[BE][UI]`, or `[DOCS]` for the corresponding product repository scope.
- Use `[BASEKIT]` for this independently versioned Codex Basekit.
- Do not add repository prefixes to branches, commits, or pull-request titles unless explicitly requested.
- Create children only for proven changes; record `no change required` on the parent for satisfied surfaces.
- Include problem and outcome, scope, out-of-scope items, affected repositories, acceptance criteria, contract impact, dependencies, verification, and rollback.
- Keep parent Feature descriptions concise and avoid duplicating child task lists, implementation hierarchy, or story-reference catalogs already represented by Linear parent and blocker relationships.
- Prefer one `[DOCS]` child for a capability's canonical product and shared-contract documentation after the behavior is verified, unless documentation is an earlier independent blocker.
- Do not add a standalone Risks section while the product is pre-launch; place concrete constraints with the relevant scope, dependency, or verification item.

### Contracts and coordination

- Treat approved Backend request and response schemas as the implementation source of truth.
- Inspect both Backend and UI for API or user-flow changes.
- Define shared request and response contracts before implementation.
- Record whether coordinated work is independent or blocked, and include deployment ordering only when concretely required.
- Do not add legacy compatibility or rollout layers unless explicitly requested.

### Documentation routing

- Keep canonical product behavior and shared contracts in `TWM_Docs/`.
- Keep Backend technical and operational documentation in `TravelWithMe/`, including prompts, n8n, FastAPI internals, runtime configuration, deployment, and troubleshooting.
- Do not duplicate Backend-only operational documentation in `TWM_Docs/`.
- Require documentation changes only for affected product behavior, user-facing flows, shared contracts, prompt behavior, state ownership, architecture, or material operational workflows.

### Verification and Git delivery

- Run relevant tests, linters, type checks, builds, and focused manual verification in every modified repository.
- Verify affected documentation matches implemented behavior.
- Stage only intended files in dirty worktrees.
- Keep commits small, traceable to the approved Linear story, and easy to revert.
- Title every TWM delivery pull request exactly `TWM#<issue-number> - <concise title>`, using the primary Linear issue number and no repository prefix.
- Start every TWM delivery pull-request description with a `## Tracking` section. Put the primary Linear issue link first, followed by companion pull requests or follow-on tracking links when applicable; place summary, acceptance-criteria coverage, verification, coordination, and rollback after Tracking.
- Keep pull-request metadata truthful when editing an open or merged pull request; remove stale scope, test, prompt-version, or deployment claims.
- Never merge a pull request without explicit user approval.
- After a pull request is merged, verify the merge and clean up its exact delivery branch completely: delete the remote branch, remove any linked worktree created for that branch, delete the local branch, and prune stale worktree metadata. Never delete a default branch, protected branch, unmerged branch, branch referenced by an open pull request, or a branch or worktree with commits or uncommitted work not contained in the merged PR. Before removing a linked worktree, verify its resolved path, confirm it is clean, and confirm it belongs to the exact merged-PR branch; never force removal or discard worktree changes. If GitHub already deleted the remote branch automatically or no linked worktree exists, verify that it is absent.

# Travel With Me Product Documentation

This repository owns canonical product behavior and shared-contract documentation.

## Scope and issue naming

- Use `[DOCS]` only for Linear implementation stories that change this repository.
- Do not create a Docs story merely because Backend or UI changed; prove a canonical product-behavior or shared-contract documentation delta.
- Keep unrelated editorial cleanup and formatting churn out of an approved documentation change.

## Canonical ownership

- Maintain the playbook, Scout and Meridian product behavior, product architecture, TripState ownership, stage transitions, CTA mappings, resume behavior, and shared API or user-flow contracts here.
- Keep Backend technical and operational documentation in `TravelWithMe/`, including prompt changelogs, FastAPI internals, n8n operations, EC2, deployment, runtime configuration, and troubleshooting.
- Do not duplicate Backend-only implementation or operational documentation.
- Treat approved product decisions as authoritative when existing documents conflict; surface unresolved conflicts before editing.

## Consistency and verification

- Identify every document affected by a product behavior or shared-contract change before editing.
- Keep terminology, state ownership, stage names, CTA behavior, and request or response examples consistent across affected documents.
- Verify links and referenced paths after changes.
- Report changed canonical behavior, cross-document checks, known limitations, and rollback instructions.

## Git delivery

- Use a Docs-specific branch and pull request.
- Stage only intended files in a dirty worktree.
<!-- twm-codex-basekit: END -->
