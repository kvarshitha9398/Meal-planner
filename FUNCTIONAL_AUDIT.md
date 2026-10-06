# Functional audit and backtesting

**Audited:** 2026-10-06  
**Scope:** Original project brief, repository implementation, backend tests, live NVIDIA image endpoint, browser-to-backend flow, configuration and frontend secret exposure.

## Executive result

The project is a usable **local prototype**, but it does not meet the full brief. The central components work: the FastAPI backend starts with NVIDIA configured, a live synthetic image request succeeded once through `/api/analyze`, the frontend called the backend and rendered catalog recipes, and the isolated API backtest passed 24 checks. A later NVIDIA smoke request returned HTTP 503, so hosted inference availability is intermittent and recognition accuracy is not established.

Several requested product capabilities are partial or absent. The recipe source is a five-item local catalog; chat is deterministic canned guidance; inventory and shopping lists are in memory; Supabase, authentication, ownership, and persistence are not wired. There is no labeled image evaluation set, recommendation user study, or waste-reduction measurement.

## Validation evidence

| Check | Evidence | Result |
|---|---|---|
| Backend unit suite | `backend\.venv\Scripts\python.exe -m pytest -q` | 14 passed; one dependency deprecation warning |
| Backend lint | `backend\.venv\Scripts\ruff.exe check app` | Passed |
| Frontend TypeScript | `pnpm typecheck` | Passed |
| Backend startup / key detection | FastAPI startup and `/health` on port 8001; key value not logged | Passed; NVIDIA reported configured |
| Live vision request | Synthetic tomato sent through `POST /api/analyze` | One HTTP 200; selected NVIDIA model returned `tomato` |
| Follow-up provider probe | Separate synthetic-image request | HTTP 503 from provider; hosted service is not consistently available |
| Isolated API backtest | Disposable `TestClient` process with seeded synthetic analysis | 24/24 checks passed for inventory, recommendations, constraints, shopping list, chat, and validation |
| Frontend-to-backend | Browser at `http://localhost:3001`, clicked “Explore all ideas” then “Find meal ideas” | Recipes rendered; CORS preflight and API request succeeded |
| Frontend secret scan | Searched source and generated frontend JavaScript for configured backend key | No match |
| Git state | `git rev-parse --show-toplevel` | No Git repository exists at this project path; no commit history to inspect |

The API backtest seeded a synthetic analysis in an isolated process; it did not depend on a second live model response. It covered analysis confirmation, normalized duplicate rejection, inventory create/read/update/delete, expiry and quantity fields, vegan/allergy/time/serving filters, priority explanations, gluten-free filtering, no-match behavior, recipe steps, missing-item calculation, purchased toggles, chat non-mutation, safety caveat, and invalid-input responses.

## Requirement matrix

| Brief requirement | Implementation | Audit result and limits |
|---|---|---|
| 8.1 Fridge image upload | `frontend/src/app/page.tsx`; `backend/app/main.py` | **Partial.** UI accepts JPEG/PNG/WebP, previews the selected image, limits it to 10 MB, and shows errors/loading. Backend validates MIME, size, decoding, and dimensions. Synthetic live `POST /api/analyze` succeeded; unsupported type and undersized-image tests reject bad input. A browser file-picker upload through the whole review flow was not automated. No multiple-photo workflow. |
| 8.2 AI ingredient identification | `backend/app/services.py`, `backend/app/config.py` | **Partial.** OpenAI and NVIDIA image paths exist. NVIDIA returned a tomato once via the real endpoint; a later request returned 503. Model output is parsed and schema-checked. Confidence labels are model-provided review hints, not calibrated scores. No real-photo accuracy, latency, cost, quota, or robustness benchmark. |
| 8.3 Ingredient review and confirmation | `frontend/src/app/page.tsx`; `/api/inventory/confirm` | **Partial.** UI supports editing, adding, removing, and confirming suggestions. Backend stores only confirmed items and normalizes/deduplicates names. API confirmation is backtested with synthetic analysis data; the full file-picker → edit → confirm sequence was not driven in the browser. |
| 8.4 Inventory | `backend/app/main.py`, `backend/app/schemas.py`; frontend inventory view | **Partial.** API create/list/update/delete, duplicate rejection, quantity/unit/expiry/notes fields, and confirmation passed. Frontend only manually adds a name and removes items; it has no quantity, unit, or expiry editor. All state is process memory and shared by callers; restart loses it. |
| 8.5 Meal suggestions and ranking | `backend/app/services.py`; `/api/recipes/recommend` | **Partial.** Browser flow returned five local recipes. Ranking uses confirmed ingredient overlap, missing items, selected cuisine bonus, and priority ingredients. Constraints and empty-result behavior passed backtests. Catalog size and score weights are small and not empirically tuned. |
| 8.6 Recipe retrieval/generation | `backend/app/services.py` local `CATALOG` | **Partial.** Five authored catalog recipes have ingredients and cooking steps. There is no recipe API, retrieval corpus, source URL/provenance, or model-generated recipe validation. This is local catalog output, not live AI recipe generation. |
| 8.7 Personalization and hard constraints | `backend/app/services.py`; `UserPreferences`; meal-ideas form | **Partial.** Vegan/vegetarian/gluten-free/dairy-free, allergies/disliked items, time limits, priority items, cuisine, and displayed serving count are represented to varying degrees. Vegan, allergy, time, serving label, and gluten-free cases were tested. Cuisine is API-only; the UI does not expose cuisine/equipment/serving selection (it sends 2). Equipment is not used. Serving counts are labels; recipe quantities are not scaled. Ingredient-list filters cannot guarantee allergen safety or cross-contamination handling. |
| 8.8 Food-waste reduction | Priority ranking and inventory expiry fields | **Partial.** “Use soon” priorities affect ranking. Expiry dates can be stored by API but are not used in ranking or exposed in the UI. Cooking does not consume inventory. No waste reduction has been measured. |
| 8.9 Shopping list | `backend/app/main.py`; shopping-list view | **Partial.** API computes missing catalog ingredients and supports purchased checkboxes; both passed backtesting. UI can generate a list from a recipe and check items. No quantity/unit calculations, manual list editing, merge workflow, or inventory update after cooking. |
| 8.10 AI interaction | `/api/chat` in `backend/app/main.py` | **Not built as AI.** Endpoint returns fixed local guidance for general, substitution, and allergy/safety prompts. It does not use a text-model API or selected recipe/history/inventory context. It correctly leaves inventory unchanged and includes a safety caveat. |
| 8.11 User interface | `frontend/src/app/page.tsx`, Next.js app | **Partial.** Local page rendered; browser-driven navigation to meal ideas and backend recipe results succeeded. Loading/error states and primary controls exist. No automated accessibility, keyboard, responsive-breakpoint, or complete photo-upload browser suite. |
| Supabase persistence/auth/RLS | `backend/migrations/001_initial.sql` | **Not wired.** A schema migration exists, but API code does not connect to Supabase. No authentication, owner scoping, RLS execution, or multi-user isolation. Credentials alone will not implement this integration. |
| Recognition/recommendation research evaluation | No evaluation harness or labeled corpus | **Not built.** No precision/recall/F1, correction-rate, held-out image set, user ratings, overlap baseline, latency/cost study, or waste measurement. Unit tests do not establish model accuracy. |
| Security and operations | `backend/app/config.py`, `backend/app/main.py`, `.gitignore` files | **Partial.** Provider key is loaded backend-only; startup validates configuration; env patterns are ignored; frontend bundle scan found no key. Upload limits, CORS allowlist, timeouts, retries, and generic provider errors exist. There is no API authentication, rate limiting, ownership, or durable data protection; do not expose this prototype publicly. |

## API and integration needs

- **Already connected for the prototype:** NVIDIA vision uses `NVIDIA_API_KEY` in `backend/.env`. No additional API is required for the current local recipe catalog. NVIDIA hosted inference succeeded once but returned 503 on a later probe, so keep retry/error handling and expect provider availability limits.
- **Optional alternative:** OpenAI vision can be selected with `VISION_PROVIDER=openai` and `OPENAI_API_KEY`; it is not required while NVIDIA is selected.
- **For persistent multi-user data:** Supabase is required by the brief, but connecting credentials alone is insufficient. The backend needs Supabase client integration, user-token validation, owner-scoped reads/writes, applied migrations, and RLS tests.
- **For contextual AI chat or generated recipes:** a text-model API and validated prompts/output are needed only after replacing the current deterministic helper/catalog. These capabilities are not presently implemented.
- **Local-model fallback:** no Ollama or other local vision provider exists in this project. The local recipe catalog remains usable without vision, but image analysis requires a configured cloud provider.

## Known limits / next work

1. Add browser end-to-end coverage for file upload, correction, confirmation, constraints, and shopping-list interaction.
2. Wire Supabase Auth and owner-scoped persistent inventory/list state before multi-user or public use.
3. Add expiry-aware recommendation and a cook-confirmation flow that updates inventory.
4. Decide whether contextual chat, generated recipes, or a recipe data API are product requirements; connect a provider only if those are implemented.
5. Evaluate image detection on a permitted labeled held-out set; report precision/recall/F1 and uncertainty calibration. Measure recommendation usefulness against a baseline.
6. Check live NVIDIA service availability again before relying on hosted image analysis; one success and one 503 do not establish service reliability.

## Run commands (current machine)

Backend:

```powershell
cd "C:\Users\ThinkPad\OneDrive\Desktop\meal planner\backend"
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload --port 8001
```

Frontend:

```powershell
cd "C:\Users\ThinkPad\OneDrive\Desktop\meal planner\frontend"
$env:NEXT_PUBLIC_API_URL = "http://localhost:8001"
pnpm dev --hostname localhost --port 3001
```

Edit only `backend/.env` for server credentials. The NVIDIA variable is `NVIDIA_API_KEY`; never use a `NEXT_PUBLIC_*` variable for a secret. Test provider connectivity with `.\.venv\Scripts\python.exe -m scripts.nvidia_vision_smoke` from `backend` (uses a generated image). Backend health is `/health`; interactive API docs are `/docs`.

## Final status

**Partially implemented; core prototype paths verified locally.** Live NVIDIA image analysis succeeded once and later returned 503. Persistence/auth, contextual AI chat, broader recipe retrieval, full browser image-review workflow, and measured research evaluation remain incomplete.
