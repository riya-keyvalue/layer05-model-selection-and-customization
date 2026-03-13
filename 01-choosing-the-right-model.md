# Choosing the Right LLM

Picking the wrong model costs you in at least one of three ways: money, accuracy, or compliance risk. This guide walks through the three core decisions every engineering team faces — which model family to use, whether to host it yourself, and whether you need multimodal capabilities — with real-world scenarios to illustrate the tradeoffs.

---

## Table of Contents

1. [Frontier Models vs Open-Source](#1-frontier-models-vs-open-source)
2. [Hosted APIs vs Self-Hosted](#2-hosted-apis-vs-self-hosted)
3. [Multimodal Model Selection](#3-multimodal-model-selection)

---

## 1. Frontier Models vs Open-Source

### What "Frontier" Means

Frontier models are the largest, most capable models available at a given point in time, trained at massive scale by well-resourced labs. They are typically accessible only through hosted APIs and represent the current ceiling of general-purpose reasoning, instruction-following, and language understanding.

There are now two distinct sub-categories within frontier models:

- **General-purpose frontier models** — optimised for balanced performance across reasoning, language, coding, and multimodal tasks (GPT-5.4, Claude Sonnet 4.6, Gemini 2.5 Pro)
- **Reasoning / "thinking" models** — optimised specifically for deep multi-step reasoning by running an internal chain-of-thought before responding. Slower and more expensive per request, but significantly more accurate on hard logic, math, and planning tasks (GPT-5.4 Thinking, o3, Claude Sonnet 4.6 with adaptive thinking, Gemini 2.5 Pro)

Current frontier models (as of March 2026):

| Model | Provider | Context Window | Modalities | Strengths | Weaknesses | Cost Tier |
|---|---|---|---|---|---|---|
| **GPT-5.4** | OpenAI | 1M tokens | Text, Images, Audio in/out, Code, Computer-use | Top benchmark scores (83% GDPval, 75% OSWorld), native computer-use for agents | Very expensive, newer model with less production track record | Very High |
| **GPT-5.4 Thinking** | OpenAI | 1M tokens | Text, Images, Audio in/out, Code, Computer-use | Dedicated reasoning variant of GPT-5.4, excels on complex multi-step problems | Slow (internal reasoning steps), highest cost tier | Very High |
| **GPT-5.3 Instant** | OpenAI | 1M tokens | Text, Images, Audio in/out, Code | Fast everyday model, improved web search, natural conversational tone | Less capable on hard reasoning vs GPT-5.4 | Medium |
| **Claude Sonnet 4.6** | Anthropic | 1M tokens (beta) | Text, Images, Code, Computer-use | Excellent coding (SWE-bench leader), adaptive thinking mode auto-decides reasoning depth, strong long-context accuracy | Adaptive thinking can increase cost unpredictably | High ($3/$15 per M tokens) |
| **Claude Opus 4.6** | Anthropic | 1M tokens (beta) | Text, Images, Code, Computer-use | Most capable Claude — sustained performance on hours-long agentic tasks, best nuanced writing | Expensive ($5/$25 per M tokens), slower than Sonnet | Very High |
| **Claude Haiku 4.5** | Anthropic | 200K tokens | Text, Images, Code | Fast, cheap ($1/$5 per M tokens), strong structured extraction and classification | Less capable on complex reasoning vs Sonnet/Opus | Low |
| **Gemini 2.5 Pro** | Google | 1M tokens | Text, Images, Audio in, Video, Code | #1 on LMArena (Elo 1470), native thinking model, strong multilingual | Can be inconsistent on nuanced writing vs Claude | High |
| **Gemini 2.5 Flash** | Google | 1M tokens | Text, Images, Audio in, Video, Code | Cost-efficient with strong agentic tool use, 24% reduction in output tokens vs predecessor | Weaker on hard reasoning tasks | Low |
| **Gemini 2.5 Flash-Lite** | Google | 1M tokens | Text, Images, Audio in, Video, Code | Cheapest in the Gemini family, 50% fewer output tokens, good for high-volume simple tasks | Significant capability gap vs Pro and Flash | Very Low |

> **Note on reasoning / thinking models:** GPT-5.4 Thinking, o3, and Claude Sonnet 4.6 in adaptive thinking mode all run an internal scratchpad before generating their final response. This dramatically improves accuracy on hard problems but increases latency to 10–60 seconds per request and raises cost by 5–20x compared to their standard counterparts. Use them selectively for tasks where accuracy matters more than speed.
>
> Claude Sonnet 4.6 introduced **adaptive thinking** — the model automatically decides when to engage deeper reasoning, giving a better cost-quality tradeoff than always-on thinking modes. Gemini 2.5 Pro is a native thinking model by default.

> **Legacy models still in API:** GPT-4o, GPT-4.1, GPT-4.1 mini, and o4-mini were retired from ChatGPT in February 2026 but remain available in the API for existing integrations. New projects should use the GPT-5.x family.

### What "Open-Source" Means

Generally speaking, open-source LLMs are models whose architecture, code, and weights are publicly released so anyone can download them, run them locally, fine-tune them, and deploy them in their own infrastructure. They give teams full control over inference, customization, data privacy, and long-term costs.

However, the term “open-source LLM” is often used loosely. Many models are openly available, but their licensing falls under open weights, not traditional open source.

Open weights here means the model parameters are published and free to download, but the license may not meet the Open Source Initiative (OSI) definition of open source. These models sometimes have restrictions, such as commercial-use limits, attribution requirements, or conditions on how they can be redistributed.

The quality gap with frontier models has largely closed through 2025–2026. Open-weight models now represent 182 of the 282 tracked models in the ecosystem, making open-source the dominant force by volume. Chinese labs (DeepSeek, Alibaba/Qwen) and Meta have been the primary drivers of this shift.

| Model | Provider | Parameters | Context Window | Modalities | Strengths | Weaknesses | Cost Tier | License |
|---|---|---|---|---|---|---|---|---|
| **Llama 4 Maverick** | Meta | 17B active / 128 experts | 1M tokens | Text, Images, Video, Code | Beats GPT-4o and Gemini 2.0 Flash on benchmarks, natively multimodal, MoE (Mixture of Experts) architecture for efficient inference | Newer model, smaller community tooling ecosystem than Llama 3.x | Medium infra cost | Llama Community (commercial ok) |
| **Llama 4 Scout** | Meta | 17B active / 16 experts | 10M tokens | Text, Images, Video, Code | Fits on a single H100, massive 10M-token context window, outperforms Gemma 3 and Gemini 2.0 Flash-Lite | Less capable than Maverick on complex tasks | Low infra cost | Llama Community (commercial ok) |
| **Llama 4 Behemoth** | Meta | 288B active / 2T total | TBD | Text, Images, Video, Code (expected) | Frontier-class — outperforms GPT-4.5 and Claude 3.7 Sonnet on STEM benchmarks in early testing | Still in training as of Mar 2026; not yet available for download | Very high infra cost | Llama Community (expected) |
| **DeepSeek V3.2** | DeepSeek | ~660B MoE | 128K tokens | Text, Code | GPT-5-level performance with integrated reasoning and agentic capabilities, MIT license | Chinese lab provenance raises data-handling concerns; massive model requires significant GPU infra | High infra cost | MIT |
| **DeepSeek R1** (0528 update) | DeepSeek | 671B MoE | 128K tokens | Text, Code | Approaches o3 performance on reasoning, math, and logic; distilled 8B version runs on single 40GB GPU | Slow inference due to thinking tokens; same provenance considerations | Infra cost | MIT |
| **Qwen 3.5** | Alibaba | 397B MoE / 17B active | 128K tokens | Text, Images, Video, Code | 201-language support, natively multimodal, hybrid Gated DeltaNet architecture, beats GPT-5 mini on many benchmarks | Same third-party trust considerations as DeepSeek; very new (Feb 2026) | Medium infra cost | Apache 2.0 |
| **Mistral Large 3** | Mistral AI | ~123B | 128K tokens | Text, Images, Code | Open-weight multimodal, strong coding, multilingual (especially European), permissive license | Lower performance ceiling than Llama 4 Maverick and DeepSeek V3.2 | Medium infra cost | Mistral Research License |
| **Devstral 2** | Mistral AI | Specialized | 256K tokens | Code only | Frontier code agents model, purpose-built for software engineering tasks | Narrow focus — not a general-purpose model | Low infra cost | Apache 2.0 |
| **Phi-4-reasoning** | Microsoft | 14B | 128K tokens | Text, Code | Approaches DeepSeek R1 performance at tiny scale, strong math/science/coding, trained on o3-mini reasoning traces | Narrow training distribution, less capable on creative/open-ended tasks | Minimal | MIT |
| **Phi-4-reasoning-vision** | Microsoft | 15B | 128K tokens | Text, Images, Code | Multimodal reasoning (image + text), adaptive reasoning that auto-decides depth, strong on UI understanding and math | Very new (Mar 2026), limited production track record | Minimal | MIT |

> **The open-source inflection point:** By early 2026, the open-weight ecosystem has matured beyond catching up — models like DeepSeek V3.2 and Qwen 3.5 match or exceed previous-generation frontier models on many tasks. Llama 4's MoE architecture means frontier-class quality at dramatically lower inference cost than dense models of equivalent capability. The primary remaining advantages of proprietary models are (1) the absolute cutting edge on the hardest reasoning tasks, (2) integrated product ecosystems (ChatGPT, Claude.ai), and (3) enterprise support and compliance guarantees.
>
> Teams subject to strict data governance should still evaluate their risk appetite before deploying DeepSeek or Qwen models due to the provenance of the training infrastructure.

### Key Differentiators to Evaluate

- **Deep reasoning**: GPT-5.4 Thinking and o3 are the proprietary ceiling. DeepSeek R1 (0528) is the open-weight alternative, approaching o3 on many benchmarks. Claude Sonnet 4.6 with adaptive thinking offers the best cost-quality tradeoff by auto-deciding when to reason deeply.
- **General-purpose balance**: Claude Sonnet 4.6 and GPT-5.4 are the strongest all-rounders for production systems that need consistent quality across diverse task types. Gemini 2.5 Pro is competitive and leads on LMArena.
- **Cost efficiency**: Gemini 2.5 Flash-Lite, Claude Haiku 4.5, and GPT-5.3 Instant are the best value for high-volume, lower-complexity tasks. Llama 4 Scout (single H100, 10M context) is the open-weight cost leader.
- **Agentic / computer-use tasks**: GPT-5.4 and Claude Opus/Sonnet 4.6 both have native computer-use capabilities. Claude Opus 4.6 can sustain performance on multi-hour agentic tasks.
- **Instruction-following**: All current-generation frontier models are strong. Llama 4 Maverick and Qwen 3.5 are the open-weight leaders.
- **Safety tuning**: Claude retains the most conservative safety profile. Mistral and DeepSeek models remain the least constrained out of the box.
- **Multilingual support**: Qwen 3.5 leads with 201-language support. Gemini 2.5 Pro is strong across a broad set. Llama 4 is natively multimodal but multilingual depth varies by language.
- **Code generation**: Claude Sonnet 4.6 (72.7% SWE-bench), Claude Opus 4.6 (72.5% SWE-bench), and Devstral 2 are the top performers. DeepSeek V3.2 is the open-weight coding leader.
- **Long context**: Llama 4 Scout at 10M tokens is the longest context window available. Gemini 2.5 Pro/Flash at 1M tokens is the longest among proprietary models. Claude Sonnet/Opus 4.6 at 1M tokens (beta) is catching up.

---

### Scenarios - Frontier vs Open-Source

---

> **Scenario A — High-Stakes Legal Document Review**
>
> A law firm wants to build a contract analysis tool that extracts obligation clauses, identifies risky terms, and flags missing standard provisions from 200-page contracts.

> **Scenario B — On-Device Mobile Assistant**
>
> A mobile app wants an embedded assistant that responds to voice notes and short text commands — no server roundtrip, must run locally on a mid-range Android device.

> **Scenario C — Multilingual Customer Support Across 20 Languages**
>
> A global SaaS company handles support tickets in 20 languages including Thai, Vietnamese, Turkish, and Polish. They need consistent quality across all languages.

> **Scenario D — Code Generation in an Air-Gapped Enterprise Environment**
>
> A defense contractor needs a coding assistant for engineers working on classified systems. No internet access is permitted; all compute must run on internal servers.
>

### Key Takeaways — Frontier vs Open-Source

- Use **frontier general-purpose models** (Claude Sonnet 4.6, GPT-5.4, Gemini 2.5 Pro) when task accuracy, safety, or long-context fidelity is non-negotiable.
- Use **reasoning/thinking models** (GPT-5.4 Thinking, o3, Claude Sonnet 4.6 adaptive thinking, DeepSeek R1) only when the task genuinely requires deep multi-step reasoning — they are 5–20x more expensive and slower than standard models. Prefer adaptive thinking (Claude) or Gemini 2.5 Pro (native thinking) for automatic cost-quality optimization.
- Use **open-source models** when data privacy, air-gap requirements, cost at scale, or on-device deployment are constraints. Llama 4 Maverick, DeepSeek V3.2, and Qwen 3.5 now match or exceed previous-generation frontier models.
- The performance gap has largely closed — always benchmark your specific task before assuming frontier is necessary. Many production classification and extraction tasks run just as well on Llama 4 Scout or Phi-4-reasoning.
- Consider **open-source + fine-tuning** as a powerful hybrid: start with a capable open-weight base model and adapt it to your domain at a fraction of frontier API cost.
- **MoE architectures** (Llama 4, Qwen 3.5, DeepSeek V3.2) deliver frontier-class quality at dramatically lower inference cost than dense models — understand this tradeoff when sizing GPU infrastructure.

### Common Mistakes

- Using legacy models (GPT-4o, Claude 3.5 Sonnet) when current-generation models (GPT-5.x, Claude 4.x) are available at similar or lower cost with significantly better performance.
- Defaulting to the most expensive model for every task — Claude Haiku 4.5, Gemini 2.5 Flash-Lite, or GPT-5.3 Instant often match quality at 10–20x lower cost for classification and extraction tasks.
- Using a dedicated reasoning model (GPT-5.4 Thinking, o3) for simple tasks — the extra cost and latency buys nothing on straightforward extraction or classification. Prefer adaptive thinking models that auto-decide.
- Ignoring MoE models — Llama 4 Scout (17B active) and Qwen 3.5 (17B active) deliver quality comparable to much larger dense models at a fraction of the inference cost.
- Not accounting for data provenance when evaluating DeepSeek or Qwen models — technically excellent, but requires a governance review before use in regulated or defense environments.
- Not evaluating models on your actual data — benchmark rankings reflect generic benchmarks, not your domain-specific distribution.

---

## 2. Hosted APIs vs Self-Hosted

### The Core Tradeoff

Every LLM deployment sits somewhere on a spectrum from fully managed API to fully self-hosted. Neither extreme is universally correct — the right answer depends on your traffic, budget, data sensitivity, and operational maturity.

| Dimension | Hosted API | Self-Hosted |
|---|---|---|
| **Cost structure** | Pay per token, no upfront cost | Upfront GPU cost, lower per-token at scale |
| **Operational burden** | Zero infrastructure to manage | Significant: model serving, scaling, monitoring |
| **Data privacy** | Data sent to third party | Data never leaves your environment |
| **Latency** | 50–500ms typical, network-dependent | 20–200ms on local network (varies with hardware) |
| **Model updates** | Automatic (can be disruptive) | You control upgrade timing |
| **Scaling** | Instant, handled by provider | Manual, requires capacity planning |
| **Compliance** | Depends on provider BAA/DPA | Full control over data residency |
| **Uptime SLA** | 99.9–99.99% (provider-dependent) | You own the SLA |

### When Hosted APIs Win

- You're prototyping or in early product stages — no GPU budget or MLOps team yet
- Your request volume is moderate — the break-even point against self-hosting a Llama 4 Maverick-class model is roughly $5,000–$15,000/month in API spend
- You need access to the absolute latest frontier models (GPT-5.4, Claude Opus 4.6) without waiting for open-source equivalents
- You need native computer-use / agentic capabilities that are only available through proprietary APIs
- Your compliance requirements permit data processing by a third party under a DPA/BAA

### When Self-Hosting Wins

- You have strict data residency or sovereignty requirements (HIPAA, GDPR in certain interpretations, government regulations)
- Your request volume is high enough that token costs exceed GPU amortization
- You need the model to operate offline or in an air-gapped network
- You want full control over model versioning and upgrade cadence
- You intend to fine-tune and serve your own custom model weights

### The Hybrid Approach

Many production systems use a combination:
- Hosted API for low-volume, high-complexity tasks (frontier reasoning)
- Self-hosted open-source model for high-volume, lower-complexity tasks (classification, extraction)
- Self-hosted model for sensitive data processing + hosted API for non-sensitive tasks

### Decision Flow

```
                          ┌─────────────────────────────────┐
                          │  Hosted vs Self-Hosted?          │
                          │  Start here                      │
                          └────────────────┬────────────────┘
                                           │
                    ┌──────────────────────▼──────────────────────┐
                    │  Do you have data privacy or compliance      │
                    │  requirements? (HIPAA, GDPR, air-gap, etc.)  │
                    └─────────────┬──────────────────┬────────────┘
                                  │                  │
                                YES                  NO
                                  │                  │
             ┌────────────────────▼──┐    ┌──────────▼─────────────────┐
             │ Is it air-gapped or   │    │ Are you in early stages     │
             │ classified? (no       │    │ or building an MVP?         │
             │ internet allowed)     │    └──────┬──────────────┬───────┘
             └───┬───────────────┬───┘           │              │
                 │               │              YES              NO
                YES              NO              │              │
                 │               │               │     ┌────────▼────────────────┐
                 │     ┌─────────▼──────────┐    │     │ Is monthly API spend    │
                 │     │ Can a cloud        │    │     │ above ~$10k/month?      │
                 │     │ provider sign a    │    │     └────────┬──────────┬─────┘
                 │     │ BAA / DPA?         │    │              │          │
                 │     └─────┬──────────┬───┘    │             YES         NO
                 │           │          │        │              │          │
                 │          YES         NO       │    ┌─────────▼──┐       │
                 │           │          │        │    │ Do you have│       │
                 │           │          │        │    │ an MLOps   │       │
                 │           │          │        │    │ team?      │       │
                 │           │          │        │    └──┬─────┬───┘       │
                 │           │          │        │      YES    NO           │
                 │           │          │        │       │     │            │
                 ▼           ▼          ▼        ▼       ▼     ▼            ▼
            ┌───────┐  ┌─────────┐ ┌───────┐ ┌─────┐ ┌────┐ ┌────────┐ ┌──────┐
            │  [1]  │  │   [2]   │ │  [3]  │ │ [4] │ │[7] │ │  [6]  │ │ [5]  │
            └───────┘  └─────────┘ └───────┘ └─────┘ └────┘ └────────┘ └──────┘
```

| # | Outcome | What to do |
|---|---|---|
| **1** | **Air-gapped self-host** | No internet allowed — only open-weight models work. Deploy Llama 4 Maverick or Scout with vLLM on on-premises H100s. |
| **2** | **Hosted API with compliance agreement** | Get a BAA/DPA signed first, then use the API. Options: Azure OpenAI (GPT-5.x), AWS Bedrock (Claude 4.x, Llama 4), Anthropic Enterprise. |
| **3** | **Self-host in your own VPC** | No cloud provider meets your compliance needs. Run Llama 4 Scout or Mistral Large 3 inside your own HIPAA/GDPR-compliant VPC. |
| **4** | **Hosted API — start here** | Don't touch infrastructure yet. Use OpenAI, Anthropic, or Google APIs. Pick a mid-tier model (Claude Haiku 4.5, GPT-5.3 Instant, Gemini 2.5 Flash). Revisit when spend crosses $8–10k/month. |
| **5** | **Hosted API — optimise cost first** | Spend is low; self-hosting isn't worth the overhead yet. Downgrade to a cheaper model tier (Gemini 2.5 Flash-Lite, GPT-5.3 Instant) and reduce token usage before considering infrastructure. |
| **6** | **Hosted API + Batch API** | Spend is high but no MLOps capacity. Use async batch endpoints for up to 50% cost savings — OpenAI Batch API and Anthropic Message Batches, with up to 24-hour turnaround. |
| **7** | **Hybrid deployment** | Self-host Llama 4 Scout (single H100) for high-volume simple tasks. Route the 5–10% of complex or reasoning-heavy requests to a hosted frontier model (Claude Sonnet 4.6, GPT-5.4). |

---

### Scenarios: Hosted vs Self-Hosted

> **Scenario A — Healthcare SaaS Storing Patient Data (PHI)**
>
> A digital health startup builds a clinical note summarization feature. Notes contain Protected Health Information (PHI) under HIPAA.

> **Scenario B — Early-Stage Startup Building an MVP**
>
> A two-engineer startup is building a B2B writing assistant. They have no GPU infrastructure, no DevOps team, and need to ship in 6 weeks.
>

> **Scenario C — High-Traffic E-Commerce Product Discovery**
>
> An e-commerce platform wants to power product search and recommendations with LLM-generated explanations. They expect 50 million API calls per month.

> **Scenario D — Government Agency with Classified Data**
>
> A national security agency needs LLM-assisted document analysis. Data is classified; no traffic may leave the classified network.

---

### Key Takeaways — Hosted vs Self-Hosted

- **Default to hosted APIs** unless you have a concrete reason not to: data compliance, volume-driven cost, or air-gap requirements.
- **The break-even point** for self-hosting has shifted — MoE models like Llama 4 Scout (single H100) make self-hosting viable at lower volume thresholds than before. Roughly $5–12k/month in equivalent API spend is the crossover point.
- **Compliance is the most common forcing function** — know your data classification requirements before choosing an API provider.
- Hosted APIs that offer **BAA/DPA agreements**: Azure OpenAI (GPT-5.x), AWS Bedrock (Claude 4.x, Llama 4), Google Cloud Vertex AI (Gemini 2.5), and Anthropic Enterprise. OpenAI direct has expanded enterprise agreements but verify BAA availability for your specific use case.

### Common Mistakes

- Self-hosting prematurely to save money before validating that the product has traction.
- Assuming all hosted APIs are equivalent for compliance — the BAA availability varies significantly across providers and tiers.
- Underestimating GPU operational complexity: model serving, version management, autoscaling, and CUDA driver maintenance are non-trivial — though MoE models have simplified the hardware requirements.
- Forgetting network latency when self-hosting in a distant region — a model on a GPU in US-East adds 80ms for users in Asia-Pacific.
- Not considering the **hybrid approach** from the start — many production systems benefit from routing simple tasks to a cheap self-hosted model and complex tasks to a frontier API.

---

## 3. Multimodal Model Selection

### What "Multimodal" Means

A multimodal model can process inputs beyond plain text — images, audio, video, documents, or structured data — and produce text (or in some cases image/audio) outputs. As of early 2026, multimodal support is no longer a differentiator among frontier models — it is table stakes. The real questions are: which modalities does your task require, how good is each model at your specific modality, and does a general multimodal model outperform a specialized one?

A major shift: open-weight models are now natively multimodal. Llama 4 was trained from the ground up on text, image, and video data — not bolted on after the fact.

### Modality Capability Matrix

| Model | Text | Vision | Audio in | Audio out | Video | Code | Computer-use |
|---|---|---|---|---|---|---|---|
| GPT-5.4 | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ |
| GPT-5.3 Instant | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| Claude Sonnet/Opus 4.6 | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Claude Haiku 4.5 | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Gemini 2.5 Pro | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Gemini 2.5 Flash | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ |
| Llama 4 Maverick | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Llama 4 Scout | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Qwen 3.5 | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| Phi-4-reasoning-vision | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| Whisper (OpenAI) | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Devstral 2 (Mistral) | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |

*Note: Capabilities evolve rapidly. Verify against provider documentation for the latest state.*

### When You Actually Need Multimodal

The right question is not "which model supports the most modalities" but "does my task require modalities beyond text, and does a general multimodal model outperform a specialized one?"

| Use Case | Approach | Why |
|---|---|---|
| Document OCR + understanding | Vision model (GPT-5.4, Gemini 2.5 Flash, Claude Sonnet 4.6) | Layout-aware extraction is better with native vision than OCR → text pipelines |
| Voice-first interface | Whisper (STT) → LLM → TTS, or GPT-5.4 native audio | Separate specialized models often give better quality + control |
| Code generation | Devstral 2, Claude Sonnet 4.6, or DeepSeek V3.2 | Code-specialized models outperform generic ones on complex codebases |
| Video understanding | Gemini 2.5 Pro, Llama 4 Maverick, or Qwen 3.5 | Multiple models now support native video — no longer a single-vendor lock-in |
| Computer-use / UI automation | GPT-5.4 or Claude Sonnet/Opus 4.6 | Native computer-use capabilities for agentic workflows |
| Image generation | DALL-E 3, Stable Diffusion XL, Midjourney | These are separate generative vision models — not the same as vision-input LLMs |

*STT - Speech to Text*
*TTS - Text to Speech*
### When Specialized Models Beat General Multimodal

Multimodal convenience has a cost: general-purpose vision models trained on broad data may underperform task-specific computer vision models on narrow domains like medical imaging, satellite imagery, or manufacturing defect detection.

```
Generalist multimodal LLM:
  Pros: One API, no pipeline complexity, handles ambiguous inputs
  Cons: Weaker on specialized domains, higher cost per image token

Specialized vision model (e.g., fine-tuned ViT, custom YOLO):
  Pros: Higher accuracy on the specific domain, lower inference cost
  Cons: Requires ML expertise, training data, and maintenance
```

---

### Scenarios: Multimodal Model Selection

> **Scenario A — Invoice Processing Pipeline**
>
> A fintech company receives 10,000 scanned invoices per day in various formats (PDFs, photos, mixed layouts). They need to extract vendor name, invoice number, line items, and total amount.
>

> **Scenario B — Voice-First Customer Support Bot**
>
> A telecom company wants customers to speak their issue naturally and receive spoken responses without a human agent. Latency must be under 2 seconds end-to-end.

> **Scenario C — Code Review Assistant for a Large Monorepo**
>
> An engineering platform team wants an LLM-powered code review bot that comments on pull requests — checking style, catching bugs, and suggesting optimizations. The codebase spans 500K lines of Python, Go, and TypeScript.

> **Scenario D — Video Content Moderation**
>
> A social media platform needs to automatically flag policy-violating video content (violence, explicit material) at upload time. They receive 100,000 video uploads per day.
>

---

### Key Takeaways — Multimodal Model Selection

- **Vision support is now table stakes** among both frontier and open-weight models. The real question is cost and accuracy on your specific document or image type.
- **Video is no longer single-vendor** — Gemini 2.5 Pro, Llama 4 Maverick, and Qwen 3.5 all support native video input. Evaluate based on your deployment model (hosted vs self-hosted) and cost.
- **Audio remains differentiated** — GPT-5.4 is the only model offering native audio-in + audio-out in a single API. All other models require a Whisper → LLM → TTS pipeline.
- **Computer-use is the newest modality** — GPT-5.4 and Claude Sonnet/Opus 4.6 can interact with UIs, navigate applications, and perform multi-step workflows. This enables a new class of agentic applications.
- **Code is just text** — don't conflate "multimodal" with "code support." Specialized code models (Devstral 2, Claude Sonnet 4.6) and large-context text models handle code better than generic multimodal models.
- **For specialized domains** (medical imaging, satellite imagery, industrial inspection), general multimodal LLMs are a starting point, not a final solution — domain-specific fine-tuned models (including Phi-4-reasoning-vision as a base) typically outperform them.
- **Separate perception from reasoning** when possible: use specialized models for the sensing/extraction layer and a general LLM for the language generation layer.

### Common Mistakes

- Using a vision-capable LLM for tasks that a purpose-built OCR or CV model handles faster and cheaper (e.g., straightforward text extraction from clean documents).
- Assuming all models that support "audio" are equivalent — Whisper is transcription-only, GPT-5.4 audio does native turn-based dialogue, and they serve different use cases.
- Not accounting for per-image token costs: a single high-resolution image can consume 800–1500 tokens, which changes cost modeling significantly at scale.
- Conflating image generation (DALL-E, Midjourney, Stable Diffusion) with vision-input models (GPT-5.4 Vision, Gemini 2.5 Vision) — these are entirely different model families.
- Overlooking open-weight multimodal options — Llama 4 is natively multimodal (text/image/video) and can be self-hosted, eliminating the need to send sensitive visual data to third-party APIs.

---
