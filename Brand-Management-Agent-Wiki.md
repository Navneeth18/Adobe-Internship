# Brand Management Agent — Wiki

> **Adobe Express AI Assistant · Internship Project**
> **Author:** D Navneeth Reddy (`@donthir`) · **Manager:** Amit Aggarwal · **Mentors:** S V Raghavendra Nuggu, Anushka Narula

---

## Table of Contents

1. [Overview](#1-overview)
2. [Approach](#2-approach)
3. [Architecture](#3-architecture)
4. [Pipeline Changes](#4-pipeline-changes)
5. [Brand Management Agent — Detailed](#5-brand-management-agent--detailed)
   - [5.1 All 23 LLM Tools](#51-all-23-llm-tools)
   - [5.2 Multi-Step Chaining Mechanism](#52-multi-step-chaining-mechanism-steprefsts)
   - [5.3 Brand & Element Resolution](#53-brand--element-resolution)
   - [5.4 PDF-to-Brand Pipeline](#54-pdf-to-brand-pipeline)
   - [5.5 Error Handling](#55-error-handling)
6. [Applying Brand to Designs (Conversational)](#6-applying-brand-to-designs-conversational)
7. [Passing Brand Context at Generation Time](#7-passing-brand-context-at-generation-time)
8. [Client-Side Architecture](#8-client-side-architecture)
   - [8.1 Action Handlers](#81-action-handlers-homechatsassiststorets)
   - [8.2 HzaBrandAssets Component](#82-hzabrandassets-component)
   - [8.3 Brand Manager UI Dialogs](#83-brand-manager-ui--manual-counterpart)
9. [Shared Types](#9-shared-types-assistant-types)
10. [Feature Flags](#10-feature-flags)
11. [Prompt Templates](#11-prompt-templates)
12. [Testing](#12-testing)
13. [Results](#13-results)
14. [Future Work](#14-future-work)
15. [File Inventory](#15-file-inventory)

---

## 1. Overview

In Adobe Express, managing a brand previously required **leaving the AI Assistant chat** and navigating to the Brands hub. This project introduces a conversational **Brand Management Agent** — a new subagent within the AI Assistant that brings the entire brand lifecycle directly into the chat interface.

Users can now:

- **Create and edit** brand libraries (create, rename, delete, list)
- **Upload and organize** brand assets (logos, graphics, colour palettes, fonts)
- **Apply a brand** to the currently open design (deterministic recolor + font swap + logo placement)
- **Generate on-brand designs** with brand context (logos, colours, fonts, guidelines) injected into the generation pipeline

The agent ships behind the `aiAssistantBrandManagementAgent` feature flag and exposes **23 LLM tools** across library-level, element-level, and design-integration categories. The implementation spans **~12,600 insertions across 94 files** in 43 commits, with the brands subagent directory alone contributing **7,597 lines** (4,392 production + 323 prompts + 2,132 tests).

---

## 2. Approach

A standalone Brand Agent prototype (built outside the monorepo) was ported into the **Horizon monorepo** as a new **conversational subagent** that plugs into the existing Atomic Steps Orchestrator, alongside `generateTemplatesAgent`, `designQueryAgent`, and `findAndPreviewAssetsAgent`.

### Key design decisions

| Decision | Rationale |
|----------|-----------|
| **Reuse `MelvilleController`** (`@hz/brands-core`) instead of a separate API client | Consistent auth handling via the real user token; no duplicate API surface |
| **Fully agentic** — the LLM drives every action through tool calls | No hardcoded shortcuts or deterministic branching; the agent adapts to any user phrasing |
| **Single atomic step** for compound brand operations | Separate atomic steps execute in parallel without shared state; chaining requires sequential execution within one step |
| **Flag-gated routing** changes | Three shared pipeline blocks modified, each byte-identical when the flag is off — zero risk to existing assistant functionality |
| **Follow repo conventions** throughout | Standard `ClientActionBasedSubagent` base class, Jest test harness, Handlebars prompts, same linting/formatting as other subagents |

---

## 3. Architecture

The architecture diagram below shows the full execution flow — from agent entry and message construction, through the LLM call and sequential execution loop, to post-processing and client action emission.

![Brand Management Agent execution architecture](./brand-management-agent-architecture.png)

### Where the Brand Agent fits in the AI Assistant pipeline

```
User Prompt → AssistantService.assist() → WorkflowRouter
    → ConversationalWorkflow
        → ConversationalIntentAnalysisBlock  ← MODIFIED (brand = converse)
        → HarmAndBiasBlock ‖ PromptCompletenessBlock  ← MODIFIED (brand = always complete)
        → ConversationalPlannerBlock  ← MODIFIED (brand = single atomic step)
        → AtomicStepsOrchestratorBlock
            → generateTemplatesAgent
            → designQueryAgent
            → findAndPreviewAssetsAgent
            → brandManagementAgent  ← NEW (registered in AgentRegistryFactory)
```

---

## 4. Pipeline Changes

Three shared conversational pipeline blocks were modified to route brand requests to the new agent. All changes are **flag-gated** behind `aiAssistantBrandManagementAgent` and byte-identical when the flag is off.

| Block | File | Change |
|-------|------|--------|
| **ConversationalIntentAnalysisBlock** | `conversational-intent-analysis-block.prompt.hbs` | Brand management requests (CRUD, upload, palette, fonts) always classified as `converse`, never `redirect` to the Brands hub. Bare confirmations to brand follow-ups also `converse`. |
| **PromptCompletenessBlock** | `PromptCompletenessBlock.ts` + `.prompt.hbs` | Brand requests = always `isComplete: true`. Attachment summaries surfaced even when `multiTurnDocumentBootstrap` flag is off. |
| **ConversationalPlannerBlock** | `conversational-planner.prompt.hbs` | Compound brand operations must be a **single atomic step** (not split — separate steps execute in parallel without shared state). `summaryMessage` describes action-in-progress, not outcome. |

---

## 5. Brand Management Agent — Detailed

**Main file:** `services/assistant/assistant-service/src/subagents/brands/BrandManagementAgent.ts` (2,657 lines)

The agent extends `ClientActionBasedSubagent` and implements the full brand management lifecycle. It uses **Azure GPT-4o (PTU)** as its LLM with a system prompt (`brand-management-system.prompt.hbs`) and 23 tool definitions.

### 5.1 All 23 LLM Tools

The `BrandAction` enum at `BrandManagementAgent.ts:166-191` defines all tool actions:

#### Library-Level (4 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandListLibraries` | List user's brand libraries | Brand names + URNs |
| `brandCreateLibrary` | Create a new brand library | `brandUrn` |
| `brandRenameLibrary` | Rename a library | — |
| `brandDeleteLibrary` | Delete a library | — |

#### Asset Upload (4 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandUploadLogo` | Upload an image as a logo | `elementId`, `name` |
| `brandUploadGraphic` | Upload an image as a graphic | `elementId`, `name` |
| `brandAddPdfImage` | Batch upload classified PDF images | `elementIds[]`, `names[]` |
| `brandReplaceElement` | Replace an element's primary image | `elementId`, `name` |

#### Element Operations (6 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandListElements` | List elements in a brand | Element details |
| `brandDeleteElement` | Delete an element | — |
| `brandRenameElement` | Rename an element | — |
| `brandDuplicateElement` | Duplicate within same brand | `elementId`, `name` |
| `brandCopyElement` | Copy across brands | `elementId`, `name` |
| `brandMoveElement` | Move across brands | `elementId`, `name` |

#### Role Management (2 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandSetElementRole` | Set `primary` / `secondary` / `none` (logos + palettes only) | — |
| `brandSetFontRole` | Set `header` / `body` / `none` (fonts only) | — |

#### Colour & Font (3 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandAddColorPalette` | Create a colour palette element | `elementId` |
| `brandExtractLogoColors` | Extract dominant colours via Adobe Sensei API → deduped `#rrggbb[]` | `colors[]` |
| `brandAddFont` | Add font(s) with roles (client-side hand-off, supports multiple fonts + roles) | — |

#### Metadata (2 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandSetGuidelines` | Set brand guidelines (overview, industry, audience, tone) | — |
| `brandSetAssetUsage` | Set per-element usage instructions (e.g., "place in top-left corner") | — |

#### Design Integration (3 tools)

| Tool | Action | Outputs |
|------|--------|---------|
| `brandApplyToDesign` | Deterministic recolor + font swap on the open design (client hand-off) | — |
| `brandAddElementToDesign` | Place an exact logo/graphic on the canvas at full-quality rendition | — |
| `brandPlaceLogoOnDesign` | Vision-based intelligent logo placement — GPT-4o analyses the canvas layout | — |

---

### 5.2 Multi-Step Chaining Mechanism (`stepRefs.ts`)

**File:** `services/assistant/assistant-service/src/subagents/brands/stepRefs.ts` (149 lines)

When the LLM emits multiple tool calls in a single response, they execute **sequentially** (not in parallel) within `postProcessActionInvocations()`. Each tool call is assigned a step ID (`s1`, `s2`, `s3`...) and can reference a prior step's output using `$sN.key` tokens.

#### How it works

1. **`parseStepRef(value)`** — parses `$sN.key` or `$sN.key[index]` strings into `StepRef` objects
2. **`collectStepRefs(args)`** — recursively walks the arguments tree, collecting all step references
3. **`resolveStepRefs(args, stepOutputs)`** — deep-replaces each `$sN.key` with the real value from the `stepOutputs` map
4. **Splice-in-place** — when `$sN.key` resolves to an array and is a single element of an outer array, items are spliced in place (e.g., `colors: ['$s3.colors']` → `['#36B38C', '#FF5733', ...]`)

#### Example: 4-step chain in one user message

```
User: "Create a brand called PulseFit, upload this logo, extract its colours, and save them as a palette"

s1: brandCreateLibrary("PulseFit")
    → stepOutputs.s1 = { brandUrn: "urn:..." }

s2: brandUploadLogo(brandName: "$s1.brandUrn")
    resolves → brandName: "urn:..."
    → stepOutputs.s2 = { elementId: "abc", name: "Logo" }

s3: brandExtractLogoColors(elementName: "$s2.name")
    resolves → elementName: "Logo"
    → stepOutputs.s3 = { colors: ["#36B38C", "#FF5733"] }

s4: brandAddColorPalette(colors: ["$s3.colors"])
    resolves → colors: ["#36B38C", "#FF5733"] (spliced)
    → stepOutputs.s4 = { elementId: "def" }
```

#### Error handling in chains

- A **dangling reference** (step not yet run) fails **only that step**
- A failed step does **not** abort the rest of the chain — subsequent steps continue

---

### 5.3 Brand & Element Resolution

The agent resolves brand and element references from user free-text using a multi-strategy approach.

#### Brand Resolution (`_resolveBrand`)

```
inputs.brandName provided?
  ├─ yes → starts with "urn:"? (step-ref resolved URN)
  │    ├─ yes → look up by URN in listBrands()
  │    └─ no  → _matchBrandByName() — case/diacritic-insensitive
  │              exact match, then partial "contains" match
  │              ├─ 1 match  → ✅ Resolved
  │              └─ 0 or 2+  → Return clarification string
  └─ no  → use attached brand (supplementalContext.brand.id)
           ├─ exists → ✅ Resolved
           └─ none   → Return clarification string
```

#### Element Resolution (`_resolveElement`)

```
1. Check by element ID (for step-ref resolved IDs)
    → match? ✅ Resolved
2. Case-insensitive exact name match
    → match? ✅ Resolved
3. _ensureElementDescriptions()
    → Lazy-caption uncaptioned logo/graphic elements
      (vision LLM call, persisted to Melville)
4. _matchElementByReference()
    → LLM JSON call via element-match.prompt.hbs
      → matched / ambiguous / none
        ├─ matched   → ✅ Resolved
        ├─ ambiguous → Return disambiguation prompt (lists candidates)
        └─ none      → Return "could not find" message
```

---

### 5.4 PDF-to-Brand Pipeline

Attaching a brand-guidelines PDF auto-builds a complete brand kit in one pass.

#### Stage 1: Text Extraction (`pdfBrandText.ts` — 110 lines)

- Uses `unpdf extractText()` + `getMeta()`
- Limits: max 30 pages, 5 MiB, 15-second timeout
- Sanitizes whitespace and form-feed characters
- Output: `{ text, pageCount, truncated, title }`

#### Stage 2: Image Extraction (`pdfBrandImages.ts` — 504 lines)

- Uses `unpdf getDocumentProxy()` to scan operator lists per page
- Finds `paintImageXObject` operations (opcode 85)
- Builds **two PNG resolutions** per image:
  - **Upload resolution:** max 1600px, 5 MiB (for Melville storage)
  - **Analysis resolution:** max 768px, 2 MiB (for vision classification)
- **SHA-256 deduplication** — identical images across pages are merged
- Filters out images < 100px and near-uniform images (luminance range + stddev check)
- Output: `PdfBrandImage[]` (max 10 images, max 3 per page)

#### Stage 3: Vision Classification (`pdfBrandImageAnalysis.ts` — 161 lines)

- **Batched vision LLM call** (Azure GPT-4o PTU) using `pdf-brand-image-analysis.prompt.hbs`
- Per-image classification:
  - `kind`: `logo` / `graphic` / `skip`
  - `suggestedName`, `role`, `caption`
  - `usageNote`, `usageConfidence`
- `mergePdfBrandImageAnalysis()` joins classifications with image data
- Drops entries with `usageConfidence < 0.6`

#### Stage 4: Agent Ingestion

The classified images and extracted text flow into `_buildGroundingLines()` as part of the user message. The LLM then calls `brandUploadLogo` / `brandUploadGraphic` / `brandAddColorPalette` / `brandSetGuidelines` per item to build the complete brand kit.

---

### 5.5 Error Handling

Each tool validates its inputs and returns structured error objects with user-friendly messages. Invalid operations (deleting the last brand, uploading an unsupported file format, referencing a non-existent brand) are caught **before** the API call. The error message guides the LLM to retry with corrected parameters within the same conversational turn, without user intervention.

When 0 brand tool calls are found in an LLM response, the agent returns a corrective message: *"I didn't make any brand change"* — prompting the LLM to retry.

---

## 6. Applying Brand to Designs (Conversational)

From chat, a user can apply their brand to the currently open design. Three tools handle different aspects:

### `brandApplyToDesign`

Triggers Adobe Express's **deterministic** apply-brand pipeline via a client-side hand-off:

1. Loads all library elements for the brand URN
2. Builds a recommended brand style
3. Dispatches `ApplyBrandActionType.applyBrandSinglePage`
4. Executes **recolor** (EMD/colour-distance clustering remaps existing vector fill/stroke colours onto the brand's colour theme) and **font swap** (ranks existing text nodes by font size — largest gets header font, rest get body font)

**Handler:** `HomeChatsAssistStore.ts:4039`

### `brandAddElementToDesign`

Places an **exact** logo or graphic onto the canvas at full-quality rendition:

1. Fetches the element rendition blob (auth-fetched via IMS token)
2. Centers it on the canvas (20% of shorter dimension, minimum 200px)
3. Resizes and places it via `assistant.addUserAsset` file-drop action

**Handler:** `HomeChatsAssistStore.ts:3792`

### `brandPlaceLogoOnDesign`

Uses a **GPT-4o vision model** to choose intelligent placement coordinates:

1. Fetches the logo rendition blob
2. Captures a canvas screenshot
3. Calls `analyzeBrandLogoPlacement` (vision LLM) with a 30-second timeout
4. Receives `{x, y, width, height}` coordinates from the model
5. Scales vision coordinates to document pixel bounds
6. Falls back to **bottom-right corner** (5% margin, 20% of shorter dimension) if vision fails
7. Optionally triggers `setCutoutAsync` for background removal

**Handler:** `HomeChatsAssistStore.ts:3625`
**Vision prompt:** `logo-placement-analysis.prompt.hbs` (14 lines)

---

## 7. Passing Brand Context at Generation Time

For net-new designs, brand context now flows into the generation pipeline through the `GenerateTemplatesSubagent`.

**File:** `services/assistant/assistant-service/src/subagents/generateTemplates/GenerateTemplatesSubagent.ts`

### Brand Resolution

The subagent resolves the target brand in two ways:

1. **Picker-attached** — the user selected a brand from the brands picker UI
2. **Free-text name match** — `_prefetchBrandMentionedInMessage` (line 141) uses `BrandContextResolver.resolveBrandMentionedInText()` to find a brand named in the user's message

### `BrandContextResolver`

**File:** `services/assistant/assistant-service/src/brands-context/BrandContextResolver.ts`

Resolves the full `BrandContext` object:
- `logos: BrandAsset[]` — with rendition URLs, roles, usage notes
- `graphics: BrandAsset[]`
- `colors` — hex colour values
- `fonts` — with header/body roles
- `guidelines: BrandGuidelines` — overview, industry, audience, tone
- `assetUsageNotes: BrandAssetUsageNote[]`

Reads `application_metadata` from Melville for guidelines and asset usage data (the same store written by both the chat agent and the manual UI dialogs).

### Logo as Generative Reference Image

**Method:** `_resolveBrandReferenceImages` (lines 625–659)

1. `_selectPrimaryBrandLogo(brand.logos)` picks the logo tagged `priority: "primary"` (falls back to `"secondary"`; does NOT fall back to an arbitrary logo)
2. `BrandContextResolver.fetchAssetImageBytes(primaryLogo)` downloads the logo at **2048 px / full rendition**
3. Re-uploads the image bytes to Colligo via `apiUtils.uploadImage()` to get a presigned URL
4. Returns `[{ presignedUrl }]` — exactly **one** brand reference image
5. On any failure, returns `[]` — never blocks generation

**Gated on:** `aiAssistantBrandAssetsInGeneration` AND absence of `aiAssistantBrandLogoPlaceholder`

### Brand Context as Text

**Method:** `_getBrandContextSummary` (lines 364–384)

Serializes the primary logo (id, name, usage note), colours, fonts, and guidelines into a JSON text block appended to the user message. Brand graphics are intentionally excluded. The system prompt (`system.prompt.hbs`) teaches the model how to interpret and apply the brand JSON.

---

## 8. Client-Side Architecture

### 8.1 Action Handlers (`HomeChatsAssistStore.ts`)

**File:** `apps/project-x/features/x-chats/src/stores/home-chats-assist-store/HomeChatsAssistStore.ts`

The store's action dispatcher (lines 6541–6571) routes server response actions to handlers:

| Action ID | Handler | Line | Description |
|-----------|---------|------|-------------|
| `brandAssets` | `_getBrandAssets()` | — | Renders in-chat brand asset cards |
| `brandAddFont` | `_executeAddFont()` | 3950 | Resolves font via `FontProviderStore.searchFont()`, creates one Melville element per role |
| `brandApply` | `_executeBrandApply()` | 4039 | Deterministic recolor + font swap via `ApplyBrandActionType` |
| `brandAddElementToDesign` | `_executeBrandAddElementToDesign()` | 3792 | Center-placed exact logo/graphic on canvas |
| `brandPlaceLogoOnDesign` | `_executeBrandPlaceLogoOnDesign()` | 3625 | Vision-based logo placement with fallback |
| `brandGuidelinesUpdated` | `_executeSetGuidelines()` | 6567 | Persists to `BrandLocalDataStore` + Melville metadata |
| `brandAssetUsageUpdated` | `_executeSetAssetUsage()` | 6570 | Persists to `BrandLocalDataStore` + Melville metadata |

### 8.2 `HzaBrandAssets` Component

**File:** `apps/project-x/features/x-chats/src/components/hza-brand-assets/HzaBrandAssets.ts`
**Tag:** `<hza-brand-assets>`

Renders a horizontal carousel of brand asset categories:

| Category | Element kinds | Display |
|----------|--------------|---------|
| **Logos** | `logo` | Image thumbnails |
| **Colors** | `color`, `colortheme` | Swatch strips (`_renderSwatches`) |
| **Fonts** | `font` | Image thumbnails with typographic "Ag" fallback tile |
| **Graphics** | `graphic` | Image thumbnails |

**Features:**

- **Auth-fetched renditions** — images fetched with IMS Bearer token + project-x API key (lines 122–157), stored as object URLs, revoked on disconnect for GC
- **Expand/collapse** — shows up to 4 items per category (`PREVIEW_COUNT = 4`), with "Show all" / "Show less" toggle
- **Drag-and-drop** — every card has `draggable="true"`, emits custom MIME `application/x-hz-brand-asset` with a `BrandAssetDragRef` payload (elementId, name, kind, role, brandUrn, brandName) + `text/plain` fallback
- **Click-to-add** — logo and graphic cards (`CANVAS_PLACEABLE_KINDS`) fire a `brand-asset-add-to-canvas` custom event

### 8.3 Brand Manager UI — Manual Counterpart

Two new dialogs in `apps/project-x/web` that write to the **same backend store** the chat agent reads:

#### `BrandsManagerGuidelinesDialog`

**File:** `apps/project-x/web/src/components/x-brands/brands-manager-details/BrandsManagerGuidelinesDialog.ts`

- Four fields: **Overview**, **Industry**, **Target Audience**, **Tone** (each max 500 chars)
- Opened from the toolbar overflow menu
- Persists to `BrandLocalDataStore.saveBrandGuidelines` (localStorage fast echo) AND Melville `application_metadata` under `GUIDELINES_METADATA_KEY`

#### `BrandsManagerElementUsageInstructionsDialog`

**File:** `apps/project-x/web/src/components/x-brands/brands-manager-details/BrandsManagerElementUsageInstructionsDialog.ts`

- Single multiline text field for usage instructions (max 1000 chars)
- Placeholder: *"e.g. Place in the top-left corner with clear space..."*
- Opened from the element right-click context menu
- Persists to `BrandLocalDataStore.saveAssetUsage` (localStorage) AND Melville `application_metadata` under `ASSET_USAGE_METADATA_KEY`

> **Key insight:** The manual UI dialogs and the conversational brand agent are two front doors into the **same** brand-intelligence data store. A user can type "don't stretch this logo" by hand in the Brands manager, or say it in chat — either way, `BrandContextResolver` picks it up identically for design generation.

---

## 9. Shared Types (`assistant-types`)

**File:** `features/assistant/assistant-types/src/shared/action/BrandAssetsTypes.ts`

### New types

| Type | Purpose |
|------|---------|
| `BrandAssetKind` | `logo` / `color` / `colortheme` / `font` / `graphic` |
| `BrandAssetItem` | Individual asset with id, name, kind, role, rendition URL |
| `BrandFontRole` | `header` / `body` / `none` |
| `BrandApplySettings` | Configuration for deterministic brand apply |
| `BrandGuidelines` | `{ overview, industry, targetAudience, tone }` |
| `BrandAssetDragRef` | Drag payload for in-chat asset cards |

### New shared action IDs

**File:** `features/assistant/assistant-types/src/shared/action/AssistantSharedActionIds.ts`

| Action ID | Line | Purpose |
|-----------|------|---------|
| `brandAssets` | 163 | In-chat card rendering |
| `brandAddFont` | 170 | Client-side font add |
| `brandGuidelinesUpdated` | 178 | Client-side guidelines persistence |
| `brandAssetUsageUpdated` | 186 | Client-side usage note persistence |
| `brandApply` | 195 | Deterministic apply-brand pipeline |
| `brandAddElementToDesign` | 204 | Exact canvas placement |
| `brandPlaceLogoOnDesign` | 214 | Vision-based canvas placement |

### Enriched types (`AssistantRequestTypes.ts`)

`BrandContext` now includes: `id`, `logos: BrandAsset[]`, `graphics: BrandAsset[]`, `guidelines: BrandGuidelines`, `assetUsageNotes: BrandAssetUsageNote[]`.

---

## 10. Feature Flags

**Flag forwarding file:** `apps/project-x/features/x-chats/src/stores/home-chats-assist-store/HomeChatsAssistantFeatureFlagPlugin.ts`

| Flag | Gates | Lines |
|------|-------|-------|
| `aiAssistantBrandManagementAgent` | Agent registration, intent carve-out, completeness carve-out, client flag forwarding. **Main gate** — when off, the entire brand agent is invisible. | 133 |
| `aiAssistantBrandsEnabled` | Brand colour/font context resolution (pre-existing flag), brands panel visibility. Parent gate for the two flags below. | 136 |
| `aiAssistantBrandAssetsInGeneration` | Brand logo + context injection into `GenerateTemplatesSubagent`. Nested under `aiAssistantBrandsEnabled`. | 141 |
| `aiAssistantBrandLogoPlaceholder` | Vision-based logo placement on canvas. Skips generative reference image and uses direct compositing instead. | 145 |

All flags are **independently toggleable**. When all flags are off, the codebase is **byte-identical** to before the feature was added — zero impact on existing assistant functionality.

---

## 11. Prompt Templates

Six Handlebars prompt files power the agent's LLM interactions:

| File | Lines | Purpose |
|------|-------|---------|
| `brand-management-system.prompt.hbs` | 196 | Main system prompt — rules (always call tools, never narrate), chaining instructions (`$sN.key` syntax), conditional logo-placement section |
| `brand-management-capabilities.prompt.hbs` | 20 | Planner-facing capabilities summary — lists all operations this agent handles (used by `ConversationalPlannerBlock`) |
| `pdf-brand-image-analysis.prompt.hbs` | 49 | Classify PDF-extracted images as `logo` / `graphic` / `skip`, suggest names, extract usage instructions with confidence |
| `element-match.prompt.hbs` | 30 | Fuzzy element matching — LLM decides `matched` / `ambiguous` / `none` from element inventory + user reference |
| `element-caption.prompt.hbs` | 14 | One-sentence image captioning for lazy element description (persisted to Melville for future lookups) |
| `logo-placement-analysis.prompt.hbs` | 14 | Vision-based logo placement — analyze design canvas + logo image, return `{ x, y, width, height }` coordinates |

---

## 12. Testing

**136 unit tests** across 4 spec files, using the repo's standard Jest harness with mocked `MelvilleController` responses.

| Spec File | Tests | Lines | Covers |
|-----------|-------|-------|--------|
| `BrandManagementAgent.spec.ts` | 90 | 1,586 | All 23 tool handlers, brand/element resolution, multi-step chaining, CTA emission, error cases, PDF ingestion flow |
| `BrandsApiService.spec.ts` | 22 | 347 | API methods, role builders (`buildLogoRoles`, `buildColorRoles`, `buildFontRoles`), Sensei colour extraction, error handling |
| `stepRefs.spec.ts` | 17 | 106 | Parse, collect, resolve, splice-in-place, dangling-ref error isolation |
| `pdfBrandImages.spec.ts` | 7 | 93 | Image extraction, SHA-256 deduplication, size/uniformity filtering |
| **Total** | **136** | **2,132** | |

---

## 13. Results

### Generation Model Comparison

Brand context was tested with three generation models by passing the brand's primary logo at **2048 px / full rendition** as a generative reference image:

| Model | Logo Reconstruction | Text Fidelity | Overall |
|-------|-------------------|---------------|---------|
| LegoDesigns | Partial | Poor | Baseline |
| Firefly3PNb2 | Moderate | Poor | Better |
| **Firefly3PGptImg2** | **Accurate symbols & shapes** | **Text deteriorates** | **Best** |

**Key findings:**

- **Firefly3PGptImg2** gave the best results — at 2048 px resolution / full rendition, **symbols and shapes are reconstructed accurately**
- **Wordmark text still deteriorates** — this is an inherent limitation of generative reference-image reconstruction (the model re-interprets the logo rather than compositing it)
- The deterioration is not a bug in the implementation but a fundamental constraint of how generative reference images work in the Firefly pipeline

---

## 14. Future Work

### Logo Placeholder Compositing

**Flag:** `aiAssistantBrandLogoPlaceholder`

Instead of passing the logo as a generative reference image (which the model re-interprets), composite the original logo **directly onto the generated canvas** at a vision-chosen position. This bypasses the generative model entirely for the logo, preserving exact wordmark fidelity.

### Proposed Metadata Fields

Two new metadata fields to enrich brand context at generation time:

1. **Brand Guidelines** — stores information about the brand's target audience and design style, persisted in Melville `application_metadata` and sent to the generation model as structured context
2. **Asset Usage Notes** — per-asset placement instructions (e.g., "place in top-left corner with clear space"), providing the generation model with explicit guidance on how to incorporate each brand asset

---

## 15. File Inventory

### Brands Subagent (server-side)

| File | Lines | Type |
|------|-------|------|
| `BrandManagementAgent.ts` | 2,657 | Agent core (all 23 handlers, resolution, chaining, PDF pipeline) |
| `BrandsApiService.ts` | 724 | Melville API wrapper (18 methods) |
| `pdfBrandImages.ts` | 504 | PDF image extraction + dedup + filtering |
| `pdfBrandImageAnalysis.ts` | 161 | Vision-based image classification |
| `pdfBrandText.ts` | 110 | PDF text extraction via unpdf |
| `stepRefs.ts` | 149 | Step reference parsing + resolution |
| `logoPlacementAnalysis.ts` | 87 | Vision-based logo placement analysis |
| 6 × `.prompt.hbs` | 323 | Handlebars prompt templates |
| 4 × `.spec.ts` | 2,132 | Unit tests (136 total) |
| **Subtotal** | **6,847** | |

### Generation Integration

| File | Change |
|------|--------|
| `GenerateTemplatesSubagent.ts` | Brand resolution, `_resolveBrandReferenceImages`, `_getBrandContextSummary` |
| `BrandContextResolver.ts` | `resolveBrandContextByName()`, `resolveBrandMentionedInText()`, `fetchAssetImageBytes()` |
| `BrandAssetItemMapper.ts` (new) | Maps `BrandContext` → `BrandAssetItem[]` for in-chat rendering |

### Client-Side

| File | Change |
|------|--------|
| `HomeChatsAssistStore.ts` | 6 brand action handlers (apply, place logo, add element, add font, set guidelines, set usage) |
| `HzaBrandAssets.ts` (new) | In-chat brand asset carousel component |
| `HzaChatThread.ts` | Renders `<hza-brand-assets>`, handles asset-add-to-canvas events, accepts PDF attachments |
| `HomeChatsAssistantFeatureFlagPlugin.ts` | 4 brand flags forwarded to server |
| `BrandsManagerGuidelinesDialog.ts` (new) | Manual brand guidelines editor |
| `BrandsManagerElementUsageInstructionsDialog.ts` (new) | Manual asset usage note editor |

### Shared Types

| File | Change |
|------|--------|
| `BrandAssetsTypes.ts` (new) | 16 new types for brand actions |
| `AssistantSharedActionIds.ts` | 7 new action IDs |
| `AssistantRequestTypes.ts` | Enriched `BrandContext` with logos, graphics, guidelines, usage notes |

### Other Modified Files

| File | Change |
|------|--------|
| `PixelEditAttachments.ts` | `allowedMimeTypes` option (enables PDF ingestion), `isExpiredAttachmentError()` helper |
| `AttachmentPromptContextUtil.ts` | `hasPdfAttachment` flag, PDF brand text/image user message builders |
| `AssistantService.ts` | Brand context resolution wired into request pipeline |
| `HzaChats.ts` | "Remove background" button when `brandAssetJustAdded` |

### Scope Summary

| Metric | Value |
|--------|-------|
| Total commits | 43 |
| Total insertions | ~12,600 |
| Files changed | 94 |
| Production code (brands subagent) | 4,392 lines |
| Prompt templates | 323 lines |
| Unit tests | 2,132 lines (136 tests) |
