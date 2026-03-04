# Tech Stack — Second Brain Research Dashboard

**Last Updated**: 2026-02-25

## Architecture Overview

Generative UI application. The backend is a pure LLM-powered JSON generator — it renders nothing. The frontend holds 56 pre-built React components. The LLM analyzes markdown, selects components, generates JSON specs, and streams them to the frontend via the AG-UI protocol (SSE). The frontend renders components by type lookup.

```
Markdown Input
    → Backend LLM (analyze content, select layout, generate component JSON)
    → AG-UI SSE stream
    → Frontend renders pre-built components by type lookup
```

---

## Backend

| Item | Detail |
|------|--------|
| Language | Python 3.11+ |
| Framework | FastAPI (Starlette) |
| Agent | Pydantic AI with AG-UI protocol support |
| LLM Gateway | OpenRouter API (default: Claude Sonnet) |
| Port | 8000 |

**Agent tools**: `analyze_content()`, `generate_components()`

**LLM prompts**: `agent/prompts.py` — content analysis, layout selection, component selection

**Component validation**: `agent/a2ui_generator.py`

---

## Frontend

| Item | Detail |
|------|--------|
| Framework | React 19 + Vite + TypeScript |
| Styling | Tailwind CSS (dark theme) |
| AG-UI Client | CopilotKit |
| State hook | `useDashboardAgent` — state management + zone grouping |
| Renderer | `A2UIRenderer` — recursive component rendering by type lookup |
| Port | 3010 |

---

## AG-UI Protocol

Not a UI framework. It is a communication protocol (SSE-based) for bidirectional state sync between the Pydantic AI agent and the CopilotKit frontend.

**Events**: `StateSnapshot`, `StateDelta`, `TextMessageContent`, `RunStarted`, `RunFinished`

---

## Shared State (`DashboardState`)

| Field | Type | Description |
|-------|------|-------------|
| `markdown_content` | string | Raw user input |
| `document_type` | string | Detected content category |
| `components` | array | Generated component specs |
| `status` | string | Pipeline status |
| `progress` | number | Completion percentage |
| `activity_log` | array | Processing log entries |

---

## Component Catalog (56 types)

| Category | Count | Components |
|----------|-------|------------|
| News | 4 | `HeadlineCard`, `TrendIndicator`, `TimelineEvent`, `NewsTicker` |
| Media | 4 | `VideoCard`, `ImageCard`, `PlaylistCard`, `PodcastCard` |
| Data & Stats | 6 | `StatCard`, `MetricRow`, `ProgressRing`, `ComparisonBar`, `DataTable`, `MiniChart` |
| Lists | 4 | `RankedItem`, `ChecklistItem`, `ProConItem`, `BulletPoint` |
| Resources | 4 | `LinkCard`, `ToolCard`, `BookCard`, `RepoCard` |
| People | 4 | `ProfileCard`, `CompanyCard`, `QuoteCard`, `ExpertTip` |
| Summary | 4 | `TLDR`, `KeyTakeaways`, `ExecutiveSummary`, `TableOfContents` |
| Comparison | 4 | `ComparisonTable`, `VsCard`, `FeatureMatrix`, `PricingTable` |
| Instructional | 4 | `StepCard`, `CodeBlock`, `CalloutCard`, `CommandCard` |
| Layout | 7 | `Section`, `Grid`, `Columns`, `Tabs`, `Accordion`, `Carousel`, `Sidebar` |
| Tags | 6+ | `TagCloud`, `CategoryBadge`, `DifficultyBadge`, `StatusIndicator`, `PriorityBadge`, `Tag`, `Badge` |

All types follow the `a2ui.ComponentName` naming pattern and are registered in `frontend/src/lib/a2ui-catalog.tsx`.

---

## Semantic Zones

Layout grouping zones used by `useDashboardAgent`:

`hero` | `metrics` | `insights` | `content` | `media` | `resources` | `tags`

---

## Key Files

| File | Description |
|------|-------------|
| `agent/main.py` | FastAPI endpoint, AG-UI wrapper, SSE event transformation |
| `agent/agent.py` | Pydantic AI agent, tools, `DashboardState` model |
| `agent/prompts.py` | Three LLM prompt templates |
| `agent/llm_orchestrator.py` | LLM calls, component generation orchestration |
| `agent/a2ui_generator.py` | Component spec validation, ID generation |
| `agent/content_analyzer.py` | Markdown parsing |
| `frontend/src/App.tsx` | Main app, zone-based layout rendering |
| `frontend/src/hooks/useDashboardAgent.ts` | AG-UI state management hook |
| `frontend/src/lib/a2ui-catalog.tsx` | Component registry (56 types) |
| `frontend/src/components/A2UIRenderer.tsx` | Recursive component renderer |
| `frontend/src/components/MarkdownInput.tsx` | User input capture |
| `frontend/src/lib/layout-engine.ts` | Width hint to grid span mapping |
