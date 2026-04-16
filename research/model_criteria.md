**Model Card Reviews for the Three Models** (as of latest available data from their official cards, March 2026 timeframe)

Here is a structured comparison of the three models based on their official model cards. All are strong open-weight (or open-model) LLMs optimized for reasoning, coding, math, and agentic tasks, but they differ significantly in scale, architecture, efficiency, and specialties.

| Aspect                  | **NVIDIA Nemotron-3 Super 120B-A12B** | **NVIDIA Nemotron-Cascade-2-30B-A3B** | **Qwen/Qwen3.5-27B** |
|-------------------------|---------------------------------------|---------------------------------------|----------------------|
| **Parameter Count**    | 120B total / 12B active (LatentMoE hybrid: Mamba-2 + MoE + Attention + Multi-Token Prediction) | 30B total / 3B active (MoE) | 27B (hybrid: Gated Delta Networks + sparse MoE; includes Vision Encoder) |
| **Context Length**     | Up to 1M tokens (RULER-100 @ 1M: 91.75%) | 262K tokens (NIAH @ 1M subset strong) | 262K native, extensible to ~1M+ (YaRN) |
| **Key Strengths (Benchmarks)** | Agentic/tool use (SWE-Bench OpenHands: 60.47%; TauBench: 61.15%), long-context reasoning, math (AIME25: 90.21%; HMMT: 93.67%), LiveCodeBench: 81.19%, MMLU-Pro: 83.73%. Configurable reasoning traces. | Exceptional math/coding (IMO 2025 gold: 35 pts; AIME 2025: 92.4; IOI gold; LiveCodeBench: 87.2; MMLU-Pro: 79.8; GPQA-Diamond: 76.1). Strong agentic (ArenaHard: 83.5). | Broad excellence: MMLU-Pro: 86.1; GPQA Diamond: 85.5; SWE-Bench: 72.4; LiveCodeBench: 80.7; high multilingual (201 languages) and multimodal (images/video: MMMU 82.3+, VideoMME 87%). Strong tool calling/IFBench. |
| **Modalities**         | Text-only | Text-only (strong tool calling) | Multimodal (text + image + video) |
| **Special Features**   | Hybrid architecture for efficiency/speed; configurable "thinking" mode; NVIDIA-optimized (NVFP4, TRT-LLM, vLLM/SGLang). | Thinking/instruct modes (<think> tags); excellent efficiency; tool-calling format. | Native multimodal; thinking mode; strong multilingual/global coverage. |
| **License & Deployment** | NVIDIA terms (commercial-ready); supports vLLM, SGLang, TRT-LLM; min ~8× H100 for full perf. | NVIDIA Open Model License; vLLM/Transformers; highly quantized variants available; very efficient. | Apache 2.0 (highly permissive); vLLM/SGLang/etc.; multimodal support. |
| **Best For**           | Ultra long-context RAG/agentic workflows, high-volume production on NVIDIA hardware. | Efficient high-performance reasoning, math, competitive programming, and agentic tasks on modest hardware. | Multimodal tasks, multilingual use cases, or when broad capabilities + vision are needed in a compact package. |

**Quick takeaway on these three**:  
- **Nemotron Super** shines for scale + extreme context/agentic depth.  
- **Cascade** offers outstanding price/performance (active params) for math/coding agents.  
- **Qwen3.5-27B** is the most versatile generalist here, especially if you need vision or 201-language support, and often leads or matches larger models on many benchmarks despite its size.

### Properties to Consider When Selecting an LLM
Model selection is **not** just about "biggest = best." It depends on your specific query/problem, constraints, and goals. Key properties (drawn from practitioner guides and industry analyses) include:

- **Task alignment** — Does the query need reasoning, math, code generation, tool use/agents, long documents, chat, creativity, or multimodal inputs?
- **Performance/quality** — Accuracy, coherence, instruction-following, low hallucination on your domain.
- **Efficiency & practicality** — Speed (latency/throughput), cost per token, hardware needs.
- **Deployment factors** — Open weights vs. API, licensing (commercial use?), privacy/data control, fine-tuning ease.
- **Safety & risks** — Alignment, guardrails, bias mitigation, compliance.
- **Other** — Multilingual support, modalities (vision/video), data freshness/cutoff, customizability.

### Key Technical Specs to Consider
These directly impact what a model can handle and how feasible it is to run:

- **Parameter count (total & active for MoE)**: Larger generally = smarter, but MoE models (like all three here) activate far fewer params for efficiency.
- **Context length/window**: Critical for long prompts, RAG, or multi-document tasks (e.g., 1M vs. 262K).
- **Architecture**: Dense vs. MoE/hybrid (affects speed/memory); extras like MTP or Gated DeltaNet for faster/better generation.
- **Precision/quantization support**: BF16/FP8/INT4/etc. — drastically reduces VRAM and speeds up inference with minimal quality loss.
- **Inference speed/latency/throughput**: Tokens per second on your hardware.
- **Hardware/VRAM requirements**: GPU count, memory footprint.
- **Tokenizer/vocab size**: Affects multilingual or code performance.
- **Modalities**: Text-only vs. vision/video/audio.

### Common Evaluation Categories & Benchmarks
Models are tested on standardized suites that map to real capabilities. Relevant ones for selection:

- **General knowledge/reasoning**: MMLU / MMLU-Pro, GPQA-Diamond, HLE.
- **Math**: GSM8K, AIME, MATH, HMMT, IMO/IOI (competitive level).
- **Coding**: HumanEval, LiveCodeBench (fresh problems), SWE-Bench (real GitHub issues), BFCL (function calling).
- **Agentic/tool use & instruction following**: TauBench, Terminal Bench, IFBench, ArenaHard, BFCL.
- **Long-context**: RULER, NIAH (needle-in-haystack), LongBench.
- **Multilingual**: MMLU-ProX, WMT translation, C-Eval.
- **Multimodal**: MMMU, VideoMME, MathVista, etc.
- **Conversational quality**: Chatbot Arena / MT-Bench (human preference/ELO).

**Note**: Benchmarks are useful but not perfect — always test on your actual prompts (or a representative sample) because real-world performance can differ.

### Goal: What Information Is Useful to Select the Right Model for a Given Problem/Query/Prompt?
To decide between models (or any others), gather/use this framework:

1. **Classify the task** — Math-heavy? Code/agentic? Needs images/video? Long document/RAG? Multilingual? Simple chat? High-stakes reasoning?
2. **Identify constraints** — Hardware budget? Latency tolerance? Cost per inference? Privacy/compliance needs? Deployment environment (NVIDIA GPUs? cloud? local)?
3. **Match to model strengths via benchmarks + specs**:
   - Ultra-long context or heavy agentic workflows on NVIDIA hardware → **Nemotron Super**.
   - Efficient math/coding/agentic on modest resources → **Nemotron Cascade**.
   - Vision/multimodal, broad multilingual, or strong generalist performance → **Qwen3.5-27B**.
4. **Practical checks** — License allows your use? Model supports required tools/structured output? Quantized versions fit your VRAM? Fine-tuning needed?
5. **Test empirically** — Run a few representative prompts on each (via playgrounds, vLLM, etc.) and compare quality, speed, and cost.
6. **Trade-off analysis** — E.g., is the extra capability of a larger model worth the higher latency/cost, or does a smaller MoE win on efficiency?

**Example**: For a long-document RAG agent with tool use on NVIDIA GPUs → prioritize Nemotron Super (1M context + agentic benchmarks). For a math competition solver with limited hardware → Cascade. For an image + text analysis query in multiple languages → Qwen3.5-27B.

This approach moves beyond "vibes" or raw size to data-driven, use-case-specific selection. If you have a specific query or additional constraints (e.g., hardware, budget, domain), I can help narrow it further or suggest testing steps!
