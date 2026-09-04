# VoiceControlledHermes — Clean Specifications

> These are the current specs, any agent is free to edit them.

Fork of Handy (Tauri 2.x, Rust + React/TS) at upstream commit `fbd4e15`. All file references verified against the current tree by a read-only code survey.*

**Decisions locked with Clement (2026-09-04):** Hermes gets an HTTP/WebSocket API · opencode driven via `opencode serve` + SDK · TTS runs Rust/llama.cpp-first (Kokoro native, Qwen3-TTS via GGUF) · classic paste-to-app dictation stays, with a raw-vs-cleaned option.

---

## 0. Summary

Keep Handy's transcription + global-shortcut flow untouched, and extend it into a voice-controlled agent front-end:

1. **Cleanup stage** — a local `S1-mini` (Superwhisper) 0.6B model normalizes the transcript into clean written text.
2. **Review overlay** — after transcription, the overlay shows the **raw** and the **cleaned** transcript side by side; both are editable; each has a **Send** button.
3. **Agent layer** — `Send` targets the chosen agent (**Hermes** or **opencode**), behind an abstract `AgentAdapter` trait so new agents are drop-in.
4. **Speech-aware framing** — every sent message carries a preamble: "This message went through speech-to-text and its answer will be spoken through text-to-speech. Be concise."
5. **Permissions in the overlay** — when an agent needs to run a command, the request is surfaced as an overlay card with Allow once / Always / Deny.
6. **Clarifications spoken** — if the agent asks a clarifying question or proposes options, the overlay shows them and TTS reads them aloud; options are clickable answers.
7. **Spoken answers** — the agent's final answer is synthesized with **Qwen3-TTS 0.6B** (default) and played back.
8. **Two new settings sections** — **Agents** and **TTS Models**. TTS (and cleanup) models are distributed through the same catalog/download machinery Handy uses for transcription models (extended), with entries for Kokoro, Qwen3-TTS 0.6B/1.7B, and VoxCPM2.

## 1. Goals / Non-goals

**Goals**
- G1: S1-mini cleanup runs locally, integrated into the existing post-processing pipeline (one pipeline, one history).
- G2: Overlay review mode: view + edit raw/cleaned, send either to the selected agent.
- G3: `AgentAdapter` abstraction; v1 ships Hermes (HTTP/WS), opencode (server API), and a generic OpenAI-compatible adapter.
- G4: Command-permission requests and clarifications are visible and actionable in the overlay.
- G5: Answers (and clarifications) are spoken via TTS; TTS engines are pluggable and catalog-distributed.
- G6: Classic dictation (auto-paste) keeps working; new setting chooses raw vs cleaned text for it.

**Non-goals (v1)**
- VoxCPM2 local runtime (Python-only today) — catalog entry ships, engine marked *future*.
- Speaking *into* the agent's clarifications by voice (answers are click-typed in v1).
- Any change to Handy's recording, VAD, or STT model management.

## 2. End-to-end pipeline

```
 mic ──▶ record (VAD) ──▶ STT (Handy, unchanged) ──▶ local cleanup (custom words,
                                                        │           fillers — existing)
                                                        ▼
                                                 raw transcript (*)
                                                        │
                                        S1-mini cleanup (local LLM, greedy)
                                                        ▼
                                              cleaned transcript (**)
                                                        │
                    ┌─────────────  Review Overlay  ─────────────┐
                    │ [raw textarea   (Send)] [cleaned (Send)]   │
                    │ agent selector · permissions · clarify     │
                    └──────────────┬─────────────┬──────────────┘
                     Send (agent)  │             │ classic mode: auto-paste
                                   ▼             ▼
                        AgentAdapter (Hermes /      clipboard.rs::paste
                        opencode / openai-compat)   (unchanged dispatch)
                                   │  events: deltas, permission requests,
                                   │          clarifications, final answer
                                   ▼
                        Overlay answer panel ──▶ TtsEngine (Qwen3-TTS 0.6B)
                                                   │
                                                   ▼
                                        playback on selected output device
```

(*) "Raw" = the pre-LLM transcript, i.e. what Handy already stores in `transcription_text` (after fuzzy custom-word + filler-word local cleanup). (**) "Cleaned" = S1-mini output, stored in `post_processed_text` like today's LLM post-processing.

## 3. Backend modules & seams

| Path                                     | Purpose                                                                                                                                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/agents/mod.rs`            | `AgentAdapter` trait, `AgentEvent`, `AgentCapabilities`, registry; `AgentsManager` managed in `lib.rs::initialize_core_logic` alongside Model/Transcription/Audio/History managers |
| `src-tauri/src/agents/opencode.rs`       | opencode server adapter (HTTP + SSE)                                                                                                                                               |
| `src-tauri/src/agents/hermes.rs`         | Hermes HTTP/WebSocket adapter                                                                                                                                                      |
| `src-tauri/src/agents/openai_compat.rs`  | Generic OpenAI `/v1/chat/completions` adapter (same transport style as `llm_client.rs`)                                                                                            |
| `src-tauri/src/llm_runtime.rs`           | Supervises on-demand `llama-server` sidecars (spawn/health/idle-stop, VRAM discipline)                                                                                             |
| `src-tauri/src/cleanup/`                 | S1-mini prompt builder + local inference (`llama-cpp-2`), plugged into the post-processing path                                                                                    |
| `src-tauri/src/tts/`                     | `TtsEngine` trait, `qwen3_tts_llamacpp.rs`, `kokoro_ort.rs`; `TtsManager`                                                                                                          |
| `src-tauri/src/commands/{agents,tts}.rs` | New Tauri commands (see §11)                                                                                                                                                       |
|                                          |                                                                                                                                                                                    |

**Where the work hooks into existing code** (verified):

- The transcription pipeline lives in **`actions.rs`** (`TranscribeAction::start/stop` → `tm.transcribe`/`finalize_stream` → `process_transcription_output` → history save → `utils::paste`). `transcription_coordinator.rs` is only the shortcut/PTT state machine — leave it alone.
- **`process_transcription_output`** (returns `ProcessedTranscription{final_text, post_processed_text, post_process_prompt}`) is the cleanup seam: the S1-mini engine is added there as a new local provider kind.
- **`clipboard.rs::paste`** (`PasteMethod` dispatch + `paste_tx` reliable paste + auto-submit/`ClipboardHandling` tails) stays as-is for classic dictation; agent-send is a *new output sink chosen in the review overlay*, sitting beside it.
- Reuse as-is: `managers/model.rs` + `model/download.rs` (downloads, sha256, resumable HTTP, progress), `catalog/`, `settings.rs` store, `managers/history.rs`, `overlay.rs` window, `audio_feedback.rs` (rodio), `audio_toolkit` output devices.

## 4. Transcript cleanup — S1-mini

### 4.1 Model facts

| | |
|---|---|
| Model | `superwhisper/s1-mini` (GGUF: `superwhisper/s1-mini-GGUF`, default quant **Q4_K_M**) |
| Base / size | Qwen3-0.6B fine-tune · 596M params · BF16 → ~0.4 GB at Q4_K_M |
| Task | ASR transcript → clean written text (fillers, self-corrections, punctuation, number/date/email formatting) |
| Limits | **English only** · input ≤ ~1000 tokens (chunk at sentence boundaries beyond) |
| License | Apache-2.0 **+ naming clause**: must keep the name *"S1-mini by Superwhisper"* (exact capitalization) wherever shown — settings UI, overlay, about page |

### 4.2 Runtime

- **Primary:** in-process via `llama-cpp-2` (safe Rust bindings; Qwen3 arch is fully supported by llama.cpp). GGUF downloaded through the catalog like any STT model.
- **Fallback:** bundled `llama-server` sidecar managed by `llm_runtime.rs` (`-hf superwhisper/s1-mini-GGUF:Q4_K_M --jinja --chat-template-kwargs '{"enable_thinking":false}' --temp 0`), selectable by setting if the in-process path proves fragile.
- Loaded on demand, unloaded after idle (mirrors the existing `model_unload_timeout` watcher) to keep VRAM free for STT/TTS/agent LLMs.

### 4.3 Prompt contract (must be exact — the model degrades otherwise)

- **System prompt (verbatim):** `You are a text normalizer for speech-to-text transcripts. The input begins with a control line specifying the styling, structure, and context settings; clean the transcript to match those settings and output only the cleaned text.`
- **User message:** control line, newline, raw transcript:
  `[Styling: {styling}] [Structure: {structure}] [Context: {context}]\n{raw}`
- **Allowed values:** `Styling`: casual | semi-casual | semi-formal | formal · `Structure`: prose | lists · `Context`: general | email.
- **Thinking flag:** assistant turn must start with the empty think block (`<|im_start|>assistant\n<think>\n\n</think>\n\n`), i.e. template applied with `enable_thinking: false`. Do **not** use `--reasoning-budget 0`. With `llama-cpp-2`, render the prompt manually with this literal prefix.
- **Sampling:** greedy (`temp 0`), `max_new_tokens = 1.3 × input_tokens + 32`; non-streaming (matches `llm_client.rs`'s `stream:false` convention).
- **Empty output is valid** (filler-only speech → empty cleaned text); pipeline must not treat it as an error (same "failure → keep raw" semantics as today's LLM post-processing).

### 4.4 Integration

- New provider kind **`local-s1mini`** in the existing post-processing system (`settings.rs`: `post_process_providers`, `post_process_enabled`, `post_process_provider_id`) rather than a parallel pipeline. It bypasses prompt selection (`post_process_selected_prompt_id` — which silently no-ops the remote-LLM path until a prompt is chosen) because the S1-mini system prompt is fixed by the model card; Styling/Structure/Context settings replace it.
- Keep the existing per-shortcut selection model: `transcribe` = plain, `transcribe_with_post_process` = with cleanup (`--toggle-post-process` untouched). The post-processing toggle keeps gating the second binding's registration.
- New settings: `cleanup_styling`, `cleanup_structure`, `cleanup_context` (defaults: semi-formal / prose / general), `cleanup_chunk_tokens: 1000`.
- Existing remote LLM post-processing providers (openai, anthropic, groq, …, `custom`/Ollama) remain fully supported; S1-mini is simply the local option.

### 4.5 Voice command tokens ("slash plan" → `/plan`)

Spoken command tokens are converted **deterministically in the pipeline — never delegated to S1-mini or STT** (S1-mini's normalization scope doesn't cover spoken symbol names, and STT output is inconsistent). Applied to the cleaned text *after* §4.3 and *before* the overlay renders the panels, so the raw/cleaned panels already show `/plan …` and the user sees (and can edit) exactly what will be sent:

- **Match rule:** start-anchored, case-insensitive `^\s*slash\s+([A-Za-z0-9_-]+)(\s+rest)?$` → `/{command} {rest}`. Start-anchoring prevents false positives when "slash" is spoken mid-sentence in normal dictation.
- **Command-only messages** (nothing after the command name, or a known agent command): routed as a *command*, not a prompt — opencode: `POST /api/session/{sessionID}/command`; Hermes: `type: "command"` in the §5.3 contract; openai-compat: sent as literal text.
- **Preamble is never prepended to commands** (§5.2) — a slash command must reach the agent verbatim — and history records that the send was a command.
- **Extensible map:** `voice_command_tokens: Vec<{ spoken: String, token: String }>` (default `slash → "/"`; entries like `at → "@"`, `dot → "."` can be added) with master toggle `voice_commands_enabled` (default on).
- M0 spike measures how STT and S1-mini actually render spoken "slash plan"; if S1-mini happens to handle it, this stage remains the guaranteed safety net.

## 5. Agent system

### 5.1 Abstraction

```rust
#[async_trait]
pub trait AgentAdapter: Send + Sync {
    fn id(&self) -> &str;
    fn name(&self) -> &str;
    fn capabilities(&self) -> AgentCapabilities;   // streaming, permissions, clarifications, sessions
    async fn send(&self, input: AgentMessage) -> Result<AgentSessionHandle>;
    async fn reply_permission(&self, req_id: &str, reply: PermissionReply) -> Result<()>; // Once | Always | Reject
    async fn answer_clarification(&self, session: &AgentSessionHandle, text: &str) -> Result<()>;
    async fn interrupt(&self, session: &AgentSessionHandle) -> Result<()>;
}

pub enum AgentEvent {
    Started { session_ref: String },
    Delta { text: String },                        // streaming answer
    ToolActivity { summary: String },              // status line in overlay
    PermissionRequested { id: String, action: String, resources: Vec<String>, message: String },
    Clarification { question: String, options: Vec<String> },
    Answer { text: String },                       // final answer → TTS
    Failed { error: String },
    Done,
}
```

Every adapter emits `AgentEvent`s through a broadcast channel; `AgentsManager` fans them out to the overlay window (typed events, §11). Adding an agent = one file implementing the trait + a registry entry.

### 5.2 Message envelope

Sent text = preamble + selected transcript:

> "This message went through speech-to-text and its answer will be spoken aloud through text-to-speech. Answer as if read out: be concise, avoid markdown, code blocks and lists unless asked."

- Stored as setting `agents_preamble_template` (per-agent override `agents[].preamble_override`), i18n'd default.
- Commands (§4.5) bypass the preamble entirely.
- The exact sent variant is recorded in history so we can audit what the agent received.

### 5.3 Built-in adapters (v1)

| Adapter | Transport | Permissions | Clarifications | Notes |
|---|---|---|---|---|
| **opencode** | Spawn/attach `opencode serve` (local install is v1.18.26); v2 HTTP API: `POST /api/session` → `POST /api/session/{id}/prompt` → SSE `GET /api/experimental/session/{sessionID}/log?follow=true`; `POST /api/session/{id}/interrupt`; `POST /api/session/{id}/command` (voice slash commands, §4.5) | Yes — surface `Permission.Request`; reply verbs **once / always / reject** | Yes — question/options arrive as session events | Settings: server URL or "auto-spawn", project directory, optional model/agent, session mode (new per message vs continue) |
| **Hermes** | HTTP + WebSocket against a localhost endpoint **to be implemented by Hermes** (contract below) | Yes (typed events) | Yes (typed events) | Settings: base URL, WS URL, bearer token |
| **OpenAI-compatible** | `POST {base_url}/v1/chat/completions`, `stream: true` | No (capability off) | No | Generic fallback; also covers "Hermes via local OpenAI-compatible endpoint" and cloud endpoints |

**Proposed Hermes contract** (to be confirmed/implemented Hermes-side):

```
POST {base}/api/v1/messages          body: { text, meta: { source: "voice", transcript_raw, transcript_clean } }
                                     → 202 { message_id }
WS   {base}/api/v1/events            server→client JSON events:
                                     { type: "delta"|"answer"|"permission_request"|"clarification"|"done",
                                       id, text?, options?, permission?: { action, resources, message } }
POST {base}/api/v1/permissions/{id}  body: { reply: "once"|"always"|"reject" }
```

If Hermes isn't reachable, the adapter emits `Failed` and the overlay shows a clear "Hermes offline" state; the OpenAI-compatible adapter keeps the feature usable meanwhile.
# VoiceControlledHermes — Clean Specifications

> These are the original specs. No agent should ever modify this unless explicitely asked.

Fork of Handy (Tauri 2.x, Rust + React/TS) at upstream commit `fbd4e15`. All file references verified against the current tree by a read-only code survey.*

**Decisions locked with Clement (2026-09-04):** Hermes gets an HTTP/WebSocket API · opencode driven via `opencode serve` + SDK · TTS runs Rust/llama.cpp-first (Kokoro native, Qwen3-TTS via GGUF) · classic paste-to-app dictation stays, with a raw-vs-cleaned option.

---

## 0. Summary

Keep Handy's transcription + global-shortcut flow untouched, and extend it into a voice-controlled agent front-end:

1. **Cleanup stage** — a local `S1-mini` (Superwhisper) 0.6B model normalizes the transcript into clean written text.
2. **Review overlay** — after transcription, the overlay shows the **raw** and the **cleaned** transcript side by side; both are editable; each has a **Send** button.
3. **Agent layer** — `Send` targets the chosen agent (**Hermes** or **opencode**), behind an abstract `AgentAdapter` trait so new agents are drop-in.
4. **Speech-aware framing** — every sent message carries a preamble: "This message went through speech-to-text and its answer will be spoken through text-to-speech. Be concise."
5. **Permissions in the overlay** — when an agent needs to run a command, the request is surfaced as an overlay card with Allow once / Always / Deny.
6. **Clarifications spoken** — if the agent asks a clarifying question or proposes options, the overlay shows them and TTS reads them aloud; options are clickable answers.
7. **Spoken answers** — the agent's final answer is synthesized with **Qwen3-TTS 0.6B** (default) and played back.
8. **Two new settings sections** — **Agents** and **TTS Models**. TTS (and cleanup) models are distributed through the same catalog/download machinery Handy uses for transcription models (extended), with entries for Kokoro, Qwen3-TTS 0.6B/1.7B, and VoxCPM2.

## 1. Goals / Non-goals

**Goals**
- G1: S1-mini cleanup runs locally, integrated into the existing post-processing pipeline (one pipeline, one history).
- G2: Overlay review mode: view + edit raw/cleaned, send either to the selected agent.
- G3: `AgentAdapter` abstraction; v1 ships Hermes (HTTP/WS), opencode (server API), and a generic OpenAI-compatible adapter.
- G4: Command-permission requests and clarifications are visible and actionable in the overlay.
- G5: Answers (and clarifications) are spoken via TTS; TTS engines are pluggable and catalog-distributed.
- G6: Classic dictation (auto-paste) keeps working; new setting chooses raw vs cleaned text for it.

**Non-goals (v1)**
- VoxCPM2 local runtime (Python-only today) — catalog entry ships, engine marked *future*.
- Speaking *into* the agent's clarifications by voice (answers are click-typed in v1).
- Any change to Handy's recording, VAD, or STT model management.

## 2. End-to-end pipeline

```
 mic ──▶ record (VAD) ──▶ STT (Handy, unchanged) ──▶ local cleanup (custom words,
                                                        │           fillers — existing)
                                                        ▼
                                                 raw transcript (*)
                                                        │
                                        S1-mini cleanup (local LLM, greedy)
                                                        ▼
                                              cleaned transcript (**)
                                                        │
                    ┌─────────────  Review Overlay  ─────────────┐
                    │ [raw textarea   (Send)] [cleaned (Send)]   │
                    │ agent selector · permissions · clarify     │
                    └──────────────┬─────────────┬──────────────┘
                     Send (agent)  │             │ classic mode: auto-paste
                                   ▼             ▼
                        AgentAdapter (Hermes /      clipboard.rs::paste
                        opencode / openai-compat)   (unchanged dispatch)
                                   │  events: deltas, permission requests,
                                   │          clarifications, final answer
                                   ▼
                        Overlay answer panel ──▶ TtsEngine (Qwen3-TTS 0.6B)
                                                   │
                                                   ▼
                                        playback on selected output device
```

(*) "Raw" = the pre-LLM transcript, i.e. what Handy already stores in `transcription_text` (after fuzzy custom-word + filler-word local cleanup). (**) "Cleaned" = S1-mini output, stored in `post_processed_text` like today's LLM post-processing.

## 3. Backend modules & seams

| Path                                     | Purpose                                                                                                                                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/agents/mod.rs`            | `AgentAdapter` trait, `AgentEvent`, `AgentCapabilities`, registry; `AgentsManager` managed in `lib.rs::initialize_core_logic` alongside Model/Transcription/Audio/History managers |
| `src-tauri/src/agents/opencode.rs`       | opencode server adapter (HTTP + SSE)                                                                                                                                               |
| `src-tauri/src/agents/hermes.rs`         | Hermes HTTP/WebSocket adapter                                                                                                                                                      |
| `src-tauri/src/agents/openai_compat.rs`  | Generic OpenAI `/v1/chat/completions` adapter (same transport style as `llm_client.rs`)                                                                                            |
| `src-tauri/src/llm_runtime.rs`           | Supervises on-demand `llama-server` sidecars (spawn/health/idle-stop, VRAM discipline)                                                                                             |
| `src-tauri/src/cleanup/`                 | S1-mini prompt builder + local inference (`llama-cpp-2`), plugged into the post-processing path                                                                                    |
| `src-tauri/src/tts/`                     | `TtsEngine` trait, `qwen3_tts_llamacpp.rs`, `kokoro_ort.rs`; `TtsManager`                                                                                                          |
| `src-tauri/src/commands/{agents,tts}.rs` | New Tauri commands (see §11)                                                                                                                                                       |
|                                          |                                                                                                                                                                                    |

**Where the work hooks into existing code** (verified):

- The transcription pipeline lives in **`actions.rs`** (`TranscribeAction::start/stop` → `tm.transcribe`/`finalize_stream` → `process_transcription_output` → history save → `utils::paste`). `transcription_coordinator.rs` is only the shortcut/PTT state machine — leave it alone.
- **`process_transcription_output`** (returns `ProcessedTranscription{final_text, post_processed_text, post_process_prompt}`) is the cleanup seam: the S1-mini engine is added there as a new local provider kind.
- **`clipboard.rs::paste`** (`PasteMethod` dispatch + `paste_tx` reliable paste + auto-submit/`ClipboardHandling` tails) stays as-is for classic dictation; agent-send is a *new output sink chosen in the review overlay*, sitting beside it.
- Reuse as-is: `managers/model.rs` + `model/download.rs` (downloads, sha256, resumable HTTP, progress), `catalog/`, `settings.rs` store, `managers/history.rs`, `overlay.rs` window, `audio_feedback.rs` (rodio), `audio_toolkit` output devices.

## 4. Transcript cleanup — S1-mini

### 4.1 Model facts

| | |
|---|---|
| Model | `superwhisper/s1-mini` (GGUF: `superwhisper/s1-mini-GGUF`, default quant **Q4_K_M**) |
| Base / size | Qwen3-0.6B fine-tune · 596M params · BF16 → ~0.4 GB at Q4_K_M |
| Task | ASR transcript → clean written text (fillers, self-corrections, punctuation, number/date/email formatting) |
| Limits | **English only** · input ≤ ~1000 tokens (chunk at sentence boundaries beyond) |
| License | Apache-2.0 **+ naming clause**: must keep the name *"S1-mini by Superwhisper"* (exact capitalization) wherever shown — settings UI, overlay, about page |

### 4.2 Runtime

- **Primary:** in-process via `llama-cpp-2` (safe Rust bindings; Qwen3 arch is fully supported by llama.cpp). GGUF downloaded through the catalog like any STT model.
- **Fallback:** bundled `llama-server` sidecar managed by `llm_runtime.rs` (`-hf superwhisper/s1-mini-GGUF:Q4_K_M --jinja --chat-template-kwargs '{"enable_thinking":false}' --temp 0`), selectable by setting if the in-process path proves fragile.
- Loaded on demand, unloaded after idle (mirrors the existing `model_unload_timeout` watcher) to keep VRAM free for STT/TTS/agent LLMs.

### 4.3 Prompt contract (must be exact — the model degrades otherwise)

- **System prompt (verbatim):** `You are a text normalizer for speech-to-text transcripts. The input begins with a control line specifying the styling, structure, and context settings; clean the transcript to match those settings and output only the cleaned text.`
- **User message:** control line, newline, raw transcript:
  `[Styling: {styling}] [Structure: {structure}] [Context: {context}]\n{raw}`
- **Allowed values:** `Styling`: casual | semi-casual | semi-formal | formal · `Structure`: prose | lists · `Context`: general | email.
- **Thinking flag:** assistant turn must start with the empty think block (`<|im_start|>assistant\n<think>\n\n</think>\n\n`), i.e. template applied with `enable_thinking: false`. Do **not** use `--reasoning-budget 0`. With `llama-cpp-2`, render the prompt manually with this literal prefix.
- **Sampling:** greedy (`temp 0`), `max_new_tokens = 1.3 × input_tokens + 32`; non-streaming (matches `llm_client.rs`'s `stream:false` convention).
- **Empty output is valid** (filler-only speech → empty cleaned text); pipeline must not treat it as an error (same "failure → keep raw" semantics as today's LLM post-processing).

### 4.4 Integration

- New provider kind **`local-s1mini`** in the existing post-processing system (`settings.rs`: `post_process_providers`, `post_process_enabled`, `post_process_provider_id`) rather than a parallel pipeline. It bypasses prompt selection (`post_process_selected_prompt_id` — which silently no-ops the remote-LLM path until a prompt is chosen) because the S1-mini system prompt is fixed by the model card; Styling/Structure/Context settings replace it.
- Keep the existing per-shortcut selection model: `transcribe` = plain, `transcribe_with_post_process` = with cleanup (`--toggle-post-process` untouched). The post-processing toggle keeps gating the second binding's registration.
- New settings: `cleanup_styling`, `cleanup_structure`, `cleanup_context` (defaults: semi-formal / prose / general), `cleanup_chunk_tokens: 1000`.
- Existing remote LLM post-processing providers (openai, anthropic, groq, …, `custom`/Ollama) remain fully supported; S1-mini is simply the local option.

### 4.5 Voice command tokens ("slash plan" → `/plan`)

Spoken command tokens are converted **deterministically in the pipeline — never delegated to S1-mini or STT** (S1-mini's normalization scope doesn't cover spoken symbol names, and STT output is inconsistent). Applied to the cleaned text *after* §4.3 and *before* the overlay renders the panels, so the raw/cleaned panels already show `/plan …` and the user sees (and can edit) exactly what will be sent:

- **Match rule:** start-anchored, case-insensitive `^\s*slash\s+([A-Za-z0-9_-]+)(\s+rest)?$` → `/{command} {rest}`. Start-anchoring prevents false positives when "slash" is spoken mid-sentence in normal dictation.
- **Command-only messages** (nothing after the command name, or a known agent command): routed as a *command*, not a prompt — opencode: `POST /api/session/{sessionID}/command`; Hermes: `type: "command"` in the §5.3 contract; openai-compat: sent as literal text.
- **Preamble is never prepended to commands** (§5.2) — a slash command must reach the agent verbatim — and history records that the send was a command.
- **Extensible map:** `voice_command_tokens: Vec<{ spoken: String, token: String }>` (default `slash → "/"`; entries like `at → "@"`, `dot → "."` can be added) with master toggle `voice_commands_enabled` (default on).
- M0 spike measures how STT and S1-mini actually render spoken "slash plan"; if S1-mini happens to handle it, this stage remains the guaranteed safety net.

## 5. Agent system

### 5.1 Abstraction

```rust
#[async_trait]
pub trait AgentAdapter: Send + Sync {
    fn id(&self) -> &str;
    fn name(&self) -> &str;
    fn capabilities(&self) -> AgentCapabilities;   // streaming, permissions, clarifications, sessions
    async fn send(&self, input: AgentMessage) -> Result<AgentSessionHandle>;
    async fn reply_permission(&self, req_id: &str, reply: PermissionReply) -> Result<()>; // Once | Always | Reject
    async fn answer_clarification(&self, session: &AgentSessionHandle, text: &str) -> Result<()>;
    async fn interrupt(&self, session: &AgentSessionHandle) -> Result<()>;
}

pub enum AgentEvent {
    Started { session_ref: String },
    Delta { text: String },                        // streaming answer
    ToolActivity { summary: String },              // status line in overlay
    PermissionRequested { id: String, action: String, resources: Vec<String>, message: String },
    Clarification { question: String, options: Vec<String> },
    Answer { text: String },                       // final answer → TTS
    Failed { error: String },
    Done,
}
```

Every adapter emits `AgentEvent`s through a broadcast channel; `AgentsManager` fans them out to the overlay window (typed events, §11). Adding an agent = one file implementing the trait + a registry entry.

### 5.2 Message envelope

Sent text = preamble + selected transcript:

> "This message went through speech-to-text and its answer will be spoken aloud through text-to-speech. Answer as if read out: be concise, avoid markdown, code blocks and lists unless asked."

- Stored as setting `agents_preamble_template` (per-agent override `agents[].preamble_override`), i18n'd default.
- Commands (§4.5) bypass the preamble entirely.
- The exact sent variant is recorded in history so we can audit what the agent received.

### 5.3 Built-in adapters (v1)

| Adapter | Transport | Permissions | Clarifications | Notes |
|---|---|---|---|---|
| **opencode** | Spawn/attach `opencode serve` (local install is v1.18.26); v2 HTTP API: `POST /api/session` → `POST /api/session/{id}/prompt` → SSE `GET /api/experimental/session/{sessionID}/log?follow=true`; `POST /api/session/{id}/interrupt`; `POST /api/session/{id}/command` (voice slash commands, §4.5) | Yes — surface `Permission.Request`; reply verbs **once / always / reject** | Yes — question/options arrive as session events | Settings: server URL or "auto-spawn", project directory, optional model/agent, session mode (new per message vs continue) |
| **Hermes** | HTTP + WebSocket against a localhost endpoint **to be implemented by Hermes** (contract below) | Yes (typed events) | Yes (typed events) | Settings: base URL, WS URL, bearer token |
| **OpenAI-compatible** | `POST {base_url}/v1/chat/completions`, `stream: true` | No (capability off) | No | Generic fallback; also covers "Hermes via local OpenAI-compatible endpoint" and cloud endpoints |

**Proposed Hermes contract** (to be confirmed/implemented Hermes-side):

```
POST {base}/api/v1/messages          body: { text, meta: { source: "voice", transcript_raw, transcript_clean } }
                                     → 202 { message_id }
WS   {base}/api/v1/events            server→client JSON events:
                                     { type: "delta"|"answer"|"permission_request"|"clarification"|"done",
                                       id, text?, options?, permission?: { action, resources, message } }
POST {base}/api/v1/permissions/{id}  body: { reply: "once"|"always"|"reject" }
```

If Hermes isn't reachable, the adapter emits `Failed` and the overlay shows a clear "Hermes offline" state; the OpenAI-compatible adapter keeps the feature usable meanwhile.

### 5.4 Permission surfacing

- `PermissionRequested` → overlay card: agent badge, action/command, affected resources, message, buttons **Allow once / Always allow / Deny** (+ keyboard `A`/`S`/`D`), auto-deny after `agents_permission_timeout_secs` (default 120).
- "Always" maps to the adapter's persistent reply (`always` for opencode; Hermes contract equivalent).

### 5.5 Clarifications

- `Clarification { question, options }` → overlay card with clickable option chips (sends the option text back via `answer_clarification`) and a free-text field.
- If `tts_speak_clarifications` is on, TTS speaks the question then each option ("Option 1: …") after current playback finishes; clicking an option cancels remaining TTS.

## 6. Overlay v2 (review mode)

### 6.1 States & windowing

`recording → streaming → transcribing → processing` exist today via the `show-overlay` string payload; add **`review`**, **`agent`**, **`answer`**. Constraints verified in code: the `recording_overlay` window is built `focusable(false)`, non-macOS transparent/always-on-top (GTK layer-shell `Layer::Overlay` on Linux, `HANDY_NO_GTK_LAYER_SHELL=1` escape hatch), macOS NSPanel via `tauri-nspanel` (floating, **non-activating**). For review mode:

- In `review`/`agent`/`answer` states the window must become **focusable and key-accepting**: flip `focusable` + call focus (Windows/Linux), layer-shell `KeyboardMode` interactive (Wayland), and an activating NSPanel variant (macOS) — this is the main platform risk, spike it in M0.
- Review mode is gated by its own setting (`overlay_review_mode`), independent of `overlay_style` (which defaults to `None` on Linux and would otherwise never show the overlay).
- Positioning/reposition rules (monitor-with-cursor, top/bottom, main-thread hop) are reused; review needs a larger window than the 256×46 pill / 400×120 streaming panel — extend `overlay_dimensions()` per state.

### 6.2 Layout

```
┌────────────────────────────────────────────────────────┐
│ ● Review            agent: [Hermes ▾]      [Settings]  │
│ ┌─ Raw ────────────────────┐ ┌─ Cleaned ──────────────┐ │
│ │ (editable textarea)      │ │ (editable textarea)    │ │
│ │            [Send →] [⧉]  │ │        [Send →] [⧉]    │ │
│ └──────────────────────────┘ └────────────────────────┘ │
│ (permissions card)  (clarification card)                │
│ ┌─ Answer ───────────────────────────────────────────┐  │
│ │ streaming text…                    [▶ replay][■]   │  │
│ └────────────────────────────────────────────────────┘  │
│ Esc = cancel · Ctrl+Enter = send cleaned                │
└────────────────────────────────────────────────────────┘
```

- Both panels editable; edits apply only to what is sent. `Send` sends **that panel's text** to the selected agent.
- Secondary actions: copy to clipboard; in classic mode (review disabled) the flow stays exactly as today (auto-paste; new setting `paste_text_source: raw | cleaned` selects which text `process_transcription_output` hands to `paste()`).
- Answer panel: streams `Delta`s; on `Answer` plays TTS (respecting `tts_auto_play`); replay/stop buttons; mute keeps text.
- Every send is captured to history (raw, cleaned, sent-variant, agent id, session ref, answer).
- The only current overlay back-channel is `commands.cancelOperation()` — the review UI adds the new commands from §11; cancel generations in `actions.rs`/`AudioRecordingManager` are reused so `--cancel`/Esc still abort everything.

## 7. TTS subsystem

### 7.1 Abstraction

```rust
#[async_trait]
pub trait TtsEngine: Send + Sync {
    fn id(&self) -> &str;
    async fn synthesize(&self, text: &str, voice: &TtsVoice) -> Result<TtsAudio>; // wav path or PCM buffer
    fn supports_streaming(&self) -> bool;
    async fn unload(&self);
}
```

- Playback reuses the rodio + selected-output-device resolution from `audio_feedback.rs` (rodio accepts any `Read + Seek`, so a TTS WAV buffer plays via `Cursor<Vec<u8>>`). Use its own sink/stream — today's helpers `sleep_until_end` per sink and would block chimes.
- TTS text is pre-processed: strip markdown fences/links (setting `tts_strip_markdown`, default on) so code blocks are not read aloud.

### 7.2 Engines

| Engine | Model | Runtime | Status |
|---|---|---|---|
| `qwen3-tts` (default) | `Qwen/Qwen3-TTS-12Hz-0.6B-Base` | GGUF via bundled `llama-server` (llama.cpp serves `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF:Q4_K_M`; **no 0.6B GGUF published yet**) | 1.7B works today; 0.6B entry ships flagged "GGUF pending", falls back to 1.7B until published |
| `kokoro` | `hexgrad/Kokoro-82M-v1.0` ONNX | Native Rust via `ort` (Handy already uses ONNX Runtime through transcribe-rs) + pure-Rust phonemizer (kokoroxide/any-tts pattern) | lightest, CPU-friendly, multi-voice |
| `voxcpm2` | `openbmb/VoxCPM2` (2B) | Python `voxcpm` package — **out of scope v1** | catalog entry marked future |

- Voice handling: Qwen3-TTS **Base** clones from a reference clip (≥3 s + transcript) — settings expose a reference-audio file picker; Kokoro exposes its voice list.
- `llm_runtime.rs` owns the llama-server lifecycle (start on first synthesis, idle-stop, health check), shared with the S1-mini fallback sidecar.

### 7.3 What gets spoken

- Final `Answer` (when TTS enabled) and `Clarification` cards (opt-in, §5.5). Tool activity and deltas are never spoken.

## 8. Settings — new sections

Plumbing follows the verified pattern end-to-end: fields on `AppSettings` (`settings.rs`, with `#[serde(default)]` + default fns + `settings_schema_version` bump) → `change_*_setting` commands registered in `collect_commands!` → `settingUpdaters` in `src/stores/settingsStore.ts` → UI components under `src/components/settings/{agents,tts}/` wired into `SECTIONS_CONFIG` in `src/components/Sidebar.tsx` (new nav entries "Agents", "TTS Models") → i18n keys in `src/i18n/locales/en/translation.json`.

**Agents**
| Field | Type | Default |
|---|---|---|
| `agents` | list: `{ id, enabled, kind: hermes|opencode|openai_compat, name, base_url, ws_url?, token?, project_dir?, model?, session_mode: per_message|continue, preamble_override? }` | Hermes (disabled), opencode (disabled) |
| `agents_default_id` | string | first enabled |
| `agents_permission_timeout_secs` | int | 120 |
| `agents_preamble_template` | string | i18n default (§5.2) |

**TTS Models**
| Field | Type | Default |
|---|---|---|
| `tts_enabled` | bool | true |
| `tts_engine_id` | string | `qwen3-tts` |
| `tts_model_id` | catalog id | `qwen3-tts-0.6b-base` |
| `tts_voice` / `tts_ref_audio_path` / `tts_ref_audio_text` | string | engine defaults |
| `tts_auto_play` | bool | true |
| `tts_speak_clarifications` | bool | true |
| `tts_strip_markdown` | bool | true |
| `tts_output_device` | reuse existing OutputDeviceSelector | default |

**General (review flow)**
`overlay_review_mode: auto | interactive` (default `interactive`), `paste_text_source: raw | cleaned` (default `cleaned`), plus §4.4 cleanup knobs.

## 9. Model catalog & downloads

Handy ships 69 transcription models in `src-tauri/src/catalog/catalog.json` (`catalog_version: 2`, mirror `blob.handy.computer`, per-file `sha256` as the trust anchor, generated by `scripts/gen_catalog.py`). Catalog entries currently map 1:1 to `EngineType::TranscribeCpp` GGUFs; downloads go through `ModelManager` (`ModelSource::HuggingFace` via hf-hub into the shared HF cache, revision-pinned, or `ModelSource::Url` with sha256 + resumable HTTP + mirror fallback). Extend, don't fork:

- **Catalog v3:** each model gains `task: "stt" (default) | "cleanup" | "tts"`; `catalog/mod.rs` maps task → new `EngineType` variants (e.g. `LlamaCleanup`, `TtsQwen3`, `TtsKokoro`) instead of hardcoding `TranscribeCpp`. Existing STT entries unchanged.
- **New entries:** `s1-mini` (cleanup, Q4_K_M default, naming-clause note), `kokoro-82m-v1.0` (tts), `qwen3-tts-0.6b-base` (tts, flagged pending), `qwen3-tts-1.7b-base` (tts), `voxcpm2` (tts, future). Sources are public HF repos (`superwhisper/s1-mini-GGUF`, `hexgrad/Kokoro-82M-v1.0-onnx`, `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF`, …) with pinned revisions + sha256 — no fork-owned mirror needed.
- The existing download manager (progress/verify/cancel events, HF cache discovery, `models_dir` fallback) is reused untouched; the model selector gets a task filter and the **TTS Models** settings page lists `task: "tts"` entries with the same download UX.

## 10. History

New SQLite migration in `managers/history.rs` (rusqlite_migration): `agent_id TEXT`, `agent_session_ref TEXT`, `agent_sent_text TEXT`, `agent_response TEXT`, `tts_played BOOLEAN DEFAULT 0`. `HistoryEntry` mirrors them; updates flow out through the existing typed `history-update-payload` event. History UI (optional, M4): badge on agent-sent entries, expandable answer.

## 11. Tauri commands & events (new)

**Commands:** `agents_list`, `agents_send(text, source: raw|cleaned)`, `agents_reply_permission(id, reply)`, `agents_answer_clarification(session_ref, text)`, `agents_interrupt`, `tts_synthesize(text)`, `tts_stop`, `cleanup_preview(raw)` (re-run cleanup on edited raw).

**Events:** follow the two existing conventions — plain kebab-case strings (`show-overlay`, `mic-level`) targeted to the overlay window with `emit_to("recording_overlay", …)`, and typed `tauri_specta::Event` structs registered in `collect_events!` (like `StreamTextEvent`, `HistoryUpdatePayload`). New typed events: `agent-event` (tagged `AgentEvent` envelope), `tts-state-event`. New overlay-targeted strings: `show-overlay` gains the new states; `overlay-permission-request` and `overlay-clarification` payloads reuse the typed structs.

## 12. i18n

Every new string lives in `src/i18n/locales/en/translation.json` under `agents.*`, `tts.*`, `overlay.review.*` (ESLint forbids hardcoded JSX strings). User-visible model name must keep the exact string *"S1-mini by Superwhisper"*.

## 13. Testing & acceptance

- **Unit:** prompt builder reproduces the exact S1-mini template (control line + empty think block, golden strings); catalog v3 parsing; `AgentEvent` serialization; adapter mocking with wiremock (permission round-trip, clarification, SSE delta parsing).
- **Integration:** opencode adapter against a real `opencode serve` (manual/nightly, version-pinned); Hermes contract test server.
- **E2E (Playwright):** settings pages render, catalog downloads progress, review overlay toggles.
- **Acceptance walkthrough:** hold shortcut → speak → overlay shows raw + cleaned → edit cleaned → Send → opencode streams → permission card appears → Allow once → answer streams → TTS plays; repeat with Hermes; classic auto-paste unchanged with both `raw`/`cleaned` sources; Kokoro ↔ Qwen3-TTS switch works; offline agent shows a graceful failure state.

## 14. Implementation phases

| Phase | Scope | Exit criteria |
|---|---|---|
| **M0 — spikes** | (a) `llama-cpp-2` runs s1-mini GGUF with exact prompt → golden outputs; (b) `llama-server` + Qwen3-TTS 1.7B GGUF produces audio incl. voice-clone ref flow; (c) opencode v2 server: create/prompt/SSE/permission reply; (d) overlay review window keyboard focus on Wayland, X11, macOS NSPanel | each spike scripted and reproducible |
| **M1 — cleanup + review** | catalog v3, S1-mini provider, settings, overlay raw/cleaned panels, paste sources | dictation with cleanup end-to-end; review overlay editable & sendable |
| **M2 — agents** | trait + registry, opencode + OpenAI-compat adapters, permission & clarify cards, preamble | acceptance walkthrough with opencode |
| **M3 — TTS** | `TtsEngine`, Qwen3-TTS via `llm_runtime`, Kokoro via ort, answer/clarify playback | spoken answers on both engines |
| **M4 — Hermes + polish** | Hermes adapter once contract is implemented, history columns, i18n, docs | full walkthrough incl. Hermes |

## 15. Risks & open questions

1. **Qwen3-TTS 0.6B GGUF** not yet published (only 1.7B-Base from ggml-org) — ship 1.7B + fallback, watch for 0.6B quants; alternative: vendor `predict-woo/qwen3-tts.cpp` (C++/GGML, no Python).
2. **llama.cpp TTS voice-clone flags** for Qwen3-TTS Base (reference-audio input via server) need M0 validation.
3. **Hermes API doesn't exist yet** — contract in §5.3 must be implemented Hermes-side; the OpenAI-compatible adapter is the bridge until then.
4. **Overlay keyboard focus** (Wayland layer-shell, macOS non-activating NSPanel) — the riskiest platform change; M0 spike.
5. **VRAM contention** (STT + S1-mini + TTS # VoiceControlledHermes — Clean Specifications

> These are the original specs. No agent should ever modify this unless explicitely asked.

Fork of Handy (Tauri 2.x, Rust + React/TS) at upstream commit `fbd4e15`. All file references verified against the current tree by a read-only code survey.*

**Decisions locked with Clement (2026-09-04):** Hermes gets an HTTP/WebSocket API · opencode driven via `opencode serve` + SDK · TTS runs Rust/llama.cpp-first (Kokoro native, Qwen3-TTS via GGUF) · classic paste-to-app dictation stays, with a raw-vs-cleaned option.

---

## 0. Summary

Keep Handy's transcription + global-shortcut flow untouched, and extend it into a voice-controlled agent front-end:

1. **Cleanup stage** — a local `S1-mini` (Superwhisper) 0.6B model normalizes the transcript into clean written text.
2. **Review overlay** — after transcription, the overlay shows the **raw** and the **cleaned** transcript side by side; both are editable; each has a **Send** button.
3. **Agent layer** — `Send` targets the chosen agent (**Hermes** or **opencode**), behind an abstract `AgentAdapter` trait so new agents are drop-in.
4. **Speech-aware framing** — every sent message carries a preamble: "This message went through speech-to-text and its answer will be spoken through text-to-speech. Be concise."
5. **Permissions in the overlay** — when an agent needs to run a command, the request is surfaced as an overlay card with Allow once / Always / Deny.
6. **Clarifications spoken** — if the agent asks a clarifying question or proposes options, the overlay shows them and TTS reads them aloud; options are clickable answers.
7. **Spoken answers** — the agent's final answer is synthesized with **Qwen3-TTS 0.6B** (default) and played back.
8. **Two new settings sections** — **Agents** and **TTS Models**. TTS (and cleanup) models are distributed through the same catalog/download machinery Handy uses for transcription models (extended), with entries for Kokoro, Qwen3-TTS 0.6B/1.7B, and VoxCPM2.

## 1. Goals / Non-goals

**Goals**
- G1: S1-mini cleanup runs locally, integrated into the existing post-processing pipeline (one pipeline, one history).
- G2: Overlay review mode: view + edit raw/cleaned, send either to the selected agent.
- G3: `AgentAdapter` abstraction; v1 ships Hermes (HTTP/WS), opencode (server API), and a generic OpenAI-compatible adapter.
- G4: Command-permission requests and clarifications are visible and actionable in the overlay.
- G5: Answers (and clarifications) are spoken via TTS; TTS engines are pluggable and catalog-distributed.
- G6: Classic dictation (auto-paste) keeps working; new setting chooses raw vs cleaned text for it.

**Non-goals (v1)**
- VoxCPM2 local runtime (Python-only today) — catalog entry ships, engine marked *future*.
- Speaking *into* the agent's clarifications by voice (answers are click-typed in v1).
- Any change to Handy's recording, VAD, or STT model management.

## 2. End-to-end pipeline

```
 mic ──▶ record (VAD) ──▶ STT (Handy, unchanged) ──▶ local cleanup (custom words,
                                                        │           fillers — existing)
                                                        ▼
                                                 raw transcript (*)
                                                        │
                                        S1-mini cleanup (local LLM, greedy)
                                                        ▼
                                              cleaned transcript (**)
                                                        │
                    ┌─────────────  Review Overlay  ─────────────┐
                    │ [raw textarea   (Send)] [cleaned (Send)]   │
                    │ agent selector · permissions · clarify     │
                    └──────────────┬─────────────┬──────────────┘
                     Send (agent)  │             │ classic mode: auto-paste
                                   ▼             ▼
                        AgentAdapter (Hermes /      clipboard.rs::paste
                        opencode / openai-compat)   (unchanged dispatch)
                                   │  events: deltas, permission requests,
                                   │          clarifications, final answer
                                   ▼
                        Overlay answer panel ──▶ TtsEngine (Qwen3-TTS 0.6B)
                                                   │
                                                   ▼
                                        playback on selected output device
```

(*) "Raw" = the pre-LLM transcript, i.e. what Handy already stores in `transcription_text` (after fuzzy custom-word + filler-word local cleanup). (**) "Cleaned" = S1-mini output, stored in `post_processed_text` like today's LLM post-processing.

## 3. Backend modules & seams

| Path                                     | Purpose                                                                                                                                                                            |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src-tauri/src/agents/mod.rs`            | `AgentAdapter` trait, `AgentEvent`, `AgentCapabilities`, registry; `AgentsManager` managed in `lib.rs::initialize_core_logic` alongside Model/Transcription/Audio/History managers |
| `src-tauri/src/agents/opencode.rs`       | opencode server adapter (HTTP + SSE)                                                                                                                                               |
| `src-tauri/src/agents/hermes.rs`         | Hermes HTTP/WebSocket adapter                                                                                                                                                      |
| `src-tauri/src/agents/openai_compat.rs`  | Generic OpenAI `/v1/chat/completions` adapter (same transport style as `llm_client.rs`)                                                                                            |
| `src-tauri/src/llm_runtime.rs`           | Supervises on-demand `llama-server` sidecars (spawn/health/idle-stop, VRAM discipline)                                                                                             |
| `src-tauri/src/cleanup/`                 | S1-mini prompt builder + local inference (`llama-cpp-2`), plugged into the post-processing path                                                                                    |
| `src-tauri/src/tts/`                     | `TtsEngine` trait, `qwen3_tts_llamacpp.rs`, `kokoro_ort.rs`; `TtsManager`                                                                                                          |
| `src-tauri/src/commands/{agents,tts}.rs` | New Tauri commands (see §11)                                                                                                                                                       |
|                                          |                                                                                                                                                                                    |

**Where the work hooks into existing code** (verified):

- The transcription pipeline lives in **`actions.rs`** (`TranscribeAction::start/stop` → `tm.transcribe`/`finalize_stream` → `process_transcription_output` → history save → `utils::paste`). `transcription_coordinator.rs` is only the shortcut/PTT state machine — leave it alone.
- **`process_transcription_output`** (returns `ProcessedTranscription{final_text, post_processed_text, post_process_prompt}`) is the cleanup seam: the S1-mini engine is added there as a new local provider kind.
- **`clipboard.rs::paste`** (`PasteMethod` dispatch + `paste_tx` reliable paste + auto-submit/`ClipboardHandling` tails) stays as-is for classic dictation; agent-send is a *new output sink chosen in the review overlay*, sitting beside it.
- Reuse as-is: `managers/model.rs` + `model/download.rs` (downloads, sha256, resumable HTTP, progress), `catalog/`, `settings.rs` store, `managers/history.rs`, `overlay.rs` window, `audio_feedback.rs` (rodio), `audio_toolkit` output devices.

## 4. Transcript cleanup — S1-mini

### 4.1 Model facts

| | |
|---|---|
| Model | `superwhisper/s1-mini` (GGUF: `superwhisper/s1-mini-GGUF`, default quant **Q4_K_M**) |
| Base / size | Qwen3-0.6B fine-tune · 596M params · BF16 → ~0.4 GB at Q4_K_M |
| Task | ASR transcript → clean written text (fillers, self-corrections, punctuation, number/date/email formatting) |
| Limits | **English only** · input ≤ ~1000 tokens (chunk at sentence boundaries beyond) |
| License | Apache-2.0 **+ naming clause**: must keep the name *"S1-mini by Superwhisper"* (exact capitalization) wherever shown — settings UI, overlay, about page |

### 4.2 Runtime

- **Primary:** in-process via `llama-cpp-2` (safe Rust bindings; Qwen3 arch is fully supported by llama.cpp). GGUF downloaded through the catalog like any STT model.
- **Fallback:** bundled `llama-server` sidecar managed by `llm_runtime.rs` (`-hf superwhisper/s1-mini-GGUF:Q4_K_M --jinja --chat-template-kwargs '{"enable_thinking":false}' --temp 0`), selectable by setting if the in-process path proves fragile.
- Loaded on demand, unloaded after idle (mirrors the existing `model_unload_timeout` watcher) to keep VRAM free for STT/TTS/agent LLMs.

### 4.3 Prompt contract (must be exact — the model degrades otherwise)

- **System prompt (verbatim):** `You are a text normalizer for speech-to-text transcripts. The input begins with a control line specifying the styling, structure, and context settings; clean the transcript to match those settings and output only the cleaned text.`
- **User message:** control line, newline, raw transcript:
  `[Styling: {styling}] [Structure: {structure}] [Context: {context}]\n{raw}`
- **Allowed values:** `Styling`: casual | semi-casual | semi-formal | formal · `Structure`: prose | lists · `Context`: general | email.
- **Thinking flag:** assistant turn must start with the empty think block (`<|im_start|>assistant\n<think>\n\n</think>\n\n`), i.e. template applied with `enable_thinking: false`. Do **not** use `--reasoning-budget 0`. With `llama-cpp-2`, render the prompt manually with this literal prefix.
- **Sampling:** greedy (`temp 0`), `max_new_tokens = 1.3 × input_tokens + 32`; non-streaming (matches `llm_client.rs`'s `stream:false` convention).
- **Empty output is valid** (filler-only speech → empty cleaned text); pipeline must not treat it as an error (same "failure → keep raw" semantics as today's LLM post-processing).

### 4.4 Integration

- New provider kind **`local-s1mini`** in the existing post-processing system (`settings.rs`: `post_process_providers`, `post_process_enabled`, `post_process_provider_id`) rather than a parallel pipeline. It bypasses prompt selection (`post_process_selected_prompt_id` — which silently no-ops the remote-LLM path until a prompt is chosen) because the S1-mini system prompt is fixed by the model card; Styling/Structure/Context settings replace it.
- Keep the existing per-shortcut selection model: `transcribe` = plain, `transcribe_with_post_process` = with cleanup (`--toggle-post-process` untouched). The post-processing toggle keeps gating the second binding's registration.
- New settings: `cleanup_styling`, `cleanup_structure`, `cleanup_context` (defaults: semi-formal / prose / general), `cleanup_chunk_tokens: 1000`.
- Existing remote LLM post-processing providers (openai, anthropic, groq, …, `custom`/Ollama) remain fully supported; S1-mini is simply the local option.

### 4.5 Voice command tokens ("slash plan" → `/plan`)

Spoken command tokens are converted **deterministically in the pipeline — never delegated to S1-mini or STT** (S1-mini's normalization scope doesn't cover spoken symbol names, and STT output is inconsistent). Applied to the cleaned text *after* §4.3 and *before* the overlay renders the panels, so the raw/cleaned panels already show `/plan …` and the user sees (and can edit) exactly what will be sent:

- **Match rule:** start-anchored, case-insensitive `^\s*slash\s+([A-Za-z0-9_-]+)(\s+rest)?$` → `/{command} {rest}`. Start-anchoring prevents false positives when "slash" is spoken mid-sentence in normal dictation.
- **Command-only messages** (nothing after the command name, or a known agent command): routed as a *command*, not a prompt — opencode: `POST /api/session/{sessionID}/command`; Hermes: `type: "command"` in the §5.3 contract; openai-compat: sent as literal text.
- **Preamble is never prepended to commands** (§5.2) — a slash command must reach the agent verbatim — and history records that the send was a command.
- **Extensible map:** `voice_command_tokens: Vec<{ spoken: String, token: String }>` (default `slash → "/"`; entries like `at → "@"`, `dot → "."` can be added) with master toggle `voice_commands_enabled` (default on).
- M0 spike measures how STT and S1-mini actually render spoken "slash plan"; if S1-mini happens to handle it, this stage remains the guaranteed safety net.

## 5. Agent system

### 5.1 Abstraction

```rust
#[async_trait]
pub trait AgentAdapter: Send + Sync {
    fn id(&self) -> &str;
    fn name(&self) -> &str;
    fn capabilities(&self) -> AgentCapabilities;   // streaming, permissions, clarifications, sessions
    async fn send(&self, input: AgentMessage) -> Result<AgentSessionHandle>;
    async fn reply_permission(&self, req_id: &str, reply: PermissionReply) -> Result<()>; // Once | Always | Reject
    async fn answer_clarification(&self, session: &AgentSessionHandle, text: &str) -> Result<()>;
    async fn interrupt(&self, session: &AgentSessionHandle) -> Result<()>;
}

pub enum AgentEvent {
    Started { session_ref: String },
    Delta { text: String },                        // streaming answer
    ToolActivity { summary: String },              // status line in overlay
    PermissionRequested { id: String, action: String, resources: Vec<String>, message: String },
    Clarification { question: String, options: Vec<String> },
    Answer { text: String },                       // final answer → TTS
    Failed { error: String },
    Done,
}
```

Every adapter emits `AgentEvent`s through a broadcast channel; `AgentsManager` fans them out to the overlay window (typed events, §11). Adding an agent = one file implementing the trait + a registry entry.

### 5.2 Message envelope

Sent text = preamble + selected transcript:

> "This message went through speech-to-text and its answer will be spoken aloud through text-to-speech. Answer as if read out: be concise, avoid markdown, code blocks and lists unless asked."

- Stored as setting `agents_preamble_template` (per-agent override `agents[].preamble_override`), i18n'd default.
- Commands (§4.5) bypass the preamble entirely.
- The exact sent variant is recorded in history so we can audit what the agent received.

### 5.3 Built-in adapters (v1)

| Adapter | Transport | Permissions | Clarifications | Notes |
|---|---|---|---|---|
| **opencode** | Spawn/attach `opencode serve` (local install is v1.18.26); v2 HTTP API: `POST /api/session` → `POST /api/session/{id}/prompt` → SSE `GET /api/experimental/session/{sessionID}/log?follow=true`; `POST /api/session/{id}/interrupt`; `POST /api/session/{id}/command` (voice slash commands, §4.5) | Yes — surface `Permission.Request`; reply verbs **once / always / reject** | Yes — question/options arrive as session events | Settings: server URL or "auto-spawn", project directory, optional model/agent, session mode (new per message vs continue) |
| **Hermes** | HTTP + WebSocket against a localhost endpoint **to be implemented by Hermes** (contract below) | Yes (typed events) | Yes (typed events) | Settings: base URL, WS URL, bearer token |
| **OpenAI-compatible** | `POST {base_url}/v1/chat/completions`, `stream: true` | No (capability off) | No | Generic fallback; also covers "Hermes via local OpenAI-compatible endpoint" and cloud endpoints |

**Proposed Hermes contract** (to be confirmed/implemented Hermes-side):

```
POST {base}/api/v1/messages          body: { text, meta: { source: "voice", transcript_raw, transcript_clean } }
                                     → 202 { message_id }
WS   {base}/api/v1/events            server→client JSON events:
                                     { type: "delta"|"answer"|"permission_request"|"clarification"|"done",
                                       id, text?, options?, permission?: { action, resources, message } }
POST {base}/api/v1/permissions/{id}  body: { reply: "once"|"always"|"reject" }
```

If Hermes isn't reachable, the adapter emits `Failed` and the overlay shows a clear "Hermes offline" state; the OpenAI-compatible adapter keeps the feature usable meanwhile.

### 5.4 Permission surfacing

- `PermissionRequested` → overlay card: agent badge, action/command, affected resources, message, buttons **Allow once / Always allow / Deny** (+ keyboard `A`/`S`/`D`), auto-deny after `agents_permission_timeout_secs` (default 120).
- "Always" maps to the adapter's persistent reply (`always` for opencode; Hermes contract equivalent).

### 5.5 Clarifications

- `Clarification { question, options }` → overlay card with clickable option chips (sends the option text back via `answer_clarification`) and a free-text field.
- If `tts_speak_clarifications` is on, TTS speaks the question then each option ("Option 1: …") after current playback finishes; clicking an option cancels remaining TTS.

## 6. Overlay v2 (review mode)

### 6.1 States & windowing

`recording → streaming → transcribing → processing` exist today via the `show-overlay` string payload; add **`review`**, **`agent`**, **`answer`**. Constraints verified in code: the `recording_overlay` window is built `focusable(false)`, non-macOS transparent/always-on-top (GTK layer-shell `Layer::Overlay` on Linux, `HANDY_NO_GTK_LAYER_SHELL=1` escape hatch), macOS NSPanel via `tauri-nspanel` (floating, **non-activating**). For review mode:

- In `review`/`agent`/`answer` states the window must become **focusable and key-accepting**: flip `focusable` + call focus (Windows/Linux), layer-shell `KeyboardMode` interactive (Wayland), and an activating NSPanel variant (macOS) — this is the main platform risk, spike it in M0.
- Review mode is gated by its own setting (`overlay_review_mode`), independent of `overlay_style` (which defaults to `None` on Linux and would otherwise never show the overlay).
- Positioning/reposition rules (monitor-with-cursor, top/bottom, main-thread hop) are reused; review needs a larger window than the 256×46 pill / 400×120 streaming panel — extend `overlay_dimensions()` per state.

### 6.2 Layout

```
┌────────────────────────────────────────────────────────┐
│ ● Review            agent: [Hermes ▾]      [Settings]  │
│ ┌─ Raw ────────────────────┐ ┌─ Cleaned ──────────────┐ │
│ │ (editable textarea)      │ │ (editable textarea)    │ │
│ │            [Send →] [⧉]  │ │        [Send →] [⧉]    │ │
│ └──────────────────────────┘ └────────────────────────┘ │
│ (permissions card)  (clarification card)                │
│ ┌─ Answer ───────────────────────────────────────────┐  │
│ │ streaming text…                    [▶ replay][■]   │  │
│ └────────────────────────────────────────────────────┘  │
│ Esc = cancel · Ctrl+Enter = send cleaned                │
└────────────────────────────────────────────────────────┘
```

- Both panels editable; edits apply only to what is sent. `Send` sends **that panel's text** to the selected agent.
- Secondary actions: copy to clipboard; in classic mode (review disabled) the flow stays exactly as today (auto-paste; new setting `paste_text_source: raw | cleaned` selects which text `process_transcription_output` hands to `paste()`).
- Answer panel: streams `Delta`s; on `Answer` plays TTS (respecting `tts_auto_play`); replay/stop buttons; mute keeps text.
- Every send is captured to history (raw, cleaned, sent-variant, agent id, session ref, answer).
- The only current overlay back-channel is `commands.cancelOperation()` — the review UI adds the new commands from §11; cancel generations in `actions.rs`/`AudioRecordingManager` are reused so `--cancel`/Esc still abort everything.

## 7. TTS subsystem

### 7.1 Abstraction

```rust
#[async_trait]
pub trait TtsEngine: Send + Sync {
    fn id(&self) -> &str;
    async fn synthesize(&self, text: &str, voice: &TtsVoice) -> Result<TtsAudio>; // wav path or PCM buffer
    fn supports_streaming(&self) -> bool;
    async fn unload(&self);
}
```

- Playback reuses the rodio + selected-output-device resolution from `audio_feedback.rs` (rodio accepts any `Read + Seek`, so a TTS WAV buffer plays via `Cursor<Vec<u8>>`). Use its own sink/stream — today's helpers `sleep_until_end` per sink and would block chimes.
- TTS text is pre-processed: strip markdown fences/links (setting `tts_strip_markdown`, default on) so code blocks are not read aloud.

### 7.2 Engines

| Engine | Model | Runtime | Status |
|---|---|---|---|
| `qwen3-tts` (default) | `Qwen/Qwen3-TTS-12Hz-0.6B-Base` | GGUF via bundled `llama-server` (llama.cpp serves `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF:Q4_K_M`; **no 0.6B GGUF published yet**) | 1.7B works today; 0.6B entry ships flagged "GGUF pending", falls back to 1.7B until published |
| `kokoro` | `hexgrad/Kokoro-82M-v1.0` ONNX | Native Rust via `ort` (Handy already uses ONNX Runtime through transcribe-rs) + pure-Rust phonemizer (kokoroxide/any-tts pattern) | lightest, CPU-friendly, multi-voice |
| `voxcpm2` | `openbmb/VoxCPM2` (2B) | Python `voxcpm` package — **out of scope v1** | catalog entry marked future |

- Voice handling: Qwen3-TTS **Base** clones from a reference clip (≥3 s + transcript) — settings expose a reference-audio file picker; Kokoro exposes its voice list.
- `llm_runtime.rs` owns the llama-server lifecycle (start on first synthesis, idle-stop, health check), shared with the S1-mini fallback sidecar.

### 7.3 What gets spoken

- Final `Answer` (when TTS enabled) and `Clarification` cards (opt-in, §5.5). Tool activity and deltas are never spoken.

## 8. Settings — new sections

Plumbing follows the verified pattern end-to-end: fields on `AppSettings` (`settings.rs`, with `#[serde(default)]` + default fns + `settings_schema_version` bump) → `change_*_setting` commands registered in `collect_commands!` → `settingUpdaters` in `src/stores/settingsStore.ts` → UI components under `src/components/settings/{agents,tts}/` wired into `SECTIONS_CONFIG` in `src/components/Sidebar.tsx` (new nav entries "Agents", "TTS Models") → i18n keys in `src/i18n/locales/en/translation.json`.

**Agents**
| Field | Type | Default |
|---|---|---|
| `agents` | list: `{ id, enabled, kind: hermes|opencode|openai_compat, name, base_url, ws_url?, token?, project_dir?, model?, session_mode: per_message|continue, preamble_override? }` | Hermes (disabled), opencode (disabled) |
| `agents_default_id` | string | first enabled |
| `agents_permission_timeout_secs` | int | 120 |
| `agents_preamble_template` | string | i18n default (§5.2) |

**TTS Models**
| Field | Type | Default |
|---|---|---|
| `tts_enabled` | bool | true |
| `tts_engine_id` | string | `qwen3-tts` |
| `tts_model_id` | catalog id | `qwen3-tts-0.6b-base` |
| `tts_voice` / `tts_ref_audio_path` / `tts_ref_audio_text` | string | engine defaults |
| `tts_auto_play` | bool | true |
| `tts_speak_clarifications` | bool | true |
| `tts_strip_markdown` | bool | true |
| `tts_output_device` | reuse existing OutputDeviceSelector | default |

**General (review flow)**
`overlay_review_mode: auto | interactive` (default `interactive`), `paste_text_source: raw | cleaned` (default `cleaned`), plus §4.4 cleanup knobs.

## 9. Model catalog & downloads

Handy ships 69 transcription models in `src-tauri/src/catalog/catalog.json` (`catalog_version: 2`, mirror `blob.handy.computer`, per-file `sha256` as the trust anchor, generated by `scripts/gen_catalog.py`). Catalog entries currently map 1:1 to `EngineType::TranscribeCpp` GGUFs; downloads go through `ModelManager` (`ModelSource::HuggingFace` via hf-hub into the shared HF cache, revision-pinned, or `ModelSource::Url` with sha256 + resumable HTTP + mirror fallback). Extend, don't fork:

- **Catalog v3:** each model gains `task: "stt" (default) | "cleanup" | "tts"`; `catalog/mod.rs` maps task → new `EngineType` variants (e.g. `LlamaCleanup`, `TtsQwen3`, `TtsKokoro`) instead of hardcoding `TranscribeCpp`. Existing STT entries unchanged.
- **New entries:** `s1-mini` (cleanup, Q4_K_M default, naming-clause note), `kokoro-82m-v1.0` (tts), `qwen3-tts-0.6b-base` (tts, flagged pending), `qwen3-tts-1.7b-base` (tts), `voxcpm2` (tts, future). Sources are public HF repos (`superwhisper/s1-mini-GGUF`, `hexgrad/Kokoro-82M-v1.0-onnx`, `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF`, …) with pinned revisions + sha256 — no fork-owned mirror needed.
- The existing download manager (progress/verify/cancel events, HF cache discovery, `models_dir` fallback) is reused untouched; the model selector gets a task filter and the **TTS Models** settings page lists `task: "tts"` entries with the same download UX.

## 10. History

New SQLite migration in `managers/history.rs` (rusqlite_migration): `agent_id TEXT`, `agent_session_ref TEXT`, `agent_sent_text TEXT`, `agent_response TEXT`, `tts_played BOOLEAN DEFAULT 0`. `HistoryEntry` mirrors them; updates flow out through the existing typed `history-update-payload` event. History UI (optional, M4): badge on agent-sent entries, expandable answer.

## 11. Tauri commands & events (new)

**Commands:** `agents_list`, `agents_send(text, source: raw|cleaned)`, `agents_reply_permission(id, reply)`, `agents_answer_clarification(session_ref, text)`, `agents_interrupt`, `tts_synthesize(text)`, `tts_stop`, `cleanup_preview(raw)` (re-run cleanup on edited raw).

**Events:** follow the two existing conventions — plain kebab-case strings (`show-overlay`, `mic-level`) targeted to the overlay window with `emit_to("recording_overlay", …)`, and typed `tauri_specta::Event` structs registered in `collect_events!` (like `StreamTextEvent`, `HistoryUpdatePayload`). New typed events: `agent-event` (tagged `AgentEvent` envelope), `tts-state-event`. New overlay-targeted strings: `show-overlay` gains the new states; `overlay-permission-request` and `overlay-clarification` payloads reuse the typed structs.

## 12. i18n

Every new string lives in `src/i18n/locales/en/translation.json` under `agents.*`, `tts.*`, `overlay.review.*` (ESLint forbids hardcoded JSX strings). User-visible model name must keep the exact string *"S1-mini by Superwhisper"*.

## 13. Testing & acceptance

- **Unit:** prompt builder reproduces the exact S1-mini template (control line + empty think block, golden strings); catalog v3 parsing; `AgentEvent` serialization; adapter mocking with wiremock (permission round-trip, clarification, SSE delta parsing).
- **Integration:** opencode adapter against a real `opencode serve` (manual/nightly, version-pinned); Hermes contract test server.
- **E2E (Playwright):** settings pages render, catalog downloads progress, review overlay toggles.
- **Acceptance walkthrough:** hold shortcut → speak → overlay shows raw + cleaned → edit cleaned → Send → opencode streams → permission card appears → Allow once → answer streams → TTS plays; repeat with Hermes; classic auto-paste unchanged with both `raw`/`cleaned` sources; Kokoro ↔ Qwen3-TTS switch works; offline agent shows a graceful failure state.

## 14. Implementation phases

| Phase | Scope | Exit criteria |
|---|---|---|
| **M0 — spikes** | (a) `llama-cpp-2` runs s1-mini GGUF with exact prompt → golden outputs; (b) `llama-server` + Qwen3-TTS 1.7B GGUF produces audio incl. voice-clone ref flow; (c) opencode v2 server: create/prompt/SSE/permission reply; (d) overlay review window keyboard focus on Wayland, X11, macOS NSPanel | each spike scripted and reproducible |
| **M1 — cleanup + review** | catalog v3, S1-mini provider, settings, overlay raw/cleaned panels, paste sources | dictation with cleanup end-to-end; review overlay editable & sendable |
| **M2 — agents** | trait + registry, opencode + OpenAI-compat adapters, permission & clarify cards, preamble | acceptance walkthrough with opencode |
| **M3 — TTS** | `TtsEngine`, Qwen3-TTS via `llm_runtime`, Kokoro via ort, answer/clarify playback | spoken answers on both engines |
| **M4 — Hermes + polish** | Hermes adapter once contract is implemented, history columns, i18n, docs | full walkthrough incl. Hermes |

## 15. Risks & open questions

1. **Qwen3-TTS 0.6B GGUF** not yet published (only 1.7B-Base from ggml-org) — ship 1.7B + fallback, watch for 0.6B quants; alternative: vendor `predict-woo/qwen3-tts.cpp` (C++/GGML, no Python).
2. **llama.cpp TTS voice-clone flags** for Qwen3-TTS Base (reference-audio input via server) need M0 validation.
3. **Hermes API doesn't exist yet** — contract in §5.3 must be implemented Hermes-side; the OpenAI-compatible adapter is the bridge until then.
4. **Overlay keyboard focus** (Wayland layer-shell, macOS non-activating NSPanel) — the riskiest platform change; M0 spike.
5. **VRAM contention** (STT + S1-mini + TTS + agent LLM) — on-demand loading + idle unload (`model_unload_timeout` semantics) and a single `llm_runtime` supervisor.
6. **S1-mini is English-only** — hide cleanup when the selected STT language ≠ en; fall back to raw.
7. **S1-mini naming clause** — ship LICENSE + NOTICE; UI shows the exact name.
8. **opencode v2 API is young** — pin the server version in settings; gate unknown endpoints behind capability detection.
9. **Clarify detection** relies on adapters emitting structured `Clarification` events; heuristic text-detection is explicitly out of scope.

## 16. References

- S1-mini: https://huggingface.co/superwhisper/s1-mini · GGUF: https://huggingface.co/superwhisper/s1-mini-GGUF
- Qwen3-TTS: https://github.com/QwenLM/Qwen3-TTS · GGUF: https://huggingface.co/ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF
- Kokoro: https://huggingface.co/hexgrad/Kokoro-82M (Rust: kokoroxide / kokorox / any-tts crates)
- VoxCPM2: https://huggingface.co/openbmb/VoxCPM2 · https://github.com/OpenBMB/VoxCPM
- opencode API: https://opencode.ai/v2/docs/api (sessions, prompt, SSE log, `Permission.Request` + reply `once|always|reject`)
- qwen3-tts.cpp (C++/GGML alternative): https://github.com/predict-woo/qwen3-tts.cpp+ agent LLM) — on-demand loading + idle unload (`model_unload_timeout` semantics) and a single `llm_runtime` supervisor.
6. **S1-mini is English-only** — hide cleanup when the selected STT language ≠ en; fall back to raw.
7. **S1-mini naming clause** — ship LICENSE + NOTICE; UI shows the exact name.
8. **opencode v2 API is young** — pin the server version in settings; gate unknown endpoints behind capability detection.
9. **Clarify detection** relies on adapters emitting structured `Clarification` events; heuristic text-detection is explicitly out of scope.

## 16. References

- S1-mini: https://huggingface.co/superwhisper/s1-mini · GGUF: https://huggingface.co/superwhisper/s1-mini-GGUF
- Qwen3-TTS: https://github.com/QwenLM/Qwen3-TTS · GGUF: https://huggingface.co/ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF
- Kokoro: https://huggingface.co/hexgrad/Kokoro-82M (Rust: kokoroxide / kokorox / any-tts crates)
- VoxCPM2: https://huggingface.co/openbmb/VoxCPM2 · https://github.com/OpenBMB/VoxCPM
- opencode API: https://opencode.ai/v2/docs/api (sessions, prompt, SSE log, `Permission.Request` + reply `once|always|reject`)
- qwen3-tts.cpp (C++/GGML alternative): https://github.com/predict-woo/qwen3-tts.cpp
### 5.4 Permission surfacing

- `PermissionRequested` → overlay card: agent badge, action/command, affected resources, message, buttons **Allow once / Always allow / Deny** (+ keyboard `A`/`S`/`D`), auto-deny after `agents_permission_timeout_secs` (default 120).
- "Always" maps to the adapter's persistent reply (`always` for opencode; Hermes contract equivalent).

### 5.5 Clarifications

- `Clarification { question, options }` → overlay card with clickable option chips (sends the option text back via `answer_clarification`) and a free-text field.
- If `tts_speak_clarifications` is on, TTS speaks the question then each option ("Option 1: …") after current playback finishes; clicking an option cancels remaining TTS.

## 6. Overlay v2 (review mode)

### 6.1 States & windowing

`recording → streaming → transcribing → processing` exist today via the `show-overlay` string payload; add **`review`**, **`agent`**, **`answer`**. Constraints verified in code: the `recording_overlay` window is built `focusable(false)`, non-macOS transparent/always-on-top (GTK layer-shell `Layer::Overlay` on Linux, `HANDY_NO_GTK_LAYER_SHELL=1` escape hatch), macOS NSPanel via `tauri-nspanel` (floating, **non-activating**). For review mode:

- In `review`/`agent`/`answer` states the window must become **focusable and key-accepting**: flip `focusable` + call focus (Windows/Linux), layer-shell `KeyboardMode` interactive (Wayland), and an activating NSPanel variant (macOS) — this is the main platform risk, spike it in M0.
- Review mode is gated by its own setting (`overlay_review_mode`), independent of `overlay_style` (which defaults to `None` on Linux and would otherwise never show the overlay).
- Positioning/reposition rules (monitor-with-cursor, top/bottom, main-thread hop) are reused; review needs a larger window than the 256×46 pill / 400×120 streaming panel — extend `overlay_dimensions()` per state.

### 6.2 Layout

```
┌────────────────────────────────────────────────────────┐
│ ● Review            agent: [Hermes ▾]      [Settings]  │
│ ┌─ Raw ────────────────────┐ ┌─ Cleaned ──────────────┐ │
│ │ (editable textarea)      │ │ (editable textarea)    │ │
│ │            [Send →] [⧉]  │ │        [Send →] [⧉]    │ │
│ └──────────────────────────┘ └────────────────────────┘ │
│ (permissions card)  (clarification card)                │
│ ┌─ Answer ───────────────────────────────────────────┐  │
│ │ streaming text…                    [▶ replay][■]   │  │
│ └────────────────────────────────────────────────────┘  │
│ Esc = cancel · Ctrl+Enter = send cleaned                │
└────────────────────────────────────────────────────────┘
```

- Both panels editable; edits apply only to what is sent. `Send` sends **that panel's text** to the selected agent.
- Secondary actions: copy to clipboard; in classic mode (review disabled) the flow stays exactly as today (auto-paste; new setting `paste_text_source: raw | cleaned` selects which text `process_transcription_output` hands to `paste()`).
- Answer panel: streams `Delta`s; on `Answer` plays TTS (respecting `tts_auto_play`); replay/stop buttons; mute keeps text.
- Every send is captured to history (raw, cleaned, sent-variant, agent id, session ref, answer).
- The only current overlay back-channel is `commands.cancelOperation()` — the review UI adds the new commands from §11; cancel generations in `actions.rs`/`AudioRecordingManager` are reused so `--cancel`/Esc still abort everything.

## 7. TTS subsystem

### 7.1 Abstraction

```rust
#[async_trait]
pub trait TtsEngine: Send + Sync {
    fn id(&self) -> &str;
    async fn synthesize(&self, text: &str, voice: &TtsVoice) -> Result<TtsAudio>; // wav path or PCM buffer
    fn supports_streaming(&self) -> bool;
    async fn unload(&self);
}
```

- Playback reuses the rodio + selected-output-device resolution from `audio_feedback.rs` (rodio accepts any `Read + Seek`, so a TTS WAV buffer plays via `Cursor<Vec<u8>>`). Use its own sink/stream — today's helpers `sleep_until_end` per sink and would block chimes.
- TTS text is pre-processed: strip markdown fences/links (setting `tts_strip_markdown`, default on) so code blocks are not read aloud.

### 7.2 Engines

| Engine | Model | Runtime | Status |
|---|---|---|---|
| `qwen3-tts` (default) | `Qwen/Qwen3-TTS-12Hz-0.6B-Base` | GGUF via bundled `llama-server` (llama.cpp serves `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF:Q4_K_M`; **no 0.6B GGUF published yet**) | 1.7B works today; 0.6B entry ships flagged "GGUF pending", falls back to 1.7B until published |
| `kokoro` | `hexgrad/Kokoro-82M-v1.0` ONNX | Native Rust via `ort` (Handy already uses ONNX Runtime through transcribe-rs) + pure-Rust phonemizer (kokoroxide/any-tts pattern) | lightest, CPU-friendly, multi-voice |
| `voxcpm2` | `openbmb/VoxCPM2` (2B) | Python `voxcpm` package — **out of scope v1** | catalog entry marked future |

- Voice handling: Qwen3-TTS **Base** clones from a reference clip (≥3 s + transcript) — settings expose a reference-audio file picker; Kokoro exposes its voice list.
- `llm_runtime.rs` owns the llama-server lifecycle (start on first synthesis, idle-stop, health check), shared with the S1-mini fallback sidecar.

### 7.3 What gets spoken

- Final `Answer` (when TTS enabled) and `Clarification` cards (opt-in, §5.5). Tool activity and deltas are never spoken.

## 8. Settings — new sections

Plumbing follows the verified pattern end-to-end: fields on `AppSettings` (`settings.rs`, with `#[serde(default)]` + default fns + `settings_schema_version` bump) → `change_*_setting` commands registered in `collect_commands!` → `settingUpdaters` in `src/stores/settingsStore.ts` → UI components under `src/components/settings/{agents,tts}/` wired into `SECTIONS_CONFIG` in `src/components/Sidebar.tsx` (new nav entries "Agents", "TTS Models") → i18n keys in `src/i18n/locales/en/translation.json`.

**Agents**
| Field | Type | Default |
|---|---|---|
| `agents` | list: `{ id, enabled, kind: hermes|opencode|openai_compat, name, base_url, ws_url?, token?, project_dir?, model?, session_mode: per_message|continue, preamble_override? }` | Hermes (disabled), opencode (disabled) |
| `agents_default_id` | string | first enabled |
| `agents_permission_timeout_secs` | int | 120 |
| `agents_preamble_template` | string | i18n default (§5.2) |

**TTS Models**
| Field | Type | Default |
|---|---|---|
| `tts_enabled` | bool | true |
| `tts_engine_id` | string | `qwen3-tts` |
| `tts_model_id` | catalog id | `qwen3-tts-0.6b-base` |
| `tts_voice` / `tts_ref_audio_path` / `tts_ref_audio_text` | string | engine defaults |
| `tts_auto_play` | bool | true |
| `tts_speak_clarifications` | bool | true |
| `tts_strip_markdown` | bool | true |
| `tts_output_device` | reuse existing OutputDeviceSelector | default |

**General (review flow)**
`overlay_review_mode: auto | interactive` (default `interactive`), `paste_text_source: raw | cleaned` (default `cleaned`), plus §4.4 cleanup knobs.

## 9. Model catalog & downloads

Handy ships 69 transcription models in `src-tauri/src/catalog/catalog.json` (`catalog_version: 2`, mirror `blob.handy.computer`, per-file `sha256` as the trust anchor, generated by `scripts/gen_catalog.py`). Catalog entries currently map 1:1 to `EngineType::TranscribeCpp` GGUFs; downloads go through `ModelManager` (`ModelSource::HuggingFace` via hf-hub into the shared HF cache, revision-pinned, or `ModelSource::Url` with sha256 + resumable HTTP + mirror fallback). Extend, don't fork:

- **Catalog v3:** each model gains `task: "stt" (default) | "cleanup" | "tts"`; `catalog/mod.rs` maps task → new `EngineType` variants (e.g. `LlamaCleanup`, `TtsQwen3`, `TtsKokoro`) instead of hardcoding `TranscribeCpp`. Existing STT entries unchanged.
- **New entries:** `s1-mini` (cleanup, Q4_K_M default, naming-clause note), `kokoro-82m-v1.0` (tts), `qwen3-tts-0.6b-base` (tts, flagged pending), `qwen3-tts-1.7b-base` (tts), `voxcpm2` (tts, future). Sources are public HF repos (`superwhisper/s1-mini-GGUF`, `hexgrad/Kokoro-82M-v1.0-onnx`, `ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF`, …) with pinned revisions + sha256 — no fork-owned mirror needed.
- The existing download manager (progress/verify/cancel events, HF cache discovery, `models_dir` fallback) is reused untouched; the model selector gets a task filter and the **TTS Models** settings page lists `task: "tts"` entries with the same download UX.

## 10. History

New SQLite migration in `managers/history.rs` (rusqlite_migration): `agent_id TEXT`, `agent_session_ref TEXT`, `agent_sent_text TEXT`, `agent_response TEXT`, `tts_played BOOLEAN DEFAULT 0`. `HistoryEntry` mirrors them; updates flow out through the existing typed `history-update-payload` event. History UI (optional, M4): badge on agent-sent entries, expandable answer.

## 11. Tauri commands & events (new)

**Commands:** `agents_list`, `agents_send(text, source: raw|cleaned)`, `agents_reply_permission(id, reply)`, `agents_answer_clarification(session_ref, text)`, `agents_interrupt`, `tts_synthesize(text)`, `tts_stop`, `cleanup_preview(raw)` (re-run cleanup on edited raw).

**Events:** follow the two existing conventions — plain kebab-case strings (`show-overlay`, `mic-level`) targeted to the overlay window with `emit_to("recording_overlay", …)`, and typed `tauri_specta::Event` structs registered in `collect_events!` (like `StreamTextEvent`, `HistoryUpdatePayload`). New typed events: `agent-event` (tagged `AgentEvent` envelope), `tts-state-event`. New overlay-targeted strings: `show-overlay` gains the new states; `overlay-permission-request` and `overlay-clarification` payloads reuse the typed structs.

## 12. i18n

Every new string lives in `src/i18n/locales/en/translation.json` under `agents.*`, `tts.*`, `overlay.review.*` (ESLint forbids hardcoded JSX strings). User-visible model name must keep the exact string *"S1-mini by Superwhisper"*.

## 13. Testing & acceptance

- **Unit:** prompt builder reproduces the exact S1-mini template (control line + empty think block, golden strings); catalog v3 parsing; `AgentEvent` serialization; adapter mocking with wiremock (permission round-trip, clarification, SSE delta parsing).
- **Integration:** opencode adapter against a real `opencode serve` (manual/nightly, version-pinned); Hermes contract test server.
- **E2E (Playwright):** settings pages render, catalog downloads progress, review overlay toggles.
- **Acceptance walkthrough:** hold shortcut → speak → overlay shows raw + cleaned → edit cleaned → Send → opencode streams → permission card appears → Allow once → answer streams → TTS plays; repeat with Hermes; classic auto-paste unchanged with both `raw`/`cleaned` sources; Kokoro ↔ Qwen3-TTS switch works; offline agent shows a graceful failure state.

## 14. Implementation phases

| Phase | Scope | Exit criteria |
|---|---|---|
| **M0 — spikes** | (a) `llama-cpp-2` runs s1-mini GGUF with exact prompt → golden outputs; (b) `llama-server` + Qwen3-TTS 1.7B GGUF produces audio incl. voice-clone ref flow; (c) opencode v2 server: create/prompt/SSE/permission reply; (d) overlay review window keyboard focus on Wayland, X11, macOS NSPanel | each spike scripted and reproducible |
| **M1 — cleanup + review** | catalog v3, S1-mini provider, settings, overlay raw/cleaned panels, paste sources | dictation with cleanup end-to-end; review overlay editable & sendable |
| **M2 — agents** | trait + registry, opencode + OpenAI-compat adapters, permission & clarify cards, preamble | acceptance walkthrough with opencode |
| **M3 — TTS** | `TtsEngine`, Qwen3-TTS via `llm_runtime`, Kokoro via ort, answer/clarify playback | spoken answers on both engines |
| **M4 — Hermes + polish** | Hermes adapter once contract is implemented, history columns, i18n, docs | full walkthrough incl. Hermes |

## 15. Risks & open questions

1. **Qwen3-TTS 0.6B GGUF** not yet published (only 1.7B-Base from ggml-org) — ship 1.7B + fallback, watch for 0.6B quants; alternative: vendor `predict-woo/qwen3-tts.cpp` (C++/GGML, no Python).
2. **llama.cpp TTS voice-clone flags** for Qwen3-TTS Base (reference-audio input via server) need M0 validation.
3. **Hermes API doesn't exist yet** — contract in §5.3 must be implemented Hermes-side; the OpenAI-compatible adapter is the bridge until then.
4. **Overlay keyboard focus** (Wayland layer-shell, macOS non-activating NSPanel) — the riskiest platform change; M0 spike.
5. **VRAM contention** (STT + S1-mini + TTS + agent LLM) — on-demand loading + idle unload (`model_unload_timeout` semantics) and a single `llm_runtime` supervisor.
6. **S1-mini is English-only** — hide cleanup when the selected STT language ≠ en; fall back to raw.
7. **S1-mini naming clause** — ship LICENSE + NOTICE; UI shows the exact name.
8. **opencode v2 API is young** — pin the server version in settings; gate unknown endpoints behind capability detection.
9. **Clarify detection** relies on adapters emitting structured `Clarification` events; heuristic text-detection is explicitly out of scope.

## 16. References

- S1-mini: https://huggingface.co/superwhisper/s1-mini · GGUF: https://huggingface.co/superwhisper/s1-mini-GGUF
- Qwen3-TTS: https://github.com/QwenLM/Qwen3-TTS · GGUF: https://huggingface.co/ggml-org/Qwen3-TTS-12Hz-1.7B-Base-GGUF
- Kokoro: https://huggingface.co/hexgrad/Kokoro-82M (Rust: kokoroxide / kokorox / any-tts crates)
- VoxCPM2: https://huggingface.co/openbmb/VoxCPM2 · https://github.com/OpenBMB/VoxCPM
- opencode API: https://opencode.ai/v2/docs/api (sessions, prompt, SSE log, `Permission.Request` + reply `once|always|reject`)
- qwen3-tts.cpp (C++/GGML alternative): https://github.com/predict-woo/qwen3-tts.cpp