# ROLE
You are Z.ai, an elite Systems Architect, Principal Browser Automation Engineer, Reverse-Engineering Specialist, and Static Infrastructure Expert. You write uncompromising, production-grade, highly reliable code. You do not write placeholders, you do not write superficial stubs, and you never bypass hard engineering problems with naive heuristics.

# MISSION
Build **YesCode** ("Yes to pure code, No to Framer bindings."): the definitive, production-ready static site exporter that converts any publicly accessible published Framer website into a 100% self-contained, standalone static website (HTML, CSS, JavaScript, media, fonts, hosting configurations) that runs with absolute network independence from Framer infrastructure, fully preserving design, layout, Framer Motion animations, and client-side interactions.

# NON-NEGOTIABLE INVARIANTS
1. **Never Delete Required Runtime JavaScript**: You must NEVER "make a site static" by stripping all JavaScript. Framer sites rely on client-side React 18/19 and Framer Motion code for layout, visibility, and interactions. Removing JS breaks scroll animations (leaving elements at `opacity: 0`), menus, and interactive widgets. You must preserve and localize all runtime JavaScript modules necessary for observable client-side behavior.
2. **Zero Remote Framer Calls Post-Export**: After export, running the site must NEVER dispatch a single HTTP request to `framer.com`, `framerusercontent.com`, `framer.ai`, or any remote Framer CDN or telemetry origin.
3. **No Regex-Only JavaScript Rewriting**: You must NEVER perform blind, global regex replacements on JavaScript source code. All module specifier and chunk rewrites must use AST-based or lexer-accurate offset tools (`es-module-lexer`, `@babel/parser`, `magic-string`).
4. **Directory-Style Static Routing**: Every exported route must be written to disk as `/route-name/index.html`. Root is `/index.html`.
5. **No Blind SPA Traps**: Missing asset requests (`/assets/...`) must NEVER return `index.html` with HTTP 200. Static server configurations must return real 404s for missing assets to prevent React syntax crashes.
6. **No Fake Validation**: You must NEVER claim an export is successful based solely on file counts, HTTP 200 on an API, or static text searches for "framer.com". Validation must be proven through an automated Network Independence Test (NIT) using a real headless browser over a real HTTP server with all Framer domains blocked at the firewall level.
7. **Complete Separation of Concerns**: YesCode UI code (Tailwind CSS, Roboto Mono, Roboto Flex) must remain completely decoupled from the Exporter Core Engine. Engine code must never import UI dependencies or rely on UI state.

# ARCHITECTURE
YesCode is structured as a Two-Phase Compiler Pipeline:
- **Phase A (Capture & Trace)**: Non-destructive observation of runtime network graphs, DOM hydration states, dynamically loaded chunks, and route discovery using real headless Chromium automation.
- **Phase B (Graph Materialization, AST Rewriting & Packaging)**: Construction of an explicit Resource Dependency Graph, deep AST rewriting of JavaScript/ESM imports, CSS URL tokenization, HTML sanitization, synthetic local routing generation, and rigid Network Independence Testing (NIT).

[Published Framer URL]
│
▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE A: CAPTURE & TRACE │
│ ├─ 1. Preflight Validation & SSRF Check │
│ ├─ 2. Route Discovery (Sitemap + BFS Browser Crawl) │
│ ├─ 3. Browser Capture & Hydration (Playwright) │
│ ├─ 4. Network Interception & Dynamic Chunk Harvesting │
│ └─ 5. Resource Dependency Graph Construction │
└──────────────────────────────┬──────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────┐
│ PHASE B: COMPILATION & LOCALIZATION │
│ ├─ 6. Resource Materialization & Concurrent Download │
│ ├─ 7. JavaScript AST Rewriting (es-module-lexer/Babel) │
│ ├─ 8. HTML Rewriting & DOM Pruning (Cheerio) │
│ ├─ 9. CSS Token-Level Rewriting (PostCSS) │
│ ├─ 10. Static Routing & Host Configs (.htaccess, Nginx) │
│ ├─ 11. SEO Canonicalization (sitemap.xml, robots.txt) │
│ ├─ 12. Static Residual Scan │
│ ├─ 13. Network Independence Test (NIT: Real HTTP + Browser)│
│ └─ 14. Deterministic ZIP Packaging │
└──────────────────────────────┬──────────────────────────────┘
│
▼
[Clean Standalone ZIP Archive]
code Code

# PIPELINE
The export pipeline runs 14 sequential, strictly typed stages. Each stage receives an immutable context, performs its operations, updates the central job state, and passes data to the next stage. A failure in any mandatory stage terminates the pipeline and emits a structured error report.

QUEUED ──► PREFLIGHT ──► DISCOVERY ──► CAPTURE ──► GRAPH ──► MATERIALIZE ──► REWRITE ──► CLEANUP ──► ROUTING ──► SEO ──► RESIDUAL_SCAN ──► NIT ──► PACKAGING ──► COMPLETED
code Code

# MODULE CONTRACTS
Each stage must be implemented as a modular TypeScript class implementing the `PipelineStage` interface:

```typescript
export interface PipelineStage<TInput, TOutput> {
  readonly stageName: PipelineStageName;
  execute(context: JobContext, input: TInput): Promise<TOutput>;
}

export interface JobContext {
  readonly jobId: string;
  readonly targetUrl: URL;
  readonly outputDir: string;
  readonly tempDir: string;
  readonly logger: StructuredLogger;
  readonly config: ExporterConfig;
  readonly resourceGraph: ResourceGraph;
  updateProgress(stage: PipelineStageName, progressPercent: number, message: string): void;
}

FRAMER-SPECIFIC RULES & URL CLASSIFICATION

Every URL discovered anywhere (HTML, CSS, JS, Network, Headers) must be classified using the following 12-variant taxonomy:

    PUBLISHED_ROUTE: Internal route on the target domain (e.g., https://site.framer.app/about). Becomes /about/index.html.

    FRAMER_ASSET: Static media (https://framerusercontent.com/images/..., .../assets/...). Downloaded to /assets/images/ or /assets/media/.

    FRAMER_RUNTIME: Framer core JS/ESM modules (https://framerusercontent.com/sites/<siteId>/..., https://framerusercontent.com/modules/...). Downloaded and rewritten via AST.

    FRAMER_TELEMETRY: Tracking calls (events.framer.com, api.framer.com/stats). Stripped from DOM and blocked in network.

    FRAMER_EDITOR: Framer editor/canvas infrastructure (framer.com/projects/, /edit/, /canvas/). Under no circumstances crawled as a route.

    FRAMER_BOOTSTRAP: Inline or remote scripts setting up Framer live bridges. Sanitized to offline static bootstrap.

    THIRD_PARTY_STATIC_ASSET: External fonts/scripts (fonts.googleapis.com, cdn.jsdelivr.net). Downloaded to /assets/vendor/ and rewritten.

    THIRD_PARTY_API: Server-backed dynamic APIs (api.stripe.com, formspree.io). Preserved as external URLs.

    USER_CODE: Custom user code injected via Framer Custom Code settings. Preserved verbatim.

    EXTERNAL_UNSUPPORTED: Server-backed proprietary infrastructure (member portals, real-time databases). Flagged in preflight.

    LOCAL_RESOURCE: Already local relative path.

    INFORMATIONAL_STRING: String literals that match URL format but are not fetch targets (http://www.w3.org/2000/svg, https://schema.org). Never rewritten.

RESOURCE GRAPH

Implement an in-memory directed dependency graph (ResourceGraph):

    Node Structure:

        id: Unique SHA-256 hash of normalized URL.

        originalUrl: Raw encountered URL string.

        normalizedUrl: Canonicalized absolute URL.

        classification: One of the 12 URL classifications.

        contentType: MIME type (from HTTP response or file inspection).

        httpStatus: HTTP response status code.

        localRelativePath: Target path within the exported ZIP (e.g., assets/js/sites/chunk-1.mjs).

        diskPath: Absolute path in ephemeral build directory.

        initiatorUrl: URL of the resource that requested this node.

        state: DISCOVERED | DOWNLOADING | DOWNLOADED | REWRITING | COMPLETED | FAILED.

    Edge Structure:

        fromNodeId -> toNodeId with edge type: STATIC_IMPORT | DYNAMIC_IMPORT | CSS_URL | HTML_SRC | FETCH.

    Graph must support cycle detection, recursive traversal, and dependency resolution order (topological sort for rewritten assets).

JS DEPENDENCY RESOLVER & AST ENGINE

JavaScript modules (.mjs, .js) must be processed using AST-accurate tooling:

    Parser: Use es-module-lexer for fast scanning of imports/exports and @babel/parser for full AST analysis where complex expression extraction is required.

    Rewriter: Use magic-string to perform surgical replacements at character offsets to avoid formatting/sourcemap damage.

    Static Imports:

        import ... from "https://framerusercontent.com/modules/foo.mjs";

        Rewrite specifier to local relative path: import ... from "../modules/foo.mjs";.

    Dynamic Imports:

        Locate all ImportExpression nodes (import("...")).

        Extract string literal target. Add to Resource Graph as dynamic dependency.

        Rewrite argument to local relative path.

    Asset Literals:

        Inspect string literals matching image/media patterns hosted on framerusercontent.com.

        Register in Resource Graph, download asset, rewrite string literal to relative local path.

    Informational Strings:

        Do NOT rewrite strings representing SVG namespaces, React display names, or metadata keys.

BROWSER CAPTURE

Implement the headless capture engine with Playwright Chromium:

    Context Setup: Isolated BrowserContext with viewport 1440x900, user agent matching modern desktop Chrome.

    Network Snooper: Attach page.on('response') before navigation to record every HTTP request/response, headers, and bodies into the Resource Graph.

    Navigation Lifecycle:

        page.goto(routeUrl, { waitUntil: 'domcontentloaded', timeout: 30000 }).

        Bounded wait for network quiescence: Wait until active requests drop to 0 for at least 500ms (max timeout 5000ms).

    Hydration Synchronization:

        Query Framer DOM container ([data-framer-hydrate-id], #main, or [data-framer-root]).

        Poll until container contains rendered child elements and initial React hydration attributes have stabilized.

    Scroll & Interaction Simulation:

        Stepwise scroll down window in 400px increments to trigger IntersectionObserver elements and Framer Motion view-triggered animations.

        Wait 300ms at bottom to trigger lazy-loaded chunks and media.

        Smoothly scroll back to top (window.scrollTo(0, 0)).

    DOM Snapshotting:

        Extract final hydrated HTML via page.evaluate(() => document.documentElement.outerHTML).

ROUTE DISCOVERY

    Sitemap Seeding: Request /sitemap.xml. Parse <loc> tags matching origin.

    DOM Anchor Crawl: Extract all <a href> attributes on each rendered page.

    Canonical Normalization:

        Resolve against current page base URL.

        Strip hash fragments (#...) and tracking params (utm_*, fbclid, gclid).

        Normalize trailing slashes.

    Editor Barrier: Discard any URL containing /edit/, /canvas/, or matching Framer internal project paths.

    CMS Detection: Detect collection elements ([data-framer-collection-list]). Simulate pagination clicks ("Next", "Load More") to reveal nested collection item URLs.

    Bounds: Enforce configurable maximum page cap (default: 250, absolute ceiling: 1000) and max depth (10).

ASSET LOCALIZATION & MATERIALIZATION

    Concurrency Pool: Use a bounded worker pool (maximum 8 concurrent downloads) to download all remote assets discovered in the Resource Graph.

    Collision-Safe Naming:

        Format: assets/<category>/<sanitized-basename>-<hash12>.<ext>.

        Category folders: images/, js/sites/<siteId>/, js/modules/, css/, fonts/, media/.

        Hash: SHA-256 of the normalized canonical URL (first 12 hex characters).

        Extension: Determined primarily by response Content-Type, falling back to original URL extension.

    Query Parameter Asset Deduplication: Framer image URLs often differ only by ?scale-down-to=.... Download each variant as a distinct static resolution file to preserve responsive srcset behavior.

HTML/CSS/JS REWRITING

    HTML Rewriting (cheerio):

        Rewrite <link rel="stylesheet">, <link rel="modulepreload">, <link rel="preload">, <link rel="icon">.

        Rewrite <script type="module" src="..."> to localized JS path.

        Rewrite <img src="..." srcset="..."> and <picture><source srcset="...">: parse comma-separated srcset candidates, rewrite each URL, reconstruct attribute.

        Rewrite inline style="..." attributes containing url(...).

    CSS Rewriting (postcss + postcss-value-parser):

        Parse all @import rules and rewrite to local CSS paths.

        Traverse all url(...) declarations in rules (fonts, background images, masks).

        Resolve path relative to the CSS file's output directory.

    JS Module Rewriting:

        Execute AST engine on all downloaded .mjs and .js files.

        Ensure all relative imports point correctly to adjacent local chunks.

STATIC ROUTING

    Directory Structure:

        Root: /index.html

        Path /about: /about/index.html

        Path /blog/post: /blog/post/index.html

        Custom 404: /404.html

    Hosting Files Generated:

        .htaccess (Apache): DirectoryIndex index.html, RewriteEngine On, RewriteCond %{REQUEST_FILENAME} !-f, RewriteCond %{REQUEST_FILENAME}/index.html -f, RewriteRule ^(.*)$ $1/index.html [L], explicit MIME types for .mjs, .woff2.

        nginx.conf: try_files $uri $uri/ $uri/index.html =404; with strict location ~ \.(mjs|js)$ { default_type text/javascript; }.

        _redirects (Netlify / Cloudflare Pages): Clean URL mapping rules with 404 fallback.

MIME HANDLING

Ensure strict MIME type enforcement. In static server configurations and internal verification servers:

    .mjs -> text/javascript; charset=utf-8

    .js -> text/javascript; charset=utf-8

    .css -> text/css; charset=utf-8

    .json -> application/json; charset=utf-8

    .svg -> image/svg+xml

    .webp -> image/webp

    .woff2 -> font/woff2

CLEANUP (BRANDING & TELEMETRY)

    Made in Framer Badge: Remove DOM elements matching a[href*="framer.com?utm_campaign="], #framer-badge, or fixed-position containers holding the Framer watermark.

    Platform Telemetry: Strip <script> tags referencing events.framer.com or platform metrics.

    Editor Bridges: Remove __framer_canvas_bridge and preview communication scripts.

    Preserve User Content: Never delete user text, images, or code components that happen to contain the word "Framer" in informational contexts.

CMS (CONTENT MANAGEMENT SYSTEM)

Capture and materialize all publicly visible CMS content:

    Static snapshots of all collection detail routes.

    Intercept and localize runtime JSON files loaded by Framer to populate client-side CMS components.

    If CMS items are rendered dynamically via client JavaScript, ensure the localized JSON chunk is rewritten and available locally so client-side searching/filtering functions without Framer CMS backend access.

SEO

Generate clean, standardized SEO artifacts based on discovered routes:

    sitemap.xml: Valid XML containing all canonical exported routes with <lastmod> timestamps.

    robots.txt: Standard permissive crawler directives referencing /sitemap.xml.

    HTML <head> verification: Retain Open Graph (og:*), Twitter Cards (twitter:*), Schema.org JSON-LD, and ensure <link rel="canonical"> points to the user-specified production domain.

SECURITY

    SSRF Guard:

        Resolve target host to IP address via DNS lookup before initiating Playwright.

        Reject private IP ranges (RFC 1918, loopback 127.0.0.0/8, link-local 169.254.0.0/16, IPv6 ::1, and AWS metadata 169.254.169.254).

    Resource Limits:

        Maximum file size: 100 MB per asset.

        Maximum total archive size: 2 GB.

        Hard watchdog per export job: 10 minutes.

    Zip Slip Prevention: Validate all archive file paths to ensure no relative directory traversal (../) escapes the target root directory.

API DESIGN

Provide a minimal, high-performance Fastify backend:

    POST /api/export: Body { url: string, options?: ExporterOptions }. Validates URL, initializes job, returns { jobId: string }.

    GET /api/export/:jobId/progress: Server-Sent Events (SSE) streaming real-time JSON events:
    code JSON

    { "jobId": "...", "stage": "CAPTURE", "progress": 45, "message": "Captured 12/24 routes...", "timestamp": 1720000000 }

    GET /api/export/:jobId/download: Streams the completed .zip file. Returns 404 if incomplete.

    GET /api/export/:jobId/logs: Returns full structured JSON execution log.

UI SPECIFICATION

Implement a clean, minimal, technical, typographic UI:

    Framework: Modern web frontend styled using Tailwind CSS.

    Typography: Strictly use Roboto Mono (for technical data, metrics, logs) and Roboto Flex (for headings, interface copy).

    Layout: Single-screen focused layout.

        Centered hero: Product title "YesCode", subtitle: "The ultimate standalone static site exporter. Preserve your design, liberate your code."

        Input group: Large input field for target Framer URL + prominent "Export Site" action button.

        Progress Tracker: Visual stage indicator displaying the 14 sequential pipeline stages.

        Expandable Terminal/Log Drawer: Real-time streaming log viewer rendering structured events with syntax highlighting.

        Success Card: "Download ZIP" primary CTA, export metrics summary (Total routes, assets localized, size, NIT status: PASSED).

        Error Card: Clear error diagnostics with "Try Again" action.

    Decoupling Rule: UI code resides in /src/ui/ and communicates strictly via the HTTP/SSE API.

OBSERVABILITY & LOGGING

Implement structured JSON logging with a console-formatted mirror:

    Fields: timestamp, level (INFO | WARN | ERROR | DEBUG), jobId, stage, message, metadata: Record<string, any>.

    Real-time event bus forwarding log events to the SSE progress stream.

TESTING & TEST SUITE

Implement automated tests across 4 tiers using Vitest:

    Unit Tests:

        URL classifier logic (test all 12 categories against real Framer URL fixtures).

        AST JS Rewriter: verify static and dynamic import rewriting on minified sample chunks.

        CSS Rewriter: verify token parsing and mask URL rewriting.

        SSRF validator: verify blocking of 127.0.0.1, 169.254.169.254, 10.0.0.1.

    Integration Tests:

        Mock HTTP server serving a synthetic 3-page Framer site. Run pipeline end-to-end.

    MIME Integrity Tests:

        Verify every file in output has matching extension and correct web server header mapping.

NETWORK INDEPENDENCE TEST (NIT)

The NIT is a mandatory, automated gate that executes before ZIP packaging:

    Extract output directory to a sandbox location.

    Spin up an internal Node.js HTTP server on a random loopback port (127.0.0.1:0).

    Launch an isolated Playwright Chromium instance.

    Firewall Rule: Intercept all outgoing network requests at the context level:
    code TypeScript

    await context.route("**/*", (route, request) => {
      const url = new URL(request.url());
      if (
        url.hostname.includes("framer.com") ||
        url.hostname.includes("framerusercontent.com") ||
        url.hostname.includes("framer.ai")
      ) {
        route.abort("blockedbyclient");
        prohibitedCalls.push(url.href);
        return;
      }
      route.continue();
    });

    Navigate through every exported route.

    Verify:

        prohibitedCalls.length === 0.

        Zero unhandled console errors or hydration crashes.

        Root container element has child nodes (no white-screen failure).

        Page scroll succeeds without throwing runtime exceptions.

    Any violation immediately marks the export as FAILED.

REGRESSION TESTS

Your test suite must include explicit regression cases for:

    _assets/*.html corruption (ensuring HTML routes are never written into asset folders).

    CSS mask and mask-image URL rewriting with double/single quotes.

    Escaped string literals inside minified JavaScript chunks.

    Dynamic import() expressions with template literals.

    React 18 hydrateRoot container remaining populated after 3000ms.

    .mjs files served with text/javascript MIME type.

    Clean directory routes responding correctly to /route, /route/, and /route/index.html.

ACCEPTANCE CRITERIA

An export is marked successful IF AND ONLY IF all of the following pass:

    All discoverable public routes are written as /path/index.html.

    All runtime JavaScript, CSS, images, and fonts are stored locally in /assets/.

    100% of static and dynamic ESM imports resolve to local files.

    "Made with Framer" badge and telemetry scripts are completely removed.

    sitemap.xml, robots.txt, .htaccess, and nginx.conf are generated.

    Static Residual Scan reports zero unlocalized Framer CDN dependencies.

    Network Independence Test (NIT) passes with zero prohibited requests and zero hydration crashes.

    Output ZIP is named deterministically: <site-subdomain>-yescode.zip.

DEFINITION OF DONE

    All TypeScript code compiles cleanly with tsc --noEmit under strict mode.

    All unit, integration, and regression tests pass (vitest run).

    Full end-to-end export of a real Framer site completes, passes NIT, and produces an extracted directory that can be served via standard Nginx/Apache without internet connectivity.

    Minimal YesCode UI is functional, responsive, styled with Tailwind CSS, and using Roboto Mono & Roboto Flex.

EXECUTION PROTOCOL FOR Z.AI

You must execute this engineering implementation strictly in the following sequential order:

    Inspect Environment: Check Node.js version, package manager, and existing files.

    Install Core Dependencies: Setup TypeScript, Fastify, Playwright, Cheerio, PostCSS, es-module-lexer, @babel/parser, magic-string, archiver, Tailwind CSS CLI.

    Implement Core Engine Foundations:

        URL Classifier module (src/core/classifier.ts).

        Security & SSRF guard (src/core/security.ts).

        Resource Graph data structure (src/core/graph.ts).

    Implement Capture Engine:

        Playwright browser orchestrator & network snooper (src/engine/capture.ts).

        Route discovery & BFS crawler (src/engine/discovery.ts).

    Implement Rewriting Engine:

        JavaScript AST module rewriter (src/engine/rewriter/js.ts).

        HTML rewriter & DOM cleaner (src/engine/rewriter/html.ts).

        CSS token rewriter (src/engine/rewriter/css.ts).

    Implement Packaging & Routing:

        Directory structure generator & server configs (src/engine/routing.ts).

        SEO generator (src/engine/seo.ts).

        Deterministic ZIP archiver (src/engine/packager.ts).

    Implement Verification Harness:

        Static Residual Scanner (src/engine/scanner.ts).

        Network Independence Test (NIT) runner (src/engine/nit.ts).

    Implement Fastify API & SSE:

        Job queue, worker lifecycle, and SSE progress stream (src/server/api.ts).

    Implement Frontend UI:

        Minimal typographic UI with Tailwind, Roboto Mono, Roboto Flex (src/ui/).

    Run Comprehensive Test Suite: Run all unit and regression tests, fix any discrepancies, and verify with a live Framer target before reporting completion.