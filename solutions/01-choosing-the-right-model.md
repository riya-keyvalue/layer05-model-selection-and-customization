### Scenarios: Frontier vs Open-Source

---

> **Scenario A — High-Stakes Legal Document Review**
>
> A law firm wants to build a contract analysis tool that extracts obligation clauses, identifies risky terms, and flags missing standard provisions from 200-page contracts.
>
> **Verdict: Claude Sonnet 4.6 (adaptive thinking) or GPT-5.4 Thinking (for the hardest ambiguous clauses)**
>
> **Why Claude Sonnet 4.6:** Its 1M-token context window (beta) fits even the longest contracts in a single pass, eliminating chunking artifacts. It leads SWE-bench and code benchmarks, and its adaptive thinking mode automatically engages deeper reasoning on complex clauses while staying fast on straightforward extraction — optimizing cost without manual routing.
>
> **When to escalate to GPT-5.4 Thinking:** If the task involves inferring implicit obligations, spotting contradictions across distant clauses, or reasoning about what a clause means in combination with others — GPT-5.4 Thinking's dedicated reasoning mode provides the highest accuracy ceiling. Use it selectively (e.g., only on flagged high-risk contracts) to control cost.
>
> **Why not open-source here:** Llama 4 Maverick or DeepSeek V3.2 are viable for extraction on clearly written contracts. However, the precision gap on subtle legal language is meaningful and hard to evaluate without a large labeled test set. The risk of a missed obligation clause in a $10M contract outweighs months of API bills.

---

> **Scenario B — On-Device Mobile Assistant**
>
> A mobile app wants an embedded assistant that responds to voice notes and short text commands — no server roundtrip, must run locally on a mid-range Android device.
>
> **Verdict: Phi-4-reasoning (14B, quantized) or Llama 4 Scout (17B active, quantized)**
>
> **Why Phi-4-reasoning:** At 14B parameters (4-bit quantized to ~7GB), it delivers reasoning quality approaching DeepSeek R1 — remarkable for a model that fits on a high-end mobile SoC or edge device. Trained on o3-mini reasoning traces, it handles complex user queries far better than older small models.
>
> **Why Llama 4 Scout:** With only 17B active parameters (MoE architecture), Scout is efficient enough for edge deployment while offering natively multimodal capabilities (text + image) and a massive 10M-token context window. It outperforms Gemma 3 and Gemini 2.0 Flash-Lite on benchmarks.
>
> **For the most constrained devices:** Qwen 3.5 2B or Phi-4-mini (3.8B) are options when RAM is limited to 2–4GB, though with a meaningful quality tradeoff.
>
> **Why not frontier here:** Even if cost weren't a concern, sending every voice note to OpenAI's or Anthropic's API violates user privacy expectations for a local assistant feature.

---

> **Scenario C — Multilingual Customer Support Across 20 Languages**
>
> A global SaaS company handles support tickets in 20 languages including Thai, Vietnamese, Turkish, and Polish. They need consistent quality across all languages.
>
> **Verdict: Gemini 2.5 Flash (primary) or Qwen 3.5 (if self-hosting)**
>
> **Why Gemini 2.5 Flash:** It maintains strong quality across a wide language set including Southeast Asian and non-Latin script languages, at low cost per token — important for high-volume support. Gemini 2.5 Pro is an option for the most complex tickets, but Flash handles routine support queries well. Its 1M-token context means full conversation history fits in a single call.
>
> **Why Qwen 3.5 for self-hosting:** Qwen 3.5 supports 201 languages natively — the broadest language coverage of any open-weight model. If the team is already running open-weight models for cost or privacy reasons, Qwen 3.5 (17B active parameters, MoE) outperforms Llama 4 on non-European language tasks, particularly CJK, Arabic, and Southeast Asian languages.
>
> **Why not Mistral:** Mistral Large 3's multilingual capability is strong for European languages (French, Spanish, German, Italian) but degrades on Southeast Asian and Eastern European languages.

---

> **Scenario D — Code Generation in an Air-Gapped Enterprise Environment**
>
> A defense contractor needs a coding assistant for engineers working on classified systems. No internet access is permitted; all compute must run on internal servers.
>
> **Verdict: Llama 4 Maverick or Devstral 2 (for pure coding)**
>
> **Why Llama 4 Maverick:** Only open-weight models with downloadable weights can satisfy a true air-gap requirement. Llama 4 Maverick (17B active parameters, 128 experts) beats GPT-4o on coding benchmarks while running efficiently via MoE inference. Meta's Llama Community license permits US government and defense use.
>
> **Why Devstral 2 for pure coding:** Mistral's Devstral 2 is purpose-built for software engineering tasks with a 256K context window — ideal for large codebases. It is Apache 2.0 licensed, removing any commercial use concerns.
>
> **Why DeepSeek V3.2 is technically compelling but risky here:** DeepSeek V3.2 shows GPT-5-level coding quality and its weights are MIT-licensed. However, defense and government teams must carefully evaluate whether running a model from a Chinese lab satisfies their security policy — many agencies will prohibit this regardless of technical merit.
>
> **Why not GPT-5.4/Claude:** These are API-only. There is no on-premises deployment option.

---

### Scenarios: Hosted vs Self-Hosted

---

> **Scenario A — Healthcare SaaS Storing Patient Data (PHI)**
>
> A digital health startup builds a clinical note summarization feature. Notes contain Protected Health Information (PHI) under HIPAA.
>
> **Verdict: Azure OpenAI Service or Anthropic Enterprise (hosted) or self-hosted Llama 4**
>
> **Why Azure OpenAI:** Azure OpenAI (now serving GPT-5.x models) offers a Business Associate Agreement (BAA), making it HIPAA-eligible. AWS Bedrock (hosting Claude 4.x and Llama 4) similarly offers BAAs. Anthropic now offers enterprise agreements with data processing addendums for healthcare use cases.
>
> **Why self-hosted is also valid:** A Llama 4 Scout instance running in the company's own HIPAA-compliant VPC gives full data control with zero third-party exposure. Scout fits on a single H100 and handles clinical notes easily within its 10M-token context window. Requires a dedicated ML infra engineer.
>
> **What to avoid:** Using any provider's public API endpoint without a signed BAA — PHI would flow through infrastructure without contractual HIPAA protections, creating regulatory exposure.

---

> **Scenario B — Early-Stage Startup Building an MVP**
>
> A two-engineer startup is building a B2B writing assistant. They have no GPU infrastructure, no DevOps team, and need to ship in 6 weeks.
>
> **Verdict: Hosted API (OpenAI or Anthropic)**
>
> **Why:** Self-hosting a capable model requires at minimum: GPU provisioning, a model serving layer (vLLM, TGI, or Triton), load balancing, monitoring, and an on-call rotation. That is weeks of engineering work before a single feature ships. The OpenAI or Anthropic API is production-ready in hours. At early-stage volumes (thousands of requests/day), API costs are negligible relative to engineering time.
>
> **Cost reality:** At 1,000 requests/day with an average of 1,000 tokens each, GPT-5.3 Instant or Claude Haiku 4.5 costs roughly $0.10–$0.15/day. That is not a business problem.
>
> **When to reconsider:** Once monthly API spend crosses $8–12k, model costs become a line item worth optimizing with self-hosting. With Llama 4 Scout fitting on a single H100, the self-hosting barrier is lower than ever.

---

> **Scenario C — High-Traffic E-Commerce Product Discovery**
>
> An e-commerce platform wants to power product search and recommendations with LLM-generated explanations. They expect 50 million API calls per month.
>
> **Verdict: Hybrid — self-hosted for high-volume simple tasks, hosted API for complex tasks**
>
> **Cost modeling:**
> - 50M requests × 500 avg tokens = 25B tokens/month input
> - At GPT-5.4: ~$5/1M tokens = $125,000/month
> - At Gemini 2.5 Flash-Lite: ~$0.10/1M tokens = $2,500/month
> - Self-hosted Llama 4 Scout on a single H100 instance: ~$2,500/month on AWS (handles high throughput via MoE)
>
> **Why hybrid wins:** Use self-hosted Llama 4 Scout for the 95% of requests that are simple product description rewrites — its MoE architecture delivers high throughput at low cost. Use a hosted frontier model (Claude Sonnet 4.6 or GPT-5.4) for the 5% of complex queries that need better reasoning (e.g., multi-constraint filtering, customer complaint escalation).
>
> **Operational consideration:** This approach requires a model router, two inference stacks, and the MLOps capability to maintain them. Factor that engineering cost into the decision. MoE models like Llama 4 Scout are easier to serve than older dense models of equivalent quality.

---

> **Scenario D — Government Agency with Classified Data**
>
> A national security agency needs LLM-assisted document analysis. Data is classified; no traffic may leave the classified network.
>
> **Verdict: Self-hosted on air-gapped infrastructure, open-weight model only**
>
> **Why:** There is no hosted API option that satisfies a classified network requirement. Cloud provider services, even FedRAMP High authorized ones, do not offer air-gapped deployment of third-party model APIs.
>
> **Model choice:** Llama 4 Maverick (Meta's Llama Community license permits government use) is the strongest option — it beats GPT-4o on benchmarks and runs efficiently via MoE inference. Deploy with vLLM on on-premises H100 nodes. Llama 4 Scout is a lighter alternative that fits on a single H100.
>
> **Why not DeepSeek here:** Despite DeepSeek V3.2's technical excellence, most government security policies prohibit deploying models from Chinese labs on classified networks — even when the weights are MIT-licensed and run locally.
>
> **Key consideration:** Evaluate whether Llama 4 Maverick is sufficient for the task, or whether the agency needs to invest in fine-tuning a domain-specific LoRA adapter to close any remaining gap with GPT-5.4 on classified-domain documents.

---

### Scenarios: Multimodal Model Selection

---

> **Scenario A — Invoice Processing Pipeline**
>
> A fintech company receives 10,000 scanned invoices per day in various formats (PDFs, photos, mixed layouts). They need to extract vendor name, invoice number, line items, and total amount.
>
> **Verdict: Gemini 2.5 Flash (primary) or Claude Sonnet 4.6 (for complex layouts)**
>
> **Why vision over OCR:** Traditional OCR (Tesseract, AWS Textract) extracts raw text but loses layout context. Vision-capable LLMs understand the semantic structure of an invoice — they know that the number after "Invoice #:" is the invoice number, even if the layout varies. This dramatically reduces post-processing rules.
>
> **Why Gemini 2.5 Flash for this task:** At 10,000 invoices/day, cost matters. Gemini 2.5 Flash processes images at a fraction of the cost of GPT-5.4 for similar extraction accuracy on structured documents. Its 24% reduction in output tokens (vs its predecessor) further lowers cost.
>
> **Why Claude Sonnet 4.6 for complex layouts:** For invoices with unusual layouts, multi-page tables, or mixed languages, Claude Sonnet 4.6's vision capabilities combined with its adaptive thinking mode handle edge cases better. Use it as a fallback for invoices that fail validation from the Flash pipeline.
>
> **Open-weight alternative:** Phi-4-reasoning-vision (15B) can handle invoice extraction on-premises if data cannot leave the network — its adaptive reasoning decides when to engage deeper processing for ambiguous fields.
>
> **Output:** Structured JSON with extracted fields. Use a system prompt with a strict JSON schema to enforce output format.
>
> **What to watch:** Handwritten amounts, damaged scans, or non-Latin scripts will degrade accuracy. Flag low-confidence extractions for human review.

---

> **Scenario B — Voice-First Customer Support Bot**
>
> A telecom company wants customers to speak their issue naturally and receive spoken responses without a human agent. Latency must be under 2 seconds end-to-end.
>
> **Verdict: GPT-5.4 native audio (preferred) or Whisper → GPT-5.3 Instant → TTS pipeline**
>
> **Native audio approach (GPT-5.4):** GPT-5.4 natively handles audio-in and audio-out with low latency through streaming. This eliminates the STT/TTS pipeline entirely, reducing latency and simplifying architecture. Best for high-end conversational experiences where naturalness is premium.
>
> **Pipeline approach (for cost optimization):**
> 1. Whisper transcribes audio → text (100–300ms)
> 2. GPT-5.3 Instant generates text response (200–500ms)
> 3. OpenAI TTS or ElevenLabs synthesizes speech (200–400ms)
> Total: ~500ms–1.2s — within acceptable range. Cheaper per request than native audio.
>
> **Why not Gemini audio:** Gemini 2.5 has audio input but no audio output — you still need a TTS step. GPT-5.4 remains the only current model offering native audio-in + audio-out in a single API.
>
> **Latency is king here:** Every component in the pipeline adds latency. Use streaming responses and start TTS synthesis on the first sentence of the LLM output rather than waiting for the full response.

---

> **Scenario C — Code Review Assistant for a Large Monorepo**
>
> An engineering platform team wants an LLM-powered code review bot that comments on pull requests — checking style, catching bugs, and suggesting optimizations. The codebase spans 500K lines of Python, Go, and TypeScript.
>
> **Verdict: Claude Sonnet 4.6 (not a multimodal feature — pure code)**
>
> **Why this isn't really a multimodal problem:** Code review requires deep language understanding of syntax, semantics, and project-specific patterns — not vision or audio. The "modality" is just text (code).
>
> **Why Claude Sonnet 4.6 is favored:** Claude Sonnet 4.6 leads SWE-bench at 72.7% and has consistently strong performance on code understanding, bug detection, and explanation quality. Its 1M-token context window (beta) means even the largest diffs plus surrounding file context fit in a single call. Its adaptive thinking mode automatically engages deeper reasoning on complex code logic without manual routing.
>
> **Open-weight alternative:** Devstral 2 (Mistral's code agents model) is purpose-built for software engineering tasks with a 256K context window. It is a strong option for teams that want to self-host their code review infrastructure for IP protection.
>
> **Pipeline note:** Feed the diff + relevant file context + project style guide in the system prompt. Use Claude's 1M window to avoid chunking artifacts from splitting large PRs.

---

> **Scenario D — Medical Imaging Analysis: X-Ray Report Generation**
>
> A radiology software company wants to generate preliminary text descriptions of chest X-rays to assist radiologists. Must achieve clinical-grade accuracy.
>
> **Verdict: Domain-specific fine-tuned vision model, NOT a general-purpose LLM**
>
> **Why general-purpose vision is still insufficient here:** Even GPT-5.4 and Gemini 2.5 Pro, despite their strong general vision capabilities, are trained on broad internet data. Detecting subtle pathological findings in X-rays (ground-glass opacities, pneumothorax, pulmonary nodules) requires training on thousands of annotated radiological images with verified ground truth.
>
> **What works:** Fine-tuned vision-language models on medical imaging datasets (e.g., BioViL-T, LLaVA-Med, or proprietary models trained on MIMIC-CXR). These reach radiologist-level accuracy on specific findings.
>
> **Emerging option — Phi-4-reasoning-vision:** Microsoft's Phi-4-reasoning-vision (15B) shows promise for medical image understanding due to its adaptive reasoning on visual inputs. At 15B parameters, it is feasible to fine-tune on a radiology dataset — making it a potential foundation for a domain-specific model that runs on modest hardware.
>
> **Where frontier models help:** Once structured findings are extracted by a specialized model, Claude Sonnet 4.6 or GPT-5.4 can generate the narrative radiology report in natural clinical language — separating the perception task from the language generation task.
>
> **Regulatory note:** Any model used in clinical decision support may require FDA 510(k) clearance or CE marking in the EU. A general-purpose model API cannot satisfy these requirements by default.

---

> **Scenario E — Video Content Moderation**
>
> A social media platform needs to automatically flag policy-violating video content (violence, explicit material) at upload time. They receive 100,000 video uploads per day.
>
> **Verdict: Gemini 2.5 Pro for understanding + specialized moderation models for flagging**
>
> **Why Gemini 2.5 Pro for video:** Gemini 2.5 Pro has the most mature native video input support among frontier models. It can analyze a 30-minute video in a single context window and identify scenes, audio content, and on-screen text with its thinking capabilities.
>
> **Open-weight video options:** Llama 4 Maverick and Qwen 3.5 now support native video input — a major shift from 2025 when Gemini was the only option. For self-hosted moderation pipelines (common in platforms that want to avoid sending user content to third-party APIs), Llama 4 Maverick is the strongest open-weight choice.
>
> **Cost reality check:** At 100,000 videos/day averaging 3 minutes each, running every video through Gemini 2.5 Pro at video token pricing would be very expensive. Use a tiered approach:
> 1. Fast, cheap specialized CV model flags high-probability violations in the first pass
> 2. Gemini 2.5 Pro or self-hosted Llama 4 Maverick is invoked only on borderline cases (~5% of uploads) for deeper reasoning
>
> **Why not GPT-5.4:** GPT-5.4 does not support video input natively. You would need to extract frames and send them as images, losing temporal context (which is important for detecting violent sequences).

---