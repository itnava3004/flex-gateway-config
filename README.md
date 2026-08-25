# EAI-3541669 — Flex Gateway Configuration

This repository contains all **MuleSoft Anypoint Flex Gateway** API configurations, policies, per-environment runtime config, and deployment automation for project **EAI-3541669**.

---

## Project Structure

```
eai-3541669-flex-gateway-config/
├── apis/                          # One folder per API (env-independent contract)
│   ├── customer-api/config.yaml
│   ├── payments-api/config.yaml
│   └── orders-api/config.yaml
├── envs/                          # Per-API, per-environment runtime config
│   ├── customer-api/
│   │   ├── dev/runtime.yaml
│   │   ├── test/runtime.yaml
│   │   ├── qa/runtime.yaml
│   │   ├── sandbox/runtime.yaml
│   │   ├── preprod/runtime.yaml
│   │   └── prod/runtime.yaml
│   ├── orders-api/<env>/runtime.yaml
│   └── payments-api/<env>/runtime.yaml
├── policies/                      # Reusable Flex Gateway policy definitions
│   ├── client-id-enforcement.yaml
│   ├── rate-limiting-sla.yaml
│   └── fxf-custom-header.yaml
├── waves/                         # Release wave manifests
│   ├── R1/manifest.yaml
│   └── R2/manifest.yaml
├── inventory/
│   └── api-inventory.yaml         # Master API inventory (source of truth)
├── src/deploy/                    # Deployment automation scripts
│   ├── deploy.sh
│   └── validate.sh
└── Jenkinsfile                    # CI/CD pipeline definition
```

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| `yq` | v4+ | `brew install yq` |
| `flexctl` | latest | [Flex Gateway CLI docs](https://docs.mulesoft.com/gateway/flex-gateway-getting-started) |
| Jenkins | 2.x+ | Internal CI/CD platform |

---

## Quick Start

### 1. Validate all YAML files
```bash
chmod +x src/deploy/validate.sh
src/deploy/validate.sh
```

### 2. Deploy to an environment (dry run)
```bash
chmod +x src/deploy/deploy.sh
src/deploy/deploy.sh dev        # Deploy all active APIs to dev
src/deploy/deploy.sh qa R1      # Deploy only Wave R1 APIs to qa
```

### 3. Deploy for real (requires `flexctl` configured with credentials)
Set `DRY_RUN=false` in the Jenkins pipeline or remove the dry-run guard in `deploy.sh`.

---

## API Inventory

| API | Wave | Status | Base Path |
|-----|------|--------|-----------|
| customer-api | R1 | active | `/customer/v1` |
| payments-api | R1 | active | `/payments/v1` |
| orders-api   | R2 | pending | `/orders/v1` |

Full details in [`inventory/api-inventory.yaml`](inventory/api-inventory.yaml).

---

## Policies Applied to All APIs

| Policy | Purpose |
|--------|---------|
| `client-id-enforcement` | Validates `client_id` / `client_secret` headers |
| `rate-limiting-sla` | Per-client request rate limiting per SLA tier |
| `fxf-custom-header` | Injects `X-Correlation-ID` and EAI tracing headers |

---

## Environment Configuration

Each API has one runtime file per environment at `envs/<app>/<env>/runtime.yaml`. It is self-contained and holds every environment-specific value for that API: the gateway `publicHostname` and `tlsCertName`, the `proxyUri` (listener port), and the backend `uri` for each endpoint.

`apis/<app>/config.yaml` stays environment-independent — endpoint names, public paths, auth patterns and policies only. Endpoints are joined by name at deploy time, so an endpoint present in `config.yaml` but missing from `runtime.yaml` fails the build with a clear error.

### Naming

Three different names are in play, and they land in different places:

| Field | Where it comes from | What it controls |
|---|---|---|
| `assetId` | `config.yaml` | The Exchange asset identifier and the asset the API Manager instance points at. This is the real identity — changing it creates a new asset. |
| `exchangeAssetName` | `config.yaml` | The **display name** shown in Exchange. Free text; spaces allowed. Falls back to the folder name if omitted. |
| *(folder name)* | `apis/<folder>/` | The API Manager instance label, as `<folder>-<env>`. |

`exchangeAssetName` is only applied when an asset is first published. Renaming it will not rename an asset that already exists in Exchange — that has to be changed in the Exchange UI.

**Never hardcode credentials.** All secrets must be referenced via Anypoint Secrets Manager:
```yaml
clientId: "${secure::prod.gateway.clientId}"
```

---

## CI/CD Pipeline

The `Jenkinsfile` defines the full promotion pipeline:

```
Validate YAML → Deploy Dev/Test/QA → Deploy PreProd → Approval Gate → Deploy Prod
```

Production deployments require:
- Branch: `main`
- Manual approval from `integration-leads` group
- `DRY_RUN=false`

---

## Contributing

1. Branch from `main` — use naming: `feature/EAI-XXXX-description`
2. Run `src/deploy/validate.sh` locally before raising a PR
3. All YAML must pass validation in the Jenkins `Validate YAML` stage
4. Tag the PR with the relevant wave (`R1` / `R2`)

---

## Contacts

| Role | Contact |
|------|---------|
| Integration Platform Lead | integration-platform-team@example.com |
| Customer Domain | customer-tech@example.com |
| Payments Domain | payments-tech@example.com |
| Orders Domain | orders-tech@example.com |
