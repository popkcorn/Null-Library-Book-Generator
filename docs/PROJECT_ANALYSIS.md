# Null Library — Complete Project Analysis

> Generated 2026-10-25 from a full source audit of this repository
> (branch `arena/5988e687-null-library-book-generator`, base commit `66e156b`).

---

## 1. What this project is

**Null Library** is a single-author, AI-powered "book factory": a web/desktop app that walks a
(non-technical) user through the entire self-publishing lifecycle for Amazon KDP — market
research → outline → chapter drafting → illustration → cover design → marketing copy → EPUB
export → (optionally) automated upload to KDP via a Playwright bot.

It goes by at least five names inside its own codebase, which tells you a lot about its history:

| Name | Where it appears |
|---|---|
| **Null Library** | Repo name, `README.md` branding ("The All-In-One Publishing Cartel", by "Blacklisted Binary Labs") |
| **FraudRob's AI Book Factory** | `index.html` `<title>`, `metadata.json`, npm package name (`fraudrobs-ai-book-factory`), Tauri copyright |
| **KDP E-Book Generator** | Tauri `productName`, installer, most docs, upstream repo `crazyrob425/KDP-E-Book-Generator` |
| **kdp-ai-booksmith** | IndexedDB storage key (`kdp-ai-booksmith-v5-db`) |
| v2.1.0 | Version in `package.json` / `tauri.conf.json` |

**Scale:** ≈11,700 lines of TypeScript/TSX (components 5,238 · services 3,845 · `App.tsx` 1,312 ·
`types.ts` 386), 550 lines of Node backend, 647 lines of Rust (Tauri), 218 lines of Electron main
process, plus a 36 KB README, 5 planning docs, installer packaging, and CI/CD workflows. There are
**zero tests** anywhere in the repo.

---

## 2. Provenance — how this code came to be

The repository's git history is a **single commit** (`66e156b`, 2026-10-18, author "Rob",
141 files, +32,062 lines) titled *"Wire the 12-stage pipeline into the UI (PipelineConsole)"*.
There is no incremental history — the project was imported wholesale (it is a snapshot/fork of
`crazyrob425/KDP-E-Book-Generator`). My sandbox commit is the only other commit.

Forensic clues about its development process:

- **Born in Google AI Studio.** `metadata.json` is an AI Studio app manifest; `index.html` still
  carries an `<script type="importmap">` pointing `react`, `@google/genai`, `fabric`, `playwright`
  etc. at `aistudiocdn.com`; `geminiService.getAi()` falls back to `process.env.API_KEY` (how AI
  Studio injects keys); Tailwind and `file-saver` are loaded from CDN `<script>` tags instead of
  npm. Under Vite these CDN/importmap remnants are mostly inert cruft (Vite rewrites bare
  specifiers to bundled paths), but they're still shipped in the page.
- **Built with AI coding agents, plural.** The repo contains instruction files for several:
  `.claude/CLAUDE.md` (mandating a "vexp" MCP pipeline), `.github/copilot-instructions.md`,
  `.mcp.json` (an `orbitalmcp` server on localhost:3002), `.orbitalmcp/summaries/`,
  `.specify/` (spec-kit), `docs/COPILOT_WORKFLOW.md`, and a GitHub workflow literally named
  `ai-suggestion-autopilot.yml`. This explains both the feature breadth and the dead code
  (agents generated modules faster than a human wired them together).
- **Response style:** the README is written as over-the-top marketing copy ("Multi-AI Spoofing",
  "API Identity Theft", "publishing cartel"). Section 12 separates those claims from reality.

---

## 3. Architecture — one frontend, three runtimes

```
                        ┌─────────────────────────────────────────────┐
                        │  React 19 + TS frontend (App.tsx + steps)    │
                        │  Vite 6 · Tailwind (CDN) · IndexedDB state   │
                        └──────────────┬──────────────────────────────┘
                                       │ services/desktopBridge.ts
                ┌──────────────────────┼───────────────────────┐
                ▼                      ▼                       ▼
         Browser bridge        Electron bridge          Tauri bridge
         (downloads via        (window.electronAPI,      (invoke() → Rust
          <a download>,         contextBridge IPC)        commands in lib.rs)
          localStorage)               │                       │
                                      ▼                       ▼
                          electron/main.ts            src-tauri (Rust)
                          ├─ Playwright bot           ├─ native menu, prefs file,
                          │  (server/automation-      │  OS-keyring secrets
                          │   worker.ts, reused!)     └─ automation = STUB (Err)
                          └─ market scrapers
                                      ▲
         server/server.ts ────────────┘  same automation-worker, exposed over
         (Express + ws, port 8080)       WebSocket instead of IPC — but its
                                         client (kdpAutomationService) is unused
```

Key architectural facts:

1. **The browser app is the real product.** All AI calls happen client-side from the browser
   using the user's own API keys. There is no app server, no database, no auth. State lives in
   IndexedDB on the user's machine.
2. **`desktopBridge.ts`** is a clean runtime-abstraction: it detects Electron
   (`window.electronAPI`) vs Tauri (`__TAURI_INTERNALS__`) vs plain browser and exposes one
   interface (save/load file, window controls, market data, automation, preferences, secrets).
3. **The KDP automation worker is transport-agnostic**: `runAutomation(payload, sendUpdate)` is an
   async *generator* that yields once — when a CAPTCHA appears — so the same code is driven by
   Electron IPC (`electron/main.ts`) or by the WebSocket server (`server/server.ts`).
4. **Two parallel automation concepts for book production** coexist and overlap:
   the manual 5-step wizard in `App.tsx`, **Batch Mode** (a hand-rolled loop in `App.tsx`), and the
   newer **12-stage `PipelineRunner`** (the most recent commit). Batch Mode does *not* reuse
   PipelineRunner — the logic is duplicated a third time.

### Runtime modes

| Mode | Command | Status |
|---|---|---|
| Web (dev) | `npm run dev` → Vite :5173 | ✅ Works (running in this sandbox) |
| Web (prod) | `npm run build` → `dist/` | ✅ Built by CI (`frontend-build` job) |
| Tauri desktop | `npm run tauri:dev` / `tauri:build` | ⚠️ App shell + prefs real; automation/trends/competitors are Rust stubs returning empty/`Err` |
| Electron desktop | *(none)* | ❌ Broken: `electron` is **not in package.json**; only a stale prebuilt `dist-electron/main.js` exists. `vite.config.ts` tries to bundle it on build via an esbuild plugin |
| Standalone bot server | `npx tsx server/server.ts` (:8080) | ⚠️ Server code is complete, but `tsx` isn't a dependency, `server/` has no `package.json` (so the `cd server && npm run start` in `tauri:dev` fails), and the frontend WebSocket client is never imported |

---

## 4. Repository map

| Path | What it is |
|---|---|
| `App.tsx` (1,312) | The monolith: ~40 `useState`s, all workflow handlers (market → outline → chapters → images → cover → humanize → marketing → automation → batch → save/load), renders the wizard |
| `index.tsx` | React root mount (17 lines of logic) |
| `index.html` | Tailwind CDN + inline theme, Google Fonts, file-saver CDN, leftover AI-Studio importmap |
| `types.ts` (386) | Every shared type (see §6) + the `ElectronAPI` / `Window` global declarations |
| `components/` (5,238) | UI: 5 step screens, batch screens, modals, cover editor, pipeline console, shared primitives (see §8) |
| `services/` (3,845) | All business logic (see §7) |
| `hooks/useAutoSave.ts` | Debounced autosave hook feeding the `AutoSaveIndicator` |
| `server/` (550) | Optional Node backend: WS server + Playwright KDP bot + market scrapers + Dockerfile (Cloud Run target) |
| `electron/` (218) | Electron main + preload. Main hosts the same bot & scrapers over IPC |
| `src-tauri/` (Rust 647) | Tauri 2 shell: window, native menu, JSON prefs store, OS-keyring secret store, stub commands |
| `installer/windows/` | Custom NSIS installer script, splash/icon assets, EULA + AI-use-notice legal texts, PowerShell pre-install scripts |
| `scripts/` (258) | Branding asset generation (icons/NSIS bitmaps via sharp), updater `latest.json` scaffold generator |
| `release/updates/` | Tauri updater manifest template |
| `docs/` | 5 docs: `IMPLEMENTATION_PLAN`, `DEPLOYMENT_ROADMAP`, `WINDOWS11_INSTALLER_PREP_PLAN`, `KDP_SELECTOR_VERIFICATION`, `COPILOT_WORKFLOW` |
| `.github/workflows/` | `ci.yml` (vite build + server tsc + cargo check), `release.yml` (tag → Tauri bundles for Win/macOS/Linux), `windows-beta-release.yml`, `ai-suggestion-autopilot.yml` |
| Root clutter | Four large decorative PNGs (~25 MB total: `null library.png`, `nullbook.png`, `nullibrary.png`, `chart.png`) sitting in the repo root |

---

## 5. The user-facing workflow

### 5.1 Single-book mode — the 5-step wizard (`AppStep` enum)

1. **Market Research** (`MarketResearchStep`) — three entry paths: *Niche Finder* (AI suggests 5
   hot genres → 5 topics each), *custom topic*, or *AI-simulated report*. Produces a
   `MarketReport` (trend analysis, audience demographics/interests/pain-points, keywords,
   suggested book types, competitor analysis), optionally enriched with real Google Trends
   (desktop scraping) and Tavily web results.
2. **Outline** (`OutlineStep`) — pick book type, chapter count, pages-per-chapter; AI returns a
   `BookOutline` (title, subtitle, table of contents with per-chapter summaries). Fiction projects
   also get a **Book Bible**; non-fiction can first run a multi-stage "research" pass.
3. **Content** (`ContentGenerationStep`) — per-chapter editor: generate (optionally streamed),
   regenerate with free-text "director's instructions", literary critique + auto-rewrite
   ("AI Grand Master"), humanize, proofreading pane (readability/sentiment/retext checks),
   plus **Generate Full Manuscript** — chapters produced 4-at-a-time in parallel
   (content → image prompt → image per chapter).
4. **Illustration** (`IllustrationStep`) — per-chapter image prompts + style picker → Gemini-image
   illustrations.
5. **Review & Publish** (`ReviewStep`, 813 lines — the biggest screen) — cover generation +
   full **Fabric.js cover editor**, author profile, KDP marketing suite (descriptions,
   categories, keywords, back-cover blurb), **AI-content disclosure** questionnaire,
   then either *Download KDP Publishing Kit* (ZIP: `manuscript.epub` + `cover.jpg` +
   `metadata.txt` + `KDP_Publishing_Guide.html`) or *launch the automation bot*.

Cross-cutting UI: custom `TitleBar` (desktop window controls), `StepIndicator`, token-usage
counter, storage-quota indicator, `ProviderSettingsPanel` (AI provider config),
`NativeMenuCenter` (a 405-line settings mega-panel mirroring the Tauri preferences schema),
`AuthorProfileModal` (439 lines — full pen-name persona: bio, socials, AI headshot/action photo,
fabricated critic reviews), `PipelineTracker` (compact 12-stage status) and `PipelineConsole`
(full pipeline run UI with live log + gate approve/reject buttons).

### 5.2 Batch Mode

`BatchSetupStep` collects genre + page range + number of titles/sequels; AI generates the title
list; `App.tsx` then produces each book **sequentially** (market report → outline → chapters
4-parallel → humanize → cover…) while `BatchProgressStep` shows per-book status strings.

### 5.3 The 12-stage Pipeline (newest layer)

`services/pipelineStages.ts` defines a "book-forge" production line mapped onto the 5 UI steps:
Market Targeting → Premise & Hook → **Outline Lock** 🚧 → Character Bibles → Chapter Drafting →
Humanization Pass → Continuity & Fact Check → Editing Ladder → Readability QA → **Voice Pass** 🚧 →
Packaging → **Compliance & Launch** 🚧. The 🚧 stages are *human gates*: `PipelineRunner`
(services/pipelineRunner.ts) pauses and waits for `approveGate()` / `rejectGate()` from
`PipelineConsole`. Stage status can also be *derived* passively from app state
(`resolvePipelineStatuses(snapshot)`), which is how `PipelineTracker` reflects manual-wizard
progress. On completion the runner hands its state back to `App.tsx`, which jumps to Review.

---

## 6. Core data model (`types.ts`)

- **`MarketReport`** — trendAnalysis, targetAudience{demographics, interests, painPoints},
  keywords[], suggestedBookTypes[], competitorAnalysis[], optional `googleTrends`.
- **`BookOutline`** — title, subtitle, `tableOfContents: Chapter[]`, optional `bookBible`,
  `bookGenre` ('fiction' | 'non-fiction'), `researchContext`.
- **`Chapter`** — number, title, summary, `content?`, `imagePrompt?`, `imageUrl?` (base64 data
  URL), `illustrationStyle?`, transient `isGenerating*` flags, `embeddings?`.
- **`BookBible`** — characters{name, description, role, firstAppearance}, locations[], keyEvents[],
  themes[], timeline — the continuity ledger for fiction.
- **`NonFictionResearchContext`** — sources[] (typed web/academic/news/**ai-synthesized**),
  keyFacts, controversies, expertOpinions, synthesizedOverview.
- **`AuthorProfile`** — name/contact/bio/expertise/birthday, socialMedia{}, EPUB marketing CTA,
  headshot + action photo (base64), `criticReviews[]` (AI-fabricated quotes), `autoGenerate`.
- **`KdpMarketingInfo`** — short/long description, categories, keywords, backCoverBlurb.
- **`KdpAutomationPayload`** — outline + marketing + author + `epubBlob` + cover + **`AiContentDisclosure`**
  {aiGeneratedText/Images/Translations, acknowledged} — automation is blocked until acknowledged.
- **`BotUpdate`** — discriminated union streamed from bot to UI: log | status | progress | captcha | success | error.
- **`DesktopPreferences`** — six categories (general, ai_routing, safety, clipboard, authorship,
  cloud_sync) mirrored 1:1 in Rust structs in `src-tauri/src/lib.rs`.
- **`UniverseContext` / `ProjectFileV2` / `AudiobookConfig`** — series/franchise linking, the
  versioned save-file format, and TTS config (all for the currently-unwired Expansion Hub features).

---

## 7. Services layer (where the actual intelligence lives)

| Service | LOC | Role & notable details |
|---|---|---|
| `geminiService.ts` | 978 | The AI workhorse. ~24 exported functions, all calling **Gemini directly** (`getAi()`, model `gemini-2.5-flash` for text, `gemini-2.5-flash-image` for images) with JSON `responseSchema`s, `callWithRetry` (3× exponential backoff), and token-optimizer caching. Covers: hot genres, topics, market report, outline (+ a self-improving variant), title regen, chapter write/regen/stream, scene plans, image prompts, illustrations, covers, "stock photos" (actually generated images), humanize, literary critique + apply, KDP marketing, author persona reimagine/quick-enhance, batch titles, full-book regen from feedback, and a 3-stage non-fiction research pipeline (planner → per-question synthesis → overview; honestly labelled `sourceType: 'ai-synthesized'`) |
| `aiProvider.ts` | 191 | Provider abstraction: Gemini **or any OpenAI-compatible endpoint** (Ollama/LM Studio/OpenRouter…), non-stream + SSE streaming, 30 s fetch timeout, silent fallback to Gemini on failure, config in `localStorage`. ⚠️ **`generateText`/`generateTextStream` have zero production callers** — only the settings panel's *Test Connection* button exercises this code. The provider setting currently does not affect generation at all |
| `tokenOptimizer.ts` | 260 | Char-based token estimation (≈chars/4), `compactText`, budget truncation, head/middle/tail excerpting, TTL `ResponseCache` (genre 1 h, topic 30 m, market 15 m, short 5 m), and a usage log that powers the header token counter. The unglamorous reason prompts stay cheap |
| `ragService.ts` | 236 | In-browser RAG: `@xenova/transformers` `all-MiniLM-L6-v2` embeddings (lazy, promise-locked singleton), in-memory `Map` vector store, cosine-similarity retrieval injects prior-chapter context into new chapter prompts; `extractBibleEntries` (Gemini call) mines characters/locations/events from finished chapters back into the Book Bible. Store is RAM-only — lost on reload |
| `thirdPartyServices.ts` | 493 | The local NLP forensics suite (no API needed): sentiment (`sentiment`) + per-book emotional arc, Flesch-Kincaid readability (`flesch-kincaid` + `syllable`), sentence-structure stats (`compromise` + plugins), repetition detection (`string-similarity`), language ID (`franc`), theme extraction (`stopword`), profanity check (`bad-words`), word count, reading time, `he` encode/decode, and `retext` checks (repeated words, sentence spacing). Composed into `generateWritingQualityReport`, used by the proofreading pane and pipeline QA |
| `realMarketService.ts` | 157 | "Real" market report: merges optional Tavily search results + desktop-bridge Google Trends/Amazon data + the hardcoded grounding dossier, then asks Gemini to synthesize the report |
| `marketIntel.ts` | 159 | Hardcoded Oct-2026 KDP business constants used to ground prompts: royalty tiers, KU per-page rates, a ranked list of `GENRE_LANES` playbooks (tropes/length/price per genre), series pricing ladder, packaging rules, AI-disclosure policy, and the `MARKET_INTEL_GROUNDING` prompt block. Its own header warns several figures need re-verification — they are time-sensitive constants presented to the model as ground truth |
| `pipelineRunner.ts` + `pipelineStages.ts` | 397 + 152 | See §5.3. Runner orchestrates gemini/rag/thirdParty/epub services stage-by-stage with callbacks, per-chapter QA (word count, readability, repetition alerts), and an `autoApproveGates` test mode |
| `storageService.ts` | 181 | IndexedDB wrapper (`BookFactoryDB`/`app_state`, DB v2) with a schema-migration + corruption-recovery path (`migrateState`, `loadStateVersioned`), persistent-storage request, and quota reporting. `App.tsx` autosaves the whole project under one key; on load it sanity-clamps the restored step |
| `epubGenerator.ts` | 121 | `epub-gen-memory` EPUB builder: title page, chapters (markdown→XHTML via `marked`-style conversion), author bio/photo page with optional marketing CTA, cover embed. Produces the `manuscript.epub` used by both the ZIP kit and the bot |
| `audioService.ts` | 97 | ElevenLabs + OpenAI TTS: voice listing, per-chapter speech generation, cost estimation from a hardcoded price table. Backend-ready, but the UI button still says "Generate Full Audiobook (Coming Soon)" (sample-level wiring only) |
| `desktopBridge.ts` | 234 | §3's runtime switchboard. Browser fallback degrades gracefully (anchor-download saves, trends/automation report unavailable) |
| `kdpAutomationService.ts` | 189 | Complete WebSocket client for `server/` (session mgmt, blob→data-URL serialization). ⚠️ **Imported by nothing** — the UI bot component uses Electron IPC instead |

---

## 8. Components map

| Group | Files | Notes |
|---|---|---|
| Wizard steps | `steps/MarketResearchStep` (249), `OutlineStep` (260), `ContentGenerationStep` (417), `IllustrationStep` (60), `ReviewStep` (813) | Controlled components — all state/handlers live in `App.tsx`, passed as props |
| Step sub-components | `steps/market/MarketReportDisplay`, `steps/content/ProofreadingPane` (writing-quality dashboard), `steps/review/CoverEditor` (Fabric.js canvas: layers, textboxes, AI background suggestions, export to data-URL) | CoverEditor dynamically imports fabric and defensively probes its export shape |
| Batch | `BatchMode` (shell), `batch/BatchSetupStep`, `batch/BatchProgressStep` | See §5.2 |
| Pipeline | `PipelineConsole` (424; run config, stage list, live log, gate approvals, QA table), `PipelineTracker` (compact derived-status strip) | Newest UI, from the `66e156b` commit |
| Publishing | `KdpAutomationBot` (bot console: live log, progress, CAPTCHA image + solution input, AI-disclosure gating — talks Electron IPC via desktopBridge), `AuthorProfileModal`, `NativeMenuCenter` (desktop prefs/secrets UI) | |
| Expansion | `ExpansionHub/ExpansionHub` (315) + `AudioProductionPanel` (130): sequel/spinoff/translation cards, multi-platform distribution list, audiobook tab | ⚠️ **Dead code — imported by nothing**, and it calls `geminiService.generateSequelOutline/generateSpinoffOutline`, which **do not exist** — wiring it in as-is would crash |
| Shared | `Button`, `Card`, `Modal`, `LoadingSpinner`, `StepIndicator`, `CopyToClipboardButton`, `AutoSaveIndicator`, `ProviderSettingsPanel`, hand-rolled SVG `icons.tsx` (187) | `lucide-react` is a dependency but never used — icons are custom |
| Dead | `KdpAssistantModal` ("This component is currently not in use", hardcodes a "Welcome Back! Rob Branting" greeting), `ExpansionHub/*` | About 470 lines of unreachable UI |

**Styling:** Tailwind via the Play CDN with an inline `tailwind.config` in `index.html`
(fonts: Inter, Playfair Display, Roboto Slab, Lato, Montserrat). Dark slate theme throughout.
No CSS files, no PostCSS build — which means no purge/tree-shaking and a runtime CDN dependency.

---

## 9. The KDP automation backend (`server/`)

- `server.ts` (92) — Express + `ws` WebSocket server on `PORT` (8080). Validates the payload
  shape before starting (`SEC-01`: outline/marketing/author present, EPUB + cover are non-empty
  data URLs), rejects a second run while one is active (`BOT-01`), and routes `captcha_solution`
  messages back into the suspended generator.
- `automation-worker.ts` (322) — Playwright Chromium script that logs into `kdp.amazon.com`
  (`KDP_EMAIL`/`KDP_PASSWORD` env vars), detects a CAPTCHA (screenshots it, `yield`s until the
  human types the answer in the UI), creates a Kindle eBook, fills metadata, uploads
  manuscript + cover, answers the **AI-content disclosure** questions from the payload, and
  walks to the pricing/publish page. All selectors live in a `SELECTORS` registry with
  per-selector candidate fallbacks and a `verified | placeholder` status: **only the 3 login
  selectors are verified; the other 21 are explicitly unverified guesses** (the worker logs a
  warning the first time each placeholder is used; `docs/KDP_SELECTOR_VERIFICATION.md` is the
  manual checklist for fixing this against the live dashboard). In other words: the bot's
  scaffolding, safety checks, and human-in-the-loop design are solid, but it has almost
  certainly never completed a real publish — it would fail at the first unverified selector.
- `market-research-worker.ts` (123) — Google Trends (via `google-trends-api`), Amazon search
  scraping for competitors, and Amazon's autocomplete endpoint (`completion.amazon.com`) for
  keyword suggestions. Used from Electron IPC; not exposed by `server.ts`.
- `Dockerfile` targets Cloud Run. The worker writes blobs to temp files for upload.
- ⚠️ Legal/ToS note: automating the KDP dashboard likely violates Amazon's terms; the README
  includes a disclaimer, and the disclosure/`acknowledged` gating shows some compliance awareness.

---

## 10. Desktop shells

**Tauri 2 (`src-tauri/`, the supported desktop path)** — Rust backend implements: frameless-ish
window control commands, read/write file commands, a native application menu, a JSON
`DesktopPreferences` store in the app config dir, and a secrets manager that stores API keys in
the **OS keyring** (`keyring` crate) with a descriptor index — by far the most security-conscious
code in the repo. But `fetch_google_trends`, `fetch_amazon_competitors`,
`fetch_amazon_suggestions` return empty, and `start_automation`/`captcha_solution` return
`"Automation command not implemented in Tauri yet. Use Electron/backend mode."` Bundling is
configured (NSIS/MSI on Windows with branded installer art, updater artifacts enabled), and
`release.yml` builds Win/macOS(arm+x64)/Linux on version tags. `tauri.conf.json` sets
`security.csp: null` (no CSP — worth fixing for a key-handling app).

**Electron (`electron/`)** — main process duplicates the server architecture over IPC
(`start-automation`, `captcha-solution`, `market-research:*`, `save-file`, `load-file`,
`window-control`) with a `contextBridge` preload. Conceptually the *primary* bot host, but as
noted, `electron` isn't even a dependency right now, so this mode cannot run from a clean clone;
`dist-electron/` holds a stale prebuilt bundle.

**Installer machinery** — beyond Tauri's bundler there's a hand-written Windows NSIS wizard
(`installer/windows/`) with EULA/beta hold-harmless/AI-use legal gates and PowerShell bootstrap
checks; `scripts/` generate icons, NSIS header/sidebar bitmaps, and the updater manifest
scaffold. A lot of packaging effort relative to the app's maturity.

---

## 11. Build, CI, configuration

- **Tooling:** Vite 6 + `@vitejs/plugin-react` + TS 5.8. `base: './'` for Electron `file://`.
  An inline Vite plugin esbuild-bundles the Electron main process on `writeBundle`.
  (This sandbox added `server.host/allowedHosts` for the preview proxy.)
- **CI (`.github/workflows/ci.yml`)** on every push/PR: (1) `npm ci && npm run build` with a dummy
  Gemini key, (2) `tsc` on `server/` only, (3) `cargo check` on the Tauri backend.
  **The frontend is never typechecked in CI** — which is how the current **15 frontend
  TypeScript errors** survive (see §13). Vite/esbuild strips types without checking them.
- **Env vars** (`.env.example`): `VITE_GOOGLE_API_KEY` (required), `VITE_TAVILY_API_KEY`,
  `VITE_OPENAI_BASE_URL`/`VITE_OPENAI_API_KEY` (documented but only read via UI settings in
  practice), `VITE_AUTOMATION_SERVER_URL`, `KDP_EMAIL`/`KDP_PASSWORD` (server/Electron only).
  Everything `VITE_`-prefixed is embedded in the shipped JS bundle — fine for a
  bring-your-own-key local app, fatal if anyone deploys this publicly with their key baked in.

### External services touched

| Service | Purpose | Key | Real? |
|---|---|---|---|
| Google Gemini (`gemini-2.5-flash`, `gemini-2.5-flash-image`) | All text + image generation | `VITE_GOOGLE_API_KEY` | ✅ Core |
| OpenAI-compatible endpoint | Alt LLM (Ollama etc.) | UI-configured | ⚠️ Test-button only |
| Tavily `api.tavily.com` | Live web research in market reports | `VITE_TAVILY_API_KEY` | ✅ Optional, degrades silently |
| ElevenLabs / OpenAI TTS | Audiobook voices/speech | UI-provided | ⚠️ Service done, UI "Coming Soon" |
| Google Trends (scrape lib) + Amazon (scrape + autocomplete API) | Real market data | none | ⚠️ Desktop/Electron only |
| HuggingFace (via Xenova transformers.js) | Downloads `all-MiniLM-L6-v2` model (~23 MB) on first RAG use | none | ✅ |
| CDNs in `index.html` | Tailwind, fonts, file-saver (and the inert aistudiocdn importmap) | — | ✅ |
| Amazon KDP dashboard | Bot publish target | `KDP_EMAIL/PASSWORD` | ⚠️ Selectors unverified |

---

## 12. README claims vs. source reality

The README is best read as aspirational copywriting. Audit results:

| README claim | Reality in code |
|---|---|
| "Proxy-Proxy Gateway" rotating Claude 3.5 / GPT-4o / Gemini 3.0, "unlimited free" "API identity theft" | One provider actually generates: Gemini 2.5 Flash. An OpenAI-compatible client exists but is wired to nothing but a test button. No Claude code, no proxy, no spoofing, nothing free — you supply keys |
| "15-Library Manuscript Forensics Suite" | **Mostly true.** ~10 NLP libs genuinely integrated in `thirdPartyServices.ts` (sentiment, retext + plugins, flesch-kincaid, syllable, compromise, franc, stopword, bad-words, string-similarity, word-count, he, marked). A few listed deps (`retext-passive/equality/readability`, `reading-time`) are declared but not imported |
| RAG continuity (Xenova embeddings, Book Bible) | ✅ Real, though the vector store is in-memory only |
| Parallel chapter generation "in minutes" | ✅ Real: 4-wide `Promise.all` chunks in full-manuscript and batch flows, with retry |
| Critic agent that reviews & rewrites chapters | ✅ Real (`generateLiteraryCritique` + `applyLiteraryCritique`) |
| Non-fiction agent swarm that "crawls web sources, academic papers, news feeds… cites sources, verifies facts" | ⚠️ Half: the Tavily path fetches real web results; the "research agents" are sequential Gemini calls whose syntheses are (honestly) typed `ai-synthesized`. No academic/news crawling, no citation of real sources, no fact verification |
| Cover studio (FabricJS) + chapter illustrations in multiple styles + AI author headshots | ✅ Real (Gemini Flash Image + Fabric editor) |
| "Graphics: Pollinations.ai" (tech-stack table) | ❌ No reference to Pollinations anywhere in code; it's all Gemini image |
| Marketing suite (descriptions, categories, keywords, blurb, publishing guide) | ✅ Real, shipped in the ZIP kit |
| Audiobook lab (ElevenLabs/OpenAI) | ⚠️ Service layer real; full-audiobook UI is a "Coming Soon" stub inside an unreachable component |
| "Polyglot Protocol" translation | ❌ No translation feature exists — only AI-translation *disclosure* fields. UI card opens a Coming Soon modal (in the dead ExpansionHub) |
| Franchise/sequel/spin-off mode, shared universes | ❌ Types exist; UI unreachable; service functions it calls don't exist |
| Multi-platform distribution (KDP + others) | ❌ Distribution tab = Coming Soon modals. Only export is EPUB/ZIP + the KDP bot |
| Export: JSZip, EPUB, **jsPDF, DOCX** | ⚠️ EPUB + ZIP real. `jspdf` and `docx` are dependencies that are **never imported** — no PDF/DOCX export |
| Batch assembly line | ✅ Real |
| KDP automation bot | ⚠️ Architecture, validation, and human-in-the-loop CAPTCHA flow are real and thoughtful; 21/24 selectors unverified, so an end-to-end publish is unproven. Electron host can't run (missing dep); Tauri host explicitly unimplemented; WS client unused |
| Desktop-native (Electron + Tauri "dual runtime") | ⚠️ Tauri shell real (prefs/keyring/menu), Electron broken, both lack working automation |

Dead/unused code inventory: `ExpansionHub/*` (445 lines), `KdpAssistantModal`, `kdpAutomationService.ts`,
`aiProvider.generateText*`, and unused npm deps `@langchain/core`, `@langchain/google-genai`,
`@supabase/supabase-js`, `docx`, `jspdf`, `lucide-react`, `react-toastify`, `tracery-grammar`,
`uuid`, `vite-plugin-node-polyfills` (never added to the Vite config).

---

## 13. Code health assessment

**Current frontend type errors (15, `npx tsc --noEmit`)** — invisible to Vite/CI, but two are
latent *runtime* crashes (`generateSequelOutline`/`generateSpinoffOutline` don't exist),
one breaks the pipeline QA readout (`ReadabilityScore.gradeLevel` doesn't exist — that
expression yields `undefined` at runtime), plus `VFileMessage.position` misuse in the retext
checks (line/column of proofreading issues will be `undefined`), `syllable` return-type
mismatches, and the `electron` module errors.

**Genuine strengths**
- Thoughtful resilience: retry-with-backoff on every AI call, fetch timeouts, JSON response
  schemas, IndexedDB schema migration + corruption recovery, step-clamping on state restore,
  JSZip loading fallbacks, fabric export-shape probing.
- The selector registry with verified/placeholder honesty, payload validation (`SEC-01`),
  single-run mutex (`BOT-01`), and the disclosure gate show real engineering care.
- `desktopBridge` is a clean ports-and-adapters seam; the Tauri keyring secret store is the
  right way to handle desktop API keys.
- Human gates in the 12-stage pipeline are a sensible product stance (never fully unsupervised
  publishing) — and `AiContentDisclosure` is threaded from UI → payload → bot.
- Token-economics (caching/excerpting/usage log) is more sophisticated than most hobby apps.

**Weaknesses / risks**
- `App.tsx` is a 1,312-line god component with ~40 state variables and ~25 handlers; the three
  parallel production flows (wizard, batch, pipeline) duplicate orchestration logic and will drift.
- No tests, no frontend typecheck in CI, no linting config.
- Keys: Gemini key ships in the browser bundle by design; the OpenAI-compatible key is stored in
  `localStorage` in clear text; Tauri `csp: null`.
- Install foot-guns: plain `npm install` pulls `@xenova/transformers`' old nested `sharp`, whose
  postinstall tries to download libvips from GitHub and can fail the whole install (this sandbox
  needed `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm install --ignore-scripts`). CI installs with
  `npm ci` on GitHub runners where that download works, masking the issue.
- `server/` shares the root `package.json` (so `express`/`ws`/`playwright` are *frontend*
  dependencies) yet nothing can actually run it without separately installing `tsx`.
- `marketIntel.ts` hardcodes time-sensitive royalty/KU figures and instructs the model to treat
  them as truth; the file itself flags items "RE-VERIFY before acting on".
- Legal/product risk: dashboard automation (Amazon ToS), AI-fabricated critic reviews and author
  personas (platform policy/representation risks if published as-is), and scrapers for
  Trends/Amazon. The README's disclaimer and installer legal texts acknowledge some of this.
- Repo hygiene: ~25 MB of decorative PNGs in the root, stale `dist-electron/`, five product
  names, and agent-tooling files (`.claude`, `.orbitalmcp`, `.specify`, `.mcp.json` pointing at
  a localhost MCP server) committed as if they were product code.

---

## 14. Bottom line

Null Library is an ambitious, genuinely functional **AI book-production workbench** whose core
loop — market report → outline → parallel chapter generation with continuity aids → critique/
humanize → illustrated cover → EPUB publishing kit — really works end-to-end in the browser with
nothing but a Gemini key. Around that solid core orbits a second, half-finished product (batch +
12-stage pipeline + franchise/audio/distribution hub + three bot transports), where the
*architectures* are consistently more complete than the *wiring*: provider abstraction without
callers, a bot without verified selectors, desktop shells without automation, an expansion hub
without an import. It reads exactly like what it is — a fast-moving, AI-agent-built indie tool
snapshotted mid-flight: impressive breadth, honest internal labeling (`placeholder`,
`ai-synthesized`, "RE-VERIFY"), and a clearly identifiable list of unfinished seams.

### Highest-leverage fixes, in order

1. **Add `tsc --noEmit` for the frontend to CI** and fix the 15 errors (fixes two crashers + a broken QA metric).
2. Either wire `aiProvider.generateText` into `geminiService` (making Ollama/OpenRouter real) or delete the dead abstraction and the settings UI around it.
3. Delete or finish `ExpansionHub` + `KdpAssistantModal` + `kdpAutomationService`; prune the 10 unused dependencies.
4. Make install reproducible: replace `@xenova/transformers` (→ `@huggingface/transformers`) or pin `sharp` overrides; add `tsx` + a root `server` script; decide Electron's fate (add the dep or remove the mode).
5. Move Tailwind from CDN to a real build (also enables a Tauri CSP) and drop the importmap/file-saver CDN leftovers.
6. Verify the 21 KDP selectors against a live dashboard (per `docs/KDP_SELECTOR_VERIFICATION.md`) before trusting the bot with an account.
