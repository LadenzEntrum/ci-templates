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
| `deploy.yml` | Kopiere das Artefakt in ein lokales Verzeichnis auf dem Self‑Hosted Runner; Prod‑Gate via Environment | `deploy_target` (staging\|production), `target-path`, `artifact-name` | – |
| `agent-notify.yml` | Result an lokalen OpenClaw-Gateway-Hook senden (Bearer); läuft auf `self-hosted` | `status`, `stage` | `OPENCLAW_GATEWAY_HOOK_TOKEN` |

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
      target-path: /mnt/HettiWeb/hue-staging
      artifact-name: ${{ needs.ci.outputs.artifact-name }}
    secrets: inherit

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    uses: LadenzEntrum/ci-templates/.github/workflows/deploy.yml@<COMMIT_SHA>
    with:
      deploy_target: production          # gated: Required Reviewer im Environment
      target-path: /mnt/HettiWeb/hue
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
  - `OPENCLAW_GATEWAY_HOOK_TOKEN` — Bearer-Token des OpenClaw-Gateways (`hooks.token`). Der `agent-notify`-Job läuft auf einem **self-hosted Runner** auf dem OpenClaw-Host und postet an `http://127.0.0.1:18789/hooks/ci-result` (loopback-only, keine Internet-Exposition).
  *(Keine Deploy‑Secrets mehr nötig, da der Workflow lokal auf dem Self‑Hosted Runner schreibt.)*

## Sicherheit

- Deploy schreibt lokal auf den NFS-Mount des self-hosted Runners — keine SSH-/Deploy-Credentials nötig.
- `permissions:` defensiv (default read-only).
- Prod nur hinter Environment-Gate mit Required Reviewer.
- Gateway-Hook ist **loopback-only** (`bind=loopback`, 127.0.0.1:18789) und per
  statischem Bearer-Token (`hooks.token`) geschützt; der `agent-notify`-Job
  erreicht ihn nur, weil er auf einem self-hosted Runner auf demselben Host läuft.
  Keine Internet-Exposition, kein offener `/hooks/agent`.

## Quellen

- https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows
- https://datagrid.com/blog/cicd-pipelines-ai-agents-guide
