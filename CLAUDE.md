# CLAUDE.md — Second Brain Research Dashboard

AI assistant intelligence file for this project.

**Project**: Generative UI dashboard that converts markdown research notes into structured, visually rich dashboards using an LLM-powered component selection pipeline.

See [TECHSTACK.md](./TECHSTACK.md) for full architecture and component catalog.

---

## Quick Start

**Backend**
```bash
cd agent && uv sync && uv run uvicorn main:app --reload --port 8000
```

**Frontend**
```bash
cd frontend && npm install && npm run dev
```

---

## Environment Variables

| File | Variable | Description |
|------|----------|-------------|
| `agent/.env` | `OPENROUTER_API_KEY` | LLM gateway API key |
| `frontend/.env` | `VITE_BACKEND_URL` | Backend URL (e.g. `http://localhost:8000`) |

---

## Key Conventions

- Component types follow the `a2ui.ComponentName` pattern (e.g. `a2ui.StatCard`, `a2ui.Section`).
- All components are registered in `frontend/src/lib/a2ui-catalog.tsx`. Adding a new component requires an entry there.
- The backend generates JSON specs, never HTML or JSX. Rendering is exclusively the frontend's responsibility.
- Layout grouping is driven by semantic zones (`hero`, `metrics`, `insights`, `content`, `media`, `resources`, `tags`) resolved in `useDashboardAgent`.
- LLM prompt changes go in `agent/prompts.py` only — do not embed prompts inline in other files.
