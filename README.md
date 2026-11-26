# malevolent-shrine

This repository is a manifest set for [Shrine](https://github.com/CarlosHPlata/shrine). It holds no application code, only `shrine/v1` YAML manifests that the `shrine` CLI turns into containers, networks and gateway routes.

This guide shows how to add your own team, resources and applications.

> The [manifest schema reference](https://carloshplata.github.io/shrine/reference/manifest-schema/) is the source of truth for every field. Check it before you rely on a field that this guide does not show.

## Layout

```
.
├── teams/                     # One Team manifest per team
│   └── observability.yml
├── appDefinitions/            # One folder per team
│   └── observability/         #   that team's Resources and Applications
│       ├── orders.yml
│       ├── loki.yml
│       └── ...
└── traefik/                   # The gateway: the URLs exposed through Traefik
    ├── traefik.yml            #   static config (entrypoints, tracing, logs)
    └── dynamic/
        ├── tls.yml            #   tailnet certificate
        └── <team>-<app>.yml   #   one route file per exposed application
```

| Path | What goes there |
|------|-----------------|
| [`teams/`](teams/) | `Team` manifests. A team is a namespace with quotas. The file name matches the team's `metadata.name`. |
| [`appDefinitions/<team>/`](appDefinitions/) | The team's `Resource` (databases, storage, other dependencies) and `Application` (deployable containers) manifests. One file per manifest, named after its `metadata.name`. |
| [`traefik/`](traefik/) | The gateway configuration. The route files in `traefik/dynamic/` list the hosts and paths at which each application is reachable. |

## Adding your own manifests

### 1. Create a team

Add `teams/<team>.yml`:

```yaml
apiVersion: shrine/v1
kind: Team
metadata:
  name: my-team
spec:
  displayName: "My Team"
  contact: "you@example.com"
  quotas:
    maxApps: 3            # 0 or omitted = unlimited
    maxResources: 1
    allowedResourceTypes: # omitted = any type
      - postgres
```

Every Resource and Application you add must name this team in `metadata.owner`. Keep the quotas in line with what you add: a deploy that exceeds `maxApps`, `maxResources` or `allowedResourceTypes` is rejected.

### 2. Add resources (optional)

A Resource is a dependency container such as Postgres or Redis. Put it in `appDefinitions/<team>/<name>.yml`:

```yaml
apiVersion: shrine/v1
kind: Resource
metadata:
  name: my-db
  owner: my-team
spec:
  type: postgres
  version: "16"
  port: 5432
  env:                      # config for the resource's own container
    - name: POSTGRES_DB
      value: app
    - name: POSTGRES_PASSWORD
      generated: true
  outputs:                  # what other manifests are allowed to read
    - name: host
    - name: port
    - name: url
      template: "postgres://postgres:{{.POSTGRES_PASSWORD}}@{{.host}}:{{.port}}/{{.POSTGRES_DB}}"
  volumes:
    - name: data
      mountPath: /var/lib/postgresql/data
```

- `spec.env` is private to the resource's container. Only names listed under `spec.outputs` can be read by other manifests.
- `spec.port` is required when you export `port`.
- `spec.version` is required unless `imagePullPolicy: Pinned`.

### 3. Add applications

Put each Application in `appDefinitions/<team>/<name>.yml`:

```yaml
apiVersion: shrine/v1
kind: Application
metadata:
  name: my-api
  owner: my-team
spec:
  image: ghcr.io/me/my-api:1.0.0
  port: 8080
  replicas: 1
  env:
    - name: DATABASE_URL
      valueFrom: resource.my-db.url        # an output of a Resource
    - name: API_KEY
      valueFrom: vault:my-project/prod/API_KEY
    - name: LOG_LEVEL
      value: info
  routing:
    domain: my-api.my-team.internal
    pathPrefix: /api
  networking:
    exposeToPlatform: true                 # required for a gateway route
```

Each `env` entry sets exactly one of `value`, `valueFrom`, `template` or (Resources only) `generated`.

### Wiring manifests together

| Need | Use |
|------|-----|
| A Resource's exported value | `valueFrom: resource.<name>.<output>` |
| Another Application's address | `valueFrom: application.<name>.host` / `.port` |
| A value built from other env vars | `template: "http://{{.CART_HOST}}:{{.CART_PORT}}"` |
| A secret | `valueFrom: vault:<project>/<environment>/<secret-name>` (see the [secrets vault guide](https://carloshplata.github.io/shrine/guides/secrets-vault/)) |
| Start order without reading any value | `spec.dependencies: [{kind, name, owner}]` |

A `valueFrom` also makes Shrine deploy the target first. Two manifests that read from each other form a cycle, which Shrine rejects. [`appDefinitions/observability/customers.yml`](appDefinitions/observability/customers.yml) shows how to avoid one by building the address from `{{.team}}`.

Never commit plaintext secrets. Use `vault:` references or `generated: true`.

## Exposing an application through the gateway

Traefik is the gateway. An application gets a route when it has **both** a `spec.routing.domain` and `spec.networking.exposeToPlatform: true`. On deploy, Shrine writes `traefik/dynamic/<team>-<app>.yml`.

To serve the application on more than one host or path, add `routing.aliases`:

```yaml
routing:
  domain: my-api.my-team.internal
  aliases:
    # Reachable from the tailnet over HTTPS, under a path
    - host: apps.tail9a6ddb.ts.net
      pathPrefix: /my-api
      stripPrefix: true    # default when pathPrefix is set
      tls: true
    # The gateway's name on the platform network, for in-cluster callers
    - host: platform.traefik
      pathPrefix: /my-api
      stripPrefix: false
```

The hosts in use today:

| Host | Reached from |
|------|--------------|
| `<app>.<team>.internal` (the `domain`) | The local network / internal DNS |
| `apps.tail9a6ddb.ts.net` with `tls: true` | The tailnet, over HTTPS, using the certificate in [`traefik/dynamic/tls.yml`](traefik/dynamic/tls.yml) |
| `platform.traefik` | Other containers on the platform network |

If you only need another app to reach your container directly, without a public route, set `exposeToPlatform: true` and leave out `routing` ([`alloy.yml`](appDefinitions/observability/alloy.yml) is an example).

### Route files and generated output

- Don't edit `traefik/dynamic/<team>-<app>.yml` to change a route. Change the Application manifest instead.
- Shrine writes a route file once and keeps it on later deploys. After you change `routing`, delete the app's route file and run `shrine deploy` to regenerate it. Commit the regenerated file.
- `traefik/traefik.yml` and `traefik/dynamic/tls.yml` are maintained by hand. Traefik reads `traefik.yml` only at startup, so run `docker restart platform.traefik` after you edit it.
- `traefik/certs/` and a few platform-owned route files are ignored by git (see [`.gitignore`](.gitignore)).

## Deploying

Register teams first, then deploy:

```bash
shrine apply teams --path .
shrine deploy --dry-run --path .   # preview the changes
shrine deploy --path .
```

To deploy a single team, run `shrine deploy team <team>`. To remove a team's containers, run `shrine teardown <team>`. Teardown leaves the team's route files in `traefik/dynamic/`, so delete those yourself.

## Checklist for a new team

- [ ] `teams/<team>.yml` with quotas that cover everything you add
- [ ] `appDefinitions/<team>/` with one file per Resource and Application
- [ ] Each manifest has `metadata.owner: <team>`
- [ ] Each `valueFrom: resource.<name>.<key>` points at a key listed in that resource's `outputs`
- [ ] Secrets use `vault:` or `generated: true`, never plaintext
- [ ] Exposed apps have `routing.domain` and `networking.exposeToPlatform: true`
- [ ] `shrine deploy --dry-run` passes, and the new `traefik/dynamic/<team>-*.yml` files are committed
