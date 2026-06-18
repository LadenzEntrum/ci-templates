# ci-templates

Zentrale, wiederverwendbare GitHub-Actions-Workflows (Reusable Workflows) für alle
OpenClaw-/LadenzEntrum-Projekte. **Reuse-first, DRY** — Projekt-Repos binden diese
Workflows per `workflow_call` ein, statt YAML zu kopieren.

Entstanden aus [Planner #6](https://github.com/LadenzEntrum/Planner/issues/6).
Erst-Anwendung (Pilot): Repo `HUE Webapp` (über [Planner #5](https://github.com/LadenzEntrum/Planner/issues/5)).

## Leitprinzipien

1. **Reuse-first** — ein Template, viele Projekte.
2. **Determinismus ↔ Intelligenz getrennt** — Build/Test/Deploy sind deterministische
   Actions-Jobs; der OpenClaw-Agent ist **Beobachter/Assistent** (Failure-Analyse,
   Telegram-Report), **nie** im kritischen Deploy-Pfad.
3. **Least Privilege & Approval Gates** — minimale `permissions:`, Prod hinter
   GitHub-Environment-Gate (Required Reviewer).
4. **Versioniert & auditierbar** — Aufrufe **per Commit-SHA** pinnen.

## Workflows

| Workflow | Zweck | Wichtige Inputs | Secrets |
|----------|-------|-----------------|---------|
| `ci.yml` | Lint + Test + Build → Artefakt | `runtime` (php\|node\|python), `runtime-version`, `working-directory`, `artifact-name`, `build-command` | – |
| `deploy.yml` | rsync des Artefakts auf `dsmuc` über Tailscale/SSH; Prod-Gate via Environment | `deploy_target` (staging\|production), `target-path`, `artifact-name` | `DEPLOY_SSH_KEY`, `DEPLOY_HOST`, `DEPLOY_USER`, `TAILSCALE_AUTHKEY` |
| `agent-notify.yml` | signierten Result-Webhook an OpenClaw-Gateway senden | `status`, `stage` | `OPENCLAW_GATEWAY_WEBHOOK_URL`, `OPENCLAW_GATEWAY_WEBHOOK_SECRET` |

## So bindest du das Template in ein neues Repo ein

Lege im Projekt-Repo `.github/workflows/pipeline.yml` an:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: LadenzEntrum/ci-templates/.github/workflows/ci.yml@<COMMIT_SHA>
    with:
      runtime: php
    # secrets werden hier nicht benötigt

  deploy-staging:
    needs: ci
    if: github.ref == 'refs/heads/main'
    uses: LadenzEntrum/ci-templates/.github/workflows/deploy.yml@<COMMIT_SHA>
    with:
      deploy_target: staging
      target-path: /volume1/web/hue-staging
      artifact-name: ${{ needs.ci.outputs.artifact-name }}
    secrets: inherit

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    uses: LadenzEntrum/ci-templates/.github/workflows/deploy.yml@<COMMIT_SHA>
    with:
      deploy_target: production          # gated: Required Reviewer im Environment
      target-path: /volume1/web/hue
      artifact-name: ${{ needs.ci.outputs.artifact-name }}
    secrets: inherit

  notify-failure:
    needs: [ci, deploy-staging, deploy-production]
    if: failure()
    uses: LadenzEntrum/ci-templates/.github/workflows/agent-notify.yml@<COMMIT_SHA>
    with:
      status: failure
    secrets: inherit
```

> **SHA-Pin:** Ersetze `<COMMIT_SHA>` durch einen konkreten Commit dieses Repos
> (`git ls-remote https://github.com/LadenzEntrum/ci-templates HEAD`). Nach jedem
> Template-Release den Pin aktualisieren (manuell oder via Dependabot).

## Voraussetzungen im Projekt-Repo (NICHT hier)

Reusable Workflows lösen **Environments und Secrets im aufrufenden Repo** auf.
Daher im Projekt-Repo (z. B. `HUE Webapp`) einrichten:

- **Environments** `staging` und `production` (Prod mit Required Reviewer = PDO).
- **Secrets** (Environment- oder Repo-/Org-Ebene):
  - `DEPLOY_SSH_KEY` — privater SSH-Key, der auf `dsmuc` autorisiert ist
  - `DEPLOY_HOST` — Tailscale-IP/Hostname von `dsmuc`
  - `DEPLOY_USER` — SSH-User auf `dsmuc`
  - `TAILSCALE_AUTHKEY` — ephemeraler Tailscale-Auth-Key für den Runner
  - `OPENCLAW_GATEWAY_WEBHOOK_URL` — Webhook-Endpunkt des OpenClaw-Gateways
  - `OPENCLAW_GATEWAY_WEBHOOK_SECRET` — Shared Secret für die HMAC-Signatur

## Sicherheit

- PAT für Deploys minimal: `repo` + `workflow`; Deploy-Credentials nur als Secrets.
- `permissions:` defensiv (default read-only).
- Prod nur hinter Environment-Gate mit Required Reviewer.
- Agent-Webhook ist HMAC-SHA256-signiert (`X-OpenClaw-Signature: sha256=…`);
  das Gateway verifiziert die Signatur, bevor es das Event annimmt.

## Quellen

- https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows
- https://datagrid.com/blog/cicd-pipelines-ai-agents-guide
