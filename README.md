# Contribution 6: Feature Request: Persistent "Baseline" Memories

**Contribution Number:** 6
**Student:** Srujana Kethamukkala
**Issue:** [GitHub issue link](https://github.com/BasedHardware/omi/issues/4631)
**PR:** [#8728 — feat: persistent baseline memories](https://github.com/BasedHardware/omi/pull/8728)
**Branch:** [feature/persistent-baseline-memories](https://github.com/srujana-keth/omi/tree/feature/persistent-baseline-memories)
**Status:** MERGED — PR #8728 merged to `BasedHardware/omi` main. All blocking reviewer feedback resolved across 4 rounds of review.

---

## Why I Chose This Issue

I chose this issue because as an AI engineer and my experience with LLM infrastructure, I have spent a lot of time working with context windows and retrieval setups, so the problem of the AI forgetting foundational user facts makes total sense to me. In production systems, standard vector database searches often drop foundational user data during top-k similarity pruning, so building a dedicated "baseline" memory anchor is a highly practical problem that directly matches my backend and FastAPI skillset.

---

## Understanding the Issue

### Problem Description

Omi AI does not automatically use saved user memories when generating responses in new chat sessions. Users must explicitly prompt the AI to reference their memories (e.g., "use my saved memories to answer") for it to factor them in. This defeats the purpose of saving memories - users have to remember to ask about things they already told the app.

### Expected Behavior

When a user starts a new chat session, Omi should automatically inject certain flagged ("baseline") memories into the context window so the AI can personalize its responses without requiring explicit prompting every time.

### Current Behavior

If a user asks a generic question like "What tech stack should I use for my project?", the AI gives a generic response even if a relevant memory exists (e.g., "I want to build agents using Langchain for my project"). The memory is only used if the user explicitly says "use my saved memories to answer."

### Affected Components

- **Backend Models**: `backend/models/memories.py` (defines `MemoryDB` fields)
- **Backend Routers**: `backend/routers/memories.py` (exposes API endpoints)
- **Backend LLM Utils**: `backend/utils/llms/memory.py` (injects memories into system prompts)
- **Backend Tests**: `backend/tests/unit/test_baseline_memories.py` (unit tests)
- **Flutter Model**: `app/lib/backend/schema/memory.dart` (`isBaseline` field + layer fields)
- **Flutter UI**: `app/lib/pages/memories/widgets/memory_edit_sheet.dart` (flag toggle + badge)
- **Flutter UI**: `app/lib/pages/memories/widgets/memory_item.dart` (baseline indicator icon)

---

## Reproduction Process

### Environment Setup

- Cloned fork
- Required: Apple Developer signing certificate in Keychain, Node.js, Xcode command line tools
- Added DEEPGRAM_API_KEY to `desktop/Backend-Rust/.env.example` for on-device transcription (uses Parakeet v3 model — needs ~20s to initialize on first run)
- Used `cp desktop/.env.example desktop/Backend-Rust/.env && ./run.sh --yolo` (quick start mode) to run the macOS desktop app against the production backend — no local Rust backend or credentials required
- Known issue in --yolo mode: prod Rust backend returns HTTP 500 (`no such table: memories/transcription_sessions`) for AgentSync — chat responses fail through the desktop agent bridge. Full chat testing requires the mobile app.

### Steps to Reproduce

1. Install the Omi app (iOS or Android) and sign in with a Google account
2. Go to Memories → tap + → manually add: "I want to create agents using Langchain for my AI project"
3. Go to the Chat tab and start a new session
4. Send: "What tech stack should I use for my project?"
5. Observe: response is generic — no mention of Langchain
6. Send: "Using my saved memories, what tech stack should I use for my project?"
7. Observe: response now correctly references Langchain
8. Result: memory exists and is accessible but is NOT auto-injected — only retrieved on explicit request.

### Reproduction Evidence

My findings: Traced the memory injection pipeline in `backend/utils/llms/memory.py`. The function `get_prompt_memories()` fetches all memories and injects them via `{memories_str}`. However, there is no prioritization mechanism — all memories are treated equally and there is no way to flag certain memories as "always inject." The `Memory` model has relevant fields (`manually_added`, `visibility`, `is_locked`) but no `is_baseline` flag.

---

## Solution Approach

### Analysis

The root cause is the absence of a "baseline" memory tier in the data model and injection logic. `get_prompt_data()` in `backend/utils/llms/memory.py` fetches all memories and passes them in bulk — there is no mechanism to guarantee that high-priority memories survive context truncation or are always prepended to the prompt. The `MemoryDB` model in `backend/models/memories.py` has extensible fields (`is_locked`, `manually_added`, `visibility`) that show this pattern is already in use for other memory properties — adding `is_baseline` follows the same pattern.

### Proposed Solution

Add a boolean `is_baseline` flag to the `Memory` model. Baseline memories are always injected first in the system prompt, regardless of total memory count or context limits. Users can mark any memory as baseline via a flag button in the mobile app. The backend exposes a `PATCH` endpoint to toggle this flag.

### Implementation Plan (UMPIRE)

**Understand:** Omi stores user memories but treats all of them equally during context injection. There is no way to pin certain memories so the AI always has access to them — users must re-prompt every session.

**Match:** The `MemoryDB` model already uses boolean flags (`is_locked`, `manually_added`) to control memory behavior. The `get_prompt_data()` function already separates memories into two buckets (user_made vs generated) — adding a third "baseline" bucket follows the same pattern.

**Plan:**
1. Add `is_baseline: bool = False` to `MemoryDB` in `backend/models/memories.py`
2. Add `PATCH /v3/memories/{memory_id}/baseline` endpoint in `backend/routers/memories.py`
3. Modify `get_prompt_data()` in `backend/utils/llms/memory.py` to fetch baseline memories separately and prepend them to the prompt string
4. Add `isBaseline` field to the Flutter `Memory` model
5. Add a flag icon toggle in `memory_edit_sheet.dart` and a baseline indicator in `memory_item.dart`
6. Write behavioral unit tests in `backend/tests/unit/test_baseline_memories.py`

**Implement:** [https://github.com/srujana-keth/omi/tree/feature/persistent-baseline-memories]

**Review:**
- Follows `black --line-length 120 --skip-string-normalization` formatting
- All imports at module top level (no in-function imports)
- New endpoint uses rate-limited auth dependency (`memories:modify` policy)
- Uses `withValues(alpha:)` instead of deprecated `withOpacity()` in Flutter
- Pre-commit hook installed and passing

**Evaluate:**
- Mark the Langchain memory as baseline
- Start a new chat session and ask a generic question with no memory hint
- Verify the response references Langchain without explicit prompting
- Run `bash backend/test.sh` to confirm existing and new tests pass

---

## Testing Strategy

### Unit Tests

- [x] **Model field** — `MemoryDB.is_baseline` defaults to `False`; can be set `True`; survives dict roundtrip
- [x] **Bucket routing** — `get_prompt_data()` puts `is_baseline=True` memories in the baseline bucket, `manually_added=True` in user_made, others in generated
- [x] **Prompt label** — `get_prompt_memories()` includes a "baseline" / "always in context" label when baseline memories exist; omits it when there are none
- [x] **Locked memories** — `is_locked=True` memories are excluded from all buckets
- [x] **Baseline precedence** — `is_baseline=True` wins over `manually_added=True` (memory lands in baseline, not user_made)
- [ ] **Rate limit** — baseline endpoint is gated by `memories:modify` rate-limit policy

All unit tests are hermetic (no Firebase, no network) via `unittest.mock.patch` on DB and system calls.

### Integration Tests

- [ ] End-to-end: mark a memory as baseline, start a new chat, verify the AI references it without prompting
- [ ] Canonical memory path: verify baseline flag persists for users on the canonical memory system

### Manual Testing

Verified backend logic using `cd backend && pytest tests/unit/test_baseline_memories.py` which mocks all database dependencies.

---

## Implementation Notes

### Week 3 Progress

- Implemented the `is_baseline` field in `MemoryDB`
- Implemented `PATCH /v3/memories/{memory_id}/baseline` endpoint
- Updated `utils/llms/memory.py` to prioritize baseline memories in context injection with a distinct label
- Verified backend logic using `pytest tests/unit/test_baseline_memories.py`
- Implemented Flutter UI: `isBaseline` field in `memory.dart`, flag toggle button in `memory_edit_sheet.dart`, blue flag indicator in `memory_item.dart`

### Week 4 Progress

- **Synced fork with upstream `main`** — resolved 5 merge conflicts caused by upstream adding the canonical memory system (`MemorySystem.CANONICAL`, `MemoryService`, `MemoryLayer`, `evidence`, `memory_tier` fields):
  - `backend/models/memories.py` — kept `is_baseline` alongside new `evidence`/`memory_tier`/`layer`
  - `backend/utils/llms/memory.py` — merged canonical path support with baseline bucket logic; `get_prompt_data()` now handles both `MemorySystem.CANONICAL` and legacy paths
  - `app/lib/backend/schema/memory.dart` — kept `isBaseline` and new `layer`/`layerIsExplicit`/`primaryCaptureDevice`/`captureDeviceIds`
  - `app/lib/pages/memories/widgets/memory_edit_sheet.dart` — baseline badge UI + `withValues(alpha:)` color API
  - `app/lib/pages/memories/widgets/memory_item.dart` — `FaIconData icon` (typed, from main)
- **Addressed PR reviewer feedback (#8728):**
  - Replaced `_validate_memory` with `_validate_mutable_memory` in the baseline endpoint — canonical-path users now get a 404 from the canonical store, not the legacy store
  - Replaced regex/source-level tests with 12 behavioral tests exercising actual `MemoryDB` instantiation, `dict()` serialization, and `get_prompt_data`/`get_prompt_memories` logic via mocked DB
- Pushed all changes to `feature/persistent-baseline-memories`

### Code Changes

**Files modified:**
- `backend/models/memories.py` — `is_baseline: bool = False` field on `MemoryDB`
- `backend/routers/memories.py` — `PATCH /v3/memories/{memory_id}/baseline` with dual-path validation
- `backend/utils/llms/memory.py` — 4-bucket `get_prompt_data()` with canonical + legacy path support
- `backend/tests/unit/test_baseline_memories.py` — 12 behavioral unit tests (hermetic)
- `app/lib/backend/schema/memory.dart` — `isBaseline` field in Flutter `Memory` model
- `app/lib/pages/memories/widgets/memory_edit_sheet.dart` — flag toggle + baseline badge
- `app/lib/pages/memories/widgets/memory_item.dart` — blue flag indicator for baseline memories

**Key commits:**
- [`7a5f6ac`](https://github.com/srujana-keth/omi/commit/7a5f6ac21375bb9c0a79a274432ec2c1a2f21da3) — initial implementation (backend + Flutter UI)
- [`55ba3ed`](https://github.com/srujana-keth/omi/commit/55ba3ed2a) — merge conflicts resolved + PR feedback addressed
- [`b1040dfb`](https://github.com/srujana-keth/omi/commit/b1040dfbd) — fix injection tests with stub_modules + load_module_fresh
- [`012360e982`](https://github.com/srujana-keth/omi/commit/012360e982) — (Jul 15, 2026) fix: 4-tuple unpack in test_lock_bypass_fixes.py + restore "you already know" prompt wording
- [`883a8cd48a`](https://github.com/srujana-keth/omi/commit/883a8cd48a) — (Jul 16, 2026) fix: canonical gate (fail-closed 503), persona postprocess trigger, response_model declaration on baseline endpoint

**Approach decisions:**
- Added `is_baseline` as an additive field defaulting to `False` — fully backward-compatible with existing Firestore documents
- Used `_validate_mutable_memory` (not the simpler `_validate_memory`) so the baseline endpoint respects the canonical/legacy routing already used by other mutation endpoints
- Baseline section is only added to the prompt when at least one baseline memory exists (no empty header)

---

## Pull Request & Contribution Status

**Draft PR:** [#8728 — feat: persistent baseline memories](https://github.com/BasedHardware/omi/pull/8728)

**Brief Summary of Contribution:**
- Added `is_baseline` boolean flag to `MemoryDB` model (backward-compatible, defaults `False`)
- Implemented `PATCH /v3/memories/{memory_id}/baseline` endpoint with canonical + legacy path routing
- Updated `get_prompt_data()` to return a 4-tuple `(user_name, baseline, user_made, generated)` and inject baseline memories first with a distinct label; works on both canonical and legacy memory systems
- Added flag toggle UI in Flutter: flag icon button in the edit sheet, blue flag indicator on the memory list item
- Replaced source-level regex tests with 12 hermetic behavioral tests

**PR Reviewer Feedback Received (and addressed):**

**Round 1 — Jul 8, 2026 (`kodjima33`):**
1. ✅ **Rebase needed** — synced fork with upstream `main`, resolved 5 merge conflicts from canonical memory system addition

**Round 2 — ~Jul 10, 2026 (`git-on-my-level`):**
2. ✅ **Breaking change in existing test** — `test_lock_bypass_fixes.py:1356` unpacked `get_prompt_data` as 3-tuple; updated to 4-tuple, completed fixture data, added `resolve_memory_system` mock. Commit: `012360e982` (Jul 15, 2026)
3. ✅ **Prompt wording regression** — "you also know" for non-baseline memories was an unintentional change from original "you already know"; restored. Commit: `012360e982` (Jul 15, 2026)
4. ✅ **Behavioral tests** — replaced 9 regex/source-level checks with 12 behavioral tests (5 model + 7 injection)

**Round 3 — ~Jul 14, 2026 (`git-on-my-level`):**
5. ✅ **Canonical write inconsistency (blocking)** — baseline endpoint was silently writing `is_baseline=True` to the legacy Firestore store for canonical users; canonical read path (MemoryService) never sees legacy writes, so the flag silently had no effect. Fixed: added `_canonical_write_enabled_or_fail_closed` gate — canonical users now receive an explicit 503. Commit: `883a8cd48a` (Jul 16, 2026)
6. ✅ **Missing persona postprocessing** — every other memory mutation endpoint triggers `submit_with_context(postprocess_executor, update_personas_async, uid)` after write; baseline endpoint did not. Added. Commit: `883a8cd48a` (Jul 16, 2026)
7. ✅ **Missing `response_model` declaration** — all other mutation endpoints declare `response_model=MemoryMutationResponse` on the router decorator; baseline endpoint did not, causing OpenAPI spec to have no documented response schema. Added. Commit: `883a8cd48a` (Jul 16, 2026)

**Round 4 — Jul 17, 2026 (`git-on-my-level` + `kodjima33`):**
- All three items from Round 3 confirmed resolved
- Minor observation (non-blocking): `safe_create_memory` constructs `MemoryDB` (requires `id`, `uid`, `created_at`, `updated_at`) instead of `Memory` (no required fields) — if any legacy Firestore doc is missing those fields it is silently skipped. Acknowledged; low-risk since properly written Firestore docs always carry these fields.
- `kodjima33` re-approved on new SHA `883a8cd48a`
- PR still open pending human product/architecture review for the feature concept

**Status:** MERGED — PR #8728 merged to `BasedHardware/omi` main

---

## Learnings & Reflections

### Technical Skills Gained

- **FastAPI dual-path routing pattern** — learned how Omi's codebase separates canonical vs legacy memory system paths using `_canonical_write_enabled_or_fail_closed` / `_validate_mutable_memory`, and how to follow that pattern in a new endpoint
- **Pydantic v2 behavioral testing** — learned to write hermetic unit tests for Pydantic models by instantiating them directly and asserting field defaults/serialization without needing any external services
- **Git merge conflict resolution in large codebases** — resolved conflicts where upstream added a new feature (canonical memory system) that touched the same files as my PR; required understanding both sides of the diff before choosing which changes to keep

### Challenges Overcome

- **Merge conflicts with canonical memory system** — upstream added `MemorySystem.CANONICAL`, `MemoryService`, and `evidence`/`memory_tier`/`layer` fields to the same files I modified. Required reading the canonical memory architecture to understand what to keep from both sides rather than simply picking one.
- **Source-level vs behavioral tests** — my initial tests used regex on file contents (e.g., `assert 'is_baseline' in source`), which the reviewer correctly flagged as not testing actual behavior. Rewrote them to instantiate real `MemoryDB` objects and mock the DB layer to test `get_prompt_data()` logic.
- **Desktop app testing limitations** — the `--yolo` mode against the production backend doesn't support the Rust agent bridge, so chat responses fail. Understood the architecture well enough to trace the memory injection path in the Python backend independently.

### What I'd Do Differently Next Time

- **Read the architecture docs first** — I could have avoided some rework by reading `AGENTS.md` and `backend/AGENTS.md` before writing the first line of code, which would have told me about the canonical memory system and the dual-path routing pattern.
- **Write behavioral tests from the start** — starting with regex/source checks was faster initially but created re-work when the reviewer asked for proper behavioral coverage.
- **Check for upstream feature flags early** — the canonical memory system was behind a feature flag; checking `canonical_write_decision` and `canonical_read_enabled` patterns in the existing code would have flagged the dual-path requirement before the first draft PR.

---

## Resources Used

- [Omi Issue #4631 — Feature Request: Persistent "Baseline" Memories](https://github.com/BasedHardware/omi/issues/4631)
- [Omi AGENTS.md — backend coding guidelines, import purity, test isolation](https://github.com/BasedHardware/omi/blob/main/AGENTS.md)
- [Omi backend/AGENTS.md — async I/O rules, executor pool assignment, test runner](https://github.com/BasedHardware/omi/blob/main/backend/AGENTS.md)
- [FastAPI dependency injection docs](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [Pydantic v2 model validation docs](https://docs.pydantic.dev/latest/concepts/models/)
- [Python unittest.mock — patch.object usage](https://docs.python.org/3/library/unittest.mock.html#patch-object)
