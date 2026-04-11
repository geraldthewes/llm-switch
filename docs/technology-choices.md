The llm-switch project should use

https://github.com/maximhq/bifrost
And be implemented in golang

Additional technology choices:

1. Orchestrator Model (1B parameter)
   - Fine-tuned Qwen 2.5 0.5B-Instruct or Llama 3.2 1B for intent and complexity classification
   - Achieves sub-40ms response times for classification
   - Provides 10x cost reduction and speed improvement over frontier models

2. Statistical Routing (NormStat/VecStat)
   - NormStat: Identifies shifts in activation magnitude for coarse-grained routing
   - VecStat: Preserves directional information in latent space for finer-grained distinctions
   - Training-free intent classification with negligible overhead

3. Hardware Telemetry Integration
   - Integrate with vLLM's and llama.cpp /metrics endpoint to monitor VRAM availability and queue depth
   - Hardware-aware routing decisions (e.g., route medium complexity tasks to RTX when DGX queue > 10)

4. Trace Accumulation (langfuse)
   - Asynchronously push request/response pairs and user feedback signals to langfuse
   - Develop reasoning engine to build persistent profiles for coding tasks
   - Identify which models consistently yield successful tool calls

5. AutoResearch Loop
   - Background agent utilizing AutoResearch pattern on dual 2080 system
   - Reviews langfuse traces from previous 24 hours to identify routing failures
   - Performs 5-minute training experiments to refine orchestrator classification boundaries
   - Judges success based on val_bpb reduction for next day's task distribution

6. Analyze each model model card to understand it's capabilities (context length, expertise, capabilities)

