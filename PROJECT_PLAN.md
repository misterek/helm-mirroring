# Helm Mirroring — Project Plan

Status: draft, pending review.

## Overview

This repo hosts a system that vendors upstream Helm charts (and the container images they reference) into registries the organization controls. It runs entirely on GitHub Actions. A daily detector workflow proposes one PR per new chart version, with scan results attached. Human PR merge approves the vendoring. A vendor workflow then mirrors images by digest, rewrites the chart's `values.yaml` to point at our registry, signs the result, and commits the vendored artifact back to the repo.

The goal is to **eliminate the default footgun**: a consumer who `helm install`s a vendored chart with no overrides never reaches out to upstream for a chart or image.

---

## Goals

1. **Eliminate the default footgun.** A vendored chart installed with zero overrides *of its default values* pulls every image from our registry, pinned by digest. A consumer who subsequently toggles a feature flag (e.g., `autoscaling.enabled: true`) may reach upstream for an image the chart gates behind that flag — that's a documented limitation, not a silent failure (see "Image discovery → conditional images").
2. **Human-in-the-loop approval.** No chart or image is vendored without a human merging a PR that carries scan results for that specific chart version.
3. **Immutable references.** Images are addressed by digest. Upstream tag churn (`ruby:3.14` silently becoming a different binary) cannot propagate into deployments.
4. **Pluggable scanning.** Chart-level and image-level scanners can be added without touching the core pipeline. Multiple scanners can run against the same artifact.
5. **Per-chart failure isolation.** A bad chart (unscan-able, missing signature, discovery failure) must not block the other charts in the same daily run.
6. **Auditable trail.** For any image in the mirror, we can answer: where did it come from, when was it first mirrored, which scanners approved it, and which vendored charts reference it.

## Assumptions

- **Single repo.** Config, vendored charts, manifest, and workflows all live together. (Splitting is a future option.)
- **Two registry backends.** Zot (local, docker-compose) for dev/test and ECR for production, abstracted behind a thin interface.
- **GHA is the only execution environment.** OIDC is the auth path to AWS.
- **Upstream charts are public.** v1 does not handle private chart repos or private image registries.
- **Consumers accept the rewrite contract.** If a consumer overrides `image.repository` themselves, they're responsible for pointing at the mirror. We only fix the chart default.
- **One PR per chart version.** Daily detector runs may open many PRs. The human reviewer approves/closes each independently.
- **Approval model is PR merge.** One approving review is sufficient to merge; branch protection enforces this.
- **Digest-pinned images are tolerable to consumers.** Most modern charts accept `sha256:…` in `image.tag` or expose `image.digest`. Charts that hardcode images in templates (not values) require a manual config override.
- **Daily cadence.** Weekly is too slow; hourly is noisy. Off-peak cron.

## In scope (v1)

- Config-driven chart list with per-chart version floor.
- Daily detector workflow opening one PR per new chart version.
- Vendor workflow on PR merge that mirrors images by digest, rewrites `values.yaml`, and commits the vendored chart.
- Mirror-manifest file as source of truth.
- Zot test harness + ECR production target behind one registry interface.
- Pluggable scanners: trivy, grype, kubelinter, helm lint, cosign verify, syft SBOM in v1.
- Per-chart isolation and idempotent re-runs.
- Cosign signing of mirrored artifacts with an org-owned KMS key.

## Out of scope (v1, intentionally)

- **Subchart / dependency vendoring.** Charts that declare `Chart.yaml` dependencies (subcharts) are not transitively vendored. Detector detects `Chart.yaml: dependencies:` and fails the chart with a specific error directing the operator to either (a) wait for subchart support in a later release, or (b) vendor the subchart separately as its own top-level entry and use Helm's `dependencies:` rewrite at install time. **Honest scope statement:** of popular charts, roughly half have meaningful subchart deps (`argo-cd`, `kube-prometheus-stack`, `bitnami/*` ship with redis/postgres subcharts). v1 is genuinely usable for single-chart workloads (`ingress-nginx`, `cert-manager`, `external-dns`, `metrics-server`) and charts whose deps are only CRDs. Subchart traversal is the #1 v2 item.
- **Private upstream sources.** Private chart repos and private upstream registries.
- **Cross-registry / cross-region replication.** Single primary target.
- **Auto-pruning of mirror.** "Unreferenced images" report only; humans delete.
- **SLSA provenance beyond cosign-signing.** We sign what we mirror; we do not generate full SLSA L3 attestations in v1.
- **Non-helm artifacts.** Helm only.
- **UI / dashboard.** GitHub PRs, job summaries, and the manifest file are the interface.

## Success criteria

- From an empty repo, a developer can: add a chart (**single-chart, no subchart deps** in v1) to config → next detector run → PR with scan results → merge → vendored chart appears under `vendored/<chart>/<version>/` with a rewritten `values.yaml` pointing at zot in local testing.
- Same flow in production pushes to ECR with images pinned by digest and signed by our cosign key.
- When upstream publishes a **new chart version**, the detector picks it up and the PR for that new version mirrors whatever image digests it references — including newly-rolled tags upstream has silently updated. Already-vendored chart versions remain pinned to their original digests and are **not** re-resolved (that's the point of digest pinning; our consumers are safe).
- **Security alert path:** if the detector observes that an already-vendored chart version's upstream **chart-tarball digest** has changed (a SemVer violation or upstream tampering signal), it opens a tracking *issue* (not a PR — there is no new version to vendor) so a human can investigate. Image-digest drift inside an already-vendored chart version is expected (tags are mutable) and does not trigger an alert.
- A chart that cannot be honestly vendored (template-hardcoded images, partially-templated registry, subchart deps) **fails loudly** with an error directing the operator to a specific fix — never silently produces a broken vendored chart.
- One chart in a 10-chart daily batch fails scanning → the other 9 PRs still open.
- Two new versions of the same chart on the same day open two PRs that can both be merged without a human-rebase step.
- "Where did this image come from?" answered by grepping the manifest for a digest.
- The system is genuinely usable on at least four real charts end-to-end in v1: `ingress-nginx`, `cert-manager`, `external-dns`, `metrics-server`.

---

## System Flow (end-to-end)

```
                       ┌──────────────────────────────┐
                       │  config/charts.yaml          │
                       │  (human-edited chart list)   │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  detect.yml  (daily cron, GHA matrix fan-out per chart)     │
    │  • list upstream versions ≥ floor, ≤ N newest               │
    │  • pull candidate chart, discover images (fail-loud paths)  │
    │  • run scan plugins INLINE in this job (chart + per-image)  │
    │  • post sticky scan-summary comment from this same job      │
    │  • open PR (needs GitHub App token to fire required checks) │
    │      branch: mirror/<chart>-<ver>                           │
    │      writes:  manifest/pending/<chart>-<ver>.yaml           │
    │              manifest/scans/<chart>/<ver>/*.json            │
    │      does NOT touch the canonical mirror-manifest.yaml      │
    └──────────────────────────────┬──────────────────────────────┘
                                   │ (human review + merge)
                                   ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  vendor.yml  (trigger: push to main, paths:                 │
    │                        manifest/pending/**)                 │
    │  • skopeo copy images by digest → our registry              │
    │  • helm push chart → our registry                           │
    │  • rewrite values.yaml in-place                             │
    │  • cosign sign images + chart with our KMS key              │
    │  • syft → SBOM attestation                                  │
    │  • consolidate: move manifest/pending/<c>-<v>.yaml into     │
    │    canonical manifest/mirror-manifest.yaml, delete pending  │
    │  • commit (uses GITHUB_TOKEN → does not re-fire workflows)  │
    └─────────────────────────────────────────────────────────────┘
```

Consumers then `helm install` from our registry; with no overrides to default values, they pull images from our registry by digest.

**Why two manifest locations.** The canonical `mirror-manifest.yaml` would become a merge-conflict hot spot if every detector PR touched it (a day with 10 new versions → 10 PRs editing overlapping YAML lines). Splitting detector output into per-version pending files (`manifest/pending/<chart>-<version>.yaml`) means two PRs for the same chart can be merged independently without a human rebase. Vendor is the **sole writer** of `mirror-manifest.yaml`, which makes concurrency reasoning trivial.

---

## Repo Layout

```
helm-mirroring/
├── .github/workflows/
│   ├── detect.yml              # daily detector (scanning inlined, not a separate workflow)
│   ├── vendor.yml              # PR-merge vendoring
│   └── test.yml                # integration tests vs zot
├── cmd/mirrorctl/              # Go CLI entry point
├── internal/                   # config, detect, vendor, scan, manifest, registry
│   └── registry/
│       ├── zot/                # local dev backend
│       └── ecr/                # production backend
├── config/
│   └── charts.yaml             # human-edited chart list + version floors
├── manifest/
│   ├── mirror-manifest.yaml    # canonical source of truth; VENDOR IS SOLE WRITER
│   ├── pending/                # detector output; one file per proposed {chart, version}
│   │   └── <chart>-<version>.yaml
│   ├── scanners.yaml           # scan plugin registration
│   ├── scans/
│   │   ├── images/sha256-<digest>/<chart>-<version>/*.json  # scoped by chart-version so two
│   │   │                                                    # PRs touching the same digest don't
│   │   │                                                    # clobber each other (and retention keeps each)
│   │   └── charts/<chart>-<version>/*.json
│   ├── sboms/
│       └── sha256-<digest>/<chart>-<version>.spdx.json
│   └── upstream-keys/          # cosign public keys for keyed upstream verification
├── scanners/                   # bundled scan plugin executables
├── vendored/<chart>/<version>/ # rewritten chart tarball + values.yaml
├── testdata/
│   ├── charts/                 # fixture upstream charts (includes adversarial fixtures for discovery)
│   └── docker-compose.yml      # zot + fixture upstream registry
├── Makefile
└── go.mod
```

`manifest/` is the single canonical home for mirror state. `config/` is the human-edited input. `vendored/` is the machine-written output. `cmd/mirrorctl` is the Go binary invoked from workflow steps.

**Filesystem-safe digest encoding.** OCI digests contain a colon (`sha256:abc…`) which is awkward on Windows checkouts and in some tooling. Throughout the repo we encode digests as `sha256-<hex>` on disk (dash, not colon). The canonical manifest keeps the colon form since it's YAML.

---

## Vendoring Pipeline

This section describes the mechanics of pulling an approved upstream chart version into the repo and registry. The vendor workflow (triggered on merge of a detector PR) performs everything below.

### Config file schema (`config/charts.yaml`)

One entry per chart. Small, hand-edited.

```yaml
# config/charts.yaml
defaults:
  target_registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  target_prefix: mirror                # <registry>/mirror/...
  scan_policy: standard                # named policy in manifest/scanners.yaml

charts:
  - name: ingress-nginx
    source:
      type: classic                    # "classic" (index.yaml) | "oci"
      repo: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    version_floor: 4.10.0              # ignore anything < this
    target_path: charts/ingress-nginx
    image_target_path: images          # <registry>/mirror/images/<name>

  - name: cert-manager
    source:
      type: oci
      repo: oci://quay.io/jetstack/charts
    chart: cert-manager
    version_floor: v1.14.0
    target_path: charts/cert-manager
    # Minimal values needed to make `helm template` render at all. NOT a vendor-time
    # override — only used during image discovery. Keeps the "chart refuses to
    # render with /dev/null" case handleable without forking the chart.
    discovery_values:
      installCRDs: true
    # manual_images pre-mirrors images the chart CAN reach but doesn't render
    # by default (conditional features the operator knows will be enabled).
    # It does NOT rewrite the chart or fix Goal #1 for template-hardcoded
    # images — those hard-fail with no escape hatch.
    manual_images:
      - upstream: quay.io/jetstack/cert-manager-acmesolver:v1.14.5
        values_path: acmesolver.image   # MUST point at a real values path that
                                        # reaches this image. A null values_path
                                        # is rejected at validation time.
    # Chart-level scan policy REPLACES the default entirely (not a merge).
    # If you want to extend the default, redeclare its scanners explicitly.
    scan_policy: strict
```

Fields:
- `source.type`: `classic` → `helm repo add` + `helm pull`; `oci` → `helm pull oci://…`.
- `version_floor`: SemVer; pre-releases skipped unless floor is a pre-release.
- `discovery_values`: optional minimal values overlay used *only* during image discovery. Charts that require inputs to render at all (license-accept booleans, required JSON-schema fields) need this to avoid failing discovery before it starts.
- `manual_images`: pre-mirror images the chart reaches via a values path but doesn't render by default (conditional features). Each entry **must** supply a `values_path` that a consumer would override to enable the feature; the discovery pass will use that path to confirm the image actually reaches the named values key. `manual_images` is **not** a fix for template-hardcoded images — those hard-fail with no escape (see "Classify rewrite scope" below). `values_path: null` is rejected at config validation.
- `scan_policy`: named policy (definitions live in `manifest/scanners.yaml`). **Chart-level `scan_policy` replaces the default entirely — it is not merged.** If you want to add one scanner on top of the default, redeclare the full policy for that chart.

### Image discovery

Input: chart tarball at a pinned version. Output: `DiscoveredImage{ upstream_ref, values_path|null, source: "template"|"manual" }`.

0. **Reject subchart deps up front.** If `Chart.yaml` declares `dependencies:`, fail the chart with a pointer to the out-of-scope note in Goals. No further work.
1. `helm pull` (classic) or `helm pull oci://…` into a temp dir. Keep the tarball — it's what we re-push.
2. `helm template discovery <chart> --values <config.discovery_values or /dev/null>` — release name is pinned to the constant `discovery` so reruns are byte-stable. Capture rendered YAML as **baseline**.
3. **Normalize non-determinism.** Helm has stdlib functions that vary by invocation (`now`, `randAlphaNum`, `uuidv4`, `genSelfSignedCert`). Apply a regex mask pass to the baseline: replace `genSelfSignedCert` PEM blocks, RFC3339 timestamps, and 32+-char random strings with fixed tokens. Every subsequent sentinel re-render gets the same mask. If the masked baseline is still unstable across two back-to-back renders, fail with a clear message directing the operator to add the chart to a non-deterministic allow-list (and fall back to `manual_images`).
4. Walk every rendered doc. Collect image refs from:
   - `spec.template.spec.containers[].image`
   - `spec.template.spec.initContainers[].image`
   - `spec.jobTemplate.spec.template.spec.containers[].image` (CronJob)
   - Any `image:` string on objects of kind `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`, `ReplicaSet`, `Rollout`.
   - Match by JSONPath pattern, not type, to catch CRDs that embed pod specs.
5. Deduplicate refs.
6. **Values-path resolution (sentinel re-render).** For each discovered ref:
   - Load the chart's default `values.yaml` (plus `discovery_values` overlay if present).
   - For every leaf string value that looks image-shaped (`repo/name:tag`, `repo/name`, or a bare tag), mutate it to a sentinel `SENTINEL_<sha1(path)>` and re-render.
   - Diff rendered-masked output against the baseline-masked output. Changed images map to the values path that produced them.
   - For split form (`image.repository` + `image.tag` (+ optional `image.registry`) under the same parent), treat the triple as one logical ref; sentinel each field and record `values_path = <parent>.image`.
7. **Classify rewrite scope (full / partial / none).** For each discovered ref, determine which components of the image reference — **registry host**, **repository path**, **tag**, and **digest** — actually move when the corresponding values are mutated.
   - **Full:** mutating the values path shifts the registry host, the repo path, *and* the tag (or the chart supports an explicit `image.digest` field we can set). Safe to rewrite. Proceed.
   - **Partial:** mutating the values path shifts only some components — typically the classic case of `image: "registry.k8s.io/foo:{{ .Values.tag }}"` where the registry is hardcoded in the template. **Cannot be vendored honestly.** A rewrite would produce a broken `values.yaml` (our tag on upstream's registry); mirroring the image without a rewrite still leaves the default install reaching upstream — Goal #1 violated silently. Hard fail the vendor run:
     ```
     ERROR: chart foo 1.2.3 renders image 'registry.k8s.io/foo:v1.2.3' whose
     values path '.tag' only controls tag, not registry. This chart cannot be
     vendored without patching its templates (out of scope for v1). Remove
     this chart from config/charts.yaml or wait for template-patching support.
     ```
   - **None:** no values path affects the image. Template-hardcoded. Same hard fail — `manual_images` does **not** close this gap because it only pre-mirrors the image; the chart template still references upstream at render time.
   - **Ambiguous (tpl construction):** if mutating multiple independent values paths all affect the same rendered image (e.g. chart does `{{ tpl .Values.imageTemplate . }}`), the rewriter has no principled way to feed our mirror back through `tpl`. Hard fail, same error.
8. Merge `manual_images` onto the discovered list before proceeding.

**Why sentinel-diff, not static values.yaml parsing.** Values files use templating (`{{ .Values.global.imageRegistry }}`) and conditionals; static parsing gets the 80% case and lies about the rest. Rendering is ground truth. Cost is a few re-renders per chart — seconds, not minutes.

**Conditional images.** Features disabled by default (`autoscaling.enabled: false`) won't render in the baseline and won't be mirrored by default discovery. Goal #1 is "zero overrides of default values" — toggling a feature is an override. Documented known limitation; callable by consumers via `manual_images` if they know they'll enable the feature.

### Mirror state: two files, two writers

To avoid the daily merge-conflict hot spot that would follow if every detector PR edited a shared file, we split mirror state across **two locations** with **two distinct writers**:

| File | Writer | Purpose |
|---|---|---|
| `manifest/pending/<chart>-<version>.yaml` | Detector (one file per proposed version) | Candidate vendoring proposal + scan results |
| `manifest/mirror-manifest.yaml` | Vendor (sole writer) | Canonical record of what has been mirrored |

Two detector PRs for different versions of the same chart edit disjoint files — they can merge independently with zero rebase work.

#### Pending-file schema (`manifest/pending/<chart>-<version>.yaml`)

Written by the detector when scans complete. Committed to the PR branch. This is the file that, once merged to `main`, fires `vendor.yml`.

```yaml
# manifest/pending/ingress-nginx-4.10.2.yaml
schema_version: 1
chart: ingress-nginx
version: 4.10.2
proposed_at: 2026-04-23T06:17:00Z
upstream_source: https://kubernetes.github.io/ingress-nginx
upstream_chart_digest: sha256:b1c3...      # digest of the chart tarball itself
upstream_signed: true                       # upstream chart signature verification
images:
  - upstream_ref: registry.k8s.io/ingress-nginx/controller:v1.10.1
    upstream_digest: sha256:4e3b...c91a
    values_path: controller.image
    rewrite_scope: full                    # full | none (never partial — partial errors earlier)
    discovery_source: template             # template | manual
    upstream_signed: true
    scan_refs:                             # under manifest/scans/images/sha256-<digest>/<chart>-<version>/
      - trivy.json
      - grype.json
      - cosign-verify.json
    sbom_ref: manifest/sboms/sha256-4e3bc91a/ingress-nginx-4.10.2.spdx.json
scan_refs_chart:                           # filenames under manifest/scans/charts/<chart>-<version>/
  - kubelinter.json
  - helm-lint.json
scan_verdict: pass                         # pass | fail | error  (aggregate across all scanners)
```

The detector writes no `mirror_repo`, no `first_seen`, no `used_by` — those fields belong to vendor-time. It **does** write full scan result refs, upstream signature verification, and SBOMs so the PR contains complete review material and the approver judges on facts, not promises.

#### Canonical manifest (`manifest/mirror-manifest.yaml`)

Vendor-only writer. Digest-keyed `images:` map; dedup across charts is automatic.

```yaml
# manifest/mirror-manifest.yaml
schema_version: 1
images:
  "sha256:4e3b...c91a":
    upstream_ref: registry.k8s.io/ingress-nginx/controller:v1.10.1
    mirror_repo: 123456789012.dkr.ecr.us-east-1.amazonaws.com/mirror/images/ingress-nginx-controller
    mirror_tags:
      - v1.10.1-mirror-20260423-4e3bc91a
    first_seen: 2026-04-23T14:02:11Z
    upstream_signed: true
    our_signed_at: 2026-04-23T14:06:00Z    # after cosign sign completes; see Idempotency
    scan:
      trivy:
        result: manifest/scans/images/sha256-4e3bc91a/ingress-nginx-4.10.2/trivy.json
        ran_at: 2026-04-23T14:05:00Z
        summary: { critical: 0, high: 2, medium: 7 }
    sboms:
      - manifest/sboms/sha256-4e3bc91a/ingress-nginx-4.10.2.spdx.json
    used_by:
      - { chart: ingress-nginx, chart_version: 4.10.1 }
      - { chart: ingress-nginx, chart_version: 4.10.2 }

charts:
  "ingress-nginx@4.10.2":
    upstream_source: https://kubernetes.github.io/ingress-nginx
    mirror_ref: oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/mirror/charts/ingress-nginx:4.10.2
    vendored_at: 2026-04-23T14:10:00Z
    images:                                # digest refs; resolve via images: map above
      - "sha256:4e3b...c91a"
      - "sha256:9a1f...02dd"
```

#### Flow

1. Detector writes `manifest/pending/<chart>-<version>.yaml` + scan JSONs in one PR.
2. Human reviews + merges.
3. Vendor consumes every file in `manifest/pending/` the planner finds, does its work, and **moves** each pending file into the canonical manifest (adding/updating `images:` entries with fresh `mirror_repo` / `first_seen` / `our_signed_at`, updating `used_by`, and appending the chart entry). Deletes the pending file as part of the same commit.
4. The canonical manifest is touched once per vendor commit, by a single writer, under a serialized concurrency group.

#### Manual edits

Manual edits to `manifest/mirror-manifest.yaml` (human-driven GC, rollback, schema migration) **do not retrigger** `vendor.yml` because vendor only watches `manifest/pending/**`. If an operator wants to force re-vendoring of a version, they drop a fresh pending file under `manifest/pending/` (easiest via `mirrorctl vendor reopen <chart> <version>`).

**GC.** No automatic pruning. `mirrorctl gc --report` lists entries with empty `used_by`. A human deletes from registry + manifest in a separate PR.

### Digest pinning & mirror tagging

Push destinations are **tags**, not digests — registries compute digests from uploaded content; you pull *from* a digest but push *to* a tag. `skopeo copy --preserve-digests` guarantees the manifest bytes are copied byte-for-byte, so the resulting destination digest equals the source digest. We verify that equality post-push.

Exact sequence per image:

1. **Mint mirror tag.** `<upstream-tag>-mirror-<YYYYMMDD>-<short-digest>` where `<short-digest>` is the first 8 hex chars of the source digest. The short-digest prefix kills the same-day collision case (same upstream tag resolving to different digests within a single day — e.g. CDN flip, fast re-publication).
2. **Push.** `skopeo copy --preserve-digests docker://<upstream>@sha256:<digest> docker://<mirror_repo>:<mirror_tag>`.
3. **Verify.** `crane digest <mirror_repo>:<mirror_tag>` (or equivalent `registry.Exists`) returns the digest; assert it equals the source digest. Mismatch → abort, mark the pending file `status: error`, write no manifest entry.
4. **Reference by digest** everywhere downstream (values.yaml, manifest, cosign). The mirror tag is a human-readable handle for logs; the digest is authoritative identity.

We do **not** mirror the upstream tag verbatim (no bare `:v1.10.1`). Forces consumers off accidental tag-based trust.

Chart push: `helm push <vendored-tarball> oci://<registry>/mirror/charts/` after values are rewritten.

### Values rewriting

In-place on the extracted chart's `values.yaml`. For each `(image_ref, values_path)`:

**Split form** (most charts):
```yaml
# before
controller:
  image:
    registry: registry.k8s.io
    image: ingress-nginx/controller
    tag: v1.10.1
    digest: sha256:4e3b...c91a   # may or may not exist upstream
# after
controller:
  image:
    registry: 123456789012.dkr.ecr.us-east-1.amazonaws.com
    image: mirror/images/ingress-nginx-controller
    tag: v1.10.1-mirror-20260423-4e3bc91a
    digest: sha256:4e3b...c91a   # always set by us
```

**Single-string form:**
```yaml
# before
redis:
  image: docker.io/bitnami/redis:7.2.4
# after
redis:
  image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/mirror/images/bitnami-redis:7.2.4-mirror-20260423-9a1f02dd@sha256:9a1f...02dd
```

We append `@sha256:…` when the chart accepts it. If the chart strips digests in its template, the repo+tag rewrite is still authoritative (tag is ours, repo is ours); the manifest is the canonical digest.

**Explicit `digest` field:** always populated (upstream digest == mirrored digest under `skopeo --preserve-digests`).

Edits use `yq eval -i` on explicit paths. No regex on YAML.

### Registry abstraction

`internal/registry/registry.go`:

```go
type Registry interface {
    EnsureRepo(ctx context.Context, repo string) error
    Exists(ctx context.Context, repo, digest string) (bool, error)
    PushByDigest(ctx context.Context, srcRef, dstRepo, digest string) error
    Tag(ctx context.Context, repo, digest, tag string) error
    ListTags(ctx context.Context, repo string) ([]string, error)
    PushChart(ctx context.Context, tarballPath, repo, version string) error
}
```

- `internal/registry/zot/` — talks to local zot over HTTP. `EnsureRepo` is a no-op (zot auto-creates on push). Used by the docker-compose test harness and CI integration tests.
- `internal/registry/ecr/` — AWS SDK for `EnsureRepo` (`CreateRepository`, idempotent with `RepositoryAlreadyExists` handling). Push goes through `skopeo` with `aws ecr get-login-password` creds from the OIDC-assumed role. **ECR requires repos to exist before push** — `EnsureRepo` is mandatory.

Factory in `cmd/mirrorctl/main.go` selects impl by env (`MIRROR_REGISTRY_KIND=zot|ecr`).

**Helm push is OCI-only in this system.** Both zot and ECR are OCI registries. We never push charts to a `classic` (index.yaml) repo — even if the upstream is classic, we mirror to OCI. This keeps `PushChart` as a single-backend code path and eliminates index.yaml generation.

**ECR-specific backoff.** ECR enforces per-second push quotas. `PushByDigest` in the ECR impl wraps `skopeo copy` in exponential backoff with jitter (base 2s, max 60s, 5 retries). Exhausted retries surface as `status: error` on the pending file, never as a silent success.

### Idempotency and failure handling

Manifest is the anchor. A vendor run for `chart@version` from `manifest/pending/<chart>-<version>.yaml`:

1. Load `manifest/mirror-manifest.yaml`.
2. For each image listed in the pending file:
   - If digest is in `images:` **and** `registry.Exists(repo, digest)` returns true **and** `our_signed_at` is set → skip push, ensure tag, update `used_by`.
   - Else: `EnsureRepo`, `PushByDigest`, `Tag`, **then `cosign sign`**, **only then** add/update the manifest entry with `our_signed_at`. An unsigned-but-pushed digest has no manifest entry; the next run's check re-runs `cosign sign --recursive` against the existing digest (idempotent) and then writes the entry.
   - **Write the manifest after each image**, not at the end. Mid-run crash leaves a consistent partial manifest; next run's checks find what landed.
3. Rewrite values, push chart, add chart entry to the canonical manifest.
4. Delete `manifest/pending/<chart>-<version>.yaml`.
5. `git add manifest/ vendored/<chart>/<version>/ && git commit`.

**Retry semantics:**
- Partial push (crash after 4/10 images): rerun → 4 skipped (exist + signed + in manifest), 6 fresh.
- Pushed but signing failed: image is in the registry, no manifest entry. Next run re-signs and writes the entry. No double-push.
- Registry has a digest, manifest doesn't (out-of-band push): update manifest, `first_seen = now`.
- Manifest has a digest, registry doesn't (registry restored from backup): re-push; keep `first_seen`.

Digest-addressed pushes are content-idempotent — pushing the same digest twice is a no-op at the registry layer.

**Specific failure modes:**

- **Upstream deletes a `pending` version** (404 on `helm pull`). Vendor records `status: upstream_gone` in the pending file and opens a tracking issue (not a new PR); the planner skips the entry on subsequent runs.
- **ECR throttle.** Per the Registry abstraction section, `PushByDigest` in the ECR impl uses exponential backoff with jitter. Exhausted retries → `status: error` on the pending file; vendor exits non-zero for that matrix job. Other charts continue.
- **Cosign signing fails after image push succeeds.** Image is in the registry but unsigned; no manifest entry yet (see step 2 above). The next run's idempotency check catches this: `Exists=true` but manifest entry missing → re-attempt `cosign sign`, write the entry on success. The image is **never** referenced by a vendored chart until signed.
- **Two team members edit `config/charts.yaml` simultaneously** affecting the same chart's `version_floor` or `manual_images`. Ordinary git merge conflict; resolve in code review. Not a system bug.
- **KMS key rotation.** cosign's KMS integration references keys by **alias** (`awskms:///alias/helm-mirror-signer`). AWS KMS rotation is transparent to verification — old signatures verify against the old key version, new signatures against the new. Re-signing existing artifacts is only required if we **replace** (not rotate) the key.

---

## Security & Scanning

### Threat model

This system defends against common supply-chain risks in consuming third-party Helm charts directly: **compromised upstream chart or image**, **typo-squatted chart references**, **silent tag overwrites**, **registry tampering downstream of our mirror**. Pinning by digest, verifying upstream signatures where available, and re-signing into our own registry constrain what a compromised upstream can do between our scans.

It does **not** defend against: an insider with merge rights on this repo (they *are* the trust root — human PR approval is the gate), a compromised scanner binary or its vuln DB, or zero-days no scanner recognizes yet. Scans raise the bar; the approver reading the PR is the ultimate control.

### Scan plugin interface

A scan plugin is an **executable** (any language) that reads a job spec on stdin and writes a structured result to a known path. Language-agnostic, trivially testable.

**Input (stdin, JSON):**
```json
{
  "kind": "image" | "chart",
  "target": "ghcr.io/foo/bar@sha256:abc..." | "/tmp/charts/foo-1.2.3.tgz",
  "output_path": "manifest/scans/images/sha256-abc.../<chart>-<version>/<scanner>.json",
  "config": { "severity_threshold": "HIGH" },
  "timeout_seconds": 600
}
```

**Output (written to `output_path`, JSON):**
```json
{
  "scanner": "trivy",
  "scanner_version": "0.50.1",
  "target": "ghcr.io/foo/bar@sha256:abc...",
  "started_at": "2026-04-23T14:02:01Z",
  "duration_seconds": 42,
  "status": "pass" | "fail" | "error",
  "summary": { "CRITICAL": 0, "HIGH": 2, "MEDIUM": 11, "LOW": 34 },
  "findings": [
    {"id": "CVE-2025-1234", "severity": "HIGH", "package": "openssl",
     "version": "3.0.2", "fixed_version": "3.0.13", "description": "..."}
  ]
}
```

**Exit codes:** `0` = pass, `1` = fail (findings exceed threshold), `2` = scanner error. The runner treats `0` and `1` as successful runs (result is authoritative); `2` (or any other non-zero) is an infra error — retried once, then recorded as `status: error`, surfaced in the PR, not killing other scanners.

**Registration** via `manifest/scanners.yaml`:
```yaml
scanners:
  - name: trivy
    kind: image
    command: ["./scanners/trivy-plugin.sh"]
    timeout_seconds: 600
    runtime_deps: [trivy, jq]             # installed in the workflow step before this plugin runs
    config: { severity_threshold: HIGH }
  - name: kubelinter
    kind: chart
    command: ["./scanners/kubelinter-plugin.sh"]
    timeout_seconds: 120
    runtime_deps: [kube-linter, jq]
policies:
  # Chart-level scan_policy REPLACES the default entirely (not merge semantics).
  standard:
    image: [trivy, grype, syft, cosign-verify]
    chart: [kubelinter, helm-lint]
  strict:
    image: [trivy, grype, syft, cosign-verify]
    chart: [kubelinter, helm-lint]
    overrides:
      trivy: { severity_threshold: MEDIUM }
```

`runtime_deps` is a list of tool names the detector job installs (via `apt-get`, `go install`, or a pinned GitHub release) before invoking the plugin. Plugins that rely only on pre-installed `ubuntu-latest` tools (bash, jq, coreutils) may omit the field.

**Hello-world plugin (`scanners/hello-plugin.sh`):**
```bash
#!/usr/bin/env bash
set -euo pipefail
# Runtime deps: jq, coreutils (date). Both are pre-installed on ubuntu-latest
# runners. Plugins relying on non-standard tooling must declare it in
# scanners.yaml under `runtime_deps:` (see registration schema above).
spec="$(cat)"
out=$(jq -r .output_path <<<"$spec")
target=$(jq -r .target <<<"$spec")
mkdir -p "$(dirname "$out")"
cat > "$out" <<EOF
{"scanner":"hello","scanner_version":"0.0.1","target":"$target",
 "started_at":"$(date -u +%FT%TZ)","duration_seconds":0,
 "status":"pass","summary":{},"findings":[]}
EOF
exit 0
```

### Built-in plugins (v1)

| Plugin | Kind | Produces | Default fail threshold |
|---|---|---|---|
| **trivy** | image | CVEs, misconfig findings | Fail on `HIGH`+ with a known fix; `CRITICAL` always fails |
| **grype** | image | CVEs (second opinion, different DB) | Fail on `CRITICAL` only (trivy primary; grype catches disagreements) |
| **syft** | image | SBOM (SPDX JSON) | Never fails; always `status: pass`, SBOM attached |
| **kubelinter** | chart | Rendered-template correctness | Fail on any `error`-level; `warning` advisory |
| **helm lint** | chart | Chart syntax / schema | Fail on any `ERROR` |
| **cosign verify** | image | Upstream signature presence + validity | Fail only if upstream *has* signatures and they don't verify. **Unsigned upstream → pass** with `notes: "upstream unsigned"` |

`checkov` is a v1.1 candidate; omitted from v1 to keep the default set small.

### Failure policy

**Always open the PR, mark the GitHub check failed when any scanner reports `fail`, post a sticky summary comment.** Rationale: the point is human approval. Hiding a failing chart by refusing to open a PR robs the approver of visibility — they won't know a new upstream version dropped. Opening with a red check is louder and more honest. Branch protection prevents merging a red PR anyway.

**Per-chart isolation:**
- Each chart is its own GHA matrix job (`fail-fast: false`). One chart crashing ≠ matrix dead.
- Within a chart's job, scanners run as subprocesses with enforced `timeout_seconds` (SIGTERM then SIGKILL +10s).
- Non-zero-non-one exit is captured, not thrown. PR check fails; other scanners continue.
- Scanners within a chart run in parallel (independent), serialized only on shared vuln-DB downloads (cached across the matrix via `actions/cache`).

### Cosign / signature verification

Anything that informs **human approval** must be known by detector-time so it appears in the PR. Vendor re-verifies for defense in depth but never discovers new facts.

**At detector-time (results committed to the pending file + PR):**

1. **Upstream image signatures.** `cosign verify <upstream-image>@<digest> --certificate-identity-regexp=… --certificate-oidc-issuer=…` when upstream uses keyless (Sigstore Fulcio). Keyed: verify against a public key committed under `manifest/upstream-keys/`. Result → `upstream_signed: true|false` per image in the pending file.
2. **Upstream chart provenance.** If `.prov` exists next to the tarball, `helm verify`. If upstream uses `cosign sign-blob` on the tarball, verify that instead. Result → pending-file `upstream_signed` on the chart.
3. No signatures present → `upstream_signed: false`. Approver sees this in the PR — judgment call, not auto-block.

**At vendor-time:**

1. **Re-verify** upstream signatures (defense against a merge-to-mirror race). Abort if verification now fails (`status: error` on the pending file).
2. **Re-sign everything with our own key** so consumers verify provenance from *our* registry:
   - Each mirrored image: `cosign sign <our-registry>/<image>@<digest>`.
   - Each mirrored chart: `cosign sign-blob` on the tarball; signature stored as an OCI artifact next to the chart.
   - Attach the SBOM: `cosign attest --predicate sbom.spdx.json --type spdx`.

**Key storage: AWS KMS for prod**, via cosign KMS integration (`cosign sign --key awskms:///alias/helm-mirror-signer`). Private key never leaves KMS; GHA's OIDC role is allowed `kms:Sign`. For test/zot, a GHA-secret-stored cosign key is acceptable. **Do not** use a plain GHA secret for the prod key — it's extractable by any repo admin.

### SBOM

**Generate at detector-time with syft** on the upstream image digest. The SBOM is committed into the PR so the approver has visibility.

- Detector: `syft <upstream>@<digest> -o spdx-json > manifest/sboms/sha256-<digest>/<chart>-<version>.spdx.json`. Pending file references it via `sbom_ref`.
- Vendor: re-generates on the *mirrored* digest to confirm byte-equivalence (they should match — `skopeo --preserve-digests`), then attaches to our-registry image as a cosign attestation (`cosign attest`).

Format: SPDX JSON. We regenerate rather than pass through upstream SBOMs — consistent format, proves the SBOM matches the bytes we mirrored.

### Scan result storage & PR surfacing

**On-disk layout:**
```
manifest/
├── scans/
│   ├── images/
│   │   └── sha256-abc123.../                 # image digest (dash-encoded)
│   │       └── my-chart-1.2.3/               # chart-version that triggered this scan run
│   │           ├── trivy.json
│   │           ├── grype.json
│   │           └── cosign-verify.json
│   └── charts/
│       └── my-chart-1.2.3/
│           ├── kubelinter.json
│           └── helm-lint.json
└── sboms/
    └── sha256-abc123.../
        └── my-chart-1.2.3.spdx.json          # SBOM path scoped same way
```

**Why per-chart-version scan paths under the digest directory.** Two PRs for different chart versions that both reference the same image digest would otherwise touch the same `trivy.json` file — reintroducing the merge-conflict hot spot we killed by splitting the manifest. Scoping scan output by `{digest, chart, chart-version}` keeps each PR's outputs disjoint *and* preserves the full historical record (a re-scan of the same digest from a different chart-version lands in its own path).

The per-chart PR commits relevant scan JSONs under `manifest/scans/…` — reviewable in the diff, greppable forever.

**PR surfacing is inline in `detect.yml`.** The per-chart detector job writes the scan results to disk, then (in the same job) uses a sticky-comment action (`marocchino/sticky-pull-request-comment`) to post / update a single scan-summary comment on the PR. There is no separate `scan-summary.yml` workflow — inlining avoids a second workflow run that would also need a GitHub App token to fire on a PR created by `GITHUB_TOKEN`.

The summary comment body:

```
## Scan Summary — my-chart 1.2.3

| Scanner | Status | Findings |
|---|---|---|
| trivy (image: app@sha256:abc) | ❌ FAIL | 2 HIGH, 11 MEDIUM |
| grype (image: app@sha256:abc) | ✅ PASS | 0 CRITICAL |
| kubelinter (chart) | ✅ PASS | 0 errors, 3 warnings |
| helm lint | ✅ PASS | — |
| cosign verify (upstream) | ⚠️ upstream unsigned |
| syft | 📦 SBOM generated (847 packages) |

<details>Top findings…</details>
```

Check status (green/red): any `fail` → red. `error` → yellow/neutral with the approver explicitly notified.

---

## GitHub Actions Workflows

### Language / runtime

**Go.** OCI-heavy (manifest fetching, digest verification, registry pushes) and YAML-heavy (parse + rewrite `values.yaml` preserving comments). First-party libraries match: `github.com/google/go-containerregistry`, `helm.sh/helm/v3` as a library (Python options wrap the helm CLI), `gopkg.in/yaml.v3` for round-trip-safe YAML. Single static binary is trivial to ship to a GHA runner and straightforward to unit-test. Python would work but the helm-as-library and OCI stories are weaker.

All non-trivial logic lives in `internal/` so it's testable without a runner. Workflows invoke the binary: `mirrorctl detect plan`, `mirrorctl vendor run --chart foo --version 1.2.3`.

### Detector workflow (`.github/workflows/detect.yml`)

**Triggers:** daily cron + manual dispatch.

**Topology:** a coordinator `plan` job reads `config/charts.yaml` and emits a JSON matrix of `{chart, candidate_version}` pairs; a `per-chart` job fans out, runs scans inline, writes a `manifest/pending/<chart>-<version>.yaml` file plus scan JSONs, opens a PR, and posts a sticky scan-summary comment from the same job.

**Critical: required check against the PR head SHA.** The detector runs on `main`; the job that ran the scans completes against `main`'s SHA, **not** the PR's head SHA. Branch protection required-checks match the PR head SHA, so a job-level success/failure on `detect.yml`'s `main` run is invisible to branch protection. The fix: after pushing the PR branch, the detector **explicitly posts a Checks API result** against the PR head SHA (`POST /repos/:owner/:repo/check-runs`) with the scan verdict as `conclusion: success|failure|neutral`. The `checks: write` permission on the bot App is required for this.

**Enforcing the required check.** Classic GitHub branch protection is **not path-scoped** — you can't say "require `mirror/scan` only on PRs touching `manifest/pending/**`." Two viable options:

1. **Repo rulesets (recommended)** — GitHub's newer ruleset system supports path-scoped required checks. Create a ruleset targeting `main` with a condition on `manifest/pending/**` paths, required status check `mirror/scan`. This is the clean answer.
2. **Fallback for repos on classic branch protection only** — make `mirror/scan` globally required on all PRs to `main`. Non-mirror PRs need a trivial emitter (a tiny `.github/workflows/scan-check-noop.yml` that posts `mirror/scan` with `conclusion: neutral` on any PR that doesn't touch `manifest/pending/**`), otherwise classic BP blocks merge on missing-required-check.

We document both; repos with rulesets available use option 1.

**PR author identity.** The detector creates PRs using a **GitHub App token**, not the default `GITHUB_TOKEN`. Reason: PRs opened with the default `GITHUB_TOKEN` do not trigger `pull_request`-triggered workflows at all. The App token (`HELM_MIRROR_BOT_APP_ID` + private key in secrets) is also the credential that posts the Checks API status above.

```yaml
name: detect
on:
  schedule: [{ cron: '17 6 * * *' }]   # avoid top-of-hour stampede
  workflow_dispatch:

concurrency:
  group: detect
  cancel-in-progress: false

permissions:
  contents: write
  pull-requests: write
  id-token: write

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs: { matrix: ${{ steps.plan.outputs.matrix }} }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - id: plan
        run: |
          mirrorctl detect plan \
            --config config/charts.yaml \
            --manifest manifest/mirror-manifest.yaml \
            --pending-dir manifest/pending \
            --output matrix > matrix.json
          echo "matrix=$(cat matrix.json)" >> "$GITHUB_OUTPUT"

  per-chart:
    needs: plan
    if: ${{ fromJSON(needs.plan.outputs.matrix).include[0] != null }}
    strategy:
      fail-fast: false       # one bad chart ≠ kill the run
      max-parallel: 8        # be polite to upstream
      matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.22' }

      # Mint a short-lived App token — this is the PR author identity
      - id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.HELM_MIRROR_BOT_APP_ID }}
          private-key: ${{ secrets.HELM_MIRROR_BOT_PRIVATE_KEY }}

      # Pull candidate chart + run image discovery (may fail-loud on partial-rewrite etc.)
      - run: mirrorctl detect pull --chart "${{ matrix.chart }}" --version "${{ matrix.version }}"

      # Scanning runs inline. Also runs cosign-verify against upstream and
      # syft against each upstream image digest so the PR carries all the
      # facts the approver needs (vendor-time re-verifies but never discovers
      # new facts — see Security & Scanning → Cosign / signature verification).
      # Exit code 0 even when scans fail — verdict is in the pending file.
      - id: scan
        run: |
          mirrorctl scan run \
            --chart "${{ matrix.chart }}" \
            --version "${{ matrix.version }}" \
            --write-pending manifest/pending/${{ matrix.chart }}-${{ matrix.version }}.yaml \
            --write-scans manifest/scans/ \
            --write-sboms manifest/sboms/ \
            --include-cosign-verify \
            --include-sbom
          echo "verdict=$(yq .scan_verdict manifest/pending/${{ matrix.chart }}-${{ matrix.version }}.yaml)" >> "$GITHUB_OUTPUT"

      # Open (or update) the PR. Always open — even on scan fail.
      - uses: peter-evans/create-pull-request@v6
        id: pr
        with:
          token: ${{ steps.app-token.outputs.token }}
          branch: mirror/${{ matrix.chart }}-${{ matrix.version }}
          title: "mirror(${{ matrix.chart }}): ${{ matrix.version }}"
          body-path: .pr-body.md
          labels: |
            mirror
            chart:${{ matrix.chart }}
            scan:${{ steps.scan.outputs.verdict }}
          add-paths: |
            manifest/pending/${{ matrix.chart }}-${{ matrix.version }}.yaml
            manifest/scans/**
            manifest/sboms/**

      # Sticky scan summary comment. One body, updated on re-run — no thread spam.
      - if: steps.pr.outputs.pull-request-number
        uses: marocchino/sticky-pull-request-comment@v2
        with:
          token: ${{ steps.app-token.outputs.token }}
          number: ${{ steps.pr.outputs.pull-request-number }}
          path: .scan-summary.md

      # Post a Checks API result against the PR HEAD SHA so branch protection
      # can enforce it. detect.yml runs on main; branch protection looks at
      # checks on the PR head SHA, so the job status alone is invisible to it.
      - if: steps.pr.outputs.pull-request-number
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          HEAD_SHA: ${{ steps.pr.outputs.pull-request-head-sha }}
          VERDICT: ${{ steps.scan.outputs.verdict }}
        run: |
          conclusion=$([ "$VERDICT" = "fail" ] && echo failure || echo success)
          gh api repos/${{ github.repository }}/check-runs \
            -f name="mirror/scan" \
            -f head_sha="$HEAD_SHA" \
            -f status=completed \
            -f conclusion="$conclusion" \
            -f "output[title]=scan verdict: $VERDICT" \
            -f "output[summary]=$(cat .scan-summary.md | head -c 60000)"

      # Also fail the job step so the detector matrix's own status is honest.
      - if: steps.scan.outputs.verdict == 'fail'
        run: |
          echo "::error::scan verdict=fail for ${{ matrix.chart }} ${{ matrix.version }}"
          exit 1

      - if: always()
        run: cat .step-summary.md >> "$GITHUB_STEP_SUMMARY"
```

**Conventions:**
- Branch: `mirror/<chart>-<version>`. Idempotent — `create-pull-request` updates an existing branch if the version was already proposed and not merged.
- PR title: `mirror(<chart>): <version>`.
- Labels: `mirror`, `chart:<name>`, `scan:<pass|warn|fail>`.
- Body: scan summary, upstream URL, source digest, diff vs. previously vendored version.
- **The PR commits a pending file** at `manifest/pending/<chart>-<version>.yaml` plus its scan JSONs and SBOM. It **does not** touch `manifest/mirror-manifest.yaml`.

**Matrix size:** GHA caps at 256 jobs. The planner caps to N most-recent versions per chart (configurable, default 5).

### Vendor workflow (`.github/workflows/vendor.yml`)

**Trigger:** push to `main` touching `manifest/pending/**`. Detector PRs add pending files; merging is the green light. We do **not** trigger on `manifest/mirror-manifest.yaml` (vendor itself writes that file and would self-loop) or `vendored/` (also vendor-written).

```yaml
name: vendor
on:
  push:
    branches: [ main ]
    paths: [ 'manifest/pending/**' ]
  workflow_dispatch:
    inputs:
      chart:   { required: false }
      version: { required: false }

concurrency:
  group: vendor-main
  cancel-in-progress: false

permissions:
  contents: write        # commit vendored/ back
  id-token: write        # OIDC → AWS
  packages: read

jobs:
  plan:
    runs-on: ubuntu-latest
    outputs: { matrix: ${{ steps.plan.outputs.matrix }} }
    steps:
      - uses: actions/checkout@v4
      - id: plan
        run: |
          mirrorctl vendor plan --pending-dir manifest/pending > matrix.json
          echo "matrix=$(cat matrix.json)" >> "$GITHUB_OUTPUT"

  vendor:
    needs: plan
    if: ${{ fromJSON(needs.plan.outputs.matrix).include[0] != null }}
    strategy:
      fail-fast: false
      max-parallel: 1      # serialize canonical manifest writes
      matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0, ref: main }   # start on the push-event SHA

      # CRITICAL: after acquiring the concurrency slot, a queued vendor run
      # may have started from a stale push SHA (the earlier run already
      # consumed and deleted pending files and updated the manifest).
      # Reset to current origin/main BEFORE planning so we never reprocess
      # a pending file that has already been vendored.
      - name: Reset to current origin/main
        run: |
          git fetch origin main
          git reset --hard origin/main
          git log -1 --oneline

      # Re-check against current state. A queued vendor run's matrix may
      # include a pending file that an earlier run already consumed and
      # deleted. We detect that here and skip ALL subsequent steps via
      # `if: steps.pending.outputs.exists == 'true'` — note that a plain
      # `exit 0` in a step only succeeds the step; the job continues.
      - name: Re-check pending file still exists
        id: pending
        run: |
          if [ -f "manifest/pending/${{ matrix.chart }}-${{ matrix.version }}.yaml" ]; then
            echo "exists=true" >> "$GITHUB_OUTPUT"
          else
            echo "::notice::pending file already consumed by an earlier run; skipping remaining steps"
            echo "exists=false" >> "$GITHUB_OUTPUT"
          fi

      - if: steps.pending.outputs.exists == 'true'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.ECR_VENDOR_ROLE_ARN }}
          aws-region: us-east-1
      - if: steps.pending.outputs.exists == 'true'
        uses: aws-actions/amazon-ecr-login@v2
      - if: steps.pending.outputs.exists == 'true'
        uses: actions/setup-go@v5
        with: { go-version: '1.22' }
      - if: steps.pending.outputs.exists == 'true'
        run: |
          mirrorctl vendor run \
            --pending manifest/pending/${{ matrix.chart }}-${{ matrix.version }}.yaml \
            --registry "${{ vars.MIRROR_REGISTRY }}"

      # Commit-back uses the default GITHUB_TOKEN on purpose: pushes made with
      # GITHUB_TOKEN do NOT fire further workflow runs, which is the sole
      # mechanism that prevents vendor.yml from re-triggering itself when it
      # writes to manifest/mirror-manifest.yaml. Do NOT swap this for a PAT or
      # App token without adding an alternative loop guard (e.g. a path filter
      # that excludes manifest/mirror-manifest.yaml).
      - if: steps.pending.outputs.exists == 'true'
        env: { GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
        run: |
          git config user.name  "helm-mirror-bot"
          git config user.email "helm-mirror-bot@users.noreply.github.com"
          git add manifest/ vendored/
          git commit -m "vendor(${{ matrix.chart }}): ${{ matrix.version }}" || exit 0
          for i in 1 2 3 4 5; do
            git pull --rebase origin main && git push origin main && exit 0
            sleep $((RANDOM % 10 + 5))
          done
          exit 1
```

**Loop guard.** The commit touches `manifest/mirror-manifest.yaml` (and deletes the consumed pending file), but `GITHUB_TOKEN`-authored pushes do not fire workflows — so `vendor.yml` does not re-trigger on its own commits. This is the **sole** loop guard; no `[skip ci]` sentinel is needed or used (which would also silently disable every other future `push:` workflow on main).

**Commit-back auth.** `GITHUB_TOKEN` with `contents: write`. Simpler than a PAT, no rotation. The "no downstream workflow trigger" behavior is a feature here.

**Manifest concurrency.** `concurrency: vendor-main` + `max-parallel: 1` serializes vendor runs; rebase-and-retry loop handles close-together pushes (two pending files landing on main back-to-back).

**Human-driven edits.** An operator manually editing `manifest/mirror-manifest.yaml` (GC, rollback, schema migration) does **not** trigger this workflow — the path filter only watches `manifest/pending/**`. To force re-vendor, drop a fresh file under `manifest/pending/` via `mirrorctl vendor reopen <chart> <version>`.

### Idempotency

- **Detector planner** consults `manifest/mirror-manifest.yaml` and existing files in `manifest/pending/`.
  - If `{chart, version}` is already vendored **and** the upstream chart-tarball digest matches the recorded `upstream_chart_digest`: skip this version entirely. (Cheap check: a `HEAD` / index.yaml fetch; no pull, no render.)
  - If `{chart, version}` is already vendored **but** the upstream chart-tarball digest has changed: this is a SemVer violation or tampering signal (see "Security alert path" in Success criteria). Emit a tracking issue with `bot: helm-mirror` label and skip vendoring — there is no new version to mirror. Do NOT re-vendor.
  - If a pending file already exists for that version (open PR in flight): skip.
  - If the PR branch already exists: `create-pull-request` updates rather than duplicates.
- **Vendor planner** finds every file in `manifest/pending/` on the tip of main; produces one matrix entry per file.
- **Mirror push** uses `skopeo copy --preserve-digests` with digest-pinned source. Repeated pushes are no-ops at the registry.

Lose `vendored/`? Re-run vendor — rebuilds from the canonical manifest. Lose the canonical manifest? Disaster scenario; git is the backup.

### Auth

- **AWS / ECR:** OIDC. One IAM role (`ECR_VENDOR_ROLE_ARN`) trusted by `repo:org/helm-mirroring:ref:refs/heads/main` with: `ecr:GetAuthorizationToken` (required by `amazon-ecr-login`), `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:CreateRepository`, `ecr:DescribeRepositories`, `kms:Sign` on the signing key alias. A read-only role (`ecr:DescribeImages`, `ecr:ListImages`, `ecr:GetAuthorizationToken`) is available to the detector for tag lookups but is not required in v1. Audience `sts.amazonaws.com`. Trust policy owned by the platform team; we document what's required.
- **GitHub — bot identity:** a GitHub App (`HELM_MIRROR_BOT_APP_ID`) installed on the repo. The detector uses this App token to open PRs (App-authored PRs fire `pull_request` workflows; `GITHUB_TOKEN`-authored PRs don't) and to post Checks API results. The App needs: `pull_requests: write`, `contents: write`, `checks: write` (required to create check-runs via the Checks API), `issues: write` (for the security-alert tracking issues on chart-tarball re-publication), and read on `metadata`. Private key stored in secret `HELM_MIRROR_BOT_PRIVATE_KEY`.
- **GitHub — vendor commit-back:** default `GITHUB_TOKEN` with `contents: write`. Chosen specifically because its pushes don't re-fire workflows — our sole loop guard on `vendor.yml`.
- **Cosign signing key:** AWS KMS (prod), referenced by alias. cosign's KMS integration. Same OIDC role assumes `kms:Sign`.

### Test harness

`testdata/docker-compose.yml` brings up:
- `zot` on `localhost:5000` (anonymous read/write, test-only)
- `registry:2` preloaded with fixture images (deterministic upstream)

`Makefile`:
```make
test-integration:
	docker compose -f testdata/docker-compose.yml up -d --wait
	go test ./... -tags=integration -registry=localhost:5000
	docker compose -f testdata/docker-compose.yml down -v
```

Integration test: pick a fixture chart from `testdata/charts/`, run `mirrorctl vendor run --registry localhost:5000`, assert (a) chart tarball present in zot, (b) every image in rewritten `values.yaml` resolves in zot by digest, (c) manifest file gained the expected entry.

In CI, `.github/workflows/test.yml` runs on PRs touching code/scripts. Uses `services:` for zot when possible; falls back to docker-compose. Runs in <5 minutes on `ubuntu-latest`.

### Observability

- **Step summaries.** Every job writes a per-chart outcome block to `$GITHUB_STEP_SUMMARY`: chart, version, scan verdict, registry digest, vendoring status. The `plan` job summary lists everything considered and why each was skipped/queued.
- **Artifacts.** Raw scan outputs uploaded with `actions/upload-artifact` keyed by `<chart>-<version>`. 30-day retention.
- **Tracking issue.** A long-lived issue (`mirror: daily status`) gets a comment each run with a one-line-per-chart status table. Easy to scroll history.

---

## Rollout & Milestones

Each milestone is end-to-end demonstrable on its own before the next layer lands.

### M1 — Local vendor round-trip (no GHA, no scans)

The smallest slice proving core mechanics.

- Repo layout in place: `config/`, `vendored/`, `manifest/`, `testdata/`, `cmd/mirrorctl/`.
- Registry interface defined; zot implementation only.
- `docker-compose.yml` bringing up zot locally.
- One hand-picked fixture chart (single image, e.g. basic nginx) in `testdata/`.
- CLI: given a chart tarball, do image discovery → push to zot by digest → write manifest entry → rewrite values → emit vendored chart dir.
- Manifest schema finalized.
- Test: `make vendor-fixture` succeeds end-to-end.

**Exit:** a second engineer clones, runs `make vendor-fixture`, sees the fixture vendored into `vendored/` with images in zot under digest and mirror-local tag.

### M2 — Detector + PR automation in GHA (test registry only)

PRs open on schedule. No scanning yet. No ECR yet.

- `detect.yml` with coordinator + matrix fan-out.
- Config-driven list with version floor.
- Per-chart logic: list upstream versions, filter new ≥ floor, for each open a branch + PR with the body template describing chart + version + upstream source.
- `vendor.yml` triggered on merge, runs against zot (still not ECR).
- Integration test workflow running full detect → vendor loop in CI against a fixture upstream hosted in `testdata/`.

**Exit:** pushing a new fixture chart version triggers the next detector run to open a PR; merging triggers the vendor workflow to mirror it into zot.

### M3 — Scanning plugin framework

- Scan plugin interface finalized per Security & Scanning.
- Built-in plugins: trivy, grype, kubelinter, helm lint, cosign verify, syft.
- Per-chart isolation: subprocess + timeout + exit-code capture.
- Scan results as sticky PR comment + job summary tables.
- Failure policy implemented: red check on fail blocks merge via branch protection; PR always opens.
- Scan results written to `manifest/scans/images/sha256-<digest>/<chart>-<version>/<scanner>.json` and `manifest/scans/charts/<chart>-<version>/<scanner>.json`; SBOMs at `manifest/sboms/sha256-<digest>/<chart>-<version>.spdx.json`.

**Exit:** a fixture chart with a deliberately vulnerable image produces a failing PR check; a clean fixture produces passing.

### M4 — Production registry (ECR)

- ECR implementation of the registry interface.
- OIDC trust policy in target AWS account (infra team owns; we document).
- ECR repo-creation in the vendor workflow (idempotent).
- Cosign signing of mirrored images + chart with KMS key.
- First real public chart vendored into prod end-to-end.

**Exit:** vendor a real public chart into ECR, verify signatures, `helm install` the vendored chart into a test cluster with every image pulled from ECR.

### M5 — Hardening & operability

- Manifest lock / concurrency validated under near-simultaneous merges.
- Unreferenced-image report (no auto-prune).
- Runbook: scan failures, PR open failures, vendor halfway-failures.
- Alerting on workflow failure.
- Docs: config reference, plugin authoring, consumer onboarding.

**Exit:** another team adopts the system without asking us questions not already answered in docs.

Non-goals for the milestones: no "v1.1 with subcharts" milestone (separate project, filed as follow-up). No multi-registry replication. No SLSA L3.

---

## Decisions made (not open anymore)

The initial round of review closed these:

- **Date-tag accumulation** (`…-mirror-<YYYYMMDD>` on re-vendor): **keep**. Audit-trail benefit outweighs tag-listing noise. Disk is cheap.
- **Scan result retention**: **keep all historical scan JSONs in the repo**. JSON is small, auditability matters, and pruning logic is complexity we don't need.
- **Grype vs Trivy disagreement**: **red check**. Either disagreeing with the pass-side is loud and the approver can triage. Silent yellow trains reviewers to ignore yellow.
- **Vuln-DB freshness**: **fail the run if trivy's DB is >7 days old**, with a clear error directing the runner to refresh the cache.
- **Re-scan cadence**: **defer to v2**. A weekly re-scan that emits new PRs conflicts with the "one PR per chart version" structure; the honest v2 shape is a separate `rescan-alert.yml` workflow that opens *issues* (not PRs) when an already-mirrored digest's severity profile changes. Not v1.
- **`[skip ci]` in vendor commits**: **removed**. `GITHUB_TOKEN` already prevents re-trigger; `[skip ci]` would silently skip every unrelated future `push:` workflow on main.
- **Scan-summary workflow**: **inlined into `detect.yml`**. Eliminates a workflow that would need an App token to fire on detector-opened PRs.
- **Manifest layout**: **two-file split** (`manifest/pending/**` and `manifest/mirror-manifest.yaml`). Avoids detector merge-conflicts on a shared file.

## Open Questions (carry to implementation)

**Pipeline mechanics**
- **Sentinel-diff case sensitivity.** Templates doing `{{ .Values.image | lower }}` will break naive sentinel matching. Mitigation: lowercase sentinels + case-insensitive diff; needs a test chart.
- **Chart provenance `.prov` files.** Verify but don't propagate, per security section. Confirm during M1.
- **Large images & runner disk.** GHA runners have ~14GB free. Charts with >10GB cumulative images need larger runners. Track; address if we hit it.
- **Multi-arch images.** Mirror full index or just `linux/amd64` + `linux/arm64`? Affects skopeo flags and storage cost.
- **Subchart traversal scope for v2.** Recurse `Chart.yaml` `dependencies:`, push subcharts under `mirror/charts/<parent>/charts/<dep>`, or require explicit top-level listing of every dep? (Leaning: recurse, with the option to re-target via config.)

**Security**
- **Cosign upstream identity allow-list** vs. accept any valid Fulcio cert. Stricter = safer, higher maintenance.
- **KMS region & DR.** Single vs multi-region replica. Out of scope per constraints; key loss = re-sign everything.

**GHA workflows**
- **Upstream rate limiting.** Per-host token bucket in the detector, or is `max-parallel: 8` polite enough?
- **Auto-merge for clean scans.** Tempting, but defeats human-approval. Park for v2.
- **Rollback.** Deprecating a bad vendored version — separate `unvendor` workflow + manifest field. Out of scope v1.
- **Runner choice.** `ubuntu-latest` until we see bandwidth saturation; then larger runners or self-hosted with pull-through cache.
