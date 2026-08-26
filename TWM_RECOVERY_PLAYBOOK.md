# TWM Recovery Playbook

This playbook answers one question:

```text
If a machine, account, database, or hosted service disappears tomorrow, what must be protected so TWM can be restored?
```

It is intentionally practical for the current pre-MVP stage. The filled secrets inventory must live outside Git.

## Recovery Target

Current production shape:

```text
UI on Vercel
  -> FastAPI Backend on Render
      -> n8n webhooks on EC2
          -> n8n Postgres and Docker volumes

Backend app data
  -> Postgres through APP_DATABASE_URL

Product contracts and decisions
  -> TWM_Docs

Runtime prompts, schemas, migrations, and n8n workflow backups
  -> TravelWithMe
```

The minimum acceptable restore is:

```text
1. Clone all repos with Git history.
2. Restore Backend runtime configuration.
3. Restore app database.
4. Restore n8n live state or re-import workflows and recreate credentials.
5. Deploy Backend and UI.
6. Verify login, trip persistence, Scout, Meridian, and planner/dashboard resume flows.
```

## Asset Inventory

| Asset | Current source | Backup need | Priority |
| --- | --- | --- | --- |
| Backend source and Git history | `TravelWithMe` GitHub repo | Independent Git backup or mirror | Critical |
| UI source and Git history | `TWM-UI` GitHub repo | Independent Git backup or mirror | Critical |
| Product docs and contracts | `TWM_Docs` GitHub repo | Independent Git backup or mirror | Critical |
| Backend app database | `APP_DATABASE_URL`, schema `twm_app` | Automated database backup plus restore drill | Critical before real users |
| n8n live state | EC2 Docker volumes `n8n_data`, `n8n_postgres_data` | Volume/Postgres backup plus stable encryption key | Critical while n8n is active |
| n8n workflow backups | `TravelWithMe/n8n/*.json` | Commit and mirror with backend repo | Critical while n8n is active |
| Runtime prompts | `TravelWithMe/twm/prompts` | Commit, mirror, and preserve prompt changelogs | Critical |
| API schemas and migrations | `TravelWithMe/twm/schemas`, `TravelWithMe/migrations` | Commit and mirror with backend repo | Critical |
| Knowledge base YAML | `TravelWithMe/kb`, `TWM_Docs/KB` | Commit and mirror | Important |
| Ingested KB database | Supabase/Postgres if populated beyond Git YAML | DB backup if rows cannot be recreated exactly | Important |
| Render service config | Render dashboard plus `TravelWithMe/render.yaml` | Secrets inventory and account recovery | Critical |
| Vercel project config | Vercel dashboard plus `TWM-UI/vercel.json` | Secrets inventory and account recovery | Critical |
| EC2 instance config | AWS account and EC2 setup docs | Snapshot or rebuild runbook | Important |
| Observability history | Axiom datasets `twm-production`, `twm-dev` | Export only if history becomes business-critical | Later |
| Analytics history | GA4 property | Account recovery; export later if needed | Later |
| User-uploaded files | No current object storage implementation found | Add object storage backup before uploads launch | Future critical |
| Research and strategy files | Workspace root PDFs/notes | Back up outside repo if not committed | Important |

## Current Local Work To Protect

These were present during the August 26, 2026 recovery inventory and are not necessarily protected by GitHub unless committed or backed up elsewhere.

| Repository | Local state |
| --- | --- |
| `TravelWithMe` | Untracked `twm/prompts/scout_old.md`, `twm/prompts/meridian_old.md` |
| `TWM_Docs` | Untracked `trip-matcher/SCOUT_OLD.md`, `trip-matcher/MERIDIAN_OLD.md` |
| `TWM-UI` | Modified `app/src/pages/TripDashboard.jsx`, `app/tests/unit/pages/TripDashboard.test.jsx`; untracked `node_modules/` should not be backed up as source |
| Workspace root | `dubai story.txt`, `product_flow_feedback_notes.md`, `Reddit User Research Summary.pdf`, `TWM_Product_and_Strategy.pdf` |

## Secrets And Account Inventory Template

Store the filled version in a password manager or other private vault, not in Git.

```text
GitHub
- Account owner:
- 2FA recovery codes location:
- Backup admin/collaborator:
- Repos:
  - TravelWithMe:
  - TWM-UI:
  - TWM_Docs:

Render
- Account:
- Service: travelwithme-api
- Service URL:
- ENVIRONMENT:
- AGENT_ENGINE:
- APP_DATABASE_URL:
- APP_DATABASE_SCHEMA:
- JWT_SECRET:
- LANGGRAPH_MODEL_PROVIDER:
- LANGGRAPH_MODEL:
- LANGGRAPH_API_KEY:
- OTEL_EXPORTER_OTLP_LOGS_ENDPOINT:
- OTEL_EXPORTER_OTLP_LOGS_PROTOCOL:
- OTEL_EXPORTER_OTLP_LOGS_HEADERS:
- TRUSTED_HOSTS:
- CORS_ALLOWED_ORIGINS:
- AVIASALES_API_TOKEN:
- AVIASALES_PARTNER_ID:

Vercel
- Account:
- Project:
- Production URL:
- TWM_BASE_URL:
- VITE_GA_MEASUREMENT_ID:

AWS / EC2
- Account:
- Region:
- Instance name/id:
- SSH key location:
- Security group notes:
- n8n public/editor URL:
- Reverse proxy/domain notes:

n8n
- Admin account:
- N8N_ENCRYPTION_KEY:
- N8N_DB_PASSWORD:
- N8N_HOST:
- N8N_PROTOCOL:
- WEBHOOK_URL:
- N8N_EDITOR_BASE_URL:
- Workflow names/IDs:
- Credential names:

Database
- Provider/account:
- Database host/project:
- Database name:
- Schema:
- Backup location:
- Restore command/runbook:

Axiom
- Account:
- Datasets:
  - twm-production:
  - twm-dev:
- Token rotation location:

Domain / DNS
- Registrar:
- Domain:
- DNS provider:
- Recovery email:
- 2FA recovery location:

LLM / provider accounts
- Groq account:
- Other model providers:

Affiliate / booking providers
- Aviasales account:
- Partner ID:
```

## Backup Cadence

| Item | Suggested cadence now | Before real users |
| --- | --- | --- |
| Git repos | Push every meaningful change; weekly independent mirror | Same |
| Local uncommitted work | End-of-day export or commit to a branch | Avoid long-lived uncommitted critical work |
| Backend app database | Manual snapshot after schema/data milestones | Automated daily backups plus retention |
| n8n Postgres and volumes | Manual backup after workflow/credential changes | Automated backup after each live workflow change |
| Secrets inventory | Update after every config/account change | Same, with quarterly recovery-code check |
| Product docs/research | Weekly backup if outside Git | Keep canonical docs in Git where possible |
| Object storage | Not applicable yet | Automated bucket backup/versioning before file upload launch |

## Restore Drill

Run this after the first real database backup exists, and again before onboarding real users.

```text
1. Start from a fresh machine or clean VM.
2. Clone TravelWithMe, TWM-UI, and TWM_Docs.
3. Install Backend dependencies and run Backend tests.
4. Install UI dependencies and run UI tests/build.
5. Restore a database backup into a non-production database.
6. Set Backend env vars against the restored database.
7. Restore n8n from backup, or import workflow JSON and recreate credentials.
8. Deploy or run Backend against restored n8n.
9. Deploy or run UI against restored Backend.
10. Verify:
    - /health returns ok
    - signup/login works
    - guest trip creation works
    - trip save/resume works
    - Scout request works
    - Meridian request works
    - recommendation history resumes
    - dashboard and selected-trip flows render
11. Record what failed and update this playbook.
```

## Restore Pass Criteria

A restore is successful only when all of these are true:

```text
Code exists with history.
Backend deploys from the restored source.
UI deploys from the restored source.
Database records are readable by the Backend.
n8n workflows are active or the selected AgentEngine replacement works.
Runtime secrets are present and rotated if compromise is suspected.
Core user flows work without manually editing production data.
```

## Immediate Actions

```text
1. Decide where independent Git mirrors or repo exports should live.
2. Put GitHub 2FA recovery codes and account recovery details in a vault.
3. Fill the private secrets/account inventory outside Git.
4. Back up or intentionally commit important untracked prompt/doc files.
5. Confirm where APP_DATABASE_URL is hosted and enable backups before real users.
6. Back up n8n volumes and N8N_ENCRYPTION_KEY while n8n remains active.
7. Keep product decisions, API contracts, prompt behavior, and state ownership in TWM_Docs/TravelWithMe instead of scattered chats.
```

