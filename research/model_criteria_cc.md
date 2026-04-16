# LLM Model Selection Criteria

A comprehensive framework for selecting large language models based on technical specifications, evaluation metrics, and operational constraints.

---

## 1. Technical Specifications

### Core Architecture Parameters

| Specification | Description | Impact |
|--------------|-------------|--------|
| **Parameter Count** | Total trainable weights (e.g., 7B, 27B, 120B) | Correlates with reasoning capability, knowledge breadth, and inference cost |
| **Architecture Type** | Dense Transformer vs. Mixture of Experts (MoE) | MoE offers speed/quality tradeoffs; Dense is more predictable |
| **Context Window** | Maximum token sequence length (8K, 32K, 128K, 1M+) | Determines document size, conversation history retention |
| **Knowledge Cutoff** | Training data end date | Critical for recency-dependent tasks (APIs, current events, recent research) |
| **Precision** | FP32, FP16, BF16, INT8, INT4 | Affects hardware requirements, latency, and quantization artifacts |

### Deployment Specifications

| Specification | Description |
|--------------|-------------|
| **VRAM Requirements** | Memory needed for inference at different batch sizes |
| **Quantization Support** | Available bit-widths (4-bit, 8-bit) for memory optimization |
| **Inference Speed** | Tokens/second at various batch sizes |
| **First Token Latency** | Time to generate first token (critical for interactive apps) |

---

## 2. Evaluation Categories

### Reasoning & Cognitive Abilities

| Benchmark | What It Measures | Target Score Range |
|-----------|------------------|-------------------|
| **MMLU** (Massive Multitask Language Understanding) | 57 academic subjects (STEM, humanities, social sciences) | 70-90% |
| **MMLU-Pro** | Advanced reasoning across 15 domains with multiple choice | 50-80% |
| **GSM8K** | Grade school math word problems (multi-step reasoning) | 80-95% |
| **BigBench Hard** | Complex reasoning tasks requiring planning | 40-80% |
| **ARC-Challenge** | Scientific reasoning and common sense | 80-95% |

### Coding & Technical Tasks

| Benchmark | What It Measures | Target Score Range |
|-----------|------------------|-------------------|
| **HumanEval** | Python function completion from docstrings | 60-90% pass@1 |
| **MBPP** | Programming problems (Python, C++, Java) | 70-90% pass@1 |
| **SWE-bench** | Real GitHub issue resolution (hard) | 15-40% resolved |
| **LiveCodeBench** | Competitive programming (Codeforces, LeetCode) | 20-50% |

### Language & Understanding

| Benchmark | What It Measures |
|-----------|------------------|
| **TruthfulQA** | Hallucination detection and factuality |
| **HellaSwag** | Commonsense reasoning |
| **BoolQ** | Boolean question answering |
| **PIQA** | Physical commonsense reasoning |

### Specialized Domains

| Domain | Key Benchmarks |
|--------|---------------|
| **Mathematics** | MATH (competition-level), GSM8K, MATH-500 |
| **Science** | ARC-Challenge, SciQ, MMLU (Science subset) |
| **Multilingual** | Flores, Multi-MMLU, XCOPA |
| **Long Context** | Needle In A Haystack (NIAH), RULER |

---

## 3. Model Card Analysis Checklist

When evaluating a model card, extract these critical pieces of information:

### Identity & Licensing
- [ ] Model name and version
- [ ] Base architecture (e.g., Llama 3, Mixtral, custom)
- [ ] License type (Apache 2.0, MIT, CC-BY-NC, custom)
- [ ] Commercial use permissions
- [ ] Attribution requirements
- [ ] Open weights status

### Intended Use
- [ ] Primary use cases (QA, coding, chat, completion)
- [ ] Target domains (general, biomedical, legal, code)
- [ ] Known limitations and failure modes
- [ ] Languages supported (beyond English)

### Performance Profile
- [ ] Benchmark scores with confidence intervals
- [ ] Comparison to baseline/competitor models
- [ ] Hallucination rates and safety filters
- [ ] Jailbreak resistance metrics

### Operational Requirements
- [ ] Hardware requirements (GPU type, VRAM)
- [ ] Recommended quantization strategies
- [ ] Inference latency benchmarks
- [ ] Throughput (tokens/sec) at various batch sizes
- [ ] Memory footprint for different precisions

---

## 4. Selection Decision Framework

### By Task Complexity

| Task Type | Recommended Model Size | Key Benchmarks |
|-----------|----------------------|----------------|
| **Simple QA / Summarization** | 7B-30B | MMLU > 65%, low latency |
| **General Chat / Conversation** | 20B-70B | TruthfulQA, dialogue benchmarks |
| **Complex Reasoning** | 70B-120B+ | MMLU > 80%, GSM8K > 85% |
| **Multi-step Logic / Planning** | 70B+ | BigBench Hard, MMLU-Pro |
| **Professional Coding** | 30B-120B | HumanEval > 70%, SWE-bench |
| **Mathematical Reasoning** | 70B+ | MATH benchmark, GSM8K |

### By Context Requirements

| Context Size | Model Considerations |
|-------------|---------------------|
| **< 4K tokens** | Any modern model; focus on quality/latency tradeoff |
| **4K-32K tokens** | Standard Llama-3-style context windows |
| **32K-128K tokens** | Check explicit long-context support; verify NIAH scores |
| **128K-1M+ tokens** | Specialized architectures (e.g., LongChat, Grok-1L); verify retrieval accuracy |

### By Operational Constraints

| Constraint | Recommendation |
|-----------|----------------|
| **Low Latency (<100ms)** | 7B-30B quantized; consider MoE architectures |
| **High Throughput (>50 t/s)** | MoE models or smaller dense models with batching |
| **Cost Sensitive** | 7B-27B quantized; consider Cascade models |
| **Accuracy Critical** | 70B+ dense models; ensemble approaches |
| **Self-Hosted / Air-gapped** | Check VRAM: 7B=12GB, 70B=140GB (FP16) |

---

## 5. Comparative Model Analysis

### Reference Models for Comparison

#### **NVIDIA Nemotron-3 Super 120B**
- **Profile**: Frontier reasoning model, ~120B parameters
- **Strengths**: High MMLU scores, complex reasoning, instruction following
- **Use Cases**: Complex QA, multi-step reasoning, professional tasks
- **Trade-offs**: High VRAM (~240GB), slower inference, expensive API

#### **NVIDIA Nemotron-Cascade 2 30B**
- **Profile**: Mid-tier MoE or dense model (~30B effective)
- **Strengths**: Balanced cost/quality, fast inference
- **Use Cases**: General chat, summarization, light reasoning
- **Trade-offs**: Lower accuracy on hard reasoning tasks

#### **Qwen3.5-27B**
- **Profile**: Mid-sized dense model with strong multilingual support
- **Strengths**: Code generation, multilingual (Chinese/English), cost-effective
- **Use Cases**: Coding assistance, multilingual applications, general chat
- **Trade-offs**: Smaller context window than frontier models

---

## 6. Evaluation Score Interpretation

### Benchmark Score Ranges

**MMLU (57 subjects)**
- 90-100%: SOTA/Expert level (o1, o1-preview)
- 85-89%: Frontier models (GPT-4-turbo, Claude 3.5)
- 80-84%: Strong 70B+ class (Llama-3-70B, Nemotron-3)
- 70-79%: Good 30B class (Llama-3-8B, Qwen-2.5-32B)
- <70%: Entry-level or specialized small models

**HumanEval (Python)**
- 85-100%: Expert coding models
- 70-84%: Strong coding capability (Llama-3-70B)
- 50-69%: Basic coding (Llama-3-8B)
- <50%: Not recommended for coding tasks

**GSM8K (Math)**
- 90%+: Expert reasoning
- 80-89%: Strong reasoning (70B+ class)
- 70-79%: Moderate reasoning
- <70%: Struggles with multi-step math

---

## 7. Practical Selection Workflow

Given a **problem/query**, follow this decision tree:

1. **Determine Context Window**
   - Calculate token count of input + expected output
   - Select models with context window > 2x this value (safety margin)

2. **Assess Reasoning Requirements**
   - Simple lookup/QA → 7B-30B
   - Multi-step logic → 70B+
   - Coding → Check HumanEval > 60%

3. **Check Operational Constraints**
   - Self-hosted: Does it fit in available VRAM?
   - API: Is latency acceptable? Is cost sustainable?

4. **Verify Domain Suitability**
   - Check specific benchmark scores for your domain
   - Verify language support if non-English

5. **Test with Representative Prompts**
   - Run 5-10 sample queries
   - Evaluate on accuracy, speed, cost

---

## 8. Red Flags in Model Cards

Avoid models that lack:
- [ ] Clear benchmark scores (vague "state-of-the-art" claims without numbers)
- [ ] Knowledge cutoff date
- [ ] Hardware requirements or VRAM specs
- [ ] Licensing clarity (vague "contact us" for commercial use)
- [ ] Safety/alignment documentation
- [ ] Comparison to established baselines (MMLU, HumanEval)

---

## 9. Quick Reference: Model Tier Classification

| Tier | Parameters | MMLU | Context | VRAM | Use Case |
|------|-----------|------|---------|------|----------|
| **Nano** | <7B | 60-70% | 8K-32K | 12-24GB | Edge, mobile, cheap inference |
| **Small** | 7B-30B | 70-80% | 32K-128K | 24-48GB | Chatbots, summarization, light coding |
| **Mid** | 30B-70B | 80-85% | 128K+ | 80-140GB | Complex QA, professional coding, reasoning |
| **Large** | 70B-120B | 85-90% | 128K-1M | 140-240GB | Expert reasoning, research, complex planning |
| **Frontier** | 120B+ | 90%+ | 1M+ | 240GB+ | SOTA reasoning, agentic workflows |

---

*Document generated: 2026-04-16*
*Version: 1.0*
