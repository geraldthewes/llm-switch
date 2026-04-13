# Technology Choices

## Section 1: Core Dependencies
1. The llm-switch project should use https://github.com/maximhq/bifrost
2. And be implemented in golang

## Section 2: Orchestrator Model (1B parameter)
- Fine-tuned Qwen 2.5 0.5B-Instruct or Llama 3.2 1B for intent and complexity classification
- Achieves sub-40ms response times for classification
- Provides 10x cost reduction and speed improvement over frontier models

## Section 3: Statistical Routing (NormStat/VecStat)
- NormStat: Identifies shifts in activation magnitude for coarse-grained routing
- VecStat: Preserves directional information in latent space for finer-grained distinctions
- Training-free intent classification with negligible overhead

## Section 4: Hardware Telemetry Integration
- Integrate with vLLM's and llama.cpp /metrics endpoint to monitor VRAM availability and queue depth
- Hardware-aware routing decisions (e.g., route medium complexity tasks to RTX when DGX queue > 10)

## Section 5: Trace Accumulation (langfuse)
- Asynchronously push request/response pairs and user feedback signals to langfuse
- Develop reasoning engine to build persistent profiles for coding tasks
- Identify which models consistently yield successful tool calls

## Section 6: AutoResearch Loop
- Background agent utilizing AutoResearch pattern on dual 2080 system
- Reviews langfuse traces from previous 24 hours to identify routing failures
- Performs 5-minute training experiments to refine orchestrator classification boundaries
- Judges success based on val_bpb reduction for next day's task distribution

## Section 7: Model Card Analysis
- Analyze each model model card to understand it's capabilities (context length, expertise, capabilities)

## Section 8: Deployment Environment
- llm-switch designed to be run inside a docker container, to be deployed on a nomad cluster infrastructure with access to consul and vault

## Section 9: Available Services in Cluster
- Current services used in the cluster. You do not need to use, if needed you can recommend other choices, but you need a reason to do so:
eph
ceph-csi
clickhouse
cluster-docs
consul
dgraph
dnsmasq
docker-registry
fabio
glm-flash
gpt-oss-20b
gpu
granite-docling
homestorage-nfs
hunyuan-ocr
jaeger
langfuse
litellm
llama-swap-gx10
llama-swap-gx10-d8ce
minimax-m2.5-rpc
nemotron-3-super
neo4j
nomad-build-service
nsq
olah
ollama
openwebui
pcopy
perplexica
postgres
prometheus
pypiserver
python-executor
qdrant
qwen35-122b
qwen35-122b-gguf
rembg
searxng
searxng-cache
tool-orchestrator
valkey
vault
video-transcription
vllm
vllm-nemotron