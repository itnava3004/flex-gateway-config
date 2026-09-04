# Omni Gateway Config — Cookbook

Task-oriented guide to this repository: how to do the things you will actually do,
and what to do when a build fails.

For *why* the repo is shaped the way it is, see [README.md](README.md). This file is
the how.

---

## The mental model

Three files describe an API, and each answers a different question:

| File | Question it answers | Changes when |
|---|---|---|
| `apis/<app>/config.yaml` | *What* is this API? Endpoints, paths, policies. | The API contract changes |
| `envs/<app>/<env>/runtime.yaml` | *Where* does it run in this environment? Gateway, port, backend host. | Environment wiring changes |
| `anypoint.yaml` | What environments and gateways exist at all. | Platform inventory changes |

`config.yaml` is environment-independent and is promoted unchanged. Anything that
differs between Dev and Prod belongs in `runtime.yaml`.

**Two version numbers, and they do different jobs:**

- `configVersion` in `config.yaml` — the current version of the contract. Bump it when
  you change `config.yaml`.
- `deployVersion` in `runtime.yaml` — the version *that environment* runs.

Equal → **publish mode**: deploy the working copy, archive it to Nexus.
Different → **replay mode**: pull that version's config from Nexus and deploy it.

That is what makes promotion and rollback a one-line change.

---

## File map

```
apis/<app>/config.yaml              the API contract           (env-independent)
envs/<app>/<env>/runtime.yaml       environment wiring         (one per app per env)
anypoint.yaml                       environments and gateways
policies/<policy>.yaml              policy definitions         (assetId, version, baseline config)
commons/policies.yaml               policies applied to every API
releases/<R>/manifest.yaml          which apps are in a release
Jenkinsfile                         the pipeline
```

---

## Recipes

### Onboard a new API

**1. Create the contract** — `apis/<app>/config.yaml`:

```yaml
app: my-api
assetId: my-api
exchangeVersion: 1.0.0
configVersion:   1.0.0

endpoints:
  - name: my-api-search
    publicPath: /my/v1/search
    methods: [GET]
    authPattern: client-id-enforcement

policies:
  client-id-enforcement:
  rate-limiting:
    rateLimits:
      - maximumRequests: 50
        timePeriodInMilliseconds: 1000
```

**2. Create one runtime file per environment** — `envs/my-api/<env>/runtime.yaml`:

```yaml
app: my-api
env: dev
deployVersion: 1.0.0
gateway: fxf-dev-gtwy
proxyUri: http://0.0.0.0:8081/my
upstreamHost: https://l7-gw.fxf.internal
```

`app` and `env` must match the file's own path — the build checks.

**3. Add it to a release** — `releases/R1/manifest.yaml`:

```yaml
apps:
  - my-api
```

**4. Dry run, then deploy.** `ENVIRONMENT=dev`, `RELEASE=R1`, `DRY_RUN=true`, then `false`.

Pick a `proxyUri` base path no other API on that gateway owns — see the port conflict
entry under Troubleshooting.

### Add an endpoint

`config.yaml` only:

```yaml
endpoints:
  - name: my-api-detail
    publicPath: /my/v1/detail
    methods: [GET]
```

Bump `configVersion`, and set `deployVersion` to match in whichever environments should
get it. No `runtime.yaml` change is needed — the new route uses the existing
`upstreamHost`.

### Remove an endpoint

Delete it from `config.yaml`. The routing reconcile removes the route from API Manager
on the next run — `config.yaml` is authoritative for removals, not just additions. If a
`runtime.yaml` still lists a path override for it you get a WARN; tidy it up.

### Change a policy for one API

Give the policy a value in that API's `config.yaml`. Only the keys you supply are
overridden; everything else falls through to `policies/<policy>.yaml`:

```yaml
policies:
  ip-allowlist:
    ips:
      - 10.40.0.0/16
```

### Bump a policy version everywhere

Edit `policies/<policy>.yaml` — one line, one place:

```yaml
policyVersion: "1.4.0"
```

No API's `config.yaml` changes; they reference the policy by name.

The catalogue is keyed by **assetId, not filename**. `policies/rate-limiting-sla.yaml`
declares `assetId: rate-limiting`, so configs reference `rate-limiting`.

### Promote a config version to the next environment

One line in the target environment's `runtime.yaml`:

```yaml
deployVersion: 1.2.0        # was 1.1.0
```

Set it equal to the current `configVersion` to deploy the working copy.

### Roll an environment back

The same edit in reverse — point `deployVersion` at a previously published version:

```yaml
deployVersion: 1.1.0        # configVersion is 1.2.0
```

The pipeline downloads that version's config from Nexus and deploys it. `config.yaml`
is not touched, so other environments are unaffected. The version must already exist in
Nexus — you cannot roll back to something never published.

### Move an API to a different gateway

`runtime.yaml` for that environment:

```yaml
gateway: fxf-dev-gtwy-2
```

Must name a gateway declared under that environment in `anypoint.yaml`.

### Add a gateway to an environment

`anypoint.yaml` — the key is the gateway's name in Runtime Manager:

```yaml
dev:
  gateways:
    fxf-dev-gtwy:
      id: 2a0abcce-051d-4199-9d5a-0bd5ca47ed94
    fxf-dev-gtwy-2:
      id: <the new gateway's id>
```

### Add a new environment

1. Declare it in `anypoint.yaml` with `environmentId`, `gatewayVersion` and `gateways`.
2. Add `envs/<app>/<newenv>/runtime.yaml` for every app.
3. If it should carry a `-SNAPSHOT` or `-RC` suffix, add it to the suffix list in
   `nexusCoords()` — otherwise it is treated as a release environment.

### Add an API to a release

`releases/<R>/manifest.yaml`:

```yaml
apps:
  - payments-api
  - my-api
```

### Deploy a subset

- One release: `RELEASE=R1`
- Everything: `RELEASE=ALL` (expands to R1–R4, in order)
- Named apps only: `RELEASE=ALL` plus `APP_FILTER=payments-api,orders-api`

`RELEASE=R1,R2` does **not** work — it is read as one literal name, no manifest matches,
and the build succeeds having deployed nothing.

---

## Field reference

### `apis/<app>/config.yaml`

| Key | Required | Meaning |
|---|---|---|
| `app` | yes | Application name |
| `assetId` | yes | Exchange asset identifier |
| `exchangeVersion` | yes | Exchange asset version |
| `configVersion` | yes | Current version of this contract |
| `endpoints[].name` | yes | Endpoint name; joins to `runtime.yaml` overrides |
| `endpoints[].publicPath` | yes | Path the route matches, and forwards to the host |
| `endpoints[].methods` | no | e.g. `[GET, POST]` — sent as a pipe-separated string |
| `endpoints[].headers` | no | Header name/value the route must match |
| `endpoints[].authPattern` | no | Documentation only; not read by the pipeline |
| `policies` | no | Map keyed by policy name; value is the overrides |

### `envs/<app>/<env>/runtime.yaml`

| Key | Required | Meaning |
|---|---|---|
| `app`, `env` | yes | Must match the file's own path |
| `deployVersion` | yes | Config version this environment deploys |
| `gateway` | yes | Gateway name from `anypoint.yaml`; there is no default |
| `proxyUri` | yes | Listener, **including base path** |
| `upstreamHost` | yes | Backend host, **scheme + host only, no path** |
| `endpoints` | no | Per-environment path overrides, keyed by endpoint name |

### Version suffixes

| Environment | Suffix | Nexus repo | Overwritable |
|---|---|---|---|
| `dev`, `test`, `sandbox`, `design` | `-SNAPSHOT` | `snapshot` | yes |
| `qa` | `-RC` | `release` | no |
| `preprod`, `prod` | *(none)* | `release` | no |

RC = release candidate: the QA-tested build that, if it passes, is promoted to the
plain version for preprod and prod.

---

## Pipeline stages

| # | Stage | Notes |
|---|---|---|
| 1 | Checkout Config | |
| 2 | Load Config | Reads and validates everything; most failures surface here |
| 3 | Resolve Config Version (Check Nexus) | Blocks a duplicate release version |
| 4 | Approval: Higher Environment Gate | Skipped for `dev`/`sandbox` and for `DRY_RUN` |
| 5 | Authenticate to Anypoint | After the gate, so the token cannot expire while waiting |
| 6 | Publish to Exchange | Skipped if the asset already exists |
| 7 | Create API Instance | Instance, upstream and routes reconciled |
| 8 | Deploy API Instance | Deploy, apply policies, poll for status |
| 9 | Archive Config to Nexus | Publish mode only; a replay is already published |
| 10 | Summary | Fails the build if any API failed |

**Parameters:** `ENVIRONMENT`, `RELEASE`, `APP_FILTER`, `DRY_RUN` (default **true**),
`SKIP_POLICIES`.

A release that fails does not stop later releases — deploys are wrapped in
`catchError`, and the build goes red at Summary.

---

## Jenkins prerequisites

| What | Value |
|---|---|
| Credential | `anypoint-connected-app-<env>` — Username+Password (CLIENT_ID / CLIENT_SECRET) |
| Credential | `3541669_nexus` — Username+Password, for config archiving |
| Managed file | `flex-gateway-approvers` — folder-scoped Properties file |
| Plugins | Pipeline Utility Steps, Config File Provider |
| Agent tools | `git`, `curl`, `python3` |

The approvers file holds:

```properties
TEST_QA_APPROVERS=integration-leads
PREPROD_PROD_APPROVERS=integration-leads
```

Approvers are deliberately **not** build parameters — a parameter is editable by
whoever triggers the build, which would let them approve their own deployment. Values
may be Jenkins user IDs or a group name; a group is preferable, since membership is
then managed in Access Management and joiners and leavers need no repo change.

---

## Troubleshooting

### `The port 8081 path / is already in use`

Anypoint's uniqueness key is **port + base path**. Two instances cannot both use
`http://0.0.0.0:8081`, whose base path is `/`.

Give each API its own base path — then any number of APIs share one port:

```yaml
proxyUri: http://0.0.0.0:8081/payments
```

This is the durable fix. Shuffling port numbers only postpones it.

### `HTTP 000` on a Nexus call

No response at all — the connection never completed, so it is not an auth or a
permissions problem. Nexus is internal and the corporate egress proxy cannot route to
it. `NO_PROXY` covers `.fedex.com`, and both Nexus calls pass `--noproxy '*'`.

If it persists, it is DNS or firewall from the agent. Confirm from the agent itself:

```bash
curl -ks --noproxy '*' -o /dev/null -w '%{http_code}\n' -I https://nexus.prod.cloud.fedex.com:8443/nexus/repository/snapshot/
```

`401` means connectivity is fine and the credential is the next thing to check.

### `HTTP 502 ... GatewayManagerError`

Usually a gateway `id` in `anypoint.yaml` that does not exist — a placeholder left in,
or an id copied from the wrong environment. Check the `targetId` in the failing
deployment against Runtime Manager.

### `Config artifact(s) already published to Nexus release repo`

A released version cannot be overwritten. Bump `deployVersion` in that environment's
`runtime.yaml`. Snapshots overwrite freely, so this only affects `qa`, `preprod` and
`prod`.

Note there is no `-RC1`/`-RC2` distinction: a rejected `1.0.0-RC` cannot be replaced,
so a fix needs a version bump even though `1.0.0` was never released.

### `Pinned config version X is not in Nexus (HTTP 404)`

`deployVersion` points at a version that was never published. Set it to one that
exists, or to the current `configVersion` to publish afresh.

### `The last upstream associated to an endpoint with proxy cannot be deleted`

Should no longer occur — the upstream in use is always kept. If it reappears, the job
is running an older Jenkinsfile than the repo.

### Duplicate upstreams in API Manager

Several upstreams with the same URL, only one carrying routes. The reconcile keeps one
and deletes the rest on the next run; no manual cleanup needed.

### `gateway missing from ...` / `app is 'x' but the file lives under ...`

Validation working as intended. `gateway` has no default, and `app`/`env` must agree
with the file's path — usually a `runtime.yaml` copied from another app or environment
and not fully edited.

### `Policy 'x' referenced in ... is not defined in policies/`

The name must match an **assetId** in `policies/*.yaml`, not a filename. The error
lists what is available.

### `upstreamHost ... must be scheme + host only`

The backend path belongs on the route rules, not the upstream. One upstream serves all
of an API's endpoints; the path distinguishes the routes.

### Deployment reports `gatewayStatus=UNKNOWN`

The pipeline warns and proceeds after several unknown polls, and marks the API OK. It
may well be serving traffic — confirm in Runtime Manager rather than assuming either
way.

### Build parameters still show a removed field

Jenkins renders the form from the last run's definitions. Run the job once and they
refresh, or delete them under Job → Configure → "This project is parameterised".
