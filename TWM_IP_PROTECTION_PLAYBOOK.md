# TWM IP Protection Playbook

This playbook covers ownership and collaborator protection. It is separate from backups:

```text
Backups protect TWM from loss.
IP protection helps prove ownership, control access, and avoid collaborator disputes.
```

This is an operational checklist, not legal advice. Use it to prepare clean evidence and questions for a lawyer before bringing on a contractor, designer, developer, advisor, or cofounder.

## Current Repository Signals

As of August 26, 2026:

| Area | Current state |
| --- | --- |
| Backend CODEOWNERS | `TravelWithMe/.github/CODEOWNERS` assigns all files to `@agarwalnaina1496` |
| UI CODEOWNERS | `TWM-UI/.github/CODEOWNERS` assigns all files to `@agarwalnaina1496` |
| Docs CODEOWNERS | `TWM_Docs/.github/CODEOWNERS` assigns all files to `@agarwalnaina1496` |
| Open-source license | No `LICENSE` file found in the main Backend, UI, or Docs repos |
| Contributor process | No `CONTRIBUTING` file found in the main Backend, UI, or Docs repos |
| Public legal pages | UI has public `privacy.html` and `terms.html`, but these are user-facing product pages, not collaborator IP agreements |

## What To Protect

| Asset | Why it matters |
| --- | --- |
| Source code and Git history | Best technical evidence of who created what and when |
| Prompts and agent behavior | TWM's product logic and proprietary execution details |
| API schemas and state contracts | Durable implementation design and product architecture |
| UX copy, pages, and design system | Copyrightable product expression |
| Product docs and research | Evidence of accumulated strategy, decisions, and product reasoning |
| Domain, accounts, invoices, vendor records | Business ownership and continuity evidence |
| Private datasets, KB profiles, workflow exports | Proprietary content and operational knowledge |
| Secrets, tokens, DB credentials | Access control, not IP, but essential to prevent misuse |

## Collaborator Access Rules

Use least privilege by default:

```text
1. Give repo access only after the written agreement is signed.
2. Prefer GitHub collaborator access over sharing account credentials.
3. Never share GitHub, Render, Vercel, AWS, database, n8n, or provider passwords.
4. Use separate user accounts wherever a platform supports it.
5. Give read-only access when review is enough.
6. Give write access only to the repo they actively work on.
7. Use pull requests for contributions.
8. Keep CODEOWNERS review required for protected branches.
9. Revoke access immediately when the engagement ends.
10. Rotate secrets if a collaborator had access to any environment where secrets were visible.
```

## Agreement Checklist

Before meaningful contribution, get a written agreement reviewed for the exact relationship.

For a contractor or freelancer, cover:

```text
- Work made for TWM belongs to TWM or is assigned to TWM.
- Source code, designs, docs, prompts, workflows, data, and product copy are included.
- The collaborator will not reuse confidential TWM materials elsewhere.
- Pre-existing tools/libraries they bring in must be disclosed.
- Open-source dependencies require approval before introduction.
- Payment terms do not leave ownership ambiguous.
- They must return/delete TWM materials when the engagement ends.
- They must not keep production credentials.
```

For a cofounder or business partner, also cover:

```text
- Equity ownership and vesting.
- Decision rights.
- Role expectations.
- IP assignment from each founder to the company/entity.
- What happens if someone leaves.
- Confidentiality.
- Domain/account ownership.
- Future company formation and transfer of assets.
```

For an advisor or reviewer, cover:

```text
- Confidentiality.
- No ownership rights unless explicitly granted.
- No permission to copy, publish, or use private TWM materials.
- Scope of feedback and any compensation.
```

## Evidence Trail

Preserve dated evidence without turning work into bureaucracy:

```text
- Keep Git commits small and meaningful.
- Push important work to private GitHub remotes.
- Keep product decisions in TWM_Docs or clearly named private notes.
- Keep invoices, contracts, vendor receipts, domain receipts, and account emails.
- Keep exports of major design/research documents.
- Keep issue/PR history connected to implementation decisions.
- Avoid making important decisions only in ephemeral chats.
```

## License Position

No open-source license was found in the main TWM repos during this scan.

Practical meaning:

```text
- Do not add an open-source license casually.
- Keep repos private while the product is pre-MVP.
- If a collaborator asks to reuse TWM code, treat that as a separate written permission decision.
- If TWM intentionally open-sources anything later, choose the license deliberately with legal/business review.
```

## Employment Check

Before onboarding collaborators or forming a company, review any current employment agreement for:

```text
- invention assignment
- moonlighting restrictions
- use of company devices/accounts
- use of company time
- conflict-of-interest rules
- obligation to disclose side projects
```

If any clause is broad or unclear, ask a lawyer before showing TWM private material to collaborators.

## Trademark And Brand

Copyright protects specific code, docs, designs, copy, and other expression. It usually does not protect a broad product idea such as an AI-based personalized travel planner.

Trademark is separate. If the `Travel With Me` / `TWM` brand becomes important:

```text
- Check name availability in the relevant markets.
- Preserve domain and social/account ownership.
- Consider trademark filing before major launch or brand investment.
- Keep brand assets and launch materials dated.
```

## Immediate Actions

```text
1. Keep GitHub repos private.
2. Keep CODEOWNERS as `@agarwalnaina1496` unless ownership structure changes.
3. Turn on branch protection before external collaborators contribute.
4. Prepare a contractor IP assignment + confidentiality template with a lawyer.
5. Prepare a cofounder/founder agreement before treating anyone as a partner.
6. Move durable product decisions out of scattered chats into TWM_Docs or private dated notes.
7. Keep a private folder for contracts, invoices, domain receipts, vendor emails, and account recovery.
8. Do not share production secrets; create least-privilege access per collaborator.
9. Review employment invention/IP clauses before broad disclosure.
10. Decide when brand/trademark protection matters based on launch timing.
```

