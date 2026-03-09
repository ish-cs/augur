# Augur v0.3.2 — Enterprise-Grade AI Accuracy Plan

**Status:** Planning
**Version:** v0.3.2
**Goal:** Eliminate hallucinations and achieve 99%+ grounded accuracy in "Ask AI" responses by migrating from Naive RAG to a State-of-the-Art Enterprise Architecture (CRAG, Contextual Chunking, Re-ranking).

---

## 1. Executive Summary: The Path to 99% Accuracy

Based on 2024–2025 enterprise AI research (Anthropic, Semi-Analysis, LangChain CRAG implementations), achieving near-100% accuracy requires abandoning simple chunk-and-search vector logic. Hallucinations stem from three failures:
1. **Context Loss:** Cutting sentences arbitrarily strips their meaning.
2. **Retrieval Blindspots:** Relying solely on vector cosine similarity misses exact keywords or temporal logic.
3. **LLM Over-confidence:** The LLM guesses when context is weak rather than admitting ignorance.

This document serves as the **actionable implementation master plan** to rebuild the `Ask AI` pipeline across `context-server.py` and `screenpipe-dashboard.html`.

---

## 2. Advanced Architectural Blueprint

The v0.3.2 pipeline will be rebuilt using these four enterprise pillars:

### Pillar A: Anthropic-Style Contextual Chunking
*Problem: `text.slice(0, 150)` destroys the meaning of the underlying code/text.*
* **Implementation**: Stop arbitrary slicing. Parse OCR captures by natural boundaries (e.g., newline groupings).
* **Context Augmentation**: Before storing/embedding a chunk, prepend standard metadata so the vector understands the source even if the text is generic. 
    * *Format:* `[App: VS Code] [Window: context-server.py] [Time: 14:32:00] -- {chunk_text}`. 
    * By appending this *before* embedding, the semantic search has an anchor.

### Pillar B: Two-Stage Hybrid Retrieval + Cross-Encoder Reranking
*Problem: Raw vector search retrieves loosely related garbage.*
* **Implementation (`context-server.py`)**:
    1. **Stage 1 (Hybrid Recall)**: Fetch the Top 50 candidates using a combination of dense vector semantic similarity (already loosely in `context-server.py`) AND sparse BM25 keyword matching. 
    2. **Stage 2 (Precision Reranking)**: Pass those 50 candidates through a lightweight Cross-Encoder (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`) or apply a strict reciprocal rank fusion (RRF) algorithm. 
    3. Only the Top 10 highly-scored chunks are sent to the LLM.

### Pillar C: Corrective RAG (CRAG) with Self-Reflection Gate
*Problem: The LLM tries to answer even when the Top 10 chunks don't actually contain the answer.*
* **Implementation (`screenpipe-dashboard.html`)**:
    * Introduce a fast "Grader" step. Before showing the user the final answer, run a lightweight evaluation prompt under the hood:
        * *Prompt:* "You are a Grader. Assess if the retrieved context definitively contains the answer to the user's query: '{query}'. Context: {context}. Respond only with 'YES' or 'NO'."
    * If `YES`: Proceed to standard Generation.
    * If `NO`: Immediately short-circuit the generation and output the fallback string: `"I cannot find the answer to this cleanly in your recent screen history."` This enforces zero hallucinations.

### Pillar D: Strict Parameterization & System Prompts
*Problem: Multi-turn conversations drop the context window, causing immediate hallucination on turn 2. Temperature is set to creative mode (0.7).*
* **Implementation (`screenpipe-dashboard.html`)**:
    * **Statefulness:** Re-inject the Top 10 chunks as a system prompt prefix on *every single turn* of the chat array.
    * **Temperature:** Hardcode to `0.1` or `0.15` max.
    * **Citation Enforcement:** Prompt must demand: `"For every claim, explicitly cite the application name and timestamp provided in the context."`

---

## 3. Step-by-Step Implementation Guide for the Agent

This section provides exact instructions for implementing the architecture across the two affected core files.

### Phase 1: Context Stability & Prompt Engineering (Frontend)
**Target File:** `screenpipe-dashboard.html`

1. **Fix Multi-Turn Context Drop (Critical Bug)**
   * Find the `getScreenpipeContext()` loop inside `handleAIKeyPress()`.
   * **Action**: Currently, context is only added if `historyMessages.length === 0`. Remove this condition. You MUST prepend the system prompt + screen context to the `userMessageWithSystem` on *every* turn. Retain the last 6 `chatHistory` messages for conversational flow, but always re-attach the grounding context to the newest query.
2. **Implement Generative Parameters**
   * Change `temperature` to `0.15`.
   * Change `max_tokens` to `1200`.
3. **Rewrite the System Prompt**
   * **Action**: Implement this exact grounding prompt:
     ```text
     "You are Augur, an AI assistant with direct access to the user's screen capture history.
     STRICT RULES:
     1. ONLY answer using the screen capture data provided below.
     2. If the data does not contain the answer, respond EXACTLY with: 'I don't see this in your recent screen history.' Do not guess.
     3. Always cite the timestamp and app name when answering (e.g., 'At 2:34 PM in VS Code...').
     4. Do not infer beyond what the text explicitly says."
     ```
4. **Implement Temporal Query Routing**
   * **Action**: Write a RegEx interceptor `parseTemporalQuery(query)` that scans the user's input. If they say "yesterday", pass `window_hours=48` to the backend. If "today", pass `window_hours=24`. If "last week", pass `168`. Append this to the `/context` fetch URL.

### Phase 2: Building the RAG Pipeline Engine (Backend)
**Target File:** `context-server.py`

1. **Implement Aggressive Deduplication**
   * **Action**: Inside `/context` processing, create a dictionary mapping `(app_name, window_name)` to captures. If a new capture belongs to the same `app` and `window` as an existing one, only keep the one with the highest semantic score. This prevents 20 identical VS Code frames from filling the context window.
2. **Contextual Text Expansion**
   * **Action**: Stop returning truncated text snippets. Ensure `text` fields are returned at up to 500 characters, cleanly stripped of `\n\n\n` whitespace blocks and repeated OCR stuttering (`aaaaaa`).
   * **Action**: Add `text_chars: len(cleaned_text)` to the JSON response so the frontend can calculate token budgets safely.
3. **Hybrid Scaling (The Re-Ranker)**
   * **Action**: Increase the initial screenpipe fetch `limit` from `20` to `50`. 
   * Apply semantic cosine similarity against the query embedding (already present). 
   * Slice the sorted array to exactly the top `15` highest scoring, deduplicated captures to send back via JSON.

### Phase 3: The CRAG Evaluation Loop (Frontend UI Flow)
**Target File:** `screenpipe-dashboard.html`

1. **Action**: Implement the Grader Loop. 
    * When `getScreenpipeContext()` returns context, do not immediately stream the generation.
    * Make a rapid, non-streaming `/v1/chat/completions` call to LM Studio requesting a `YES` or `NO` on whether the context answers the user's query.
    * If the response string starts with `"NO"`, bypass the streaming generator and immediately append the hardcoded "I don't see this..." message to the UI.

---

## 4. Verification & Testing Matrix

To prove 99% accuracy has been met, the agent must ensure the following tests pass manually:

| Test ID | Scenario | Expected Behavior | Pass/Fail Criteria |
|---------|----------|-------------------|--------------------|
| **T1: Fiction Test** | User asks "What did I write in my AWS config file?" (when no such file was opened). | The CRAG grader intercepts and the AI responds: "I don't see this in your recent screen history." | Must NOT attempt to guess generic AWS commands. |
| **T2: Context Retention** | User asks "What was the function name?" -> AI answers. User asks "What parameters did it take?" | Model correctly answers the follow-up because the `contextBlock` was injected perfectly into turn 2. | Must NOT say "As an AI, I don't see previous context." |
| **T3: Grounding Citations** | User asks "What was I reading on the browser?" | AI answers: "At 1:45 PM in Arc Browser, you were reading..." | Must include Timestamp and App explicitly. |
| **T4: Temporal Scoping** | User asks: "What did I work on last week?" | Backend receives `window_hours=168` and returns older data. | Must return data older than 24h. |

---

## 5. Constraints & Guidelines for Implementing Agent
- **No Extra Endpoints**: Keep changes confined to existing server routing in `context-server.py`.
- **Local Priority**: Ensure all LLM Grader and Generation calls point to `localhost:1234` (LM Studio).
- **Vanilla JS**: Modifying `screenpipe-dashboard.html` must remain framework-free vanilla JS.
- **Fail Gracefully**: If the Context Server is offline, fallback immediately to "Knowledge not available" safely.
