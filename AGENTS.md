# AGENTS.md

## What this repository is

This repository holds no application code. Every YAML file in it (except the generated Traefik config) is a **Shrine manifest** — a declarative definition consumed by the `shrine` CLI to deploy Docker-based apps and their dependencies.

Shrine project: https://github.com/CarlosHPlata/shrine

Before editing or generating any manifest here, read the schema reference — do not guess field names or behavior from examples alone:

**https://carloshplata.github.io/shrine/reference/manifest-schema/**

If deeper behavior needs confirming beyond the schema page (e.g. how vault resolution or templating actually executes), check the rest of the docs under https://carloshplata.github.io/shrine/ or the `shrine` source at https://github.com/CarlosHPlata/shrine before assuming.

## The three manifest kinds

Every manifest starts with `apiVersion: shrine/v1` and a `kind`:

- **Team** — a namespace with quotas (`maxApps`, `maxResources`, `allowedResourceTypes`). Must be applied (`shrine apply teams`) before any Resource/Application that references it as `metadata.owner`.
- **Resource** — a managed dependency container (Postgres, Redis, MariaDB, etc.). Has two distinct, non-overlapping blocks:
  - `spec.env` — runtime config injected into the resource's *own* container.
  - `spec.outputs` — a strict export allowlist of what other manifests may read via `valueFrom`. An env var not listed in `outputs` is private to the resource.
- **Application** — a deployable container with routing (`spec.routing`), dependency wiring (`spec.dependencies`, `spec.env[].valueFrom`), and optional platform exposure / localhost publishing (`spec.networking`).

Full field-by-field detail (required/optional, defaults, `valueFrom` reference formats like `resource.<name>.<output>` and `vault:<project>/<env>/<key>`, templating rules, `networking.publish` port ranges, alias/routing rules) is in the schema reference linked above — treat it as the source of truth over anything inferred from existing files in this repo.

## How this repo is organized

See [README.md](README.md) for the concrete layout: which Team manifest owns which Resources/Applications, and where each lives (`ops/`, `test/`, top-level Team files). `traefik/` is generated output from applications with `networking.exposeToPlatform: true` — don't hand-edit it to fix a routing problem; fix the owning Application manifest and redeploy.

## Working conventions for this repo

- A Resource/Application's `metadata.owner` must match an already-defined Team's `metadata.name`.
- Cross-manifest references use `valueFrom: resource.<name>.<output>` or `application.<name>.<host|port>` — the referenced name must expose that key under its own `outputs` (Resources) or be a valid built-in (Applications).
- Secrets come from `vault:<project>/<environment>/<secret-name>` — see the Secrets vault guide linked from the schema reference, not ad-hoc plaintext values.
- Validate field requirements against the schema before assuming a shape is valid — e.g. `spec.port` is required on a Resource only when `port` is exported; each `env` entry sets exactly one of `value`/`valueFrom`/`template`/`generated`.
