# CBA Hands-On Labs — Organized by Exam Domain

Target environment: your existing Backstage on EKS + a local clone on macOS for dev-workflow labs.
Exam weights: Customizing 32% · Development Workflow 24% · Infrastructure 22% · Catalog 22%

---

## Domain 1: Backstage Catalog (22%)

### Lab 1.1 — Register all entity kinds and observe relations

Create a single YAML file with the full entity model and register it. This cements the kind hierarchy (Domain → System → Component/API/Resource) that the exam loves.

```yaml
# catalog-demo.yaml
apiVersion: backstage.io/v1alpha1
kind: Domain
metadata:
  name: payments
spec:
  owner: group:default/platform-team
---
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: checkout
spec:
  owner: group:default/platform-team
  domain: payments
---
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: checkout-api
  annotations:
    github.com/project-slug: yourorg/checkout-api
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: group:default/platform-team
  system: checkout
  providesApis: [checkout-rest]
  dependsOn: [resource:checkout-db]
---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: checkout-rest
spec:
  type: openapi
  lifecycle: production
  owner: group:default/platform-team
  system: checkout
  definition: |
    openapi: 3.0.0
    info: { title: Checkout, version: 1.0.0 }
    paths: {}
---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: checkout-db
spec:
  type: database
  owner: group:default/platform-team
  system: checkout
---
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: platform-team
spec:
  type: team
  children: []
---
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: jacky
spec:
  memberOf: [platform-team]
```

Steps:
1. Push this file to a Git repo (or serve via raw URL).
2. Register via UI: **Create → Register Existing Component** → paste URL. Note that this creates a `Location` entity behind the scenes — check **Catalog → filter Kind=Location** to see it.
3. Open `checkout-api` and inspect the **Relations** in the About card and the graph view: `partOf`, `ownedBy`, `providesApi`, `dependsOn`.
4. Delete the entity from the UI, then watch it reappear (the Location still exists — this is a classic exam gotcha: entities are re-ingested from their location; you must unregister the Location).

Exam points to internalize:
- Which fields are required per kind (`spec.type`, `spec.lifecycle`, `spec.owner` for Component).
- `metadata.name` vs `metadata.title`; namespace defaults to `default`.
- Well-known annotations: `backstage.io/techdocs-ref`, `github.com/project-slug`, `backstage.io/kubernetes-id`, `backstage.io/managed-by-location`.

### Lab 1.2 — Static locations vs discovery

In `app-config.yaml`:

```yaml
catalog:
  rules:
    - allow: [Component, System, API, Resource, Domain, Location, User, Group, Template]
  locations:
    - type: url
      target: https://github.com/yourorg/catalog/blob/main/catalog-demo.yaml
    - type: file
      target: ../../examples/entities.yaml
      rules:
        - allow: [User, Group]
```

Experiments:
1. Remove `Template` from the global `rules.allow` and try to register a Template — observe the rejection error. This teaches how catalog rules gate ingestion.
2. Note per-location `rules` override behavior.
3. Set `catalog.processingInterval` (e.g. `{ minutes: 5 }`), edit the YAML in Git, and time how long until the change appears. Understand: processors run on a loop; refresh is not instant and doesn't require restart.

### Lab 1.3 — Entity providers vs processors (conceptual + config)

Add GitHub org discovery (or just read the config carefully if you don't have an org):

```yaml
catalog:
  providers:
    github:
      myorg:
        organization: 'yourorg'
        catalogPath: '/catalog-info.yaml'
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

Know the distinction cold: **providers** fetch entities from external sources into the processing pipeline; **processors** validate/enrich/emit relations during processing. `LocationEntityProcessor`, `BuiltinKindsEntityProcessor` are processors.

---

## Domain 2: Customizing Backstage (32%) — biggest domain

### Lab 2.1 — Install a frontend + backend plugin pair (do this locally)

Install the Kubernetes plugin (fits your environment perfectly):

```bash
# from repo root
yarn --cwd packages/app add @backstage/plugin-kubernetes
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```

Frontend wiring in `packages/app/src/components/catalog/EntityPage.tsx`:

```tsx
import { EntityKubernetesContent } from '@backstage/plugin-kubernetes';

// inside serviceEntityPage:
<EntityLayout.Route path="/kubernetes" title="Kubernetes">
  <EntityKubernetesContent refreshIntervalMs={30000} />
</EntityLayout.Route>
```

Backend wiring in `packages/backend/src/index.ts` (New Backend System):

```ts
backend.add(import('@backstage/plugin-kubernetes-backend'));
```

app-config:

```yaml
kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: https://<your-eks-endpoint>
          name: eks-prod
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_SA_TOKEN}
          skipTLSVerify: false
          caData: ${K8S_CA_DATA}
```

Then annotate a component: `backstage.io/kubernetes-id: checkout-api` and label your workloads `backstage.io/kubernetes-id=checkout-api`. Verify pods show in the Kubernetes tab.

Exam takeaways: frontend plugins are added to `packages/app`, backend plugins to `packages/backend`; new backend system uses `backend.add(import(...))` — one line, no router wiring.

### Lab 2.2 — Create your own plugin

```bash
yarn new
# choose: frontend-plugin, id: hello-world
yarn dev
# visit http://localhost:3000/hello-world
```

Then a backend plugin:

```bash
yarn new
# choose: backend-plugin, id: hello-world
curl http://localhost:7007/api/hello-world/health
```

Inspect what `yarn new` generated:
- `plugins/hello-world/src/plugin.ts` — `createPlugin`, `createRoutableExtension`
- `plugins/hello-world/src/routes.ts` — `RouteRef`
- how the plugin is registered in `packages/app/src/App.tsx` (`<Route path="/hello-world" element={<HelloWorldPage />} />`)
- backend: `createBackendPlugin`, `coreServices` dependency injection (httpRouter, logger, config)

Exam loves: `createPlugin` vs `createRoutableExtension` vs `createComponentExtension`, RouteRefs, and `coreServices.*` names.

### Lab 2.3 — Customize the app: theme, sidebar, homepage

1. Sidebar: edit `packages/app/src/components/Root/Root.tsx` — add a `SidebarItem` linking to your hello-world plugin.
2. Theme: in `packages/app/src/App.tsx`, add a custom theme via `createUnifiedTheme` and `app: { themes: [...] }` — change the primary color, verify in UI.
3. Change the app title/org in app-config:

```yaml
app:
  title: RJF Trust IDP
organization:
  name: RJF Trust
```

Key exam fact: `app.title` and org name changes require a frontend **rebuild** in production (config is baked in at build time), but hot-reload locally.

### Lab 2.4 — Customize the software templates (scaffolder)

Create a minimal template:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-service
  title: Node.js Service
spec:
  owner: group:default/platform-team
  type: service
  parameters:
    - title: Fill in details
      required: [name]
      properties:
        name:
          title: Name
          type: string
          ui:field: EntityNamePicker
  steps:
    - id: fetch
      name: Fetch skeleton
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
    - id: publish
      name: Publish
      action: publish:github
      input:
        repoUrl: github.com?owner=yourorg&repo=${{ parameters.name }}
    - id: register
      name: Register
      action: catalog:register
      input:
        catalogInfoUrl: ${{ steps.publish.output.repoUrl }}/blob/main/catalog-info.yaml
  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
```

Register it, run it, then break it deliberately (wrong action name, missing required parameter) and read the errors. Know: `apiVersion: scaffolder.backstage.io/v1beta3`, parameters use JSON Schema + `ui:` extensions, steps run built-in actions (`fetch:template`, `publish:github`, `catalog:register`), `${{ }}` templating syntax.

### Lab 2.5 — TechDocs end to end

You've covered TechDocs modes in study — now do it:

```yaml
techdocs:
  builder: 'local'          # vs 'external'
  generator:
    runIn: 'local'          # vs 'docker'
  publisher:
    type: 'local'           # vs awsS3 / googleGcs / azureBlobStorage
```

1. Add `mkdocs.yml` + `docs/index.md` to a repo, annotate the entity with `backstage.io/techdocs-ref: dir:.`, view the Docs tab.
2. Switch `publisher.type` to `awsS3` with a bucket and redeploy on EKS — this is the recommended production pattern (build in CI, publish to S3, Backstage serves from S3).

Exam distinction: `builder: local` = Backstage builds docs on the fly (fine for demo); `builder: external` = CI builds and publishes, Backstage only reads.

---

## Domain 3: Backstage Development Workflow (24%) — must be done locally

### Lab 3.1 — Scaffold and run from scratch

```bash
npx @backstage/create-app@latest
cd my-backstage-app
yarn install
yarn dev        # frontend :3000 + backend :7007 together
```

Explore the monorepo:
- `packages/app` — React frontend
- `packages/backend` — Node backend
- `plugins/` — your plugins
- root `package.json` — yarn workspaces
- `app-config.yaml` vs `app-config.local.yaml` vs `app-config.production.yaml` (merge order: later files override earlier; `--config` flags control which load)

Run these and understand what each does:

```bash
yarn tsc            # type-check (noEmit against a full compile? check tsconfig)
yarn build:backend  # bundles backend
yarn build:all
yarn test
yarn lint
yarn backstage-cli versions:bump   # upgrade all @backstage packages consistently
```

### Lab 3.2 — Build the Docker image

Host build (the documented default):

```bash
yarn install --immutable
yarn tsc
yarn build:backend
docker build . -f packages/backend/Dockerfile -t backstage:local
docker run -p 7007:7007 backstage:local
```

Key facts the exam checks:
- The production Docker image runs **only the backend**; the frontend is built into static assets served by the backend (`app-backend` plugin).
- `yarn build:backend` creates `packages/backend/dist/bundle.tar.gz` consumed by the Dockerfile.
- Multi-stage builds are the alternative when you want the build inside Docker.

### Lab 3.3 — Config-driven behavior

```bash
yarn start --config app-config.yaml --config app-config.local.yaml
NODE_ENV=production node packages/backend --config app-config.yaml --config app-config.production.yaml
```

Test env var substitution: `${MY_SECRET}` in config, run with and without the variable set, observe the failure mode. Also test `$include:` and `$env:` syntax.

---

## Domain 4: Backstage Infrastructure (22%) — your EKS deployment

### Lab 4.1 — Production config hygiene

Review your EKS deployment against these:

```yaml
app:
  baseUrl: https://backstage.rjftrust.com
backend:
  baseUrl: https://backstage.rjftrust.com
  listen: ':7007'
  cors:
    origin: https://backstage.rjftrust.com
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${GITHUB_CLIENT_ID}
        clientSecret: ${GITHUB_CLIENT_SECRET}
```

Exercises:
1. If you're on SQLite (`client: better-sqlite3`), migrate to Postgres and confirm catalog survives pod restarts. Know: SQLite is in-memory/ephemeral by default — dev only.
2. Break `app.baseUrl` vs `backend.baseUrl` deliberately (mismatch) and observe CORS/auth failures. Understand why frontend needs backend.baseUrl baked in at build time.
3. Scale the deployment to 2 replicas — works fine with Postgres, breaks with SQLite. Understand why (shared DB = stateless backend).

### Lab 4.2 — Auth provider + sign-in resolver

Add GitHub (or Google) OAuth:

1. Create OAuth app, callback: `https://backstage.rjftrust.com/api/auth/github/handler/frame`.
2. Configure as above; wire `SignInPage` in `packages/app/src/App.tsx`:

```tsx
import { githubAuthApiRef } from '@backstage/core-plugin-api';
components: {
  SignInPage: props => (
    <SignInPage {...props} auto provider={{
      id: 'github-auth-provider',
      title: 'GitHub',
      message: 'Sign in with GitHub',
      apiRef: githubAuthApiRef,
    }} />
  ),
},
```

3. Backend: `backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));` and configure a **sign-in resolver** (`resolvers: [usernameMatchingUserEntityName]`) — sign-in fails if no matching User entity exists unless you use `dangerouslyAllowSignInWithoutUserInCatalog`. That failure mode is exam material — trigger it on purpose.

### Lab 4.3 — Proxy backend

```yaml
proxy:
  endpoints:
    '/prometheus/api':
      target: http://prometheus.monitoring.svc:9090/api/v1
      changeOrigin: true
      credentials: dangerously-allow-unauthenticated
      headers:
        Authorization: Bearer ${PROM_TOKEN}
```

Then `curl https://backstage.rjftrust.com/api/proxy/prometheus/api/query?query=up`. Know: proxy exists so the frontend can reach services without CORS issues and without exposing secrets to the browser; keys stay server-side.

### Lab 4.4 — Client-server architecture drill

Answer these from your running system (all exam-style):
- Which process serves the frontend static assets in production? (backend, via app-backend plugin)
- Where do backend plugins mount? (`/api/<plugin-id>`)
- What happens to `app-config` values referenced by frontend code at build time? (inlined into the JS bundle — secrets must never be in frontend-visible keys)
- Which config keys are frontend-visible? (those under `app.*` and any marked `visibility: frontend` in schema)

---

## Suggested schedule (2 weeks)

| Days | Labs |
|------|------|
| 1–2  | 3.1, 3.2, 3.3 (local from scratch — foundation for everything) |
| 3–4  | 1.1, 1.2, 1.3 (catalog) |
| 5–8  | 2.1 → 2.5 (customizing — biggest weight, take your time) |
| 9–11 | 4.1 → 4.4 (infrastructure on EKS) |
| 12–14| Re-read app-config reference, plugin docs, mock questions |

## Break-things checklist (exam questions are mostly "why is this broken")

- Register an entity with missing `spec.owner` → read the processor error
- Annotate techdocs-ref wrong (`url:` vs `dir:`) → observe docs failure
- Mismatch OAuth callback URL → read the auth error
- Remove a kind from catalog rules → registration rejected
- Frontend plugin installed but route not added → page 404s
- Backend plugin added to config but not `backend.add()` → API 404s
