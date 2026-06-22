# Contribution 3: Feature Request: Persistent "Baseline" Memories

**Contribution Number:** 3  
**Student:** Srujana Kethamukkala  
**Issue:** [GitHub issue link](https://github.com/BasedHardware/omi/issues/4631)
**Status:** Phase III in-progress

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

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

  - Cloned fork
  - Required: Apple Developer signing certificate in Keychain, Node.js, Xcode command line tools
  - Added DEEPGRAM_API_KEY to desktop/Backend-Rust/.env.example for on-device transcription (uses Parakeet v3 model - needs ~20s to initialize on first run)
  - Used  cp desktop/.env.example desktop/Backend-Rust/.env ./run.sh --yolo (quick start mode) to run the macOS desktop app against the production backend - no local Rust backend or credentials required
  - Known issue in --yolo mode: prod Rust backend returns HTTP 500: no such table: memories/transcription_sessions for AgentSync — chat responses fail through the desktop agent bridge. Full chat testing requires the mobile app.

### Steps to Reproduce

  1. Install the Omi app (iOS or Android) and sign in with a Google account
  2. Go to Memories → tap + → manually add: "I want to create agents using Langchain for my AI project"
  3. Go to the Chat tab and start a new session
  4. Send: "What tech stack should I use for my project?"
  5. Observe: response is generic — no mention of Langchain
  6. Send: "Using my saved memories, what tech stack should I use for my project?"
  7. Observe: response now correctly references Langchain
  8. Result: Memory exists and is accessible but is NOT auto-injected into context — only retrieved on explicit request.

### Reproduction Evidence

  - Branch: https://github.com/srujana-keth/omi/tree/feat/baseline-memories
  - My findings: Traced the memory injection pipeline in backend/utils/llms/memory.py. The function get_prompt_memories() fetches all memories and injects them into every chat prompt via {memories_str}. However, there is no prioritization mechanism — all memories are treated equally and there is no way to flag certain memories as "always inject." The Memory model has relevant fields (manually_added, visibility, is_locked) but no is_baseline flag. The desktop chat also fails in --yolo mode due to backend DB schema issues (no such table: transcription_sessions, no such table: memories) — unrelated but worth noting for #4631 testing.

---

## Solution Approach

### Analysis

The root cause is the absence of a "baseline" memory tier in the data model and injection logic. get_prompt_data() in backend/utils/llms/memory.py fetches all memories and passes them in bulk — there is no mechanism to guarantee that high-priority memories survive context truncation or are always prepended to the prompt. The MemoryDB model in backend/models/memories.py has extensible fields (is_locked, manually_added, visibility) that show this pattern is already in use for other memory properties — adding is_baseline follows the same pattern.

### Proposed Solution

Add a boolean is_baseline flag to the Memory model. Baseline memories are always injected first in the system prompt, regardless of total memory count or context limits. Users can mark any memory as baseline via a long-press action in the mobile app. The backend exposes a PATCH endpoint to toggle this flag.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Omi stores user memories but treats all of them equally during context injection. There is no way to pin certain memories so the AI always has access to them — users must re-prompt every session.

**Match:** The MemoryDB model already uses boolean flags (is_locked, manually_added) to control memory behavior. The get_prompt_data() function already separates memories into two buckets (user_made vs generated) — adding a third "baseline" bucket follows the same pattern.

**Plan:** 
  1. Add is_baseline: bool = False to MemoryDB in backend/models/memories.py
  2. Add a PATCH /v3/memories/{memory_id}/baseline endpoint in backend/routers/memories.py to toggle the flag
  3. Modify get_prompt_data() in backend/utils/llms/memory.py to fetch baseline memories separately and prepend them to the prompt string before all others
  4. Update Memory.get_memories_as_str() to label baseline memories clearly in the prompt (e.g., [Always in context])
  5. Add a long-press "Pin as Baseline" action in the Flutter memory list UI (app/lib/)
  6. Write unit tests for the new get_prompt_data() logic in backend/tests/unit/

**Implement:** [https://github.com/srujana-keth/omi/tree/feat/baseline-memories]

**Review:** 
  - Follows black --line-length 120 --skip-string-normalization formatting
  - All imports at module top level (no in-function imports)
  - New endpoint uses uid: str = Depends(get_current_user_uid) for auth
  - No sync requests.* calls in async code
  - Pre-commit hook installed and passing

**Evaluate:** 
  - Mark the Langchain memory as baseline
  - Start a new chat session and ask a generic question with no memory hint
  - Verify the response references Langchain without explicit prompting
  - Run bash backend/test.sh to confirm existing tests pass
  - Add a unit test: get_prompt_data() always includes baseline memories at the top of the returned string

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Verified model field, endpoint existence, rate-limiting, and injection logic using `test_baseline_memories.py`.

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

Verified backend logic using `test_baseline_memories.py` which mocks database dependencies.

---

## Implementation Notes

### Week 3 Progress
  - Implemented the `is_baseline` field in `MemoryDB`.
  - Implemented `PATCH /v3/memories/{memory_id}/baseline` endpoint.
  - Updated `utils/llms/memory.py` to prioritize baseline memories in context injection.
  - Verified all backend logic using `pytest tests/unit/test_baseline_memories.py`.
  - *Note: Client-side UI integration (Flutter) is pending.*

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** 
  - `omi/backend/models/memories.py`
  - `omi/backend/routers/memories.py`
  - `omi/backend/utils/llms/memory.py`
  - `omi/backend/tests/unit/test_baseline_memories.py`
- **Key commits:** [https://github.com/srujana-keth/omi/commit/7a5f6ac21375bb9c0a79a274432ec2c1a2f21da3]
- **Approach decisions:** Added `is_baseline` as an additive field to `MemoryDB` to ensure backward compatibility with existing Firestore documents.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
