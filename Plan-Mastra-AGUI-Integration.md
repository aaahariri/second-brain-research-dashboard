# Plan: Mastra + AG-UI Backend Integration

**Goal**: Build a Next.js backend using Mastra that serves the existing frontend via the AG-UI protocol. The frontend requires zero changes — it already consumes AG-UI events through CopilotKit.

---

## Frontend Contract (What the Backend Must Satisfy)

### AG-UI Transport

The frontend creates an `HttpAgent` from `@ag-ui/client` (v0.0.43) pointing at a backend URL (default `http://localhost:8000`). It connects via `CopilotKitProvider` with `agentId: "dashboard_agent"`.

```tsx
// frontend/src/main.tsx — existing, do not modify
const dashboardAgent = new HttpAgent({
  url: BACKEND_URL,
  agentId: "dashboard_agent",
});
```

The backend must accept `POST /` with AG-UI protocol messages and respond with SSE (`text/event-stream`). Event types must be `SCREAMING_SNAKE_CASE` (e.g. `STATE_SNAPSHOT`, `RUN_STARTED`).

### DashboardState Shape

The frontend reads `agent.state` and casts it to `DashboardState`. The backend must emit `STATE_SNAPSHOT` events with this exact shape:

```typescript
interface DashboardState {
  markdown_content: string;
  document_title: string;
  document_type: string;        // "tutorial" | "research" | "article" | "guide" | "notes" | "technical_doc" | "overview"
  content_analysis: Record<string, any>;
  layout_type: string;          // "instructional" | "data" | "news" | "list" | "summary" | "reference" | "media"
  components: A2UIComponent[];  // The generated component specs
  status: "idle" | "analyzing" | "generating" | "complete" | "error";
  progress: number;             // 0–100
  current_step: string;         // Human-readable status text
  activity_log: Array<{
    id: string;
    message: string;
    timestamp: string;           // ISO 8601
    status: "in_progress" | "completed" | "error";
  }>;
  error_message: string | null;
}
```

### A2UI Component Spec

Each component in the `components[]` array must follow:

```typescript
interface A2UIComponent {
  id: string;                    // Unique, e.g. "comp-1"
  type: string;                  // "a2ui.StatCard", "a2ui.CodeBlock", etc.
  props: Record<string, any>;    // Component-specific data
  children?: A2UIComponent[];    // For layout components (Section, Grid, etc.)
  layout?: {
    width?: "full" | "half" | "third" | "quarter";
  };
  zone?: "hero" | "metrics" | "insights" | "content" | "media" | "resources" | "tags";
}
```

56 component types are registered in `frontend/src/lib/a2ui-catalog.tsx`. See `TECHSTACK.md` for the full catalog.

### Frontend Behavior During Generation

1. Frontend calls `agent.setState()` with `{ markdown_content, status: "analyzing" }`
2. Frontend calls `agent.addMessage()` with user message containing the markdown
3. Frontend calls `agent.runAgent()`
4. Frontend watches `state.status` transitions: `analyzing` → `generating` → `complete`
5. Frontend renders `state.components` progressively as they appear in state snapshots
6. Frontend groups components by `zone` field and renders in order: hero → metrics → insights → content → media → resources → tags

---

## Phase 1: Project Scaffolding

### 1.1 Initialize Next.js project

Create `backend-mastra/` directory in the project root.

```bash
npx create-next-app@latest backend-mastra --typescript --app --tailwind --eslint --src-dir --use-npm
```

Minimal config — this is an API-only backend (no UI pages needed).

### 1.2 Install dependencies

```bash
npm install @mastra/core @mastra/core/agents @ag-ui/mastra zod
npm install @mastra/core/tools @mastra/core/workflows
```

### 1.3 Environment setup

Create `backend-mastra/.env`:

```env
OPENROUTER_API_KEY=<key>
OPENROUTER_MODEL=anthropic/claude-sonnet-4-20250514
BACKEND_PORT=8000
ALLOWED_ORIGINS=http://localhost:3010,http://localhost:3011,http://localhost:3012
```

### 1.4 Project structure

```
backend-mastra/
├── src/
│   ├── app/
│   │   └── api/
│   │       └── copilotkit/
│   │           └── route.ts          # AG-UI endpoint (POST /)
│   ├── mastra/
│   │   ├── index.ts                  # Mastra instance configuration
│   │   ├── agents/
│   │   │   └── dashboard-agent.ts    # Main dashboard agent
│   │   ├── tools/
│   │   │   ├── analyze-content.ts    # Content analysis tool
│   │   │   └── generate-components.ts # Component generation tool
│   │   ├── prompts/
│   │   │   ├── content-analysis.ts   # Content analysis prompt
│   │   │   ├── layout-selection.ts   # Layout selection prompt
│   │   │   └── component-selection.ts # Component selection prompt
│   │   └── lib/
│   │       ├── content-analyzer.ts   # Markdown parsing logic
│   │       ├── component-validator.ts # A2UI component validation
│   │       └── types.ts              # DashboardState, A2UIComponent types
│   └── middleware.ts                 # CORS middleware
├── .env
├── package.json
├── tsconfig.json
└── next.config.js
```

---

## Phase 2: Type Definitions & Shared Types

### 2.1 Define DashboardState and A2UIComponent types

File: `src/mastra/lib/types.ts`

- Define `DashboardState` interface matching the frontend contract exactly (see above)
- Define `A2UIComponent` interface with `id`, `type`, `props`, `children`, `layout`, `zone`
- Define `SemanticZone` union type
- Define `DocumentType` and `LayoutType` union types
- Define Zod schemas for each type (used by Mastra tools for validation)

### 2.2 Define valid component type list

File: `src/mastra/lib/component-validator.ts`

- Export `VALID_COMPONENT_TYPES` array containing all 56 `a2ui.*` type strings
- Export `isValidComponentType(type: string): boolean` helper
- Export `generateComponentId(): string` sequential ID generator (e.g. `comp-1`, `comp-2`)
- Export validation function that checks a component spec has required fields and a valid type

---

## Phase 3: Content Analysis

### 3.1 Markdown parser

File: `src/mastra/lib/content-analyzer.ts`

Port the markdown parsing logic. This is pure string processing (no LLM):
- Extract title from first `# heading`
- Extract sections (h2/h3 headings with their content)
- Detect code blocks (language, content)
- Detect tables
- Detect links (URLs, YouTube links, GitHub repos)
- Count statistics (sections, code blocks, links, etc.)
- Return a `ParsedContent` object

### 3.2 Content analysis prompt

File: `src/mastra/prompts/content-analysis.ts`

Port the content analysis prompt from the existing `prompts.py` (lines 21–104). This prompt:
- Classifies the document type (tutorial, research, article, guide, notes, technical_doc, overview)
- Extracts key entities (technologies, tools, languages, concepts)
- Identifies the document's intent and structure
- Returns structured JSON

### 3.3 Analyze content tool

File: `src/mastra/tools/analyze-content.ts`

Create a Mastra tool using `createTool()`:
- Input: `{ markdown: string }` (Zod schema)
- Calls `parseMarkdown()` for structural analysis
- Calls the LLM with the content analysis prompt
- Returns: `{ document_title, document_type, content_analysis }` (Zod schema)

---

## Phase 4: Component Generation

### 4.1 Layout selection prompt

File: `src/mastra/prompts/layout-selection.ts`

Port the layout selection prompt from `prompts.py` (lines 111–241). This prompt:
- Takes the content analysis as input
- Weighs: structure (40%), type (30%), intent (20%), length (10%)
- Selects one of 7 layout types
- Lists component priorities for the selected layout

### 4.2 Component selection prompt

File: `src/mastra/prompts/component-selection.ts`

Port the component selection prompt from `prompts.py` (lines 248–566). This is the largest prompt. It:
- Lists all 56 available component types with their props
- Provides pattern-matching rules (e.g. "statistics → StatCard", "code → CodeBlock")
- Enforces variety rules (min 4 types, max 2 consecutive same type, no type >40%)
- Instructs the LLM to return a JSON array of component specs
- Each spec includes: `type`, `props`, `zone`, `layout.width`

### 4.3 Generate components tool

File: `src/mastra/tools/generate-components.ts`

Create a Mastra tool using `createTool()`:
- Input: `{ markdown: string, content_analysis: object, document_type: string }` (Zod schema)
- Calls the LLM with layout selection prompt → gets `layout_type`
- Calls the LLM with component selection prompt → gets component array
- Validates each component (type must be in `VALID_COMPONENT_TYPES`, required fields present)
- Assigns sequential IDs
- Returns: `{ layout_type, components: A2UIComponent[] }` (Zod schema)

---

## Phase 5: Dashboard Agent

### 5.1 Agent definition

File: `src/mastra/agents/dashboard-agent.ts`

```typescript
import { Agent } from "@mastra/core/agents";

export const dashboardAgent = new Agent({
  name: "dashboard_agent",
  instructions: `You are a specialized AI assistant that transforms Markdown research documents
into interactive dashboard components.

Your workflow:
1. Analyze the markdown content to understand its structure and type
2. Generate A2UI dashboard components that best represent the information

When the user provides markdown content:
1. First call analyze_content to understand and classify the document
2. Then call generate_components to create the UI components

Always explain what you're doing as you work.`,
  model: "openrouter/anthropic/claude-sonnet-4-20250514",
  tools: {
    analyze_content: analyzeContentTool,
    generate_components: generateComponentsTool,
  },
});
```

### 5.2 Mastra instance

File: `src/mastra/index.ts`

```typescript
import { Mastra } from "@mastra/core";

export const mastra = new Mastra({
  agents: { dashboard_agent: dashboardAgent },
});
```

---

## Phase 6: AG-UI Endpoint

### 6.1 CopilotKit / AG-UI route

File: `src/app/api/copilotkit/route.ts`

This is the critical integration point. Use `@ag-ui/mastra` to bridge Mastra → AG-UI protocol:

```typescript
import { registerCopilotKit } from "@ag-ui/mastra";
import { mastra } from "@/mastra";

// Creates AG-UI compatible route handler
const copilotKit = registerCopilotKit(mastra);

export const POST = copilotKit;
```

Key requirement: The AG-UI handler must emit `STATE_SNAPSHOT` events with the full `DashboardState` object as the agent progresses. This means the tool implementations need to update the state incrementally:

1. After `analyze_content`: emit state with `status: "analyzing"`, `progress: 40`, analysis results
2. During `generate_components`: emit state updates as each component is generated, `status: "generating"`, incrementing `progress`
3. On completion: emit final state with `status: "complete"`, `progress: 100`

### 6.2 State synchronization strategy

The AG-UI protocol sends the current frontend state to the backend with each request. The backend must:

1. Read `markdown_content` from the incoming state
2. Progressively update the state object via `STATE_SNAPSHOT` events
3. Include the full `DashboardState` in each snapshot (not just deltas)

**Critical**: Research how `@ag-ui/mastra` exposes state snapshot emission from within tools. If `registerCopilotKit` does not natively support incremental state snapshots from tool execution, an alternative approach may be needed:

- Option A: Use Mastra's `onStepFinish` callback to emit state snapshots after each tool call
- Option B: Structure the agent as a Mastra workflow where each step emits state
- Option C: Manually construct SSE responses in the route handler and use the Mastra agent imperatively (not via registerCopilotKit)

This is the highest-risk integration point. Prototype Phase 6 early and validate that state snapshots stream correctly to the frontend before investing in prompt porting.

### 6.3 CORS middleware

File: `src/middleware.ts`

Configure CORS to allow requests from `http://localhost:3010` (and other allowed origins from env).

### 6.4 Root route redirect

If the frontend points at `http://localhost:8000/` (root), ensure the AG-UI endpoint is at `POST /`. Options:
- Put the route handler at `src/app/api/route.ts` (serves at `/api`) and update `VITE_BACKEND_URL`
- Or use Next.js rewrites in `next.config.js` to proxy `/` → `/api/copilotkit`
- Or configure the frontend to point at `http://localhost:3001/api/copilotkit` (requires changing `VITE_BACKEND_URL`)

Simplest approach: change `VITE_BACKEND_URL` in `frontend/.env` to the new Mastra endpoint URL.

---

## Phase 7: Prompt Porting

### 7.1 Port content analysis prompt

Copy the logic from `agent/prompts.py` lines 21–104. Adapt from Python f-string to TypeScript template literal. Keep the prompt identical in meaning — the LLM behavior should not change.

### 7.2 Port layout selection prompt

Copy from `agent/prompts.py` lines 111–241. Same approach.

### 7.3 Port component selection prompt

Copy from `agent/prompts.py` lines 248–566. This is the largest prompt (~300 lines). It contains:
- Full component type catalog with props definitions
- Pattern matching rules
- Variety enforcement rules
- Zone assignment guidance
- Width hint guidance

Port verbatim. Do not rewrite or simplify — the prompt is heavily tuned.

### 7.4 Port variety validation

Port the post-generation validation from `prompts.py` lines 684–755:
- Minimum 4 different component types
- No more than 2 consecutive same type
- No single type exceeds 40% of total
- If validation fails, re-prompt the LLM with specific feedback

---

## Phase 8: Integration Testing

### 8.1 Validate AG-UI handshake

1. Start the Mastra backend (`npm run dev` in `backend-mastra/`)
2. Update `frontend/.env` to point `VITE_BACKEND_URL` at the Mastra endpoint
3. Open the frontend, paste a sample markdown document
4. Verify the browser's Network tab shows:
   - `POST` to the backend URL
   - Response content-type: `text/event-stream`
   - SSE events with `STATE_SNAPSHOT` type
   - State includes `status`, `progress`, `components`

### 8.2 Validate progressive rendering

1. During generation, confirm the frontend shows:
   - Progress bar advancing (`state.progress`)
   - Status text updating (`state.current_step`)
   - Activity log entries appearing (`state.activity_log`)
   - Components rendering as they appear in `state.components`

### 8.3 Validate component rendering

1. After generation completes, confirm:
   - Components are grouped by zone correctly
   - Zone sections appear in order: hero → metrics → insights → content → media → resources → tags
   - Component types resolve to actual React components (no yellow error cards)
   - Layout widths are applied correctly (full/half/third/quarter)

### 8.4 Test with all sample documents

Run each file in `sample-documents/` through the system:
- `agentic-workflows-tutorial.md` — should produce instructional components (StepCard, CodeBlock)
- `ai-industry-statistics.md` — should produce data components (StatCard, DataTable, MiniChart)
- `ai-news-weekly.md` — should produce news components (HeadlineCard, TrendIndicator)
- `claude-vs-gpt-comparison.md` — should produce comparison components (ComparisonTable, VsCard)
- `top-10-coding-tools.md` — should produce list/resource components (RankedItem, ToolCard)

---

## Phase 9: Polish & Parity

### 9.1 Error handling

- If the LLM call fails, set `status: "error"` and `error_message` in state
- If component validation fails, log the invalid component and skip it (don't crash the generation)
- If the AG-UI connection drops, the frontend already handles reconnection via CopilotKit

### 9.2 Activity log

Emit activity log entries at each stage:
- "Starting content analysis" (in_progress)
- "Analysis complete: {document_type}" (completed)
- "Generating dashboard components..." (in_progress)
- "Generated {count} components" (completed)

### 9.3 URL validation

Port the URL validation logic — if a component references an external URL (e.g. VideoCard, LinkCard, RepoCard), validate it's a properly formed URL. Don't make HTTP requests to validate; just check format.

---

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| `@ag-ui/mastra` may not support incremental state snapshots from within tool execution | High — frontend won't show progressive rendering | Prototype Phase 6 first. Fallback: manually construct SSE stream. |
| `registerCopilotKit` endpoint path may not match what the frontend expects | Medium — connection failure | Test early. Use Next.js rewrites or adjust `VITE_BACKEND_URL`. |
| Mastra's OpenRouter model string format may differ from expected | Low — agent won't start | Check Mastra docs for exact model identifier syntax. |
| SSE event type casing (PascalCase vs SCREAMING_SNAKE_CASE) mismatch | Medium — frontend ignores events | The frontend's `@ag-ui/client` expects SCREAMING_SNAKE_CASE. Verify `@ag-ui/mastra` output format. |
| Prompt porting introduces subtle behavior differences | Low — different component selection | Test with all 5 sample documents and compare output quality. |

---

## Execution Order

**Recommended order to minimize risk:**

1. **Phase 1** — Scaffold the project
2. **Phase 2** — Define types
3. **Phase 6** — AG-UI endpoint (prototype with a stub agent that returns hardcoded state) ← **validate frontend connection first**
4. **Phase 5** — Wire up the real agent
5. **Phase 3** — Content analysis tool
6. **Phase 4** — Component generation tool
7. **Phase 7** — Port prompts
8. **Phase 8** — Integration testing
9. **Phase 9** — Polish

Phase 6 comes before 3/4/5 because the AG-UI integration is the highest-risk unknown. Validating the SSE handshake with a stub agent before investing in prompt porting avoids wasted work.
