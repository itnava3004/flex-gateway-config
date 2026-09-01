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
├── release/                       # Release manifests
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
src/deploy/deploy.sh qa R1      # Deploy only Release R1 APIs to qa
```

### 3. Deploy for real (requires `flexctl` configured with credentials)
Set `DRY_RUN=false` in the Jenkins pipeline or remove the dry-run guard in `deploy.sh`.

---

## API Inventory

| API | Release | Status | Base Path |
|-----|------|--------|-----------|
| customer-api | R1 | active | `/customer/v1` |
| payments-api | R1 | active | `/payments/v1` |
| orders-api   | R2 | pending | `/orders/v1` |

Full details in [`inventory/api-inventory.yaml`](inventory/api-inventory.yaml).

---

## Policies

Each policy is defined once in `policies/<policy>.yaml` — `assetId`, `policyVersion`, `groupId` and its `defaults`:

```yaml
assetId: ip-allowlist
policyVersion: "1.1.2"
groupId: "68ef9520-..."
defaults:
  ipExpression: "#[attributes.headers['x-forwarded-for']]"
  ips: []
```

`apis/<app>/config.yaml` carries **references only** — never the definition. Every entry uses `ref:`. Add `config:` to layer API-specific values over the defaults; omit it to take the defaults unchanged:

```yaml
policies:
  - ref: client-id-enforcement     # defaults, unchanged
  - ref: ip-allowlist              # defaults + this API's CIDR ranges
    config:
      ips:
        - 10.40.0.0/16
```

Only the keys you supply are overridden; everything else falls through to the policy file. A `policyVersion` bump is therefore a one-line change in `policies/`, not an edit across every API that uses it.

The catalogue is indexed by `assetId`, **not** filename — `policies/rate-limiting-sla.yaml` declares `assetId: rate-limiting`, so that is what config.yaml references. Referencing a policy that is not in `policies/` fails the build and lists what is available.

`common/policies.yaml` uses the same reference form and applies to every API, unless that API declares the same policy itself — in which case the API's entry wins.

| Policy | Purpose |
|--------|---------|
| `client-id-enforcement` | Validates `client_id` / `client_secret` headers |
| `rate-limiting` | Per-client request rate limiting |
| `ip-allowlist` | Restricts access to trusted CIDR blocks |
| `fxf-custom-header` | Injects FXF internal tracing headers |

---

## Route Rules

Endpoints may constrain which requests match a route, beyond the path:

```yaml
endpoints:
  - name: customer-consent
    publicPath: /customer/v1/consent
    methods: [GET, POST]
    headers:
      X-FXF-Channel: internal
```

`methods` and `headers` are both optional. Omit them and the route matches on path alone. The pipeline sends `methods` to API Manager as a pipe-separated string (`GET|POST`) and `headers` as a name/value map, alongside the path rule.

---

## Environment Configuration

Each API has one runtime file per environment at `envs/<app>/<env>/runtime.yaml`. It is self-contained and holds every environment-specific value for that API: the gateway `publicHostname` and `tlsCertName`, the `proxyUri` (listener port), and the backend `uri` for each endpoint.

`apis/<app>/config.yaml` stays environment-independent — endpoint names, public paths, auth patterns and policies only. Endpoints are joined by name at deploy time, so an endpoint present in `config.yaml` but missing from `runtime.yaml` fails the build with a clear error.

## Config Versioning (Nexus)

Two fields work together:

| Field | File | Meaning |
|---|---|---|
| `apiVersion` | `apis/<app>/config.yaml` | The **current** version of this API's config contract. Bump it when you change `config.yaml`. |
| `deployVersion` | `envs/<app>/<env>/runtime.yaml` | The version **that environment deploys**. |

Both are separate from `config.yaml`'s `version:`, which is the Exchange asset version.

The pipeline compares them per API:

- **Equal → publish mode.** The working-copy `config.yaml` is deployed and archived to Nexus as that version.
- **Different → replay mode.** The pinned version already exists in Nexus, so the pipeline downloads that artifact and deploys *its* `config.yaml` instead of the working copy. Nothing is re-archived.

Because the artifact version always comes from `config.yaml`, a given version number always means the same contract content in every environment.

`runtime.yaml` itself is never replayed — environment wiring (`proxyUri`, `publicHostname`, endpoint backend URIs) always comes from the current Git checkout. Only the contract is pinned.

**Promoting** new config to an environment is therefore a one-line, reviewable commit: set that environment's `deployVersion` to the new `apiVersion`. **Rolling back** is the same edit in reverse — point `deployVersion` at the previous version and re-run; the old contract comes back from Nexus without touching `config.yaml`.

At deploy time the pipeline applies an environment suffix, following the same scheme as the eapi application pipeline:

| Environment | Suffix | Nexus repo |
|---|---|---|
| `dev`, `test`, `sandbox` | `-SNAPSHOT` | `snapshot` |
| `qa` | `-RC` | `release` |
| `preprod`, `prod` | *(none)* | `release` |

Two stages use it:

- **Resolve Config Version (Check Nexus)** runs before the approval gate. In publish mode, if the version already exists in a *release* repo the build fails with "bump apiVersion" — a released config version is never overwritten. Snapshots may be overwritten freely. Skipped for a pinned replay, whose existence is already proven by the download.
- **Archive Config to Nexus** runs after a successful deploy. For each API that deployed OK in publish mode it zips `apis/<app>/config.yaml` plus that environment's `runtime.yaml` and uploads it as `<app>-<version>.zip`. A pinned replay is not re-archived — that version is already published.

In publish mode Git is the deploy source and Nexus is the record. In replay mode Nexus is the source of the contract, which is what makes a pinned rollback possible without editing `config.yaml`.

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
- Manual approval from an authorised approver (see below)
- `DRY_RUN=false`

### Approvers

Any environment other than `dev` / `sandbox` stops at an approval gate. The authorised approvers are **not** build parameters — a parameter is editable by whoever triggers the build, which would let them name themselves as approver and self-approve. They live in a folder-scoped managed properties file instead.

**Folder → Config Files → `flex-gateway-approvers`** (type: Properties file):

```properties
TEST_QA_APPROVERS=integration-leads
PREPROD_PROD_APPROVERS=integration-leads
```

| Key | Applies to |
|---|---|
| `TEST_QA_APPROVERS` | `test`, `qa` |
| `PREPROD_PROD_APPROVERS` | `preprod`, `prod` |

Each value is either a comma-separated list of Jenkins user IDs (`3672738,1673670`) or a **group name** (`integration-leads`). A group is preferable — membership is then managed in Access Management / LDAP, and joiners and leavers need no pipeline or Jenkins config change.

Only the file's **ID** appears in the repo (`APPROVERS_CONFIG_ID` in the Jenkinsfile); the approver names never do. If the relevant key is missing the build fails at the gate rather than deploying unapproved.

Note that edit rights on a folder config file are folder-Configure, not Jenkins-admin — so whoever can configure the folder can change who approves.

---

## Contributing

1. Branch from `main` — use naming: `feature/EAI-XXXX-description`
2. Run `src/deploy/validate.sh` locally before raising a PR
3. All YAML must pass validation in the Jenkins `Validate YAML` stage
4. Tag the PR with the relevant release (`R1` / `R2`)

---

## Contacts

| Role | Contact |
|------|---------|
| Integration Platform Lead | integration-platform-team@example.com |
| Customer Domain | customer-tech@example.com |
| Payments Domain | payments-tech@example.com |
| Orders Domain | orders-tech@example.com |
