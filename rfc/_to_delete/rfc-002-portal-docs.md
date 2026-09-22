# RFC 0002: Docs — Component Documentation in the Naira Portal

| Field           | Value                                                    |
|-----------------|----------------------------------------------------------|
| RFC             | 0002                                                     |
| Title           | Docs — Component Documentation in the Naira Portal       |
| Author(s)       | @mkorbi                                                  |
| Target Milestone| TBD                                                      |
| Status          | Draft                                                    |
| Type            | Feature                                                  |
| Created         | 2026-08-20                                               |

## Abstract

This RFC introduces **Docs**, a documentation feature for Naira: every Naira
component (and, later, every cataloged asset that ships documentation) gets a
browsable documentation site **inside the Naira portal**, next to the catalog
and dashboard views the user already works in.

The design follows the docs-as-code model: documentation lives as Markdown in
each component's repository, is built in CI with **plain MkDocs + Material for
MkDocs**, published as a static site to S3-compatible object storage, and
surfaced in the OpenMFP portal as a Luigi iframe micro-frontend. A small,
shared set of primitives — a pinned build container, an inherited base
`mkdocs.yml`, a reusable GitHub Actions workflow, and a fixed object-storage
path scheme — keeps every docs site consistent without a heavyweight docs
platform.

We explicitly do **not** adopt Backstage TechDocs. The evaluation behind that
decision is summarized in *Rationale and Alternatives*: in Naira's
architecture (static docs in a Luigi iframe, no Backstage), the TechDocs layer
adds Backstage coupling and version pins with no rendering benefit, and every
genuinely useful part of it is trivially replicated in CI.

## Motivation

Naira's stated purpose is to pull the fragmented AI engineering experience
into one coherent platform. Documentation is currently the odd one out:

- Component documentation is scattered across repository READMEs
  (`naira/docs/plugin-authoring.md`, per-repo READMEs), the public website
  (`naira.github.io`, VitePress), and the architecture repository. There is
  no single place where a platform user — DevSecOps Dean, AI Engineer rAIner,
  App Dev Abdel — can read the docs for the components they operate without
  leaving the portal.
- As the plugin ecosystem grows (mlflow, litellm, fluxcd, openmetadata
  collectors today; community plugins tomorrow), each plugin needs a
  discoverable home for its usage, configuration, and troubleshooting docs.
  Forcing users out to N GitHub repositories defeats the point of an
  Internal Development Platform.
- Without a shared build pipeline, each repository invents its own docs
  toolchain, and consistency (theme, navigation, search, quality gates like
  `--strict` link checking) erodes immediately.

An IDP that catalogs AI assets but sends users to GitHub to read how those
assets work is incomplete. Docs closes that gap.

## Goals

1. **Docs are browsable in the Naira portal.** A "Documentation" section in
   the portal navigation lists all published docs sites; each renders inside
   the portal (Luigi iframe), with the full Material navigation, per-site
   search, and working deep links.
2. **Docs-as-code.** Documentation lives as Markdown next to the code it
   describes, in the component's own repository, and is versioned, reviewed,
   and released through the same pull-request flow as code.
3. **One consistent pipeline, many repositories.** A single pinned build
   image, a shared base configuration, and one reusable workflow (hosted in
   `naira-github-workflows`) produce uniform sites from every repo. Adding
   docs to a new repository is: add a `docs/` folder and a ~5-line
   `mkdocs.yml`, call the reusable workflow.
4. **Standard, boring output.** The published artifact is a plain static
   Material for MkDocs site plus a small `metadata.json` (component, commit,
   timestamp). Anything that can serve static files can serve it; nothing in
   the pipeline is Naira-proprietary.
5. **Portal-wide search.** A single portal search box federates across all
   published docs sites, so a user can search "how do I register a LiteLLM
   endpoint" without knowing which component documents it.
6. **Self-hostable and air-gap friendly.** The whole pipeline (build image,
   object storage, search index) runs inside the operator's cluster with no
   external SaaS dependency, consistent with Naira's deployment philosophy.

## Non-Goals

- **Not a hosted docs product or WYSIWYG editor.** Authoring happens in git,
  in Markdown, through pull requests.
- **Not Backstage TechDocs, and not a reimplementation of its Backstage-side
  features.** The TechDocs Reader UI, Addons framework, entity annotations,
  and search collator are Backstage-runtime concepts with no equivalent in an
  OpenMFP/Luigi world; we do not rebuild them (see Rationale).
- **Not a replacement for `naira.github.io`.** The public marketing/landing
  website remains VitePress. Docs covers component/product documentation
  surfaced *inside* the portal.
- **Versioned docs (multiple live versions per component) are out of scope
  for v1.** The path scheme reserves room for them; see Future Work.

## Specification

### Terminology

- **Docs site**: the built static site of one component (one MkDocs project).
- **Component**: the unit a docs site belongs to (e.g. `catalog`,
  `plugin-mlflow`, `portal`). Component IDs are lowercase slugs and form the
  URL path.
- **Docs bucket**: the S3-compatible bucket holding all published sites.
- **Docs index**: the machine-readable list of published sites the portal
  uses to build navigation (aggregated from each site's `metadata.json`).

### High-level architecture

```
  component repo A          component repo B           naira repo
  docs/ + mkdocs.yml        docs/ + mkdocs.yml         docs/ + mkdocs.yml
        │                         │                         │
        └───────────── reusable workflow (naira-github-workflows)
                       runs in pinned image ghcr.io/naira-project/mkdocs-build
                                  │
                    mkdocs build --strict  +  metadata.json
                                  │
                                  ▼
                 ┌─────────────────────────────────────┐
                 │  Docs bucket (MinIO / any S3 API)   │
                 │  docs/<component>/latest/…          │
                 └──────────────┬──────────────────────┘
                                │ static HTTP (via ingress/proxy,
                                │ CSP frame-ancestors → portal origin)
                                ▼
                 ┌─────────────────────────────────────┐
                 │  OpenMFP Portal (Luigi)             │
                 │  "Documentation" nodes → iframes    │
                 └──────────────┬──────────────────────┘
                                │
                                ▼
                     per-site search (Material/lunr)
                     portal-wide search (Meilisearch federation)
```

### Authoring model

Each participating repository contains:

```
docs/           # Markdown sources
mkdocs.yml      # ~5 lines, inherits the shared base
```

Per-repo `mkdocs.yml`:

```yaml
INHERIT: /base/mkdocs.base.yml   # path inside the shared build image
site_name: Naira Catalog
site_url: https://docs.<naira-domain>/catalog/latest/
```

The shared base config (owned by this feature, shipped inside the build
image and versioned with it):

```yaml
# mkdocs.base.yml
theme:
  name: material
  features: [navigation.tabs, navigation.sections, search.suggest, content.code.annotate]
markdown_extensions:
  admonition: {}
  toc: {permalink: true}
  pymdownx.superfences: {}
  pymdownx.tabbed: {alternate_style: true}
  pymdownx.highlight: {}
  pymdownx.details: {}
plugins:
  search: {}
use_directory_urls: false   # emit X.html — simplifies raw bucket serving
```

Notes:

- MkDocs `INHERIT` deep-merges, but only when `plugins` /
  `markdown_extensions` use **map** syntax (key/value pairs), not list
  syntax — the base config and all overrides MUST use map syntax.
- `site_url` MUST be set per site, including the sub-path prefix and a
  trailing slash. Material derives canonical URLs, sitemap, and asset paths
  from it; a missing or wrong `site_url` breaks sites served from a bucket
  sub-path.
- `use_directory_urls: false` avoids the "pretty URL" `page/` →
  `page/index.html` resolution that raw S3/MinIO static serving does not do
  reliably for nested paths. If we later front the bucket with a proxy that
  rewrites directory requests, this can be revisited.

### Build: pinned image + reusable workflow

A single build image, e.g. `ghcr.io/naira-project/mkdocs-build`:

```dockerfile
FROM python:3.12-slim
RUN pip install --no-cache-dir \
    mkdocs==1.6.* \
    mkdocs-material==9.7.* \
    pymdown-extensions
COPY mkdocs.base.yml /base/mkdocs.base.yml
```

Exact versions are pinned in the image; repositories never install docs
tooling themselves. The plugin set is intentionally minimal and mainstream
(see the Zensical consideration under Drawbacks).

A reusable workflow in `naira-github-workflows`
(e.g. `.github/workflows/docs-publish.yml`), called by each docs-bearing
repository:

```yaml
on:
  workflow_call:
    inputs: { component: {required: true, type: string} }
jobs:
  build-publish:
    runs-on: ubuntu-latest
    container: ghcr.io/naira-project/mkdocs-build:1
    steps:
      - uses: actions/checkout@v4
      - run: mkdocs build --strict -d site
      - name: Emit metadata
        run: |
          cat > site/metadata.json <<EOF
          {"component":"${{ inputs.component }}",
           "commit":"${{ github.sha }}",
           "built_at":"$(date -u +%FT%TZ)"}
          EOF
      - name: Publish
        env:
          MC_HOST_docs: https://${{ secrets.DOCS_S3_KEY }}:${{ secrets.DOCS_S3_SECRET }}@${{ vars.DOCS_S3_ENDPOINT }}
        run: mc mirror --overwrite --remove site/ docs/docs/${{ inputs.component }}/latest/
```

`mkdocs build --strict` is the quality gate: broken internal links and
misconfigured nav fail the build.

### Publishing: object storage and path scheme

- Target is any S3-compatible store; the dev/reference deployment adds
  **MinIO** to the platform's dev infrastructure (alongside the existing
  postgres/mlflow/litellm manifests under `deploy/dev/infra/kubernetes/`).
- Path scheme: `docs/<component>/<version>/`, with `latest` as the only
  version in v1. The `<version>` segment exists from day one so that
  versioned docs (Future Work) need no migration.
- Each site root contains `metadata.json` (component, commit SHA, build
  timestamp). This is the portal's source for "last updated" display and for
  building the docs index. It deliberately mirrors the useful part of
  TechDocs' `techdocs_metadata.json` without its Backstage semantics.

### Serving

Docs are served over plain HTTP(S) from the bucket via the cluster
ingress/proxy. Two hard requirements on the serving layer:

- **iframe embedding**: responses MUST NOT carry `X-Frame-Options:
  DENY/SAMEORIGIN`; instead set
  `Content-Security-Policy: frame-ancestors 'self' <portal-origin>`.
  `frame-ancestors` supersedes `X-Frame-Options` in CSP-aware browsers and
  MUST be sent as an HTTP header (it cannot be set via `<meta>`).
- Correct `Content-Type` for `.html`, `.css`, `.js`, `search_index.json`.

### Portal integration (OpenMFP / Luigi)

The portal already registers Luigi nodes through
`naira-openmfp-portal/backend/src/service-provider.ts`
(`SERVICE_PROVIDERS` → `contentConfiguration` → `luigiConfigFragment`).
Docs adds:

1. A **Documentation entity node** under the existing `naira` entity, with
   one child node per published docs site, e.g.:

```ts
{
  pathSegment: 'docs',
  label: 'Documentation',
  entityType: 'naira',
  children: [
    {
      pathSegment: 'catalog',
      label: 'Naira Catalog',
      url: `${DOCS_BASE_URL}/catalog/latest/index.html`,
      loadingIndicator: { enabled: true },
    },
    // one node per component …
  ],
}
```

2. In v1 the node list MAY be static (env-configured), but the target state
   is that the portal backend builds it from the **docs index** — listing the
   bucket (or a small aggregated `index.json` written by CI) and generating
   one child node per site, so publishing a new docs site requires no portal
   change.
3. Luigi renders nodes in iframes and communicates via `postMessage`; the
   iframe cannot read the parent `window.location`, so deep links into doc
   pages are carried in the Luigi node path/urlSuffix. This is sufficient for
   v1 (open portal → docs → component → navigate inside the site).

### Search

Two layers, both in scope for v1:

- **Per-site search** comes for free: Material's built-in client-side search
  (lunr-based `search/search_index.json`) works standalone inside the
  iframe with zero infrastructure.
- **Portal-wide search**: CI pushes each site's `search/search_index.json`
  into a central **Meilisearch** instance (one index per docs site,
  documents keyed by page URL + section). The portal search box queries
  Meilisearch `/multi-search` with the `federation` parameter (Meilisearch
  ≥ 1.10), which returns a single merged, relevance-ranked result list
  across all docs indexes. Result URLs map back to Luigi deep links into the
  corresponding docs node. Meilisearch is deployed as one small stateful
  service in the platform, keeping the self-hosting guarantee.

### Deployment summary (new platform pieces)

- `ghcr.io/naira-project/mkdocs-build` image (new, owned by this feature).
- Reusable `docs-publish` workflow in `naira-github-workflows` (new).
- MinIO (or operator-provided S3 endpoint) + docs bucket (new in dev infra).
- Meilisearch deployment + a small CI step pushing search documents (new).
- Luigi node additions in `naira-openmfp-portal` (change).

## Rationale and Alternatives

**Why plain MkDocs + Material rather than Backstage TechDocs?** TechDocs was
evaluated in depth; the conclusion is that for Naira it is the wrong layer,
not merely an optional one:

- **`mkdocs-techdocs-core` is a wrapper, not magic.** It bundles ordinary
  MkDocs/PyMdown plugins (search, monorepo, admonitions, superfences,
  tabbed, PlantUML/Graphviz, etc.) that we can pin ourselves — and its own
  version pins on MkDocs/Material are a coupling liability.
- **Its static output is a complete Material site anyway.** The
  chrome-stripping and re-theming people associate with TechDocs happen
  *client-side inside the Backstage React reader*
  (`TechDocsReaderPageContent/dom.tsx`), not at build time. Served
  standalone in an iframe, a techdocs-core build renders as plain Material —
  so outside Backstage the plugin contributes nothing to rendering while
  inheriting its quirks (e.g. code annotations breaking under
  techdocs-core).
- **Everything genuinely useful is replicable in CI**, and this RFC does so
  one-for-one: pinned build container ↔ `spotify/techdocs` image; `INHERIT`
  base config ↔ auto-injected techdocs-core defaults; reusable workflow +
  `mc mirror` with a fixed path scheme ↔ `techdocs-cli publish` and its
  entity-triplet layout; `metadata.json` ↔ `techdocs_metadata.json`.
- **The rest is Backstage-only** (Reader UI, Addons, entity annotations,
  search collator) and has no meaning in an OpenMFP/Luigi portal.
- **Forward compatibility.** MkDocs core is effectively unmaintained (last
  release 1.6.1, 2024-08-30) and Material for MkDocs entered maintenance
  mode on 2025-11-05 with EOL scheduled for 2026-11-05. The designated
  successor, **Zensical** (same team, MIT), reads `mkdocs.yml` natively but
  supports only a mainstream plugin subset — which a plain, minimal Material
  config keeps us inside, and techdocs-core's bespoke bundle would not.
  Backstage's own RFC #33990 explores Zensical for the same reason.

**Why not extend the VitePress site (`naira.github.io`)?** VitePress serves
the public website well, but component docs need to live in each component's
repository and build independently, and MkDocs/Material is the de-facto
standard for docs-as-code in the cloud-native/platform space our
contributors come from. Centralizing all component docs into the website
repo would recreate exactly the disaggregation problem Naira exists to
solve. If the project later prefers one SSG everywhere, the pipeline shape
(build → bucket → iframe node) is SSG-agnostic; only the build image
changes.

**Why not render Markdown natively in the portal UI (no iframe)?** A native
renderer means reimplementing navigation, search, code highlighting,
admonitions, and theming — permanently, for every Material feature authors
use. The iframe boundary keeps the docs sites standard and independently
usable (a docs site URL works in a plain browser tab too) at the cost of
some visual seams, which Material theming (palette matched to the portal)
mitigates.

**Why object storage rather than serving docs from a git branch / GitHub
Pages?** The portal must work in self-hosted and air-gapped deployments;
GitHub Pages is external, unauthenticated-only, and cannot sit behind the
portal's ingress with the required CSP headers. A bucket in the platform
is uniform across dev (kind + MinIO) and production (any S3 API).

## Drawbacks

- **Toolchain sunset risk.** Material for MkDocs is in maintenance mode with
  EOL 2026-11-05, and an announced MkDocs 2.0 rewrite would remove the
  plugin system entirely. Mitigation: exact version pins in the build image
  (the stack does not rot just because upstream stops moving), a minimal
  mainstream plugin set, and a planned evaluation of Zensical once it
  reaches plugin/feature parity (Future Work). This risk argued *for*
  dropping techdocs-core, which would otherwise sit between us and any
  migration.
- **Iframe seams.** Docs render in an iframe with their own scrollbar,
  theme, and history behavior. Deep-link and browser-history integration
  via Luigi is workable but not free.
- **New stateful components.** MinIO (dev) and Meilisearch are added to the
  platform footprint. Both are small, standard, and self-hostable, but they
  are additional operational surface.
- **CI-coupled freshness.** Docs update when CI publishes, not live;
  `metadata.json` makes staleness visible rather than eliminating it.
- **Search duplication.** Per-site lunr search and portal-wide Meilisearch
  coexist; ranking and coverage can differ between them. Acceptable for v1;
  can be unified later by driving in-site search from Meilisearch too.

## Future Work (out of scope for this RFC)

- **Versioned docs.** Add `mike` (or an equivalent CI loop) to publish
  `docs/<component>/<x.y>/` alongside `latest`, with Material's version
  selector. The v1 path scheme already reserves the version segment.
- **Catalog integration.** Docs sites as first-class catalog assets: a
  cataloged plugin or model links to its docs node, and `metadata.json`
  feeds a "docs freshness" signal into the catalog.
- **Zensical migration.** Re-evaluate when Zensical reaches 1.0 / supports
  our plugin set; the config is deliberately kept Zensical-friendly.
- **Community plugin docs.** Let out-of-org plugin repositories publish into
  a Naira instance's docs bucket via a token-scoped publish endpoint,
  extending the same pipeline to the ecosystem.
- **Unified search experience.** Replace per-site lunr with a Meilisearch-
  backed in-site search for one consistent ranking.

## References

- MkDocs: https://www.mkdocs.org/ (v1.6.1, 2024-08-30)
- Material for MkDocs: https://squidfunk.github.io/mkdocs-material/
  (maintenance mode announced 2025-11-05; EOL 2026-11-05)
- Zensical: https://zensical.org/
- Backstage TechDocs / `mkdocs-techdocs-core`:
  https://github.com/backstage/mkdocs-techdocs-core
- Backstage RFC #33990 — "Exploring Zensical as the Next TechDocs
  Documentation Engine": https://github.com/backstage/backstage/issues/33990
- Luigi micro-frontend framework: https://luigi-project.io/
- OpenMFP: https://openmfp.org/
- Meilisearch federated multi-search:
  https://www.meilisearch.com/docs/reference/api/multi_search
- CSP `frame-ancestors`:
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/frame-ancestors
- MinIO: https://min.io/
- `mike` (versioned MkDocs deploys): https://github.com/jimporter/mike

## Changelog

- 2026-08-20: Initial draft.
