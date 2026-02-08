# NanoBot - Model Architecture Analysis

> **Repository:** [HKUDS/NanoBot](https://github.com/HKUDS/NanoBot)
> **Version analyzed:** v0.1.3.post4 (commit date: 2026-02-06)
> **Analyst perspective:** Senior AI Researcher (Edge AI / On-device LLMs / Multimodal Systems)

---

## 1. Executive Summary

**NanoBot is NOT an on-device/edge AI agent.** Despite the "nano" prefix suggesting lightweight on-device inference, NanoBot is a **cloud-first, API-driven agent framework** that is "nano" only in terms of its **codebase size** (~3,400 lines of Python). It does **not** ship its own model weights, does **not** use quantization, and does **not** include an embedded inference engine. Instead, it delegates all LLM inference to external API providers through the **LiteLLM** abstraction layer.

This is a critical finding for anyone approaching this repository from an Edge AI research perspective.

---

## 2. Model Backbone

### 2.1 No Embedded Model — Pure API Proxy Architecture

NanoBot uses **no local model backbone**. The `LiteLLMProvider` class (`nanobot/providers/litellm_provider.py`) acts as a unified interface to **11+ cloud LLM providers**:

| Provider | Identifier | Default Model Example |
|----------|------------|----------------------|
| OpenRouter | `openrouter` | `anthropic/claude-opus-4-5` |
| Anthropic | `anthropic` | `anthropic/claude-sonnet-4-5` |
| OpenAI | `openai` | `gpt-4o` |
| DeepSeek | `deepseek` | `deepseek-chat` |
| Google Gemini | `gemini` | `gemini-pro` |
| Groq | `groq` | `llama-3.1-70b` |
| DashScope (Alibaba) | `dashscope` | `qwen-max` |
| Zhipu AI | `zhipu` | `glm-4` |
| Moonshot/Kimi | `moonshot` | `moonshot-v1-8k` |
| AiHubMix | `aihubmix` | (gateway, any model) |
| **vLLM (local)** | `vllm` | User-specified (e.g., `meta-llama/Llama-3.1-8B-Instruct`) |

**Default model:** `anthropic/claude-opus-4-5` (as defined in `nanobot/config/schema.py:53`)

### 2.2 vLLM Local Model Support

The **only local inference pathway** is through vLLM, which is treated as an OpenAI-compatible API server:

```python
# From litellm_provider.py:41
self.is_vllm = bool(api_base) and not self.is_openrouter and not self.is_aihubmix
```

When vLLM is configured, NanoBot:
1. Prefixes the model name with `hosted_vllm/` for LiteLLM routing
2. Passes the user-specified `api_base` URL (e.g., `http://localhost:8000/v1`)
3. Uses a dummy API key (no authentication needed for local servers)

**Key insight:** Even the "local" mode requires a **separate vLLM server process** running on the same machine. NanoBot itself performs **zero** model loading, weight management, or inference computation. It simply calls the vLLM HTTP API, which is functionally identical to calling any cloud API.

```json
// Example vLLM config from README
{
  "providers": {
    "vllm": {
      "apiKey": "dummy",
      "apiBase": "http://localhost:8000/v1"
    }
  },
  "agents": {
    "defaults": {
      "model": "meta-llama/Llama-3.1-8B-Instruct"
    }
  }
}
```

### 2.3 Provider Selection Logic

The provider resolution follows a keyword-matching + fallback strategy defined in `Config.get_provider()` (`nanobot/config/schema.py:131-150`):

```
1. Match model name keywords → specific provider (e.g., "claude" → anthropic)
2. Fallback: try gateways first (OpenRouter, AiHubMix) → then specific providers
3. Return first provider with a valid API key
```

Model name auto-prefixing rules (`litellm_provider.py:104-114`):
- `glm*` / `zhipu*` → `zai/` prefix
- `qwen*` / `dashscope*` → `dashscope/` prefix
- `moonshot*` / `kimi*` → `moonshot/` prefix
- `gemini*` → `gemini/` prefix
- OpenRouter key detected → `openrouter/` prefix
- Custom api_base → `hosted_vllm/` prefix

---

## 3. Multimodal Integration

### 3.1 Vision (Image) Input Processing

NanoBot has **rudimentary multimodal support** — limited to base64 image encoding for the OpenAI Vision API format. There is **no dedicated Vision Encoder**, **no Projection Layer**, and **no visual feature extraction pipeline**.

The image processing is handled in `ContextBuilder._build_user_content()` (`nanobot/agent/context.py:161-177`):

```python
def _build_user_content(self, text: str, media: list[str] | None) -> str | list[dict[str, Any]]:
    """Build user message content with optional base64-encoded images."""
    if not media:
        return text
    
    images = []
    for path in media:
        p = Path(path)
        mime, _ = mimetypes.guess_type(path)
        if not p.is_file() or not mime or not mime.startswith("image/"):
            continue
        b64 = base64.b64encode(p.read_bytes()).decode()
        images.append({"type": "image_url", "image_url": {"url": f"data:{mime};base64,{b64}"}})
    
    if not images:
        return text
    return images + [{"type": "text", "text": text}]
```

**Mechanism:**
1. Media files (images) are downloaded by channel handlers (Telegram, Discord) to `~/.nanobot/media/`
2. The `InboundMessage.media` field carries local file paths
3. During context building, images are base64-encoded and formatted as OpenAI `image_url` content parts
4. The multimodal message is sent to the LLM API, which must support vision (e.g., GPT-4V, Claude 3.x, Gemini Pro Vision)

**Limitations:**
- **No video processing** — only `image/*` MIME types are handled
- **No streaming/frame extraction** — video files are ignored entirely
- **No local vision model** — all visual understanding is delegated to the cloud LLM
- **No image preprocessing** — no resizing, cropping, or normalization
- **No visual grounding** — no bounding boxes, segmentation, or spatial reasoning
- The multimodal roadmap item is explicitly listed as TODO: `"Multi-modal — See and hear (images, voice, video)"`

### 3.2 Audio/Voice Input Processing

Voice transcription is handled by a **separate Groq Whisper API** integration (`nanobot/providers/transcription.py`):

```python
class GroqTranscriptionProvider:
    """Voice transcription using Groq's Whisper API (whisper-large-v3)."""
    
    async def transcribe(self, file_path: str | Path) -> str:
        # POST to https://api.groq.com/openai/v1/audio/transcriptions
        # Model: whisper-large-v3
```

**Flow:**
1. Telegram/Discord channel receives voice/audio message
2. Audio file is downloaded to `~/.nanobot/media/`
3. `GroqTranscriptionProvider.transcribe()` sends the file to Groq's Whisper API
4. Transcribed text is appended to the user message as `[transcription: ...]`
5. The text-based message is then processed by the main agent loop

**No local ASR model** — voice transcription requires an external Groq API key.

---

## 4. Quantization & Inference Engine Optimization

### 4.1 Quantization

**None.** NanoBot does not perform any model quantization. Since it doesn't load model weights, there is no need for:
- GPTQ, AWQ, GGUF, or any weight quantization format
- INT4/INT8 computation
- Mixed-precision inference
- KV-cache quantization

### 4.2 Inference Engine

**None embedded.** NanoBot uses **no inference engine**:
- **No MLC-LLM** — not referenced anywhere in the codebase
- **No ONNX Runtime** — not a dependency
- **No TFLite** — not applicable
- **No llama.cpp** — not integrated
- **No vLLM embedded** — vLLM is an *external server*, not embedded

The entire inference stack is:

```
User Input → NanoBot (Python) → HTTP/API Call → LiteLLM → Provider API → Response
```

### 4.3 Dependencies Analysis

From `pyproject.toml`, the runtime dependencies are:

```toml
dependencies = [
    "typer>=0.9.0",           # CLI framework
    "litellm>=1.0.0",         # Multi-provider LLM abstraction
    "pydantic>=2.0.0",        # Data validation
    "pydantic-settings>=2.0.0",
    "websockets>=12.0",       # WebSocket client
    "websocket-client>=1.6.0",
    "httpx>=0.25.0",          # HTTP client
    "loguru>=0.7.0",          # Logging
    "readability-lxml>=0.8.0",# Web content extraction
    "rich>=13.0.0",           # Terminal UI
    "croniter>=2.0.0",        # Cron expression parsing
    "python-telegram-bot>=21.0", # Telegram integration
    "lark-oapi>=1.0.0",       # Feishu/Lark integration
]
```

**Notable absences:**
- No `torch`, `transformers`, `accelerate`, `bitsandbytes`
- No `onnxruntime`, `tflite`, `mlc-llm`
- No `llama-cpp-python`, `ctransformers`
- No `numpy`, `scipy`, `scikit-learn`

This confirms: **zero local ML computation capability**.

---

## 5. Configuration Architecture

### 5.1 Config File Structure

Configuration is stored in `~/.nanobot/config.json` and parsed by Pydantic models in `nanobot/config/schema.py`:

```
Config (root)
├── agents
│   └── defaults
│       ├── workspace: str = "~/.nanobot/workspace"
│       ├── model: str = "anthropic/claude-opus-4-5"
│       ├── max_tokens: int = 8192
│       ├── temperature: float = 0.7
│       └── max_tool_iterations: int = 20
├── providers
│   ├── anthropic: {api_key, api_base}
│   ├── openai: {api_key, api_base}
│   ├── openrouter: {api_key, api_base}
│   ├── deepseek: {api_key, api_base}
│   ├── groq: {api_key, api_base}
│   ├── gemini: {api_key, api_base}
│   ├── zhipu: {api_key, api_base}
│   ├── dashscope: {api_key, api_base}
│   ├── vllm: {api_key, api_base}
│   ├── moonshot: {api_key, api_base}
│   └── aihubmix: {api_key, api_base, extra_headers}
├── channels
│   ├── telegram: {enabled, token, allow_from, proxy}
│   ├── discord: {enabled, token, allow_from, gateway_url, intents}
│   ├── whatsapp: {enabled, bridge_url, allow_from}
│   └── feishu: {enabled, app_id, app_secret, ...}
├── tools
│   ├── web.search: {api_key, max_results}
│   ├── exec: {timeout}
│   └── restrict_to_workspace: bool
└── gateway
    ├── host: str = "0.0.0.0"
    └── port: int = 18790
```

### 5.2 Key Configuration Files in Workspace

```
~/.nanobot/workspace/
├── AGENTS.md       # Agent behavioral instructions (system prompt component)
├── SOUL.md         # Personality/character definition
├── USER.md         # User preferences and context
├── TOOLS.md        # Tool documentation (for agent reference)
├── HEARTBEAT.md    # Periodic task list (checked every 30 min)
├── IDENTITY.md     # Optional identity customization
├── memory/
│   ├── MEMORY.md   # Long-term persistent memory
│   └── YYYY-MM-DD.md  # Daily notes
└── skills/
    └── {skill-name}/SKILL.md  # Custom skill definitions
```

---

## 6. Comparison: What "Nano" Actually Means

| Dimension | Expectation (Edge AI) | Reality (NanoBot) |
|-----------|----------------------|-------------------|
| Model size | <7B parameters, quantized | No local model — API calls only |
| Inference engine | MLC, ONNX, TFLite, llama.cpp | None — uses LiteLLM as API wrapper |
| Quantization | INT4/INT8/GPTQ/AWQ | None |
| Vision encoder | Lightweight (MobileViT, SigLIP) | None — base64 images sent to cloud API |
| Memory footprint | <4GB RAM | Minimal (~50MB Python process) |
| Latency | <1s on-device | Network-dependent (API latency) |
| Offline capability | Full offline operation | **Requires internet** for all LLM calls |
| **What is "nano"** | Small model | **Small codebase** (~3,400 LOC) |

---

## 7. Key Source Files for Architecture

| File | Purpose | Lines |
|------|---------|-------|
| `nanobot/providers/base.py` | LLMProvider abstract interface | 70 |
| `nanobot/providers/litellm_provider.py` | Multi-provider LLM implementation | 198 |
| `nanobot/providers/transcription.py` | Groq Whisper voice transcription | 66 |
| `nanobot/config/schema.py` | Pydantic configuration models | 170 |
| `nanobot/config/loader.py` | Config loading/saving/migration | 107 |
| `nanobot/agent/context.py` | System prompt & multimodal context builder | 230 |

---

## 8. Conclusion

NanoBot's "nano" refers exclusively to its **codebase minimalism** — achieving full agent functionality in ~3,400 lines of Python, compared to 430,000+ lines in its inspiration (Clawdbot/OpenClaw). From an Edge AI perspective, NanoBot is:

- **Not an on-device model** — it's an API orchestrator
- **Not optimized for edge deployment** — requires constant internet connectivity
- **Not using any inference optimization** — no quantization, no custom runtime
- **Minimally multimodal** — basic image pass-through via base64, no vision encoder
- **Architecturally simple** — single LiteLLM provider handles all model diversity

The value proposition is in its **clean, extensible agent framework design** rather than in model-level innovation. For Edge AI research, the relevant takeaway is the agent orchestration pattern, not the model architecture.
