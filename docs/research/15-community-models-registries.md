# 15 — Community models and registries

Scope: this report fills the Year96 gap around community/registry evidence: Hugging Face model choices for core roles, Docker Hub/container-image supply chain, Reddit/Hacker News practitioner signals, and public Facebook/Meta sources. I treat the current architecture as the baseline: §6.2.1 quarantined membrane, §6.3 scope-effect cascade, §6.4 memory, §6.5 hub/voice, §6.10 assurance, §6.11 L4 model-routing promotion, §8 provider catalog, and §9 deployment. Core means permissive licenses only: MIT, Apache-2.0, BSD, ISC, public domain. Llama/Gemma terms, non-commercial Creative Commons, OpenRAIL/RAIL, custom research licenses, and acceptable-use-policy-restricted weights are excluded from the core even if technically strong.

## TL;DR for the Year96 architect

- Add an explicit **Model Profile** to §8/§9. Today the architecture names vLLM/SGLang/llama.cpp/Ollama but does not choose default weights. The core should ship `solo` and `cluster` profiles, each with model IDs, license metadata, benchmark gates, and promotion status.
- Default embeddings: **Qwen/Qwen3-Embedding-0.6B** for solo and **Qwen/Qwen3-Embedding-4B or 8B** for cluster, all Apache-2.0 on HF, multilingual, MTEB-oriented, and current in 2026 [1]. Keep **BAAI/bge-m3** (MIT) as the cheaper multilingual fallback [3].
- Default reranker: **BAAI/bge-reranker-v2-m3** (Apache-2.0) for CPU/GPU solo; **Qwen/Qwen3-Reranker-4B/8B** for cluster only after latency/cost gates. bge-reranker-v2-m3 is multilingual and easy to serve [4].
- Default classifier/router family: **answerdotai/ModernBERT-base** (Apache-2.0, ONNX artifacts, 4.25M monthly downloads at fetch time) fine-tuned per Year96 route label; small DeBERTa/ModernBERT heads are the right stage-1/2 cascade shape [5].
- Prompt-injection guard default: **protectai/deberta-v3-base-prompt-injection-v2** (Apache-2.0, ONNX, 2026 update) as a cheap membrane/hub classifier [12]. Exclude Meta Prompt Guard from core if under Llama community terms; use only as external integration.
- PII/NER default: **urchade/gliner_multi_pii-v1** (Apache-2.0, EN/FR/DE/ES/PT/IT) plus Microsoft Presidio (MIT, not re-verified here) rules. **Isotonic ai4privacy v2 is CC-BY-NC-4.0 and excluded from core** despite being a useful benchmark [13][14].
- Local LLM defaults: solo = **microsoft/Phi-4-mini-instruct** (MIT) or **Qwen/Qwen3-4B** (Apache-2.0); cluster = **Mistral-7B-Instruct-v0.3** (Apache-2.0), **allenai/OLMo-2-1124-7B-Instruct** (Apache-2.0), **IBM Granite 4.0 micro** (Apache-2.0) [6][7][8][9]. Exclude Llama/Gemma from core; permit external providers.
- Vision verifier defaults: solo = **microsoft/Florence-2-base** (MIT) for OCR/caption/localization and **HuggingFaceTB/SmolVLM-500M-Instruct** (Apache-2.0) for small VLM checks; cluster can trial permissive InternVL/Phi-Vision variants only after license verification. **Qwen2.5-VL-3B uses `qwen-research`; excluded from core** even though it has strong screenshot-agent benchmarks [10][11].
- Speech defaults: STT = **openai/whisper-large-v3** (Apache-2.0, multilingual) or smaller Whisper variants; TTS = **hexgrad/Kokoro-82M** (Apache-2.0) for solo and **parler-tts-mini-v1** or **Dia-1.6B** (Apache-2.0) for richer English voices; **Piper voices** (MIT) for lightweight multilingual/offline voices [15][16][17][18][19].
- Serving/runtime licenses are compatible: vLLM and SGLang Apache-2.0; llama.cpp, Ollama, Infinity, ONNX Runtime MIT; text-embeddings-inference and sentence-transformers Apache-2.0 [20]-[27]. Architecture Appendix/§8 already has most of these; add Infinity/ONNX/sentence-transformers to the model-serving row.
- Container policy must become a first-class repo gate. Docker Official/Verified badges are not enough. Pin every image by digest, mirror into a Year96 registry, verify cosign/Sigstore attestations where available, generate Year96 SBOMs for every image, and fail CI on prohibited licenses/CVEs/no provenance.
- Bitnami’s 2025 public-catalog change is real enough to affect Helm choices: do not depend on public Bitnami charts/images for core. Use upstream/vendor images, official Helm charts, or Year96-built images; treat `bitnamilegacy` as emergency-only [33].
- Practitioner signals converge on the same architecture lessons: long-running agents need hibernate/wake state, budget and kill-switches; MCP/tool metadata is hostile input; context rot is real; memory is poisonable; swarms amplify costs. Year96 already covers much in §6.2.1, §6.3, §6.4, §6.10, but lacks concrete model/container policy and a “tool-description integrity” gate.

## Landscape

### Hugging Face model registry: candidates by core role

**1) Text embeddings for retrieval, dedupe and memory search.**

| Candidate | License verified | Size | Benchmark signal | CPU? | Multilingual? | Verdict |
|---|---:|---:|---|---|---|---|
| Qwen/Qwen3-Embedding-0.6B | Apache-2.0 in HF API/card [1] | 0.6B | Qwen3 embedding release targets MTEB/MMTEB-class retrieval; HF tags `arxiv:2506.05176` | Yes with ONNX/TEI/quantization, but GPU preferred for high QPS | Yes | Adopt solo default |
| Qwen/Qwen3-Embedding-4B/8B | Apache-2.0 family [1] | 4B/8B | higher MTEB/MMTEB headroom | No for normal CPU; GPU pool | Yes | Adopt cluster default after latency gates |
| BAAI/bge-m3 | MIT verified on HF [3] | XLM-R size, multi-vector/sparse/dense | strong MIRACL/MKQA/BEIR-style card assets; 36M downloads at fetch time | Yes, slower; ONNX artifacts | Yes | Adopt fallback |
| nomic/mixedbread/e5 derivatives | license varies | small-large | useful but must be per-checkpoint verified | Often | varies | Watch/Trial |

Year96 needs one embedding interface for §6.3 stage 1, §6.2.1 dedupe/cluster, and §6.4 memory search; the same model should not be used blindly for all three. Dedupe emphasizes stable semantic locality and false-positive control; memory search emphasizes recall and time-aware facts; scope-effect retrieval emphasizes weak-signal discovery. The promotion suite should therefore include MTEB, multilingual MTEB/MIRACL, a duplicate-news cluster corpus, and a Year96 dormant-thread recall corpus.

**2) Rerankers / late interaction.**

| Candidate | License verified | Size | Benchmark signal | CPU? | Multilingual? | Verdict |
|---|---:|---:|---|---|---|---|
| BAAI/bge-reranker-v2-m3 | Apache-2.0 HF [4] | XLM-R sequence classifier | BEIR/MIRACL/CMTEB assets on card; 17M downloads | Yes for low QPS | Yes | Adopt solo |
| Qwen/Qwen3-Reranker-0.6B | Apache-2.0 HF [2] | 0.6B generative ranker | Qwen3 reranker paper/card; text-ranking tag | Borderline CPU; better GPU | Yes | Trial |
| Qwen/Qwen3-Reranker-4B/8B | Apache-2.0 family [2] | 4B/8B | higher multilingual RAG ranking | No | Yes | Adopt cluster if p95 OK |
| ColBERT-style late interaction (ColBERTv2/AnswerAI variants) | commonly MIT/Apache, verify per repo | 100M-500M | high recall/late interaction | CPU index build possible; GPU better | Mostly EN | Watch |

Late interaction is useful for §6.3 top-k before LLM adjudication and §6.4 rehydration. Do not start with a ColBERT index as the default: it complicates storage and scaling. Start cross-encoder/generative rerankers, then trial ColBERT for high-value memory/search domains.

**3) Small classifiers/routers for cascade stages 1-2.**

- **answerdotai/ModernBERT-base**: Apache-2.0 in HF API, English, long-context, ONNX quantized artifacts, high downloads [5]. Use as the default base for `route`, `rights`, `scope-risk`, `thread-type`, `needs-llm-adjudication`, and `wake-priority` heads. CPU: yes with ONNX/int8. Multilingual: no.
- Multilingual alternatives: XLM-RoBERTa/DeBERTa multilingual fine-tunes. Trial only after license/data card review.
- Do not use generic closed LLM classification for cheap stages: it destroys the cost cascade and weakens §6.3 calibration.

**4) Prompt-injection / jailbreak detectors.**

- **protectai/deberta-v3-base-prompt-injection-v2**: Apache-2.0, English, prompt-injection/security tags, ONNX artifact, updated 2026 [12]. Adopt for membrane ingress (§6.2.1), MCP/tool descriptions (§6.5), and retrieved memory before it reaches an LLM.
- Meta/Llama Prompt Guard: strong and widely discussed, but if the actual weights are under Llama terms, mark **Excluded-license (core)**. It can be an external integration where legal accepts the terms.
- Classifiers are not sufficient. Microsoft’s MCP warning is specifically about **tool-description poisoning**; Year96 needs a manifest hash/signature gate for tool schemas plus runtime context separation, not just text scoring [36].

**5) PII / NER detectors.**

- **urchade/gliner_multi_pii-v1**: Apache-2.0 verified, GLiNER-based, multilingual EN/FR/DE/ES/PT/IT [13]. Adopt as the learned PII recognizer for §6.2.1.
- **Isotonic/deberta-v3-base_finetuned_ai4privacy_v2**: CC-BY-NC-4.0 verified; **Excluded-license (core)** [14]. Keep as benchmark-only if redistribution rights allow.
- Combine learned NER with deterministic detectors (emails, phones, credentials, keys), Presidio-style recognizers, and tenant-specific entity allow/deny lists. Every PII decision becomes evidence in the membrane event.

**6) Small/medium local LLMs for adjudication, fact extraction, offline/sensitive work.**

| Candidate | License verified | Size | CPU? | Multilingual? | Best role | Verdict |
|---|---|---:|---|---|---|---|
| microsoft/Phi-4-mini-instruct | MIT HF [7] | mini (~3-4B class) | Yes quantized; single GPU better | Yes languages listed | solo adjudicator, tool-light fact extraction | Adopt solo |
| Qwen/Qwen3-4B | Apache-2.0 HF [6] | 4B | Yes quantized; single GPU | Strong multilingual | solo general local LLM | Adopt solo/cluster |
| mistralai/Mistral-7B-Instruct-v0.3 | Apache-2.0 HF [9] | 7B | Possible quantized; GPU preferred | Mostly EN/EU | cluster/default 7B | Adopt |
| allenai/OLMo-2-1124-7B-Instruct | Apache-2.0 HF [8] | 7B | quantized possible | EN | auditable/open-data-sensitive work | Trial |
| ibm-granite/granite-4.0-h-micro | Apache-2.0 HF [6] | micro MoE/hybrid | GPU preferred | Enterprise/code | structured extraction | Trial |
| Llama, Gemma | custom/community terms or Gemma terms | many | yes | yes | strong external providers | Excluded-license (core) |

The issue is not only license. Qwen/Mistral/Phi/Granite/OLMo are permissive weights, but data transparency varies. OLMo is valuable for auditability; Qwen is valuable for multilingual quality; Phi is efficient; Granite has enterprise posture; Mistral is mature in vLLM/SGLang. Year96 §6.11 L4 should let different thread types select champions.

**7) Vision-language models for screenshot proof verification.**

- **microsoft/Florence-2-base**: MIT verified, small vision caption/OCR/localization model, good for deterministic-ish “is the expected UI element visible?” prechecks [11]. Adopt solo.
- **HuggingFaceTB/SmolVLM-500M-Instruct**: Apache-2.0 verified, 500M, small VLM [10]. Adopt for cheap screenshot semantic checks.
- **Qwen/Qwen2.5-VL-3B-Instruct**: strong screenshot/agent benchmarks on the model card (ScreenSpot, AndroidWorld) but model card says `license_name: qwen-research`; **Excluded-license (core)** [28]. This is an example where the best technical fit fails the license rule.
- Cluster should evaluate permissive InternVL/Phi-Vision/Florence-large variants per exact checkpoint. Vision verifiers should be redundant: OCR+DOM+pixel+VLM; a VLM alone cannot attest §6.10 proof.

**8) Speech-to-text and text-to-speech for voice gateways.**

- **openai/whisper-large-v3**: Apache-2.0 HF, multilingual ASR, adopt cluster/STT default; smaller Whisper/Distil-Whisper for solo [15].
- **rhasspy/piper-voices**: MIT verified, ONNX, many languages; adopt solo/offline TTS where naturalness is secondary [19].
- **hexgrad/Kokoro-82M**: Apache-2.0 verified, 82M, high downloads; adopt solo English high-quality TTS [16].
- **parler-tts/parler-tts-mini-v1** and **nari-labs/Dia-1.6B**: Apache-2.0 verified; trial richer expressive TTS, mostly English [17][18].

### Serving and runtime registry

The runtime pieces fit the permissive core: vLLM Apache-2.0 [20], SGLang Apache-2.0 [21], llama.cpp MIT [22], Ollama MIT [23], text-embeddings-inference Apache-2.0 [24], Infinity MIT [25], ONNX Runtime MIT [26], sentence-transformers Apache-2.0 [27]. Add explicit provider choices: `TextEmbeddingsInferenceProvider` for embeddings/rerankers on GPU, `InfinityProvider` for mixed CPU/GPU embedding/reranker serving, `OnnxRuntimeProvider` for classifiers and speech/PII CPU paths, `VllmProvider`/`SglangProvider` for cluster LLMs, `LlamaCppProvider`/`OllamaProvider` for solo quantized GGUF.

### Docker Hub and container supply chain

Docker’s trusted-content docs define Docker Official Images, Hardened Images/charts, Verified Publisher images and Docker-Sponsored OSS images [29]. Docker Hardened Images (DHIs) add signed attestations, SLSA Build Level 3 practices, OCI metadata, SBOMs, Docker Scout and cosign verification [30][31]. Year96 should not assume every Docker Hub image has DHI-grade provenance. Treat badges as input signals, not proof.

| Image | Publisher | Official/verified? | Signing/provenance/SBOM state | License/base notes | Year96 policy |
|---|---|---|---|---|---|
| postgres | Docker Official Image `_ /postgres` | Official [34] | DOI provenance varies; DHI alternative may have attestations | PostgreSQL License; Debian/Alpine variants | Use official, pin digest; consider DHI/Chainguard for prod |
| nats | Docker Official Image `_ /nats` | Official [35] | DOI; generate own SBOM | NATS Apache-2.0; likely scratch/alpine variants | Adopt, pin digest |
| temporalio/server, admin-tools, cli/dev server | temporalio | Vendor image, not DOI [37] | Verify if cosign exists; otherwise mirror+SBOM | Temporal server MIT; base image license must be scanned | Adopt only via mirror and SBOM gate |
| openfga/openfga | OpenFGA | Vendor image [32] | Need cosign/SBOM verification in CI | Apache-2.0 binary; base scanned | Adopt, pin/mirror |
| apicurio/apicurio-registry | Apicurio, often Quay/GHCR | Not DOI | Prefer upstream release provenance | Apache-2.0 | Mirror; do not rely only on Docker Hub |
| otel/opentelemetry-collector-contrib | OpenTelemetry | CNCF/vendor | Upstream release provenance; generate SBOM | Apache-2.0 | Adopt, pin/mirror |
| jaegertracing/all-in-one | Jaeger/CNCF | Vendor | Upstream provenance; SBOM required | Apache-2.0 | Dev only; prod uses collector+storage components |
| clickhouse/clickhouse-server or `_ /clickhouse` | ClickHouse / DOI | Official DOI exists [38] | Generate SBOM; DHI if available | Apache-2.0 server; base scanned | Adopt for observability analytics |
| chrislusf/seaweedfs | project maintainer | Not DOI/verified enough | likely no standard attestations | Apache-2.0 app; scan base | Prefer building Year96 image or upstream signed alternative |
| vllm/vllm-openai | vLLM project | Vendor; Hub overview empty [39] | GPU images often lack full attestations; scan CUDA bases | Apache-2.0 app; NVIDIA CUDA image terms | Mirror; GPU base-license review |
| ghcr.io/berriai/litellm or litellm/litellm | LiteLLM | Vendor | verify per release | MIT core; enterprise dirs separate | Build from source or mirror verified image |

**2025-2026 change to account for:** Bitnami public images moved/restructured around Aug/Sep 2025; free production-safe versioned images were reduced/archived, with `bitnamilegacy` carrying no updates and paid Bitnami Secure Images positioned for production [33]. This breaks Helm charts that quietly use Bitnami PostgreSQL/Redis/etc. Year96 must be **Bitnami-free by default**: upstream charts, official images, or Year96-owned builds.

### Reddit, HN and practitioner signals

Direct Reddit pages are inconsistently accessible/searchable without login or through SEO mirrors; I used public web-search results and primary/near-primary technical posts where available. Record this limitation: r/LocalLLaMA, r/AI_Agents, r/ClaudeAI and r/ExperiencedDevs are valuable signals, but exact thread retrieval should be automated later via Reddit API under policy.

| Signal (2025-2026) | Link | Implication for Year96 | Covered today? |
|---|---|---|---|
| Long-running agents need hibernate/wake, persistent state, and human strategic checkpoints. Meta REA explicitly runs days/weeks with hibernate-and-wake [40]. | Meta REA | Make hibernate/wake a first-class runtime primitive, not a prompt trick. | Mostly §6.9 actors/timers, §6.3 wake; add model profile for REA-like ML duties. |
| Agent platforms encode senior domain expertise into reusable skills; Meta capacity agents use standardized tool interfaces and recover hundreds of MW [41]. | Meta Capacity | Skills/playbooks need versioning, eval and tool manifests. | §6.11 + R11; add tool-description hash gate. |
| MCP/tool-description poisoning is a concrete production risk; Microsoft warns malicious tool descriptions can exfiltrate data [36]. | Microsoft MCP | Treat tool metadata as hostile membrane input. Sign/pin tool manifests and run injection classifiers on descriptions. | Partly §6.2.1 hostile feeds; gap in §6.5 MCP registry. |
| Context rot: longer context degrades quality before window limit; Chroma/Hamel report lost signal [42][43]. | Chroma/Hamel | Memory cannot be “dump more tokens.” Rehydration must be measured for precision/positioning. | Strong §6.4; add context-rot eval gate to model promotion. |
| Memory poisoning/persistent false facts are emerging attack class [44]. | THN MemGhost | Facts need provenance, supersession, quarantine, and PII/injection scan before retrieval. | §6.4 bitemporal facts; add red-team corpus. |
| Agent costs blow up from hidden integration/security/observability overhead [45]. | Cleanlab/agent cost writeups | Budget ledger must include model, vector, reranker, verifier, sandbox and human-review costs. | §6.1/§6.3 budgets; add per-model cost SLO. |
| Swarms amplify loops and compromised tools. | CSA/MCP security note [46] | Clone/thread/thought budgets must be ancestry-based with kill switches. | Strong §6.3; verify with chaos tests. |
| “Agents in production” succeeds less by autonomy, more by narrowed workflows and evals. | Cleanlab [45] | Promote narrow duties, not general swarms; every model has task-specific evals. | §6.10/§6.11 covers. |
| Local model users prize privacy/offline fallback but struggle with VRAM, quantization and quality cliffs. | HF model/runtime pages [1]-[27] | Ship explicit solo profile with quantized models and CPU degradation path. | Gap in §8/§9. |
| Container supply-chain changes (Bitnami) can break clusters without code changes. | Bitnami change [33] | Image provenance/mirror is core OS state and proof input. | Gap in §9/deploy. |
| Voice agents require end-to-end latency, turn-taking and transcripts, not just STT/TTS. | Meta/Muse + Pipecat/LiveKit architecture (§6.5) | Add SpeechProvider gates: WER, diarization, latency, barge-in, PII redaction. | §6.5 names voice gateways; gap in models. |

### Facebook / Meta source attempt

Public Facebook groups/pages were not practically accessible in this non-interactive environment and often require login; I do not use private/group content. Public Meta corporate sources count. The useful Meta evidence is strong: REA has hibernate/wake multiday ML workflows, budgets and human checkpoints [40]; Capacity Efficiency uses standardized tool interfaces and reusable skills at hyperscale [41]; Meta Enterprise Platform positions agents/models/infrastructure as a business platform with security/privacy claims [47]; Llama 4 posts show Meta’s model/safeguard strategy, but Llama-family licenses are excluded from Year96 core unless legal explicitly allows them [48].

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| Qwen3-Embedding-0.6B | HF model | Default embeddings | Apache-2.0 [1] | Qwen/Alibaba, 2026, 9M downloads | Adopt |
| BAAI/bge-m3 | HF model | Fallback multilingual embeddings | MIT [3] | BAAI, 36M downloads | Adopt |
| BAAI/bge-reranker-v2-m3 | HF model | Default reranker | Apache-2.0 [4] | 17M downloads | Adopt |
| Qwen3-Reranker | HF model | Cluster reranking | Apache-2.0 [2] | 2026 Qwen | Trial/Adopt cluster |
| ModernBERT-base | HF model | Classifier/router base | Apache-2.0 [5] | Answer.AI, ONNX, 4M downloads | Adopt |
| protectai prompt-injection v2 | HF model | Injection guard | Apache-2.0 [12] | 2026, ONNX, 813k downloads | Adopt |
| GLiNER multi PII | HF model | PII NER | Apache-2.0 [13] | multilingual, 62k downloads | Adopt |
| Isotonic ai4privacy v2 | HF model | PII benchmark only | CC-BY-NC-4.0 [14] | useful but non-commercial | Excluded-license |
| Phi-4-mini-instruct | HF model | Solo local LLM | MIT [7] | Microsoft, multilingual | Adopt |
| Qwen3-4B | HF model | Solo/cluster local LLM | Apache-2.0 [6] | Qwen, 6M downloads | Adopt |
| Mistral-7B-Instruct-v0.3 | HF model | Cluster LLM | Apache-2.0 [9] | Mistral, mature serving | Adopt |
| OLMo-2-7B-Instruct | HF model | Auditable LLM | Apache-2.0 [8] | AI2 open science | Trial |
| Granite 4.0 micro | HF model | enterprise extraction | Apache-2.0 [6] | IBM | Trial |
| Florence-2-base | HF model | Screenshot OCR/localization | MIT [11] | Microsoft, 3M downloads | Adopt |
| SmolVLM-500M-Instruct | HF model | tiny screenshot VLM | Apache-2.0 [10] | HF TB | Adopt |
| Qwen2.5-VL-3B | HF model | strong UI VLM | qwen-research [28] | strong benchmarks | Excluded-license |
| Whisper-large-v3 | HF model | STT | Apache-2.0 [15] | OpenAI, 4M downloads | Adopt |
| Kokoro-82M | HF model | TTS | Apache-2.0 [16] | 11M downloads | Adopt |
| Piper voices | HF model | offline multilingual TTS | MIT [19] | rhasspy, ONNX | Adopt |
| Docker Official Images | registry program | curated base/service images | image-specific; Docker ToS [29] | Docker-backed | Adopt with gates |
| Docker Hardened Images | product | signed SBOM/SLSA attestations | product terms; image-specific [30][31] | Docker-backed | Trial for prod bases |
| Bitnami public catalog | registry/product | formerly common Helm dependency | mixed; access changed [33] | Broadcom | Avoid core |

## How I would build this part of Year96

### Default Model Profiles

`solo` is a single workstation: CPU plus optional one consumer GPU, no assumption of always-on cloud. `cluster` is Kubernetes plus GPU pools. The same interfaces run both; only provider config differs.

```ts
export type ModelLicenseVerdict = 'core-allowed' | 'excluded-core' | 'external-only' | 'unknown';
export interface ModelCardRef { repoId: string; revision: string; license: string; licenseVerdict: ModelLicenseVerdict; sourceUrl: string; verifiedAt: string; }
export interface EvalGate { name: string; dataset: string; metric: string; threshold: number; owner: 'assurance'|'evolution'|'security'; }

export interface EmbeddingProvider { embed(input: {texts: string[]; purpose: 'scope'|'dedupe'|'memory'; lang?: string}): Promise<{vectors: Float32Array[]; model: ModelCardRef}>; }
export interface RerankerProvider { rerank(q: string, docs: {id:string;text:string}[], k: number): Promise<{id:string; score:number}[]>; }
export interface ClassifierProvider { classify(input: {text:string; labels:string[]; context?: object}): Promise<{label:string; p:number; calibrated:boolean}>; }
export interface GuardModelProvider { screen(input: {text:string; channel:'feed'|'tool'|'memory'|'user'}): Promise<{injectionRisk:number; pii: EntitySpan[]; action:'allow'|'redact'|'quarantine'}>; }
export interface LocalLLMProvider { generate(req: {task:'adjudicate'|'extractFacts'|'offlineSensitive'; prompt:string; refs:string[]; maxCostUsd:number}): Promise<{text:string; citations:string[]; model:ModelCardRef}>; }
export interface VisionVerifierProvider { verify(req: {screenshotUri:string; expected: ExpectedVisualPredicate[]}): Promise<VisualVerdict[]>; }
export interface SpeechProvider { transcribe(audioUri:string): Promise<Transcript>; synthesize(text:string, voice:VoiceSpec): Promise<AudioArtifact>; }
```

**Solo profile:** Qwen3-Embedding-0.6B; bge-reranker-v2-m3; ModernBERT-base fine-tunes; protectai injection v2; GLiNER PII; Phi-4-mini or Qwen3-4B via llama.cpp/Ollama; Florence-2-base + SmolVLM-500M; Whisper small/large depending hardware; Piper/Kokoro TTS. **Cluster profile:** Qwen3-Embedding-4B/8B; Qwen3-Reranker or bge-reranker; ModernBERT served through ONNX Runtime; Qwen3-4B/Mistral/Granite/OLMo pools through vLLM/SGLang; Whisper-large-v3; Kokoro/Parler/Dia; TEI/Infinity for embeddings.

**Promotion gates linked to §6.11 L4:** (1) license gate from HF card/LICENSE at pinned revision; (2) reproducible serving image with SBOM and digest; (3) task eval: MTEB/MMTEB/MIRACL/BEIR plus Year96 corpora; (4) calibration: Brier/log-loss for classifiers and §6.3 pMatters; (5) security eval: prompt-injection, PII leakage, memory poisoning; (6) latency/cost budgets by profile; (7) regression replay on frozen worlds; (8) shadow/canary through OpenFeature; (9) proof bundle signed per §6.10.

### Container Supply-Chain Policy

1. **Every deploy reference is digest-pinned**: `image: registry.year96.local/postgres@sha256:...`; tags are comments only.
2. **Mirror first**: deployment pulls only from `registry.year96.local` or cloud equivalent. Mirror records upstream URL, digest, fetchedAt, scanner versions, SBOM digest, signature/provenance result.
3. **Verify signatures/attestations**: `cosign verify`, Docker Scout attest where DHI, in-toto/SLSA provenance where upstream supports it. If no signature, mark `unsigned-allowed-dev` or build internally.
4. **Generate SBOMs for all images** using Syft/CycloneDX/SPDX; store in state fabric; diff SBOMs on upgrades.
5. **License gate**: fail core images containing AGPL/GPL/LGPL/MPL/SSPL/BSL/Elastic/Commons Clause/custom restricted packages unless explicitly allowed as external/non-core. Base OS packages must be accounted for.
6. **Base image strategy**: prefer upstream official minimal images for data stores; for Year96-built services use distroless or Wolfi/Chainguard-style minimal bases after terms review; never `latest` in prod.
7. **Bitnami-free Helm**: no chart may pull `bitnami/*`, `bitnamilegacy/*`, or `bitnamisecure/*` by default. Use official charts/operators or Year96 overlays with explicit images.
8. **GPU images**: CUDA/NVIDIA bases require separate license/redistribution review and larger CVE budget; isolate in GPU namespace.
9. **Proof**: admission controller rejects unmirrored/unpinned/unscanned images; quarterly rebuild drill proves every image can be rebuilt or re-fetched.

### Typical flow

A world-feed item arrives (§6.2.1). The GuardModelProvider screens PII/injection; embeddings dedupe it; scope-effect retrieval uses Qwen3/bge vectors; a ModernBERT router decides whether reranking is needed; bge/Qwen reranker selects top evidence; LocalLLM adjudicator explains top-k only; proposals enter the membrane. If a UI proof is required (§6.10), VisionVerifierProvider runs OCR/DOM/pixel/SmolVLM checks and signs evidence. Every model call carries `ModelCardRef`, eval gate version and cost into the ledger. If §6.11 proposes a new model, it must pass the same gates on replay before canary.

## What is still unsolved (late 2026)

- **License/data provenance mismatch.** A model can have Apache/MIT weights but opaque or problematic training data. Core should record a `dataConcern` flag and allow owners to pick stricter profiles.
- **Prompt-injection detection is not solved.** Classifiers reduce risk but do not provide security. Tool/context separation, capability gates, signed tool manifests and least privilege are mandatory.
- **Multilingual safety parity.** Embeddings and speech are multilingual; injection/PII classifiers are weaker outside English/major EU languages.
- **VLM proof reliability.** VLMs hallucinate. Visual proof must be multi-oracle and re-executable, not “the VLM said yes.”
- **Container provenance inconsistency.** Many excellent OSS images still lack signed SLSA provenance. Year96 must be able to build from source and sign internally.
- **Cost/calibration over years.** Long-lived threads need model drift tracking; a better leaderboard score may worsen calibration for Alice’s dormant legal thread.

## Sources

1. https://huggingface.co/api/models/Qwen/Qwen3-Embedding-0.6B
2. https://huggingface.co/api/models/Qwen/Qwen3-Reranker-0.6B
3. https://huggingface.co/api/models/BAAI/bge-m3
4. https://huggingface.co/api/models/BAAI/bge-reranker-v2-m3
5. https://huggingface.co/api/models/answerdotai/ModernBERT-base
6. https://huggingface.co/api/models/Qwen/Qwen3-4B and https://huggingface.co/api/models/ibm-granite/granite-4.0-h-micro
7. https://huggingface.co/api/models/microsoft/Phi-4-mini-instruct
8. https://huggingface.co/api/models/allenai/OLMo-2-1124-7B-Instruct
9. https://huggingface.co/api/models/mistralai/Mistral-7B-Instruct-v0.3
10. https://huggingface.co/api/models/HuggingFaceTB/SmolVLM-500M-Instruct
11. https://huggingface.co/api/models/microsoft/Florence-2-base
12. https://huggingface.co/api/models/protectai/deberta-v3-base-prompt-injection-v2
13. https://huggingface.co/api/models/urchade/gliner_multi_pii-v1
14. https://huggingface.co/api/models/Isotonic/deberta-v3-base_finetuned_ai4privacy_v2
15. https://huggingface.co/api/models/openai/whisper-large-v3
16. https://huggingface.co/api/models/hexgrad/Kokoro-82M
17. https://huggingface.co/api/models/parler-tts/parler-tts-mini-v1
18. https://huggingface.co/api/models/nari-labs/Dia-1.6B
19. https://huggingface.co/api/models/rhasspy/piper-voices
20. https://raw.githubusercontent.com/vllm-project/vllm/main/LICENSE
21. https://raw.githubusercontent.com/sgl-project/sglang/main/LICENSE
22. https://raw.githubusercontent.com/ggml-org/llama.cpp/master/LICENSE
23. https://raw.githubusercontent.com/ollama/ollama/main/LICENSE
24. https://raw.githubusercontent.com/huggingface/text-embeddings-inference/main/LICENSE
25. https://raw.githubusercontent.com/michaelfeil/infinity/main/LICENSE
26. https://raw.githubusercontent.com/microsoft/onnxruntime/main/LICENSE
27. https://raw.githubusercontent.com/UKPLab/sentence-transformers/master/LICENSE
28. https://huggingface.co/Qwen/Qwen2.5-VL-3B-Instruct/raw/main/README.md
29. https://docs.docker.com/docker-hub/image-library/trusted-content/
30. https://docs.docker.com/dhi/explore/security-concepts/attestations/
31. https://docs.docker.com/dhi/explore/security-concepts/sbom/
32. https://hub.docker.com/r/openfga/openfga
33. https://github.com/bitnami/containers/issues/83267 and search result summary for “Upcoming changes to the Bitnami catalog (effective August 28th, 2025)”
34. https://hub.docker.com/_/postgres
35. https://hub.docker.com/_/nats
36. https://developer.microsoft.com/blog/protecting-against-indirect-injection-attacks-mcp/
37. https://hub.docker.com/r/temporalio/server
38. https://hub.docker.com/_/clickhouse
39. https://hub.docker.com/r/vllm/vllm-openai
40. https://engineering.fb.com/2026/03/17/developer-tools/ranking-engineer-agent-rea-autonomous-ai-system-accelerating-meta-ads-ranking-innovation/
41. https://engineering.fb.com/2026/04/16/developer-tools/capacity-efficiency-at-meta-how-unified-ai-agents-optimize-performance-at-hyperscale/
42. https://www.trychroma.com/research/context-rot
43. https://hamel.dev/notes/llm/rag/p6-context_rot.html
44. https://thehackernews.com/2026/07/new-memghost-attack-plants-persistent.html
45. https://cleanlab.ai/ai-agents-in-production-2025/
46. https://labs.cloudsecurityalliance.org/research/csa-research-note-mcp-security-crisis-20260504-csa-styled/
47. https://about.fb.com/news/2026/09/launching-meta-enterprise-platform/
48. https://ai.meta.com/blog/llama-4-multimodal-intelligence/
