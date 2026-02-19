Explanation:

<explanation> ## 1) What this project is (type + purpose) This repo is a **large TypeScript monorepo** that contains: 1) **The tldraw SDK**: a React/TypeScript library for building infinite canvas / whiteboard experiences (`packages/*`, especially `packages/tldraw` + `packages/editor`). 2) **First-party web apps** built on the SDK: - **tldraw.com product app** (`apps/dotcom/client`) + its backend services (multiple Cloudflare Workers in `apps/dotcom/*`). - **Docs site** (`apps/docs`, Next.js). - **Examples app** (`apps/examples`, Vite + React) for local development and demos. - **Analytics widget/app** (`apps/analytics`) and **analytics worker** (`apps/analytics-worker`). - Several other workers (e.g. `apps/bemo-worker`). 3) **Developer tooling & distribution**: - Build / publish scripts (`internal/scripts/*`) - CI workflows (`.github/workflows/*`) - Templates and starters (`templates/*`, `packages/create-tldraw`) So your architecture diagram should not be “a single app”; it’s a **platform**: - a set of **published libraries (npm packages)**, - plus **multiple deployed web properties**, - plus **edge backend services** (Cloudflare Workers / Durable Objects) and storage layers. --- ## 2) How to read the file tree into architecture boundaries ### A. Monorepo boundaries Top-level structure tells you what to put in the diagram as “containers”: - `packages/*` → **SDK / libraries** (published artifacts, used by apps + external users) - `apps/*` → **deployed applications** (tldraw.com, docs, examples, analytics, workers) - `templates/*` → **reference architectures / starter kits** (not production runtime, but important as distribution & onboarding) - `internal/*` → **build/publish/deploy tooling** (CI, scripts, release automation) ### B. Key runtime domains you should diagram From the file tree, there are a few “systems” that deserve their own container-level view: 1) **SDK Runtime (in-browser)** `packages/tldraw`, `packages/editor`, `packages/store`, `packages/state`, `packages/sync`, `packages/sync-core`, `packages/tlschema`, `packages/utils`, `packages/assets`, etc. 2) **tldraw.com Runtime (product)** - Frontend: `apps/dotcom/client` - Backend edge services (Cloudflare): - `apps/dotcom/sync-worker` - `apps/dotcom/asset-upload-worker` - `apps/dotcom/image-resize-worker` - `apps/dotcom/tldrawusercontent-worker` - Storage/DB: - `apps/dotcom/zero-cache` (Postgres + migrations + pgbouncer config) - `sync-worker/src/r2.ts` indicates **Cloudflare R2** usage - `sync-worker/src/postgres.ts` indicates **Postgres** usage - `sync-worker/src/utils/createSupabaseClient.ts` indicates **Supabase** integration (likely auth / data access) 3) **Docs runtime** - `apps/docs` (Next.js app + scripts like Algolia indexing) 4) **Analytics runtime** - `apps/analytics` + `apps/analytics-worker` and client-side integrations (GA4, GTM, Hubspot, Posthog, etc) 5) **CI/CD + release** - `.github/workflows/*`, `internal/scripts/*`, `lerna.json`, `yarnrc.yml`, `sst.config.ts` (deployment orchestration is part of “system design” for this repo) --- ## 3) What the “main components” are (what to draw) ### A. SDK / library packages (draw as a dependency graph) At minimum, show these packages as boxes with arrows “depends-on / imports”: - `packages/tldraw` (public React component API: `Tldraw`, UI, default tools/shapes) - depends on `packages/editor` (core editor engine + React integration) - depends on `packages/tlschema` (records, migrations, schema) - depends on `packages/store` (record store) - depends on `packages/state` and `packages/state-react` (reactive primitives) - depends on `packages/utils` - uses `packages/assets` (icons/fonts/translations; URLs/self-hosting helpers) - optionally integrates with `packages/sync` (hooks) / `packages/sync-core` (protocol/storage adapters) - `packages/editor` - depends on `store/state/utils/tlschema` (core engine layering) - includes export pipeline (`src/lib/exports/*`) - includes licensing/watermark (`src/lib/license/*`) - `packages/sync` + `packages/sync-core` - client/server socket adapters, storage abstractions, durable object sqlite wrappers, protocol types - `packages/create-tldraw` - CLI scaffolding that generates projects from `templates/*` **Diagram tip:** Treat `packages/*` as **modules**; show directionality of dependencies (leaf utilities at bottom, higher-level UI packages at top). --- ### B. tldraw.com “product system” containers (draw as runtime architecture) For a production system diagram, include: **Client (browser)** - `apps/dotcom/client` (Vite + React) - uses SDK packages: primarily `tldraw` (and dotcom-specific shared libs: `packages/dotcom-shared`) - talks to: - Sync APIs (HTTP + WebSocket style flows) via `apps/dotcom/sync-worker` - Asset upload endpoints via `apps/dotcom/asset-upload-worker` / `tldrawusercontent-worker` - Image resize endpoints via `apps/dotcom/image-resize-worker` **Edge backend (Cloudflare Workers + Durable Objects)** - `apps/dotcom/sync-worker` - Durable Objects: `TLDrawDurableObject`, `TLFileDurableObject`, `TLUserDurableObject`, `TLStatsDurableObject`, `TLLoggerDurableObject` - Routes: `src/routes/*` and `src/routes/tla/*` (API surface) - Uses: - Postgres access (`src/postgres.ts`) - R2 object storage (`src/r2.ts`) - Replicator logic (`src/replicator/*`, `TLPostgresReplicator.ts`) - Rate limit/throttle utilities (`src/utils/rateLimit.ts`, `throttle.ts`) - Feature flags (`src/utils/featureFlags.ts`) - Supabase client (`src/utils/createSupabaseClient.ts`) (external auth/data service) - `apps/dotcom/asset-upload-worker` - handles direct upload flows, likely signs URLs / validates payloads - `apps/dotcom/tldrawusercontent-worker` - serves user-uploaded content (often a separate worker for origin isolation / caching) - `apps/dotcom/image-resize-worker` - image transforms at the edge (resizing pipeline) **Data plane** - **Postgres** (see `apps/dotcom/zero-cache/migrations/*`, docker compose, pgbouncer) - **Cloudflare R2** (asset blobs) - Possibly **Durable Object storage / SQLite** (common pattern for DO state; also see `packages/sync-core/*DurableObjectSqliteSyncWrapper*`) **Observability / external** - Sentry appears in dotcom client (`sentry.client.config.ts`) and worker-shared has sentry helpers (`packages/worker-shared/src/sentry.ts`) - Analytics integrations (also separate `apps/analytics`) --- ### C. Docs system (separate diagram or separate lane) - `apps/docs` (Next.js) - Content: `apps/docs/content/*` (MDX) - Search indexing: scripts include `update-algolia-index.ts` and `utils/algolia.ts` → **Algolia** should be drawn as an external dependency - Deployed likely to Vercel (has `vercel.json` patterns in other apps; docs has vercel build scripts in `internal/scripts/vercel/*`) --- ### D. Examples app (for dev workflow diagram) - `apps/examples` (Vite + React) used by `yarn dev` per README - imports local workspace packages (`packages/tldraw`, etc.) - E2E: Playwright tests in `apps/examples/e2e/*` This is important for a “developer experience architecture” diagram: how code changes flow into running examples, tests, and builds. --- ## 4) Relationships & interactions to explicitly show (arrows) ### A. Primary runtime data flows (tldraw.com) Include arrows with protocols: 1) **Realtime collaboration / sync** - Browser `apps/dotcom/client` → (WebSocket-like or long-lived connection / fetch) → `apps/dotcom/sync-worker` → Durable Objects (`TLDrawDurableObject`, `TLFileDurableObject`, etc.) → Postgres (`apps/dotcom/zero-cache` DB) and/or DO storage → R2 for snapshots/assets (where applicable) 2) **Asset upload** - Browser `apps/dotcom/client` → `asset-upload-worker` (request signed upload / policy) → uploads to storage (R2 or equivalent) → assets served via `tldrawusercontent-worker` (CDN/origin isolation) 3) **Image transforms** - Browser or worker → `image-resize-worker` → fetch source from storage → returns resized variants (cacheable) 4) **Auth / identity** - Browser / sync-worker → Supabase (external) (based on `createSupabaseClient.ts`) Show this as an external auth provider dependency. ### B. Package dependency flow (SDK) - Apps (`apps/*`, templates) → import `packages/tldraw` → which composes `packages/editor` + foundational packages This should be a separate “module” diagram to avoid mixing runtime and build-time concerns. ### C. Build & release flow (CI/CD) Use `.github/workflows/*` + `internal/scripts/*` to draw: - GitHub Actions pipelines (checks, publish, deploy) - publish flows to npm (Lerna/Yarn workspaces) - deployment flows to Vercel / Cloudflare / Fly.io (hinted by scripts and config files) --- ## 5) Architectural patterns worth highlighting (because they affect the diagram) 1) **Layered SDK architecture** - Low-level primitives (`utils`, `state`, `store`, `tlschema`) - Editor engine (`editor`) - Productized UI + defaults (`tldraw`) This layering should be visually obvious in your package diagram. 2) **Edge-first backend** - Cloudflare Workers + Durable Objects appear to be the main backend runtime for dotcom. - Durable Objects are stateful boundaries: draw them explicitly as “stateful compute units” rather than generic stateless services. 3) **Separation of concerns for assets** - Upload, serving, resizing are split into different workers. That’s an explicit architecture decision; diagram them separately. 4) **Replication / caching layer around Postgres** - `zero-cache` looks like a DB + replication/caching subsystem that sync-worker interacts with. - Replicator code in `sync-worker/src/replicator/*` suggests a deliberate replication pipeline: show it as a subcomponent inside sync-worker. --- ## 6) How to tailor the diagram to this repo (recommended diagram set) Because this repo is both a **library** and a **set of deployed services**, do **at least 3 diagrams** (C4-style), each very detailed: ### Diagram 1 — System Context (big picture) Audience: stakeholders / new engineers Include: - External developers using `tldraw` npm package - tldraw.dev docs users - tldraw.com end users - External services: Supabase, Algolia, Analytics vendors, Sentry - Hosting: Cloudflare (workers/R2), Vercel (docs/analytics apps if applicable), Fly.io (zero-cache template suggests it) ### Diagram 2 — Container diagram for tldraw.com Audience: principal engineer reviewing production architecture Include containers: - Dotcom web client - Sync-worker (routes + DOs) - Asset-upload-worker - tldrawusercontent-worker - image-resize-worker - Postgres (zero-cache) - R2 - Supabase - Observability (Sentry) Then annotate major endpoints (sync, upload, share/publish, snapshots). ### Diagram 3 — SDK Module/Package diagram (monorepo internal architecture) Audience: SDK maintainers Include: - Key `packages/*` nodes and arrows - Identify which packages are “public API” vs “internal utilities” - Mark build artifacts: “published to npm” for `tldraw`, `editor`, etc. Optional but useful: - Diagram 4 — CI/CD pipeline (GitHub Actions + internal scripts) - Diagram 5 — Template ecosystem (templates map to create-tldraw scaffolding) --- ## 7) What to include in the diagram (labels, arrows, shapes, color coding) ### A. Labels For each box, label: - **Name** (e.g. `apps/dotcom/sync-worker`) - **Runtime** (Cloudflare Worker, Durable Object, Next.js app, Vite SPA, Node script) - **Primary responsibility** (e.g. “room sync & replication”, “asset upload signing”, “docs rendering”) ### B. Directional arrows (be explicit) Use different arrow styles: - Solid arrow = request/response (HTTP fetch) - Double-ended / lightning = realtime (WebSocket/room session) - Dashed arrow = build-time dependency (imports / bundling) - Dotted arrow = async pipeline (replication, background jobs, alarms/cron—see `AlarmScheduler.ts`) Annotate arrows with: - protocol (HTTP, WS) - data type (snapshots, presence, assets) - auth mode (cookie/JWT/etc. if known) ### C. Color/shape scheme (simple but high-signal) - **Blue rectangles**: browser/client apps - **Orange hexagons**: edge workers (stateless) - **Red cylinders**: databases (Postgres) - **Purple buckets**: object storage (R2) - **Green rounded rectangles**: shared libraries/packages - **Gray boxes**: CI/CD + scripts Also add a legend. --- ## 8) Concrete mapping from repo paths to diagram nodes (starter list) ### Runtime (prod) - `apps/dotcom/client` → “tldraw.com web app (SPA)” - `apps/dotcom/sync-worker` → “Sync API + Durable Objects” - DOs: `TLDrawDurableObject`, `TLFileDurableObject`, `TLUserDurableObject`, `TLStatsDurableObject`, `TLLoggerDurableObject` - `apps/dotcom/asset-upload-worker` → “Asset upload worker” - `apps/dotcom/image-resize-worker` → “Image resize/transform worker” - `apps/dotcom/tldrawusercontent-worker` → “usercontent serving worker” - `apps/dotcom/zero-cache` → “Postgres + replication/cache layer” - External: Supabase, Cloudflare R2, Sentry ### Runtime (docs) - `apps/docs` → “tldraw.dev docs (Next.js)” - External: Algolia (indexing/search) ### SDK (packages) - `packages/tldraw` → “Public React SDK” - `packages/editor` → “Core editor engine” - `packages/sync` / `packages/sync-core` → “Sync protocol + storage adapters” - `packages/store`, `packages/state`, `packages/state-react` → “State/store foundation” - `packages/tlschema` → “Schema + migrations” - `packages/assets` → “Static assets + self-hosting helpers” - `packages/utils`, `packages/validate`, `packages/worker-shared` → “Utilities” --- ## 9) Don’t overthink it: maximize component separation For this repo, “best and most accurate” means you should: - **Split along repo boundaries** first (`apps/` vs `packages/` vs `templates/` vs `internal/`) - Then split runtime services further (each Worker and each Durable Object is its own box) - Then split storage/external systems (Postgres vs R2 vs Supabase vs Algolia vs analytics vendors) - Only after that, optionally group boxes into “domains” (SDK, Dotcom, Docs, Analytics, Tooling) If you do the above, the diagram will naturally match the repo’s real structure and will stay maintainable as the monorepo evolves. </explanation>

Mapping:

<component_mapping>
1. tldraw SDK (Public React SDK package): packages/tldraw
2. Core editor engine (Editor package): packages/editor
3. Schema + migrations (tlschema): packages/tlschema
4. Record store foundation (store): packages/store
5. Reactive primitives (state): packages/state
6. React bindings for state (state-react): packages/state-react
7. Sync hooks / higher-level sync client API (sync): packages/sync
8. Sync protocol + storage adapters (sync-core): packages/sync-core
9. Shared utilities (utils): packages/utils
10. Validation utilities (validate): packages/validate
11. Static assets + self-hosting helpers (assets package): packages/assets
12. Worker shared utilities (incl. Sentry helpers): packages/worker-shared (notably packages/worker-shared/src/sentry.ts)

13. tldraw.com web client (SPA): apps/dotcom/client (entry: apps/dotcom/client/src/main.tsx)
14. Dotcom client Sentry config: apps/dotcom/client/sentry.client.config.ts
15. Dotcom sync API + Durable Objects (Cloudflare Worker): apps/dotcom/sync-worker (entry: apps/dotcom/sync-worker/src/worker.ts)
16. Durable Object - TLDrawDurableObject: apps/dotcom/sync-worker/src/TLDrawDurableObject.ts
17. Durable Object - TLFileDurableObject: apps/dotcom/sync-worker/src/TLFileDurableObject.ts
18. Durable Object - TLUserDurableObject: apps/dotcom/sync-worker/src/TLUserDurableObject.ts
19. Durable Object - TLStatsDurableObject: apps/dotcom/sync-worker/src/TLStatsDurableObject.ts
20. Durable Object - TLLoggerDurableObject: apps/dotcom/sync-worker/src/TLLoggerDurableObject.ts
21. Sync-worker routes (API surface): apps/dotcom/sync-worker/src/routes
22. Sync-worker TLA routes: apps/dotcom/sync-worker/src/routes/tla
23. Postgres integration in sync-worker: apps/dotcom/sync-worker/src/postgres.ts
24. Cloudflare R2 integration in sync-worker: apps/dotcom/sync-worker/src/r2.ts
25. Supabase integration in sync-worker: apps/dotcom/sync-worker/src/utils/createSupabaseClient.ts
26. Replication pipeline (replicator code): apps/dotcom/sync-worker/src/replicator
27. Postgres replicator implementation: apps/dotcom/sync-worker/src/TLPostgresReplicator.ts
28. Rate limiting + throttling utilities: apps/dotcom/sync-worker/src/utils/rateLimit.ts, apps/dotcom/sync-worker/src/utils/throttle.ts
29. Feature flags utilities: apps/dotcom/sync-worker/src/utils/featureFlags.ts
30. Alarm/cron scheduler (background timing): apps/dotcom/sync-worker/src/AlarmScheduler.ts
31. Zero integration layer used by sync-worker: apps/dotcom/sync-worker/src/zero

32. Asset upload worker (Cloudflare Worker): apps/dotcom/asset-upload-worker (entry: apps/dotcom/asset-upload-worker/src/worker.ts)
33. Image resize/transform worker (Cloudflare Worker): apps/dotcom/image-resize-worker (entry: apps/dotcom/image-resize-worker/src/worker.ts)
34. User content serving worker (Cloudflare Worker): apps/dotcom/tldrawusercontent-worker (entry: apps/dotcom/tldrawusercontent-worker/src/worker.ts)

35. Postgres + migrations / “zero-cache” data layer: apps/dotcom/zero-cache (migrations: apps/dotcom/zero-cache/migrations, docker: apps/dotcom/zero-cache/docker)

36. Docs site (Next.js): apps/docs (routing/app dir: apps/docs/app)
37. Docs content (MDX): apps/docs/content
38. Algolia indexing script + Algolia client utilities: apps/docs/scripts/update-algolia-index.ts, apps/docs/utils/algolia.ts

39. Analytics widget/app (Vite): apps/analytics (entry: apps/analytics/src/index.ts)
40. Analytics service integrations (GA4/GTM/Hubspot/Posthog/etc): apps/analytics/src/analytics-services
41. Analytics worker (Cloudflare Worker): apps/analytics-worker (entry: apps/analytics-worker/src/worker.ts)

42. Examples app (Vite + React): apps/examples (entry: apps/examples/src/index.tsx)
43. Examples E2E tests (Playwright): apps/examples/e2e

44. Bemo worker (Cloudflare Worker + DO): apps/bemo-worker (DO: apps/bemo-worker/src/BemoDO.ts; entry: apps/bemo-worker/src/worker.ts)

45. create-tldraw CLI scaffolding tool: packages/create-tldraw (entry: packages/create-tldraw/src/main.ts; templates selector: packages/create-tldraw/src/templates.ts)

46. Templates / starter kits (scaffold targets): templates
47. Sync Cloudflare template (reference architecture): templates/sync-cloudflare (worker: templates/sync-cloudflare/worker; client: templates/sync-cloudflare/client)
48. Simple server example template (reference architecture): templates/simple-server-example/src/server
49. Socket.io server example template (reference architecture): templates/socketio-server-example/src/server

50. CI/CD workflows (GitHub Actions): .github/workflows
51. Build/publish/deploy scripts: internal/scripts
52. Vercel build helper scripts: internal/scripts/vercel
53. Deploy scripts for properties: internal/scripts/deploy-dotcom.ts, internal/scripts/deploy-docs-on-release-edit.yml, internal/scripts/deploy-analytics.ts (and workflow: .github/workflows/deploy-analytics.yml)
54. Monorepo tooling configs (workspaces/publishing): lerna.json, yarnrc.yml, package.json
55. Deployment orchestration config (SST): sst.config.ts
</component_mapping>
Mermaid.js diagram:

flowchart TD
  %% =========================
  %% Legend (colors / shapes)
  %% =========================
  subgraph "Legend"
    direction TB
    L1["Client App (Browser)"]:::client
    L2{{"Edge Worker (Cloudflare)"} }:::worker
    L3[("Durable Object (Stateful)")]:::durable
    L4[("Database (Postgres)")]:::db
    L5[("Object Storage (R2)")]:::object
    L6(["Library / Package (npm)"]):::pkg
    L7["Tooling / CI-CD"]:::tooling
    L8["External Service"]:::external
  end

  %% ==========================================================
  %% Diagram 1 — System Context (big picture, stakeholders)
  %% ==========================================================
  subgraph "Diagram 1 — System Context"
    direction TB

    EndUsers["End Users\n(tldraw.com)"]:::external
    Devs["External Developers\n(npm consumers)"]:::external
    DocsUsers["Docs Users\n(tldraw.dev)"]:::external

    subgraph "Published SDK (npm)"
      direction TB
      PkgTldraw["tldraw (React SDK)\nPublished package"]:::pkg
      PkgEditor["editor (Core engine)\nPublished package"]:::pkg
      PkgSync["sync (Sync hooks)\nPublished package"]:::pkg
      PkgSyncCore["sync-core (Protocol + adapters)\nPublished package"]:::pkg
    end

    subgraph "Deployed Web Properties"
      direction TB
      DotcomClient["tldraw.com Web App\n(Vite + React SPA)"]:::client
      DocsApp["tldraw.dev Docs\n(Next.js)"]:::client
      AnalyticsApp["Analytics Widget/App\n(Vite)"]:::client
    end

    subgraph "Edge Backends (Cloudflare)"
      direction TB
      DotcomSyncWorker{{"Dotcom Sync API\n(Worker + Durable Objects)"} }:::worker
      AssetUploadWorker{{"Asset Upload Worker"} }:::worker
      UserContentWorker{{"User Content Serving Worker"} }:::worker
      ImageResizeWorker{{"Image Resize/Transform Worker"} }:::worker
      AnalyticsWorker{{"Analytics Worker"} }:::worker
      BemoWorker{{"Bemo Worker\n(Worker + DO)"} }:::worker
    end

    subgraph "Data Plane"
      direction TB
      Postgres[("Postgres\n(dotcom/zero-cache)")]:::db
      R2[("Cloudflare R2\n(Asset blobs)")]:::object
      DOStorage[("Durable Object Storage / SQLite\n(state)")]:::db
    end

    subgraph "External Services"
      direction TB
      Supabase["Supabase\n(Auth/identity/data access)"]:::external
      Algolia["Algolia\n(Search indexing + query)"]:::external
      Sentry["Sentry\n(Errors + performance)"]:::external
      Vendors["Analytics Vendors\n(GA4/GTM/Hubspot/Posthog/...)"]:::external
    end

    EndUsers -->|"uses"| DotcomClient
    DocsUsers -->|"browses"| DocsApp
    Devs -->|"installs"| PkgTldraw

    DotcomClient -->|"HTTP/WS-sync"| DotcomSyncWorker
    DotcomClient -->|"HTTP-upload"| AssetUploadWorker
    DotcomClient -->|"HTTP-assets"| UserContentWorker
    DotcomClient -->|"HTTP-images"| ImageResizeWorker

    DotcomSyncWorker -->|"reads/writes"| Postgres
    DotcomSyncWorker -->|"stores/reads blobs"| R2
    DotcomSyncWorker -->|"state"| DOStorage
    DotcomSyncWorker -->|"auth/identity"| Supabase

    DocsApp -->|"index/search"| Algolia

    DotcomClient -->|"errors"| Sentry
    DotcomSyncWorker -->|"errors"| Sentry

    AnalyticsApp -->|"collect/events"| AnalyticsWorker
    AnalyticsWorker -->|"forwards"| Vendors

    BemoWorker -->|"state"| DOStorage

    DotcomClient -.->|"imports"| PkgTldraw
    DocsApp -.->|"imports"| PkgTldraw
    AnalyticsApp -.->|"imports"| PkgTldraw
  end

  %% ==========================================================
  %% Diagram 2 — Container diagram for tldraw.com (production)
  %% ==========================================================
  subgraph "Diagram 2 — tldraw.com Container Diagram (Runtime)"
    direction TB

    subgraph "Browser"
      direction TB
      DotcomSPA["apps/dotcom/client\nVite + React SPA"]:::client
      DotcomSentryCfg["Sentry client config"]:::tooling
    end

    subgraph "Cloudflare Workers (Dotcom)"
      direction TB

      SyncWorker{{"apps/dotcom/sync-worker\nSync API + Durable Objects"} }:::worker
      SyncRoutes["Routes\n(src/routes/*)"]:::worker
      SyncTlaRoutes["TLA Routes\n(src/routes/tla/*)"]:::worker

      subgraph "Durable Objects (Stateful compute)"
        direction TB
        DO_Draw[("TLDrawDurableObject")]:::durable
        DO_File[("TLFileDurableObject")]:::durable
        DO_User[("TLUserDurableObject")]:::durable
        DO_Stats[("TLStatsDurableObject")]:::durable
        DO_Logger[("TLLoggerDurableObject")]:::durable
      end

      subgraph "Sync-worker internals"
        direction TB
        PgAdapter["Postgres integration\n(postgres.ts)"]:::worker
        R2Adapter["R2 integration\n(r2.ts)"]:::worker
        SupabaseClient["Supabase client\n(createSupabaseClient.ts)"]:::worker
        ReplicatorDir["Replication pipeline\n(src/replicator/*)"]:::worker
        PgReplicator["TLPostgresReplicator.ts"]:::worker
        ZeroIntegration["Zero integration\n(src/zero/*)"]:::worker
        AlarmScheduler["Alarm/cron scheduler\n(AlarmScheduler.ts)"]:::worker
        RateLimit["Rate limit utils\n(rateLimit.ts)"]:::worker
        Throttle["Throttle utils\n(throttle.ts)"]:::worker
        FeatureFlags["Feature flags\n(featureFlags.ts)"]:::worker
      end

      AssetWorker{{"apps/dotcom/asset-upload-worker\nAsset upload signing/validation"} }:::worker
      UsercontentWorker{{"apps/dotcom/tldrawusercontent-worker\nServe user content"} }:::worker
      ResizeWorker{{"apps/dotcom/image-resize-worker\nEdge image transforms"} }:::worker
    end

    subgraph "Data Stores"
      direction TB
      ZeroCache[("apps/dotcom/zero-cache\nPostgres + migrations + pgbouncer")]:::db
      CF_R2[("Cloudflare R2\nAsset blobs") ]:::object
      DO_DB[("Durable Object Storage / SQLite\nState") ]:::db
    end

    subgraph "External Dependencies"
      direction TB
      ExtSupabase["Supabase\n(Auth/identity)"]:::external
      ExtSentry["Sentry"]:::external
    end

    DotcomSPA -->|"HTTP/WS-like realtime"| SyncWorker
    DotcomSPA -->|"HTTP-request policy/signature"| AssetWorker
    DotcomSPA -->|"HTTP-fetch assets"| UsercontentWorker
    DotcomSPA -->|"HTTP-resize requests"| ResizeWorker

    %% Sync API surface
    SyncWorker -->|"dispatches"| SyncRoutes
    SyncWorker -->|"dispatches"| SyncTlaRoutes

    %% Durable objects
    SyncWorker <-->|"realtime rooms/files/users"| DO_Draw
    SyncWorker <-->|"file sessions"| DO_File
    SyncWorker <-->|"user sessions"| DO_User
    SyncWorker <-->|"stats"| DO_Stats
    SyncWorker <-->|"logging"| DO_Logger

    %% Data plane
    SyncWorker -->|"read/write"| PgAdapter -->|"SQL"| ZeroCache
    SyncWorker -->|"blob IO"| R2Adapter -->|"S3 API"| CF_R2
    DO_Draw -->|"state"| DO_DB
    DO_File -->|"state"| DO_DB
    DO_User -->|"state"| DO_DB
    DO_Stats -->|"state"| DO_DB
    DO_Logger -->|"state"| DO_DB

    %% Replication/caching pattern
    SyncWorker ..>|"async replication"| ReplicatorDir
    ReplicatorDir -->|"uses"| PgReplicator -->|"writes/reads"| ZeroCache
    SyncWorker -->|"uses"| ZeroIntegration -->|"backing DB"| ZeroCache

    %% Control-plane utilities
    SyncWorker -->|"schedules"| AlarmScheduler
    SyncWorker -->|"limits"| RateLimit
    SyncWorker -->|"limits"| Throttle
    SyncWorker -->|"toggles"| FeatureFlags

    %% Auth/identity + observability
    DotcomSPA -->|"auth"| ExtSupabase
    SyncWorker -->|"auth/identity"| SupabaseClient -->|"API"| ExtSupabase
    DotcomSPA -->|"error reporting"| ExtSentry
    SyncWorker -->|"error reporting"| ExtSentry
    DotcomSentryCfg -.->|"configures"| ExtSentry

    %% Asset flow details
    AssetWorker -->|"store blobs"| CF_R2
    UsercontentWorker -->|"read blobs"| CF_R2
    ResizeWorker -->|"fetch source"| CF_R2
    ResizeWorker -->|"serve transformed"| DotcomSPA
  end

  %% ==========================================================
  %% Diagram 3 — SDK Module/Package diagram (monorepo internals)
  %% ==========================================================
  subgraph "Diagram 3 — SDK Packages (Layered Architecture)"
    direction TB

    subgraph "Apps import SDK (build-time)"
      direction TB
      AppDotcom["apps/dotcom/client"]:::client
      AppDocs["apps/docs"]:::client
      AppExamples["apps/examples"]:::client
      AppAnalytics["apps/analytics"]:::client
    end

    subgraph "Public API (higher-level packages)"
      direction TB
      TldrawPkg["packages/tldraw\nPublic React SDK"]:::pkg
      EditorPkg["packages/editor\nCore editor engine"]:::pkg
      SyncPkg["packages/sync\nSync hooks/client API"]:::pkg
      SyncCorePkg["packages/sync-core\nProtocol + adapters"]:::pkg
      CreateTldrawPkg["packages/create-tldraw\nCLI scaffolder"]:::pkg
    end

    subgraph "Foundation (lower-level packages)"
      direction TB
      TLSchemaPkg["packages/tlschema\nSchema + migrations"]:::pkg
      StorePkg["packages/store\nRecord store"]:::pkg
      StatePkg["packages/state\nReactive primitives"]:::pkg
      StateReactPkg["packages/state-react\nReact bindings"]:::pkg
      UtilsPkg["packages/utils\nShared utilities"]:::pkg
      ValidatePkg["packages/validate\nValidation utilities"]:::pkg
      AssetsPkg["packages/assets\nStatic assets + helpers"]:::pkg
      WorkerSharedPkg["packages/worker-shared\nWorker utilities (incl. Sentry)"]:::pkg
      WorkerSharedSentry["worker-shared sentry helpers\n(src/sentry.ts)"]:::pkg
    end

    %% Apps -> SDK
    AppDotcom -.->|"imports"| TldrawPkg
    AppDocs -.->|"imports"| TldrawPkg
    AppExamples -.->|"imports"| TldrawPkg
    AppAnalytics -.->|"imports"| TldrawPkg

    %% Layering: tldraw at top
    TldrawPkg -.->|"depends-on"| EditorPkg
    TldrawPkg -.->|"depends-on"| TLSchemaPkg
    TldrawPkg -.->|"depends-on"| StorePkg
    TldrawPkg -.->|"depends-on"| StatePkg
    TldrawPkg -.->|"depends-on"| StateReactPkg
    TldrawPkg -.->|"depends-on"| UtilsPkg
    TldrawPkg -.->|"uses"| AssetsPkg
    TldrawPkg -.->|"optional sync"| SyncPkg

    %% editor depends on foundations
    EditorPkg -.->|"depends-on"| TLSchemaPkg
    EditorPkg -.->|"depends-on"| StorePkg
    EditorPkg -.->|"depends-on"| StatePkg
    EditorPkg -.->|"depends-on"| UtilsPkg
    EditorPkg -.->|"depends-on"| ValidatePkg

    %% sync layering
    SyncPkg -.->|"depends-on"| SyncCorePkg
    SyncPkg -.->|"depends-on"| StorePkg
    SyncPkg -.->|"depends-on"| StatePkg
    SyncPkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"depends-on"| TLSchemaPkg
    SyncCorePkg -.->|"depends-on"| StorePkg
    SyncCorePkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"worker helpers"| WorkerSharedPkg
    WorkerSharedPkg -.->|"includes"| WorkerSharedSentry
  end

  %% ==========================================================
  %% Diagram 4 — CI/CD + release automation (build/deploy)
  %% ==========================================================
  subgraph "Diagram 4 — CI/CD + Release"
    direction TB

    subgraph "Repo Tooling"
      direction TB
      GHWorkflows[".github/workflows\nGitHub Actions pipelines"]:::tooling
      InternalScripts["internal/scripts\nBuild/publish/deploy scripts"]:::tooling
      VercelScripts["internal/scripts/vercel\nVercel helpers"]:::tooling
      DeployDotcom["deploy-dotcom.ts"]:::tooling
      DeployAnalytics["deploy-analytics.ts"]:::tooling
      DeployDocsWorkflow["deploy-docs-on-release-edit.yml"]:::tooling
      DeployAnalyticsWorkflow["deploy-analytics.yml"]:::tooling
      LernaCfg["lerna.json"]:::tooling
      Yarnrc["yarnrc.yml"]:::tooling
      RootPkgJson["package.json"]:::tooling
      SSTCfg["sst.config.ts\nDeployment orchestration"]:::tooling
    end

    subgraph "Deploy Targets"
      direction TB
      NPM["npm registry\n(Published packages)"]:::external
      Cloudflare["Cloudflare\n(Workers/DO/R2)"]:::external
      Vercel["Vercel\n(Next.js deployments)"]:::external
    end

    GHWorkflows -->|"runs"| InternalScripts
    GHWorkflows -->|"uses"| LernaCfg
    GHWorkflows -->|"uses"| Yarnrc
    GHWorkflows -->|"uses"| RootPkgJson
    GHWorkflows -->|"orchestrates"| SSTCfg

    InternalScripts -->|"deploy"| DeployDotcom -->|"targets"| Cloudflare
    InternalScripts -->|"deploy"| DeployAnalytics -->|"targets"| Cloudflare
    GHWorkflows -->|"docs release"| DeployDocsWorkflow
    GHWorkflows -->|"analytics deploy"| DeployAnalyticsWorkflow

    VercelScripts -->|"build/deploy docs"| Vercel
    GHWorkflows -->|"publish packages"| NPM
  end

  %% ==========================================================
  %% Diagram 5 — Templates ecosystem (create-tldraw + templates)
  %% ==========================================================
  subgraph "Diagram 5 — Templates & Starters"
    direction TB

    TemplatesDir["templates/*\nStarter kits / reference architectures"]:::tooling
    SyncCloudflareTpl["sync-cloudflare template\n(worker + client)"]:::tooling
    SimpleServerTpl["simple-server-example template\n(server)"]:::tooling
    SocketIOTpl["socketio-server-example template\n(server)"]:::tooling

    CreateTldrawCli["create-tldraw CLI\n(scaffold projects)"]:::tooling

    CreateTldrawCli -->|"selects/installs"| TemplatesDir
    TemplatesDir -->|"includes"| SyncCloudflareTpl
    TemplatesDir -->|"includes"| SimpleServerTpl
    TemplatesDir -->|"includes"| SocketIOTpl
  end

  %% =========================
  %% Click events (paths only)
  %% =========================

  %% Packages
  click TldrawPkg "packages/tldraw"
  click PkgTldraw "packages/tldraw"
  click EditorPkg "packages/editor"
  click PkgEditor "packages/editor"
  click TLSchemaPkg "packages/tlschema"
  click StorePkg "packages/store"
  click StatePkg "packages/state"
  click StateReactPkg "packages/state-react"
  click SyncPkg "packages/sync"
  click PkgSync "packages/sync"
  click SyncCorePkg "packages/sync-core"
  click PkgSyncCore "packages/sync-core"
  click UtilsPkg "packages/utils"
  click ValidatePkg "packages/validate"
  click AssetsPkg "packages/assets"
  click WorkerSharedPkg "packages/worker-shared"
  click WorkerSharedSentry "packages/worker-shared/src/sentry.ts"
  click CreateTldrawPkg "packages/create-tldraw"
  click CreateTldrawCli "packages/create-tldraw/src/main.ts"

  %% Dotcom client + config
  click DotcomSPA "apps/dotcom/client"
  click DotcomClient "apps/dotcom/client"
  click AppDotcom "apps/dotcom/client"
  click DotcomSentryCfg "apps/dotcom/client/sentry.client.config.ts"

  %% Sync worker + internals
  click SyncWorker "apps/dotcom/sync-worker"
  click DotcomSyncWorker "apps/dotcom/sync-worker"
  click SyncRoutes "apps/dotcom/sync-worker/src/routes"
  click SyncTlaRoutes "apps/dotcom/sync-worker/src/routes/tla"
  click DO_Draw "apps/dotcom/sync-worker/src/TLDrawDurableObject.ts"
  click DO_File "apps/dotcom/sync-worker/src/TLFileDurableObject.ts"
  click DO_User "apps/dotcom/sync-worker/src/TLUserDurableObject.ts"
  click DO_Stats "apps/dotcom/sync-worker/src/TLStatsDurableObject.ts"
  click DO_Logger "apps/dotcom/sync-worker/src/TLLoggerDurableObject.ts"
  click PgAdapter "apps/dotcom/sync-worker/src/postgres.ts"
  click R2Adapter "apps/dotcom/sync-worker/src/r2.ts"
  click SupabaseClient "apps/dotcom/sync-worker/src/utils/createSupabaseClient.ts"
  click ReplicatorDir "apps/dotcom/sync-worker/src/replicator"
  click PgReplicator "apps/dotcom/sync-worker/src/TLPostgresReplicator.ts"
  click RateLimit "apps/dotcom/sync-worker/src/utils/rateLimit.ts"
  click Throttle "apps/dotcom/sync-worker/src/utils/throttle.ts"
  click FeatureFlags "apps/dotcom/sync-worker/src/utils/featureFlags.ts"
  click AlarmScheduler "apps/dotcom/sync-worker/src/AlarmScheduler.ts"
  click ZeroIntegration "apps/dotcom/sync-worker/src/zero"

  %% Other dotcom workers
  click AssetWorker "apps/dotcom/asset-upload-worker"
  click AssetUploadWorker "apps/dotcom/asset-upload-worker"
  click ResizeWorker "apps/dotcom/image-resize-worker"
  click ImageResizeWorker "apps/dotcom/image-resize-worker"
  click UsercontentWorker "apps/dotcom/tldrawusercontent-worker"
  click UserContentWorker "apps/dotcom/tldrawusercontent-worker"

  %% Zero-cache
  click ZeroCache "apps/dotcom/zero-cache"

  %% Docs
  click DocsApp "apps/docs"
  click AppDocs "apps/docs"

  %% Analytics
  click AnalyticsApp "apps/analytics"
  click AppAnalytics "apps/analytics"
  click AnalyticsWorker "apps/analytics-worker"

  %% Examples
  click AppExamples "apps/examples"

  %% Bemo
  click BemoWorker "apps/bemo-worker"

  %% Templates
  click TemplatesDir "templates"
  click SyncCloudflareTpl "templates/sync-cloudflare"
  click SimpleServerTpl "templates/simple-server-example/src/server"
  click SocketIOTpl "templates/socketio-server-example/src/server"

  %% CI/CD + scripts + configs
  click GHWorkflows ".github/workflows"
  click InternalScripts "internal/scripts"
  click VercelScripts "internal/scripts/vercel"
  click DeployDotcom "internal/scripts/deploy-dotcom.ts"
  click DeployAnalytics "internal/scripts/deploy-analytics.ts"
  click DeployDocsWorkflow "internal/scripts/deploy-docs-on-release-edit.yml"
  click DeployAnalyticsWorkflow ".github/workflows/deploy-analytics.yml"
  click LernaCfg "lerna.json"
  click Yarnrc "yarnrc.yml"
  click RootPkgJson "package.json"
  click SSTCfg "sst.config.ts"

  %% =========================
  %% Styles
  %% =========================
  classDef client fill:#1e88e5,stroke:#0d47a1,color:#ffffff,stroke-width:1.5px
  classDef worker fill:#fb8c00,stroke:#e65100,color:#111111,stroke-width:1.5px
  classDef durable fill:#c62828,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef db fill:#b71c1c,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef object fill:#6a1b9a,stroke:#4a148c,color:#ffffff,stroke-width:1.5px
  classDef pkg fill:#2e7d32,stroke:#1b5e20,color:#ffffff,stroke-width:1.5px
  classDef tooling fill:#546e7a,stroke:#263238,color:#ffffff,stroke-width:1.5px
  classDef external fill:#eeeeee,stroke:#616161,color:#111111,stroke-width:1.2px
Syntax Repair Loop

Attempt 3/3

Fixing Mermaid syntax (attempt 3/3)...

message: Parse error on line 163:
...O_DB    SyncWorker ..>|"async replicati
----------------------^
Expecting 'SEMI', 'NEWLINE', 'EOF', 'AMP', 'START_LINK', 'LINK', 'LINK_ID', got 'NODE_STRING'
line: 162
token: NODE_STRING
expected: 'SEMI', 'NEWLINE', 'EOF', 'AMP', 'START_LINK', 'LINK', 'LINK_ID'
Candidate Mermaid fix (streaming)

flowchart TD
  %% =========================
  %% Legend (colors / shapes)
  %% =========================
  subgraph "Legend"
    direction TB
    L1["Client App (Browser)"]:::client
    L2{{"Edge Worker (Cloudflare)"}}:::worker
    L3[("Durable Object (Stateful)")]:::durable
    L4[("Database (Postgres)")]:::db
    L5[("Object Storage (R2)")]:::object
    L6(["Library / Package (npm)"]):::pkg
    L7["Tooling / CI-CD"]:::tooling
    L8["External Service"]:::external
  end

  %% ==========================================================
  %% Diagram 1 — System Context (big picture, stakeholders)
  %% ==========================================================
  subgraph "Diagram 1 — System Context"
    direction TB

    EndUsers["End Users\n(tldraw.com)"]:::external
    Devs["External Developers\n(npm consumers)"]:::external
    DocsUsers["Docs Users\n(tldraw.dev)"]:::external

    subgraph "Published SDK (npm)"
      direction TB
      PkgTldraw["tldraw (React SDK)\nPublished package"]:::pkg
      PkgEditor["editor (Core engine)\nPublished package"]:::pkg
      PkgSync["sync (Sync hooks)\nPublished package"]:::pkg
      PkgSyncCore["sync-core (Protocol + adapters)\nPublished package"]:::pkg
    end

    subgraph "Deployed Web Properties"
      direction TB
      DotcomClient["tldraw.com Web App\n(Vite + React SPA)"]:::client
      DocsApp["tldraw.dev Docs\n(Next.js)"]:::client
      AnalyticsApp["Analytics Widget/App\n(Vite)"]:::client
    end

    subgraph "Edge Backends (Cloudflare)"
      direction TB
      DotcomSyncWorker{{"Dotcom Sync API\n(Worker + Durable Objects)"}}:::worker
      AssetUploadWorker{{"Asset Upload Worker"}}:::worker
      UserContentWorker{{"User Content Serving Worker"}}:::worker
      ImageResizeWorker{{"Image Resize/Transform Worker"}}:::worker
      AnalyticsWorker{{"Analytics Worker"}}:::worker
      BemoWorker{{"Bemo Worker\n(Worker + DO)"}}:::worker
    end

    subgraph "Data Plane"
      direction TB
      Postgres[("Postgres\n(dotcom/zero-cache)")]:::db
      R2[("Cloudflare R2\n(Asset blobs)")]:::object
      DOStorage[("Durable Object Storage / SQLite\n(state)")]:::db
    end

    subgraph "External Services"
      direction TB
      Supabase["Supabase\n(Auth/identity/data access)"]:::external
      Algolia["Algolia\n(Search indexing + query)"]:::external
      Sentry["Sentry\n(Errors + performance)"]:::external
      Vendors["Analytics Vendors\n(GA4/GTM/Hubspot/Posthog/...)"]:::external
    end

    EndUsers -->|"uses"| DotcomClient
    DocsUsers -->|"browses"| DocsApp
    Devs -->|"installs"| PkgTldraw

    DotcomClient -->|"HTTP/WS-sync"| DotcomSyncWorker
    DotcomClient -->|"HTTP-upload"| AssetUploadWorker
    DotcomClient -->|"HTTP-assets"| UserContentWorker
    DotcomClient -->|"HTTP-images"| ImageResizeWorker

    DotcomSyncWorker -->|"reads/writes"| Postgres
    DotcomSyncWorker -->|"stores/reads blobs"| R2
    DotcomSyncWorker -->|"state"| DOStorage
    DotcomSyncWorker -->|"auth/identity"| Supabase

    DocsApp -->|"index/search"| Algolia

    DotcomClient -->|"errors"| Sentry
    DotcomSyncWorker -->|"errors"| Sentry

    AnalyticsApp -->|"collect/events"| AnalyticsWorker
    AnalyticsWorker -->|"forwards"| Vendors

    BemoWorker -->|"state"| DOStorage

    DotcomClient -.->|"imports"| PkgTldraw
    DocsApp -.->|"imports"| PkgTldraw
    AnalyticsApp -.->|"imports"| PkgTldraw
  end

  %% ==========================================================
  %% Diagram 2 — Container diagram for tldraw.com (production)
  %% ==========================================================
  subgraph "Diagram 2 — tldraw.com Container Diagram (Runtime)"
    direction TB

    subgraph "Browser"
      direction TB
      DotcomSPA["apps/dotcom/client\nVite + React SPA"]:::client
      DotcomSentryCfg["Sentry client config"]:::tooling
    end

    subgraph "Cloudflare Workers (Dotcom)"
      direction TB

      SyncWorker{{"apps/dotcom/sync-worker\nSync API + Durable Objects"}}:::worker
      SyncRoutes["Routes\n(src/routes/*)"]:::worker
      SyncTlaRoutes["TLA Routes\n(src/routes/tla/*)"]:::worker

      subgraph "Durable Objects (Stateful compute)"
        direction TB
        DO_Draw[("TLDrawDurableObject")]:::durable
        DO_File[("TLFileDurableObject")]:::durable
        DO_User[("TLUserDurableObject")]:::durable
        DO_Stats[("TLStatsDurableObject")]:::durable
        DO_Logger[("TLLoggerDurableObject")]:::durable
      end

      subgraph "Sync-worker internals"
        direction TB
        PgAdapter["Postgres integration\n(postgres.ts)"]:::worker
        R2Adapter["R2 integration\n(r2.ts)"]:::worker
        SupabaseClient["Supabase client\n(createSupabaseClient.ts)"]:::worker
        ReplicatorDir["Replication pipeline\n(src/replicator/*)"]:::worker
        PgReplicator["TLPostgresReplicator.ts"]:::worker
        ZeroIntegration["Zero integration\n(src/zero/*)"]:::worker
        AlarmScheduler["Alarm/cron scheduler\n(AlarmScheduler.ts)"]:::worker
        RateLimit["Rate limit utils\n(rateLimit.ts)"]:::worker
        Throttle["Throttle utils\n(throttle.ts)"]:::worker
        FeatureFlags["Feature flags\n(featureFlags.ts)"]:::worker
      end

      AssetWorker{{"apps/dotcom/asset-upload-worker\nAsset upload signing/validation"}}:::worker
      UsercontentWorker{{"apps/dotcom/tldrawusercontent-worker\nServe user content"}}:::worker
      ResizeWorker{{"apps/dotcom/image-resize-worker\nEdge image transforms"}}:::worker
    end

    subgraph "Data Stores"
      direction TB
      ZeroCache[("apps/dotcom/zero-cache\nPostgres + migrations + pgbouncer")]:::db
      CF_R2[("Cloudflare R2\nAsset blobs")]:::object
      DO_DB[("Durable Object Storage / SQLite\nState")]:::db
    end

    subgraph "External Dependencies"
      direction TB
      ExtSupabase["Supabase\n(Auth/identity)"]:::external
      ExtSentry["Sentry"]:::external
    end

    DotcomSPA -->|"HTTP/WS-like realtime"| SyncWorker
    DotcomSPA -->|"HTTP-request policy/signature"| AssetWorker
    DotcomSPA -->|"HTTP-fetch assets"| UsercontentWorker
    DotcomSPA -->|"HTTP-resize requests"| ResizeWorker

    %% Sync API surface
    SyncWorker -->|"dispatches"| SyncRoutes
    SyncWorker -->|"dispatches"| SyncTlaRoutes

    %% Durable objects
    SyncWorker <-->|"realtime rooms/files/users"| DO_Draw
    SyncWorker <-->|"file sessions"| DO_File
    SyncWorker <-->|"user sessions"| DO_User
    SyncWorker <-->|"stats"| DO_Stats
    SyncWorker <-->|"logging"| DO_Logger

    %% Data plane
    SyncWorker -->|"read/write"| PgAdapter -->|"SQL"| ZeroCache
    SyncWorker -->|"blob IO"| R2Adapter -->|"S3 API"| CF_R2
    DO_Draw -->|"state"| DO_DB
    DO_File -->|"state"| DO_DB
    DO_User -->|"state"| DO_DB
    DO_Stats -->|"state"| DO_DB
    DO_Logger -->|"state"| DO_DB

    %% Replication/caching pattern
    SyncWorker ..>|"async replication"| ReplicatorDir
    ReplicatorDir -->|"uses"| PgReplicator -->|"writes/reads"| ZeroCache
    SyncWorker -->|"uses"| ZeroIntegration -->|"backing DB"| ZeroCache

    %% Control-plane utilities
    SyncWorker -->|"schedules"| AlarmScheduler
    SyncWorker -->|"limits"| RateLimit
    SyncWorker -->|"limits"| Throttle
    SyncWorker -->|"toggles"| FeatureFlags

    %% Auth/identity + observability
    DotcomSPA -->|"auth"| ExtSupabase
    SyncWorker -->|"auth/identity"| SupabaseClient -->|"API"| ExtSupabase
    DotcomSPA -->|"error reporting"| ExtSentry
    SyncWorker -->|"error reporting"| ExtSentry
    DotcomSentryCfg -.->|"configures"| ExtSentry

    %% Asset flow details
    AssetWorker -->|"store blobs"| CF_R2
    UsercontentWorker -->|"read blobs"| CF_R2
    ResizeWorker -->|"fetch source"| CF_R2
    ResizeWorker -->|"serve transformed"| DotcomSPA
  end

  %% ==========================================================
  %% Diagram 3 — SDK Module/Package diagram (monorepo internals)
  %% ==========================================================
  subgraph "Diagram 3 — SDK Packages (Layered Architecture)"
    direction TB

    subgraph "Apps import SDK (build-time)"
      direction TB
      AppDotcom["apps/dotcom/client"]:::client
      AppDocs["apps/docs"]:::client
      AppExamples["apps/examples"]:::client
      AppAnalytics["apps/analytics"]:::client
    end

    subgraph "Public API (higher-level packages)"
      direction TB
      TldrawPkg["packages/tldraw\nPublic React SDK"]:::pkg
      EditorPkg["packages/editor\nCore editor engine"]:::pkg
      SyncPkg["packages/sync\nSync hooks/client API"]:::pkg
      SyncCorePkg["packages/sync-core\nProtocol + adapters"]:::pkg
      CreateTldrawPkg["packages/create-tldraw\nCLI scaffolder"]:::pkg
    end

    subgraph "Foundation (lower-level packages)"
      direction TB
      TLSchemaPkg["packages/tlschema\nSchema + migrations"]:::pkg
      StorePkg["packages/store\nRecord store"]:::pkg
      StatePkg["packages/state\nReactive primitives"]:::pkg
      StateReactPkg["packages/state-react\nReact bindings"]:::pkg
      UtilsPkg["packages/utils\nShared utilities"]:::pkg
      ValidatePkg["packages/validate\nValidation utilities"]:::pkg
      AssetsPkg["packages/assets\nStatic assets + helpers"]:::pkg
      WorkerSharedPkg["packages/worker-shared\nWorker utilities (incl. Sentry)"]:::pkg
      WorkerSharedSentry["worker-shared sentry helpers\n(src/sentry.ts)"]:::pkg
    end

    %% Apps -> SDK
    AppDotcom -.->|"imports"| TldrawPkg
    AppDocs -.->|"imports"| TldrawPkg
    AppExamples -.->|"imports"| TldrawPkg
    AppAnalytics -.->|"imports"| TldrawPkg

    %% Layering: tldraw at top
    TldrawPkg -.->|"depends-on"| EditorPkg
    TldrawPkg -.->|"depends-on"| TLSchemaPkg
    TldrawPkg -.->|"depends-on"| StorePkg
    TldrawPkg -.->|"depends-on"| StatePkg
    TldrawPkg -.->|"depends-on"| StateReactPkg
    TldrawPkg -.->|"depends-on"| UtilsPkg
    TldrawPkg -.->|"uses"| AssetsPkg
    TldrawPkg -.->|"optional sync"| SyncPkg

    %% editor depends on foundations
    EditorPkg -.->|"depends-on"| TLSchemaPkg
    EditorPkg -.->|"depends-on"| StorePkg
    EditorPkg -.->|"depends-on"| StatePkg
    EditorPkg -.->|"depends-on"| UtilsPkg
    EditorPkg -.->|"depends-on"| ValidatePkg

    %% sync layering
    SyncPkg -.->|"depends-on"| SyncCorePkg
    SyncPkg -.->|"depends-on"| StorePkg
    SyncPkg -.->|"depends-on"| StatePkg
    SyncPkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"depends-on"| TLSchemaPkg
    SyncCorePkg -.->|"depends-on"| StorePkg
    SyncCorePkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"worker helpers"| WorkerSharedPkg
    WorkerSharedPkg -.->|"includes"| WorkerSharedSentry
  end

  %% ==========================================================
  %% Diagram 4 — CI/CD + release automation (build/deploy)
  %% ==========================================================
  subgraph "Diagram 4 — CI/CD + Release"
    direction TB

    subgraph "Repo Tooling"
      direction TB
      GHWorkflows[".github/workflows\nGitHub Actions pipelines"]:::tooling
      InternalScripts["internal/scripts\nBuild/publish/deploy scripts"]:::tooling
      VercelScripts["internal/scripts/vercel\nVercel helpers"]:::tooling
      DeployDotcom["deploy-dotcom.ts"]:::tooling
      DeployAnalytics["deploy-analytics.ts"]:::tooling
      DeployDocsWorkflow["deploy-docs-on-release-edit.yml"]:::tooling
      DeployAnalyticsWorkflow["deploy-analytics.yml"]:::tooling
      LernaCfg["lerna.json"]:::tooling
      Yarnrc["yarnrc.yml"]:::tooling
      RootPkgJson["package.json"]:::tooling
      SSTCfg["sst.config.ts\nDeployment orchestration"]:::tooling
    end

    subgraph "Deploy Targets"
      direction TB
      NPM["npm registry\n(Published packages)"]:::external
      Cloudflare["Cloudflare\n(Workers/DO/R2)"]:::external
      Vercel["Vercel\n(Next.js deployments)"]:::external
    end

    GHWorkflows -->|"runs"| InternalScripts
    GHWorkflows -->|"uses"| LernaCfg
    GHWorkflows -->|"uses"| Yarnrc
    GHWorkflows -->|"uses"| RootPkgJson
    GHWorkflows -->|"orchestrates"| SSTCfg

    InternalScripts -->|"deploy"| DeployDotcom -->|"targets"| Cloudflare
    InternalScripts -->|"deploy"| DeployAnalytics -->|"targets"| Cloudflare
    GHWorkflows -->|"docs release"| DeployDocsWorkflow
    GHWorkflows -->|"analytics deploy"| DeployAnalyticsWorkflow

    VercelScripts -->|"build/deploy docs"| Vercel
    GHWorkflows -->|"publish packages"| NPM
  end

  %% ==========================================================
  %% Diagram 5 — Templates ecosystem (create-tldraw + templates)
  %% ==========================================================
  subgraph "Diagram 5 — Templates & Starters"
    direction TB

    TemplatesDir["templates/*\nStarter kits / reference architectures"]:::tooling
    SyncCloudflareTpl["sync-cloudflare template\n(worker + client)"]:::tooling
    SimpleServerTpl["simple-server-example template\n(server)"]:::tooling
    SocketIOTpl["socketio-server-example template\n(server)"]:::tooling

    CreateTldrawCli["create-tldraw CLI\n(scaffold projects)"]:::tooling

    CreateTldrawCli -->|"selects/installs"| TemplatesDir
    TemplatesDir -->|"includes"| SyncCloudflareTpl
    TemplatesDir -->|"includes"| SimpleServerTpl
    TemplatesDir -->|"includes"| SocketIOTpl
  end

  %% =========================
  %% Click events (paths only)
  %% =========================

  %% Packages
  click TldrawPkg "packages/tldraw"
  click PkgTldraw "packages/tldraw"
  click EditorPkg "packages/editor"
  click PkgEditor "packages/editor"
  click TLSchemaPkg "packages/tlschema"
  click StorePkg "packages/store"
  click StatePkg "packages/state"
  click StateReactPkg "packages/state-react"
  click SyncPkg "packages/sync"
  click PkgSync "packages/sync"
  click SyncCorePkg "packages/sync-core"
  click PkgSyncCore "packages/sync-core"
  click UtilsPkg "packages/utils"
  click ValidatePkg "packages/validate"
  click AssetsPkg "packages/assets"
  click WorkerSharedPkg "packages/worker-shared"
  click WorkerSharedSentry "packages/worker-shared/src/sentry.ts"
  click CreateTldrawPkg "packages/create-tldraw"
  click CreateTldrawCli "packages/create-tldraw/src/main.ts"

  %% Dotcom client + config
  click DotcomSPA "apps/dotcom/client"
  click DotcomClient "apps/dotcom/client"
  click AppDotcom "apps/dotcom/client"
  click DotcomSentryCfg "apps/dotcom/client/sentry.client.config.ts"

  %% Sync worker + internals
  click SyncWorker "apps/dotcom/sync-worker"
  click DotcomSyncWorker "apps/dotcom/sync-worker"
  click SyncRoutes "apps/dotcom/sync-worker/src/routes"
  click SyncTlaRoutes "apps/dotcom/sync-worker/src/routes/tla"
  click DO_Draw "apps/dotcom/sync-worker/src/TLDrawDurableObject.ts"
  click DO_File "apps/dotcom/sync-worker/src/TLFileDurableObject.ts"
  click DO_User "apps/dotcom/sync-worker/src/TLUserDurableObject.ts"
  click DO_Stats "apps/dotcom/sync-worker/src/TLStatsDurableObject.ts"
  click DO_Logger "apps/dotcom/sync-worker/src/TLLoggerDurableObject.ts"
  click PgAdapter "apps/dotcom/sync-worker/src/postgres.ts"
  click R2Adapter "apps/dotcom/sync-worker/src/r2.ts"
  click SupabaseClient "apps/dotcom/sync-worker/src/utils/createSupabaseClient.ts"
  click ReplicatorDir "apps/dotcom/sync-worker/src/replicator"
  click PgReplicator "apps/dotcom/sync-worker/src/TLPostgresReplicator.ts"
  click RateLimit "apps/dotcom/sync-worker/src/utils/rateLimit.ts"
  click Throttle "apps/dotcom/sync-worker/src/utils/throttle.ts"
  click FeatureFlags "apps/dotcom/sync-worker/src/utils/featureFlags.ts"
  click AlarmScheduler "apps/dotcom/sync-worker/src/AlarmScheduler.ts"
  click ZeroIntegration "apps/dotcom/sync-worker/src/zero"

  %% Other dotcom workers
  click AssetWorker "apps/dotcom/asset-upload-worker"
  click AssetUploadWorker "apps/dotcom/asset-upload-worker"
  click ResizeWorker "apps/dotcom/image-resize-worker"
  click ImageResizeWorker "apps/dotcom/image-resize-worker"
  click UsercontentWorker "apps/dotcom/tldrawusercontent-worker"
  click UserContentWorker "apps/dotcom/tldrawusercontent-worker"

  %% Zero-cache
  click ZeroCache "apps/dotcom/zero-cache"

  %% Docs
  click DocsApp "apps/docs"
  click AppDocs "apps/docs"

  %% Analytics
  click AnalyticsApp "apps/analytics"
  click AppAnalytics "apps/analytics"
  click AnalyticsWorker "apps/analytics-worker"

  %% Examples
  click AppExamples "apps/examples"

  %% Bemo
  click BemoWorker "apps/bemo-worker"

  %% Templates
  click TemplatesDir "templates"
  click SyncCloudflareTpl "templates/sync-cloudflare"
  click SimpleServerTpl "templates/simple-server-example/src/server"
  click SocketIOTpl "templates/socketio-server-example/src/server"

  %% CI/CD + scripts + configs
  click GHWorkflows ".github/workflows"
  click InternalScripts "internal/scripts"
  click VercelScripts "internal/scripts/vercel"
  click DeployDotcom "internal/scripts/deploy-dotcom.ts"
  click DeployAnalytics "internal/scripts/deploy-analytics.ts"
  click DeployDocsWorkflow "internal/scripts/deploy-docs-on-release-edit.yml"
  click DeployAnalyticsWorkflow ".github/workflows/deploy-analytics.yml"
  click LernaCfg "lerna.json"
  click Yarnrc "yarnrc.yml"
  click RootPkgJson "package.json"
  click SSTCfg "sst.config.ts"

  %% =========================
  %% Styles
  %% =========================
  classDef client fill:#1e88e5,stroke:#0d47a1,color:#ffffff,stroke-width:1.5px
  classDef worker fill:#fb8c00,stroke:#e65100,color:#111111,stroke-width:1.5px
  classDef durable fill:#c62828,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef db fill:#b71c1c,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef object fill:#6a1b9a,stroke:#4a148c,color:#ffffff,stroke-width:1.5px
  classDef pkg fill:#2e7d32,stroke:#1b5e20,color:#ffffff,stroke-width:1.5px
  classDef tooling fill:#546e7a,stroke:#263238,color:#ffffff,stroke-width:1.5px
  classDef external fill:#eeeeee,stroke:#616161,color:#111111,stroke-width:1.2pxflowchart TD
  %% =========================
  %% Legend (colors / shapes)
  %% =========================
  subgraph "Legend"
    direction TB
    L1["Client App (Browser)"]:::client
    L2{{"Edge Worker (Cloudflare)"}}:::worker
    L3[("Durable Object (Stateful)")]:::durable
    L4[("Database (Postgres)")]:::db
    L5[("Object Storage (R2)")]:::object
    L6(["Library / Package (npm)"]):::pkg
    L7["Tooling / CI-CD"]:::tooling
    L8["External Service"]:::external
  end

  %% ==========================================================
  %% Diagram 1 — System Context (big picture, stakeholders)
  %% ==========================================================
  subgraph "Diagram 1 — System Context"
    direction TB

    EndUsers["End Users\n(tldraw.com)"]:::external
    Devs["External Developers\n(npm consumers)"]:::external
    DocsUsers["Docs Users\n(tldraw.dev)"]:::external

    subgraph "Published SDK (npm)"
      direction TB
      PkgTldraw["tldraw (React SDK)\nPublished package"]:::pkg
      PkgEditor["editor (Core engine)\nPublished package"]:::pkg
      PkgSync["sync (Sync hooks)\nPublished package"]:::pkg
      PkgSyncCore["sync-core (Protocol + adapters)\nPublished package"]:::pkg
    end

    subgraph "Deployed Web Properties"
      direction TB
      DotcomClient["tldraw.com Web App\n(Vite + React SPA)"]:::client
      DocsApp["tldraw.dev Docs\n(Next.js)"]:::client
      AnalyticsApp["Analytics Widget/App\n(Vite)"]:::client
    end

    subgraph "Edge Backends (Cloudflare)"
      direction TB
      DotcomSyncWorker{{"Dotcom Sync API\n(Worker + Durable Objects)"}}:::worker
      AssetUploadWorker{{"Asset Upload Worker"}}:::worker
      UserContentWorker{{"User Content Serving Worker"}}:::worker
      ImageResizeWorker{{"Image Resize/Transform Worker"}}:::worker
      AnalyticsWorker{{"Analytics Worker"}}:::worker
      BemoWorker{{"Bemo Worker\n(Worker + DO)"}}:::worker
    end

    subgraph "Data Plane"
      direction TB
      Postgres[("Postgres\n(dotcom/zero-cache)")]:::db
      R2[("Cloudflare R2\n(Asset blobs)")]:::object
      DOStorage[("Durable Object Storage / SQLite\n(state)")]:::db
    end

    subgraph "External Services"
      direction TB
      Supabase["Supabase\n(Auth/identity/data access)"]:::external
      Algolia["Algolia\n(Search indexing + query)"]:::external
      Sentry["Sentry\n(Errors + performance)"]:::external
      Vendors["Analytics Vendors\n(GA4/GTM/Hubspot/Posthog/...)"]:::external
    end

    EndUsers -->|"uses"| DotcomClient
    DocsUsers -->|"browses"| DocsApp
    Devs -->|"installs"| PkgTldraw

    DotcomClient -->|"HTTP/WS-sync"| DotcomSyncWorker
    DotcomClient -->|"HTTP-upload"| AssetUploadWorker
    DotcomClient -->|"HTTP-assets"| UserContentWorker
    DotcomClient -->|"HTTP-images"| ImageResizeWorker

    DotcomSyncWorker -->|"reads/writes"| Postgres
    DotcomSyncWorker -->|"stores/reads blobs"| R2
    DotcomSyncWorker -->|"state"| DOStorage
    DotcomSyncWorker -->|"auth/identity"| Supabase

    DocsApp -->|"index/search"| Algolia

    DotcomClient -->|"errors"| Sentry
    DotcomSyncWorker -->|"errors"| Sentry

    AnalyticsApp -->|"collect/events"| AnalyticsWorker
    AnalyticsWorker -->|"forwards"| Vendors

    BemoWorker -->|"state"| DOStorage

    DotcomClient -.->|"imports"| PkgTldraw
    DocsApp -.->|"imports"| PkgTldraw
    AnalyticsApp -.->|"imports"| PkgTldraw
  end

  %% ==========================================================
  %% Diagram 2 — Container diagram for tldraw.com (production)
  %% ==========================================================
  subgraph "Diagram 2 — tldraw.com Container Diagram (Runtime)"
    direction TB

    subgraph "Browser"
      direction TB
      DotcomSPA["apps/dotcom/client\nVite + React SPA"]:::client
      DotcomSentryCfg["Sentry client config"]:::tooling
    end

    subgraph "Cloudflare Workers (Dotcom)"
      direction TB

      SyncWorker{{"apps/dotcom/sync-worker\nSync API + Durable Objects"}}:::worker
      SyncRoutes["Routes\n(src/routes/*)"]:::worker
      SyncTlaRoutes["TLA Routes\n(src/routes/tla/*)"]:::worker

      subgraph "Durable Objects (Stateful compute)"
        direction TB
        DO_Draw[("TLDrawDurableObject")]:::durable
        DO_File[("TLFileDurableObject")]:::durable
        DO_User[("TLUserDurableObject")]:::durable
        DO_Stats[("TLStatsDurableObject")]:::durable
        DO_Logger[("TLLoggerDurableObject")]:::durable
      end

      subgraph "Sync-worker internals"
        direction TB
        PgAdapter["Postgres integration\n(postgres.ts)"]:::worker
        R2Adapter["R2 integration\n(r2.ts)"]:::worker
        SupabaseClient["Supabase client\n(createSupabaseClient.ts)"]:::worker
        ReplicatorDir["Replication pipeline\n(src/replicator/*)"]:::worker
        PgReplicator["TLPostgresReplicator.ts"]:::worker
        ZeroIntegration["Zero integration\n(src/zero/*)"]:::worker
        AlarmScheduler["Alarm/cron scheduler\n(AlarmScheduler.ts)"]:::worker
        RateLimit["Rate limit utils\n(rateLimit.ts)"]:::worker
        Throttle["Throttle utils\n(throttle.ts)"]:::worker
        FeatureFlags["Feature flags\n(featureFlags.ts)"]:::worker
      end

      AssetWorker{{"apps/dotcom/asset-upload-worker\nAsset upload signing/validation"}}:::worker
      UsercontentWorker{{"apps/dotcom/tldrawusercontent-worker\nServe user content"}}:::worker
      ResizeWorker{{"apps/dotcom/image-resize-worker\nEdge image transforms"}}:::worker
    end

    subgraph "Data Stores"
      direction TB
      ZeroCache[("apps/dotcom/zero-cache\nPostgres + migrations + pgbouncer")]:::db
      CF_R2[("Cloudflare R2\nAsset blobs")]:::object
      DO_DB[("Durable Object Storage / SQLite\nState")]:::db
    end

    subgraph "External Dependencies"
      direction TB
      ExtSupabase["Supabase\n(Auth/identity)"]:::external
      ExtSentry["Sentry"]:::external
    end

    DotcomSPA -->|"HTTP/WS-like realtime"| SyncWorker
    DotcomSPA -->|"HTTP-request policy/signature"| AssetWorker
    DotcomSPA -->|"HTTP-fetch assets"| UsercontentWorker
    DotcomSPA -->|"HTTP-resize requests"| ResizeWorker

    %% Sync API surface
    SyncWorker -->|"dispatches"| SyncRoutes
    SyncWorker -->|"dispatches"| SyncTlaRoutes

    %% Durable objects
    SyncWorker <-->|"realtime rooms/files/users"| DO_Draw
    SyncWorker <-->|"file sessions"| DO_File
    SyncWorker <-->|"user sessions"| DO_User
    SyncWorker <-->|"stats"| DO_Stats
    SyncWorker <-->|"logging"| DO_Logger

    %% Data plane
    SyncWorker -->|"read/write"| PgAdapter -->|"SQL"| ZeroCache
    SyncWorker -->|"blob IO"| R2Adapter -->|"S3 API"| CF_R2
    DO_Draw -->|"state"| DO_DB
    DO_File -->|"state"| DO_DB
    DO_User -->|"state"| DO_DB
    DO_Stats -->|"state"| DO_DB
    DO_Logger -->|"state"| DO_DB

    %% Replication/caching pattern
    SyncWorker ..>|"async replication"| ReplicatorDir
    ReplicatorDir -->|"uses"| PgReplicator -->|"writes/reads"| ZeroCache
    SyncWorker -->|"uses"| ZeroIntegration -->|"backing DB"| ZeroCache

    %% Control-plane utilities
    SyncWorker -->|"schedules"| AlarmScheduler
    SyncWorker -->|"limits"| RateLimit
    SyncWorker -->|"limits"| Throttle
    SyncWorker -->|"toggles"| FeatureFlags

    %% Auth/identity + observability
    DotcomSPA -->|"auth"| ExtSupabase
    SyncWorker -->|"auth/identity"| SupabaseClient -->|"API"| ExtSupabase
    DotcomSPA -->|"error reporting"| ExtSentry
    SyncWorker -->|"error reporting"| ExtSentry
    DotcomSentryCfg -.->|"configures"| ExtSentry

    %% Asset flow details
    AssetWorker -->|"store blobs"| CF_R2
    UsercontentWorker -->|"read blobs"| CF_R2
    ResizeWorker -->|"fetch source"| CF_R2
    ResizeWorker -->|"serve transformed"| DotcomSPA
  end

  %% ==========================================================
  %% Diagram 3 — SDK Module/Package diagram (monorepo internals)
  %% ==========================================================
  subgraph "Diagram 3 — SDK Packages (Layered Architecture)"
    direction TB

    subgraph "Apps import SDK (build-time)"
      direction TB
      AppDotcom["apps/dotcom/client"]:::client
      AppDocs["apps/docs"]:::client
      AppExamples["apps/examples"]:::client
      AppAnalytics["apps/analytics"]:::client
    end

    subgraph "Public API (higher-level packages)"
      direction TB
      TldrawPkg["packages/tldraw\nPublic React SDK"]:::pkg
      EditorPkg["packages/editor\nCore editor engine"]:::pkg
      SyncPkg["packages/sync\nSync hooks/client API"]:::pkg
      SyncCorePkg["packages/sync-core\nProtocol + adapters"]:::pkg
      CreateTldrawPkg["packages/create-tldraw\nCLI scaffolder"]:::pkg
    end

    subgraph "Foundation (lower-level packages)"
      direction TB
      TLSchemaPkg["packages/tlschema\nSchema + migrations"]:::pkg
      StorePkg["packages/store\nRecord store"]:::pkg
      StatePkg["packages/state\nReactive primitives"]:::pkg
      StateReactPkg["packages/state-react\nReact bindings"]:::pkg
      UtilsPkg["packages/utils\nShared utilities"]:::pkg
      ValidatePkg["packages/validate\nValidation utilities"]:::pkg
      AssetsPkg["packages/assets\nStatic assets + helpers"]:::pkg
      WorkerSharedPkg["packages/worker-shared\nWorker utilities (incl. Sentry)"]:::pkg
      WorkerSharedSentry["worker-shared sentry helpers\n(src/sentry.ts)"]:::pkg
    end

    %% Apps -> SDK
    AppDotcom -.->|"imports"| TldrawPkg
    AppDocs -.->|"imports"| TldrawPkg
    AppExamples -.->|"imports"| TldrawPkg
    AppAnalytics -.->|"imports"| TldrawPkg

    %% Layering: tldraw at top
    TldrawPkg -.->|"depends-on"| EditorPkg
    TldrawPkg -.->|"depends-on"| TLSchemaPkg
    TldrawPkg -.->|"depends-on"| StorePkg
    TldrawPkg -.->|"depends-on"| StatePkg
    TldrawPkg -.->|"depends-on"| StateReactPkg
    TldrawPkg -.->|"depends-on"| UtilsPkg
    TldrawPkg -.->|"uses"| AssetsPkg
    TldrawPkg -.->|"optional sync"| SyncPkg

    %% editor depends on foundations
    EditorPkg -.->|"depends-on"| TLSchemaPkg
    EditorPkg -.->|"depends-on"| StorePkg
    EditorPkg -.->|"depends-on"| StatePkg
    EditorPkg -.->|"depends-on"| UtilsPkg
    EditorPkg -.->|"depends-on"| ValidatePkg

    %% sync layering
    SyncPkg -.->|"depends-on"| SyncCorePkg
    SyncPkg -.->|"depends-on"| StorePkg
    SyncPkg -.->|"depends-on"| StatePkg
    SyncPkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"depends-on"| TLSchemaPkg
    SyncCorePkg -.->|"depends-on"| StorePkg
    SyncCorePkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"worker helpers"| WorkerSharedPkg
    WorkerSharedPkg -.->|"includes"| WorkerSharedSentry
  end

  %% ==========================================================
  %% Diagram 4 — CI/CD + release automation (build/deploy)
  %% ==========================================================
  subgraph "Diagram 4 — CI/CD + Release"
    direction TB

    subgraph "Repo Tooling"
      direction TB
      GHWorkflows[".github/workflows\nGitHub Actions pipelines"]:::tooling
      InternalScripts["internal/scripts\nBuild/publish/deploy scripts"]:::tooling
      VercelScripts["internal/scripts/vercel\nVercel helpers"]:::tooling
      DeployDotcom["deploy-dotcom.ts"]:::tooling
      DeployAnalytics["deploy-analytics.ts"]:::tooling
      DeployDocsWorkflow["deploy-docs-on-release-edit.yml"]:::tooling
      DeployAnalyticsWorkflow["deploy-analytics.yml"]:::tooling
      LernaCfg["lerna.json"]:::tooling
      Yarnrc["yarnrc.yml"]:::tooling
      RootPkgJson["package.json"]:::tooling
      SSTCfg["sst.config.ts\nDeployment orchestration"]:::tooling
    end

    subgraph "Deploy Targets"
      direction TB
      NPM["npm registry\n(Published packages)"]:::external
      Cloudflare["Cloudflare\n(Workers/DO/R2)"]:::external
      Vercel["Vercel\n(Next.js deployments)"]:::external
    end

    GHWorkflows -->|"runs"| InternalScripts
    GHWorkflows -->|"uses"| LernaCfg
    GHWorkflows -->|"uses"| Yarnrc
    GHWorkflows -->|"uses"| RootPkgJson
    GHWorkflows -->|"orchestrates"| SSTCfg

    InternalScripts -->|"deploy"| DeployDotcom -->|"targets"| Cloudflare
    InternalScripts -->|"deploy"| DeployAnalytics -->|"targets"| Cloudflare
    GHWorkflows -->|"docs release"| DeployDocsWorkflow
    GHWorkflows -->|"analytics deploy"| DeployAnalyticsWorkflow

    VercelScripts -->|"build/deploy docs"| Vercel
    GHWorkflows -->|"publish packages"| NPM
  end

  %% ==========================================================
  %% Diagram 5 — Templates ecosystem (create-tldraw + templates)
  %% ==========================================================
  subgraph "Diagram 5 — Templates & Starters"
    direction TB

    TemplatesDir["templates/*\nStarter kits / reference architectures"]:::tooling
    SyncCloudflareTpl["sync-cloudflare template\n(worker + client)"]:::tooling
    SimpleServerTpl["simple-server-example template\n(server)"]:::tooling
    SocketIOTpl["socketio-server-example template\n(server)"]:::tooling

    CreateTldrawCli["create-tldraw CLI\n(scaffold projects)"]:::tooling

    CreateTldrawCli -->|"selects/installs"| TemplatesDir
    TemplatesDir -->|"includes"| SyncCloudflareTpl
    TemplatesDir -->|"includes"| SimpleServerTpl
    TemplatesDir -->|"includes"| SocketIOTpl
  end

  %% =========================
  %% Click events (paths only)
  %% =========================

  %% Packages
  click TldrawPkg "packages/tldraw"
  click PkgTldraw "packages/tldraw"
  click EditorPkg "packages/editor"
  click PkgEditor "packages/editor"
  click TLSchemaPkg "packages/tlschema"
  click StorePkg "packages/store"
  click StatePkg "packages/state"
  click StateReactPkg "packages/state-react"
  click SyncPkg "packages/sync"
  click PkgSync "packages/sync"
  click SyncCorePkg "packages/sync-core"
  click PkgSyncCore "packages/sync-core"
  click UtilsPkg "packages/utils"
  click ValidatePkg "packages/validate"
  click AssetsPkg "packages/assets"
  click WorkerSharedPkg "packages/worker-shared"
  click WorkerSharedSentry "packages/worker-shared/src/sentry.ts"
  click CreateTldrawPkg "packages/create-tldraw"
  click CreateTldrawCli "packages/create-tldraw/src/main.ts"

  %% Dotcom client + config
  click DotcomSPA "apps/dotcom/client"
  click DotcomClient "apps/dotcom/client"
  click AppDotcom "apps/dotcom/client"
  click DotcomSentryCfg "apps/dotcom/client/sentry.client.config.ts"

  %% Sync worker + internals
  click SyncWorker "apps/dotcom/sync-worker"
  click DotcomSyncWorker "apps/dotcom/sync-worker"
  click SyncRoutes "apps/dotcom/sync-worker/src/routes"
  click SyncTlaRoutes "apps/dotcom/sync-worker/src/routes/tla"
  click DO_Draw "apps/dotcom/sync-worker/src/TLDrawDurableObject.ts"
  click DO_File "apps/dotcom/sync-worker/src/TLFileDurableObject.ts"
  click DO_User "apps/dotcom/sync-worker/src/TLUserDurableObject.ts"
  click DO_Stats "apps/dotcom/sync-worker/src/TLStatsDurableObject.ts"
  click DO_Logger "apps/dotcom/sync-worker/src/TLLoggerDurableObject.ts"
  click PgAdapter "apps/dotcom/sync-worker/src/postgres.ts"
  click R2Adapter "apps/dotcom/sync-worker/src/r2.ts"
  click SupabaseClient "apps/dotcom/sync-worker/src/utils/createSupabaseClient.ts"
  click ReplicatorDir "apps/dotcom/sync-worker/src/replicator"
  click PgReplicator "apps/dotcom/sync-worker/src/TLPostgresReplicator.ts"
  click RateLimit "apps/dotcom/sync-worker/src/utils/rateLimit.ts"
  click Throttle "apps/dotcom/sync-worker/src/utils/throttle.ts"
  click FeatureFlags "apps/dotcom/sync-worker/src/utils/featureFlags.ts"
  click AlarmScheduler "apps/dotcom/sync-worker/src/AlarmScheduler.ts"
  click ZeroIntegration "apps/dotcom/sync-worker/src/zero"

  %% Other dotcom workers
  click AssetWorker "apps/dotcom/asset-upload-worker"
  click AssetUploadWorker "apps/dotcom/asset-upload-worker"
  click ResizeWorker "apps/dotcom/image-resize-worker"
  click ImageResizeWorker "apps/dotcom/image-resize-worker"
  click UsercontentWorker "apps/dotcom/tldrawusercontent-worker"
  click UserContentWorker "apps/dotcom/tldrawusercontent-worker"

  %% Zero-cache
  click ZeroCache "apps/dotcom/zero-cache"

  %% Docs
  click DocsApp "apps/docs"
  click AppDocs "apps/docs"

  %% Analytics
  click AnalyticsApp "apps/analytics"
  click AppAnalytics "apps/analytics"
  click AnalyticsWorker "apps/analytics-worker"

  %% Examples
  click AppExamples "apps/examples"

  %% Bemo
  click BemoWorker "apps/bemo-worker"

  %% Templates
  click TemplatesDir "templates"
  click SyncCloudflareTpl "templates/sync-cloudflare"
  click SimpleServerTpl "templates/simple-server-example/src/server"
  click SocketIOTpl "templates/socketio-server-example/src/server"

  %% CI/CD + scripts + configs
  click GHWorkflows ".github/workflows"
  click InternalScripts "internal/scripts"
  click VercelScripts "internal/scripts/vercel"
  click DeployDotcom "internal/scripts/deploy-dotcom.ts"
  click DeployAnalytics "internal/scripts/deploy-analytics.ts"
  click DeployDocsWorkflow "internal/scripts/deploy-docs-on-release-edit.yml"
  click DeployAnalyticsWorkflow ".github/workflows/deploy-analytics.yml"
  click LernaCfg "lerna.json"
  click Yarnrc "yarnrc.yml"
  click RootPkgJson "package.json"
  click SSTCfg "sst.config.ts"

  %% =========================
  %% Styles
  %% =========================
  classDef client fill:#1e88e5,stroke:#0d47a1,color:#ffffff,stroke-width:1.5px
  classDef worker fill:#fb8c00,stroke:#e65100,color:#111111,stroke-width:1.5px
  classDef durable fill:#c62828,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef db fill:#b71c1c,stroke:#7f0000,color:#ffffff,stroke-width:1.5px
  classDef object fill:#6a1b9a,stroke:#4a148c,color:#ffffff,stroke-width:1.5px
  classDef pkg fill:#2e7d32,stroke:#1b5e20,color:#ffffff,stroke-width:1.5px
  classDef tooling fill:#546e7a,stroke:#263238,color:#ffffff,stroke-width:1.5px
  classDef external fill:#eeeeee,stroke:#616161,color:#111111,stroke-width:1.2pxflowchart TD
  %% =========================
  %% Legend (colors / shapes)
  %% =========================
  subgraph "Legend"
    direction TB
    L1["Client App (Browser)"]:::client
    L2{{"Edge Worker (Cloudflare)"}}:::worker
    L3[("Durable Object (Stateful)")]:::durable
    L4[("Database (Postgres)")]:::db
    L5[("Object Storage (R2)")]:::object
    L6(["Library / Package (npm)"]):::pkg
    L7["Tooling / CI-CD"]:::tooling
    L8["External Service"]:::external
  end

  %% ==========================================================
  %% Diagram 1 — System Context (big picture, stakeholders)
  %% ==========================================================
  subgraph "Diagram 1 — System Context"
    direction TB

    EndUsers["End Users\n(tldraw.com)"]:::external
    Devs["External Developers\n(npm consumers)"]:::external
    DocsUsers["Docs Users\n(tldraw.dev)"]:::external

    subgraph "Published SDK (npm)"
      direction TB
      PkgTldraw["tldraw (React SDK)\nPublished package"]:::pkg
      PkgEditor["editor (Core engine)\nPublished package"]:::pkg
      PkgSync["sync (Sync hooks)\nPublished package"]:::pkg
      PkgSyncCore["sync-core (Protocol + adapters)\nPublished package"]:::pkg
    end

    subgraph "Deployed Web Properties"
      direction TB
      DotcomClient["tldraw.com Web App\n(Vite + React SPA)"]:::client
      DocsApp["tldraw.dev Docs\n(Next.js)"]:::client
      AnalyticsApp["Analytics Widget/App\n(Vite)"]:::client
    end

    subgraph "Edge Backends (Cloudflare)"
      direction TB
      DotcomSyncWorker{{"Dotcom Sync API\n(Worker + Durable Objects)"}}:::worker
      AssetUploadWorker{{"Asset Upload Worker"}}:::worker
      UserContentWorker{{"User Content Serving Worker"}}:::worker
      ImageResizeWorker{{"Image Resize/Transform Worker"}}:::worker
      AnalyticsWorker{{"Analytics Worker"}}:::worker
      BemoWorker{{"Bemo Worker\n(Worker + DO)"}}:::worker
    end

    subgraph "Data Plane"
      direction TB
      Postgres[("Postgres\n(dotcom/zero-cache)")]:::db
      R2[("Cloudflare R2\n(Asset blobs)")]:::object
      DOStorage[("Durable Object Storage / SQLite\n(state)")]:::db
    end

    subgraph "External Services"
      direction TB
      Supabase["Supabase\n(Auth/identity/data access)"]:::external
      Algolia["Algolia\n(Search indexing + query)"]:::external
      Sentry["Sentry\n(Errors + performance)"]:::external
      Vendors["Analytics Vendors\n(GA4/GTM/Hubspot/Posthog/...)"]:::external
    end

    EndUsers -->|"uses"| DotcomClient
    DocsUsers -->|"browses"| DocsApp
    Devs -->|"installs"| PkgTldraw

    DotcomClient -->|"HTTP/WS-sync"| DotcomSyncWorker
    DotcomClient -->|"HTTP-upload"| AssetUploadWorker
    DotcomClient -->|"HTTP-assets"| UserContentWorker
    DotcomClient -->|"HTTP-images"| ImageResizeWorker

    DotcomSyncWorker -->|"reads/writes"| Postgres
    DotcomSyncWorker -->|"stores/reads blobs"| R2
    DotcomSyncWorker -->|"state"| DOStorage
    DotcomSyncWorker -->|"auth/identity"| Supabase

    DocsApp -->|"index/search"| Algolia

    DotcomClient -->|"errors"| Sentry
    DotcomSyncWorker -->|"errors"| Sentry

    AnalyticsApp -->|"collect/events"| AnalyticsWorker
    AnalyticsWorker -->|"forwards"| Vendors

    BemoWorker -->|"state"| DOStorage

    DotcomClient -.->|"imports"| PkgTldraw
    DocsApp -.->|"imports"| PkgTldraw
    AnalyticsApp -.->|"imports"| PkgTldraw
  end

  %% ==========================================================
  %% Diagram 2 — Container diagram for tldraw.com (production)
  %% ==========================================================
  subgraph "Diagram 2 — tldraw.com Container Diagram (Runtime)"
    direction TB

    subgraph "Browser"
      direction TB
      DotcomSPA["apps/dotcom/client\nVite + React SPA"]:::client
      DotcomSentryCfg["Sentry client config"]:::tooling
    end

    subgraph "Cloudflare Workers (Dotcom)"
      direction TB

      SyncWorker{{"apps/dotcom/sync-worker\nSync API + Durable Objects"}}:::worker
      SyncRoutes["Routes\n(src/routes/*)"]:::worker
      SyncTlaRoutes["TLA Routes\n(src/routes/tla/*)"]:::worker

      subgraph "Durable Objects (Stateful compute)"
        direction TB
        DO_Draw[("TLDrawDurableObject")]:::durable
        DO_File[("TLFileDurableObject")]:::durable
        DO_User[("TLUserDurableObject")]:::durable
        DO_Stats[("TLStatsDurableObject")]:::durable
        DO_Logger[("TLLoggerDurableObject")]:::durable
      end

      subgraph "Sync-worker internals"
        direction TB
        PgAdapter["Postgres integration\n(postgres.ts)"]:::worker
        R2Adapter["R2 integration\n(r2.ts)"]:::worker
        SupabaseClient["Supabase client\n(createSupabaseClient.ts)"]:::worker
        ReplicatorDir["Replication pipeline\n(src/replicator/*)"]:::worker
        PgReplicator["TLPostgresReplicator.ts"]:::worker
        ZeroIntegration["Zero integration\n(src/zero/*)"]:::worker
        AlarmScheduler["Alarm/cron scheduler\n(AlarmScheduler.ts)"]:::worker
        RateLimit["Rate limit utils\n(rateLimit.ts)"]:::worker
        Throttle["Throttle utils\n(throttle.ts)"]:::worker
        FeatureFlags["Feature flags\n(featureFlags.ts)"]:::worker
      end

      AssetWorker{{"apps/dotcom/asset-upload-worker\nAsset upload signing/validation"}}:::worker
      UsercontentWorker{{"apps/dotcom/tldrawusercontent-worker\nServe user content"}}:::worker
      ResizeWorker{{"apps/dotcom/image-resize-worker\nEdge image transforms"}}:::worker
    end

    subgraph "Data Stores"
      direction TB
      ZeroCache[("apps/dotcom/zero-cache\nPostgres + migrations + pgbouncer")]:::db
      CF_R2[("Cloudflare R2\nAsset blobs")]:::object
      DO_DB[("Durable Object Storage / SQLite\nState")]:::db
    end

    subgraph "External Dependencies"
      direction TB
      ExtSupabase["Supabase\n(Auth/identity)"]:::external
      ExtSentry["Sentry"]:::external
    end

    DotcomSPA -->|"HTTP/WS-like realtime"| SyncWorker
    DotcomSPA -->|"HTTP-request policy/signature"| AssetWorker
    DotcomSPA -->|"HTTP-fetch assets"| UsercontentWorker
    DotcomSPA -->|"HTTP-resize requests"| ResizeWorker

    %% Sync API surface
    SyncWorker -->|"dispatches"| SyncRoutes
    SyncWorker -->|"dispatches"| SyncTlaRoutes

    %% Durable objects
    SyncWorker <-->|"realtime rooms/files/users"| DO_Draw
    SyncWorker <-->|"file sessions"| DO_File
    SyncWorker <-->|"user sessions"| DO_User
    SyncWorker <-->|"stats"| DO_Stats
    SyncWorker <-->|"logging"| DO_Logger

    %% Data plane
    SyncWorker -->|"read/write"| PgAdapter -->|"SQL"| ZeroCache
    SyncWorker -->|"blob IO"| R2Adapter -->|"S3 API"| CF_R2
    DO_Draw -->|"state"| DO_DB
    DO_File -->|"state"| DO_DB
    DO_User -->|"state"| DO_DB
    DO_Stats -->|"state"| DO_DB
    DO_Logger -->|"state"| DO_DB

    %% Replication/caching pattern
    SyncWorker -.->|"async replication"| ReplicatorDir
    ReplicatorDir -->|"uses"| PgReplicator -->|"writes/reads"| ZeroCache
    SyncWorker -->|"uses"| ZeroIntegration -->|"backing DB"| ZeroCache

    %% Control-plane utilities
    SyncWorker -->|"schedules"| AlarmScheduler
    SyncWorker -->|"limits"| RateLimit
    SyncWorker -->|"limits"| Throttle
    SyncWorker -->|"toggles"| FeatureFlags

    %% Auth/identity + observability
    DotcomSPA -->|"auth"| ExtSupabase
    SyncWorker -->|"auth/identity"| SupabaseClient -->|"API"| ExtSupabase
    DotcomSPA -->|"error reporting"| ExtSentry
    SyncWorker -->|"error reporting"| ExtSentry
    DotcomSentryCfg -.->|"configures"| ExtSentry

    %% Asset flow details
    AssetWorker -->|"store blobs"| CF_R2
    UsercontentWorker -->|"read blobs"| CF_R2
    ResizeWorker -->|"fetch source"| CF_R2
    ResizeWorker -->|"serve transformed"| DotcomSPA
  end

  %% ==========================================================
  %% Diagram 3 — SDK Module/Package diagram (monorepo internals)
  %% ==========================================================
  subgraph "Diagram 3 — SDK Packages (Layered Architecture)"
    direction TB

    subgraph "Apps import SDK (build-time)"
      direction TB
      AppDotcom["apps/dotcom/client"]:::client
      AppDocs["apps/docs"]:::client
      AppExamples["apps/examples"]:::client
      AppAnalytics["apps/analytics"]:::client
    end

    subgraph "Public API (higher-level packages)"
      direction TB
      TldrawPkg["packages/tldraw\nPublic React SDK"]:::pkg
      EditorPkg["packages/editor\nCore editor engine"]:::pkg
      SyncPkg["packages/sync\nSync hooks/client API"]:::pkg
      SyncCorePkg["packages/sync-core\nProtocol + adapters"]:::pkg
      CreateTldrawPkg["packages/create-tldraw\nCLI scaffolder"]:::pkg
    end

    subgraph "Foundation (lower-level packages)"
      direction TB
      TLSchemaPkg["packages/tlschema\nSchema + migrations"]:::pkg
      StorePkg["packages/store\nRecord store"]:::pkg
      StatePkg["packages/state\nReactive primitives"]:::pkg
      StateReactPkg["packages/state-react\nReact bindings"]:::pkg
      UtilsPkg["packages/utils\nShared utilities"]:::pkg
      ValidatePkg["packages/validate\nValidation utilities"]:::pkg
      AssetsPkg["packages/assets\nStatic assets + helpers"]:::pkg
      WorkerSharedPkg["packages/worker-shared\nWorker utilities (incl. Sentry)"]:::pkg
      WorkerSharedSentry["worker-shared sentry helpers\n(src/sentry.ts)"]:::pkg
    end

    %% Apps -> SDK
    AppDotcom -.->|"imports"| TldrawPkg
    AppDocs -.->|"imports"| TldrawPkg
    AppExamples -.->|"imports"| TldrawPkg
    AppAnalytics -.->|"imports"| TldrawPkg

    %% Layering: tldraw at top
    TldrawPkg -.->|"depends-on"| EditorPkg
    TldrawPkg -.->|"depends-on"| TLSchemaPkg
    TldrawPkg -.->|"depends-on"| StorePkg
    TldrawPkg -.->|"depends-on"| StatePkg
    TldrawPkg -.->|"depends-on"| StateReactPkg
    TldrawPkg -.->|"depends-on"| UtilsPkg
    TldrawPkg -.->|"uses"| AssetsPkg
    TldrawPkg -.->|"optional sync"| SyncPkg

    %% editor depends on foundations
    EditorPkg -.->|"depends-on"| TLSchemaPkg
    EditorPkg -.->|"depends-on"| StorePkg
    EditorPkg -.->|"depends-on"| StatePkg
    EditorPkg -.->|"depends-on"| UtilsPkg
    EditorPkg -.->|"depends-on"| ValidatePkg

    %% sync layering
    SyncPkg -.->|"depends-on"| SyncCorePkg
    SyncPkg -.->|"depends-on"| StorePkg
    SyncPkg -.->|"depends-on"| StatePkg
    SyncPkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"depends-on"| TLSchemaPkg
    SyncCorePkg -.->|"depends-on"| StorePkg
    SyncCorePkg -.->|"depends-on"| UtilsPkg
    SyncCorePkg -.->|"worker helpers"| WorkerSharedPkg
    WorkerSharedPkg -.->|"includes"| WorkerSharedSentry
  end

  %% ==========================================================
  %% Diagram 4 — CI/CD + release automation (build/deploy)
  %% ==========================================================
  subgraph "Diagram 4 — CI/CD + Release"
    direction TB

    subgraph "Repo Tooling"
      direction TB
      GHWorkflows[".github/workflows\nGitHub Actions pipelines"]:::tooling
      InternalScripts["internal/scripts\nBuild/publish/deploy scripts"]:::tooling
      VercelScripts["internal/scripts/vercel\nVercel helpers"]:::tooling
      DeployDotcom["deploy-dotcom.ts"]:::tooling
      DeployAnalytics["deploy-analytics.ts"]:::tooling
      DeployDocsWorkflow["deploy-docs-on-release-edit.yml"]:::tooling
      DeployAnalyticsWorkflow["deploy-analytics.yml"]:::tooling
      LernaCfg["lerna.json"]:::tooling
      Yarnrc["yarnrc.yml"]:::tooling
      RootPkgJson["package.json"]:::tooling
      SSTCfg["sst.config.ts\nDeployment orchestration"]:::tooling
    end

    subgraph "Deploy Targets"
      direction TB
      NPM["npm registry\n(Published packages)"]:::external
      Cloudflare["Cloudflare\n(Workers/DO/R2)"]:::external
      Vercel["Vercel\n(Next.js deployments)"]:::external
    end

    GHWorkflows -->|"runs"| InternalScripts
    GHWorkflows -->|"uses"| LernaCfg
    GHWorkflows -->|"uses"| Yarnrc
    GHWorkflows -->|"uses"| RootPkgJson
    GHWorkflows -->|"orchestrates"| SSTCfg

    InternalScripts -->|"deploy"| DeployDotcom -->|"targets"| Cloudflare
    InternalScripts -->|"deploy"| DeployAnalytics -->|"targets"| Cloudflare
    GHWorkflows -->|"docs release"| DeployDocsWorkflow
    GHWorkflows -->|"analytics deploy"| DeployAnalyticsWorkflow

    VercelScripts -->|"build/deploy docs"| Vercel
    GHWorkflows -->|"publish packages"| NPM
  end

  %% ==========================================================
  %% Diagram 5 — Templates ecosystem (create-tldraw + templates)
  %% ==========================================================
  subgraph "Diagram 5 — Templates & Starters"
    direction TB

    TemplatesDir["templates/*\nStarter kits / reference architectures"]:::tooling
    SyncCloudflareTpl["sync-cloudflare template\n(worker + client)"]:::tooling
    SimpleServerTpl["simple-server-example template\n(server)"]:::tooling
    SocketIOTpl["socketio-server-example template\n(server)"]:::tooling

    CreateTldrawCli["create-tldraw CLI\n(scaffold projects)"]:::tooling

    CreateTldrawCli -->|"selects/installs"| TemplatesDir
    TemplatesDir -->|"includes"| SyncCloudflareTpl
    TemplatesDir -->|"includes"| SimpleServerTpl
    TemplatesDir -->|"includes"| SocketIOTpl
  end

  %% =========================
  %% Click events (paths only)
  %% =========================

  %% Packages
  click TldrawPkg "packages/tldraw"
  click PkgTldraw "packages/tldraw"
  click EditorPkg "packages/editor"
  click PkgEditor "packages/editor"
  click TLSchemaPkg "packages/tlschema"
  click