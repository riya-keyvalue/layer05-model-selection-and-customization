# Fine-Tuning LLMs

Fine-tuning is one of the most misunderstood tools in an engineer's LLM toolkit. It is often reached for too early — when better prompting or retrieval would solve the problem at a fraction of the cost and effort — and sometimes not reached for at all, when it is genuinely the right answer. This guide explains what fine-tuning actually does to a model, when it is the right choice over RAG or prompting, and what you need to know about efficient fine-tuning techniques like LoRA without implementing them yourself.

---

## Table of Contents

1. [What Fine-Tuning Changes About a Model](#1-what-fine-tuning-changes-about-a-model)
2. [Fine-Tuning vs RAG vs Prompting](#2-fine-tuning-vs-rag-vs-prompting)
3. [PEFT & LoRA — Awareness Level](#3-peft--lora--awareness-level)

---

## 1. What Fine-Tuning Changes About a Model

### The Mental Model

A pre-trained language model has learned a statistical representation of language from a massive corpus — it knows how words, sentences, and concepts relate to each other. An instruction-tuned model (like Claude, GPT-5, or Llama 4 Instruct) has been further trained to follow instructions and behave helpfully.

Fine-tuning takes that model and continues training it on a curated, task-specific dataset. The process updates the model's weights — the numerical parameters that encode everything the model "knows" — to shift its behaviour toward what your dataset demonstrates.

```
Pre-trained base model
        │
        ▼
Instruction tuning (RLHF, SFT)   ← done by the model provider
        │
        ▼
General-purpose instruction-following model (e.g. Llama 4 Instruct)
        │
        ▼
Your fine-tuning run on task-specific data
        │
        ▼
Fine-tuned model — shifted toward your domain, style, format, or task
```

### What Fine-Tuning Changes

**1. Output distribution**
The model's probability distribution over tokens shifts. If you fine-tuned on formal legal writing, the model will assign higher probability to formal phrasing and lower probability to casual language — even without being told to in the prompt.

**2. Task-specific priors**
The model develops strong priors for your task format. A model fine-tuned to extract JSON from invoices no longer needs detailed instructions in every prompt — it has learned the pattern into its weights.

**3. Style, tone, and persona**
Fine-tuning is the most reliable way to enforce a consistent voice. Prompting can nudge style, but fine-tuning bakes it in at the weight level.

**4. Domain vocabulary and conventions**
Medical, legal, financial, or engineering-specific terminology that the base model treats as low-frequency gets up-weighted. The model responds more accurately to domain jargon and produces more appropriate domain-specific outputs.

**5. Behaviour under constraint**
A fine-tuned model can learn to always refuse certain topics, always return a specific structure, or always cite a source — behaviours that are brittle when implemented via system prompt alone.

### What Fine-Tuning Does NOT Change

This is equally important to understand. Fine-tuning is commonly expected to do things it cannot.

| What people expect | What actually happens |
|---|---|
| Update the model's knowledge to recent events | Does NOT work — the model's knowledge cutoff is baked into pre-training weights, not recoverable via fine-tuning |
| Teach the model new factual information reliably | Unreliable — fine-tuning on facts causes hallucination and inconsistency; use RAG for factual grounding |
| Expand the context window | Does NOT change — context window is an architectural property of the model |
| Fix reasoning bugs | Limited improvement — fine-tuning on reasoning tasks can help at the margins but does not fundamentally improve the underlying reasoning capability |
| Make the model aware of your proprietary data | Only if that data is in the training set; fine-tuning is not a retrieval mechanism |

> **The key insight:** Fine-tuning changes *how* the model behaves, not *what* it knows. Use it to shape behaviour, style, and format. Use RAG to supply knowledge.

### The Training Process (Conceptual)

Fine-tuning is supervised learning. You provide input-output pairs that demonstrate the behaviour you want:

```
Input:  "Summarise this support ticket in one sentence."
        [ticket text here]

Output: "Customer reports intermittent login failures since the v2.3 update,
         affecting mobile app only."
```

The model is trained to maximise the probability of generating the correct output given the input. Over many examples, the weights shift to reproduce the demonstrated pattern.

**Data requirements:** Quality matters far more than quantity. 500 high-quality, diverse examples often outperform 10,000 noisy ones. The dataset should cover the full distribution of inputs you expect at inference time.

**Training cost:** Full fine-tuning of a 70B model requires multiple high-memory GPUs and hours of compute. This is why parameter-efficient methods like LoRA exist — covered in Section 3.

---

### Scenarios: What Fine-Tuning Changes

---

> **Scenario A — Base Model vs Fine-Tuned Model on Customer Support Transcripts**
>
> A SaaS company has 50,000 historical support conversations where agents successfully resolved issues. They want an LLM to handle Tier 1 support autonomously.
>
> **Without fine-tuning (base Llama 4 Instruct):**
> The model gives generic, helpful responses but misses company-specific language, escalation policies, and the exact resolution patterns that work for this product. Engineers spend heavily on system prompt engineering, and responses still feel "off-brand." The prompt must include extensive instructions on every request.
>
> **With fine-tuning on support transcripts:**
> The model learns the company's specific resolution playbook — how to handle billing disputes, what questions to ask before escalating, which error codes map to which fixes. Responses become consistent with the company's tone and process. The system prompt shrinks from 2,000 tokens to 200 tokens because the behaviour is in the weights, not the prompt.
>
> **What changed in the model:**
> - Output distribution shifted toward the company's resolution language
> - Strong prior for the escalation decision format
> - Domain vocabulary (product names, error codes, internal terms) correctly understood
>
> **What did NOT change:**
> - The model still has no awareness of new product features released after its training cutoff — those still need to be supplied via RAG or context
> - Context window remains identical

---

> **Scenario B — Style Transfer: General Model to Formal Legal Tone**
>
> A legaltech company wants every LLM-generated clause, memo, and summary to read in the precise, formal style of senior partners at a corporate law firm — not the friendly, conversational tone the base model defaults to.
>
> **Why prompting is insufficient here:**
> Instructions like "write in formal legal style" produce inconsistent results. The model interprets "formal" differently depending on the task. Edge cases (humorous content, casual questions) often slip through. Legal teams reject outputs regularly.
>
> **What fine-tuning achieves:**
> The model is fine-tuned on a curated set of 2,000 examples: input text paired with partner-reviewed legal rewrites. After fine-tuning, the style is consistent without any style instruction in the prompt. The model no longer needs to be told how to write — it just writes that way.
>
> **Dataset composition matters:**
> - Cover the full range of document types: memos, clauses, summaries, objection letters
> - Include edge cases: casual input that needs heavy transformation
> - Negative examples help: show what outputs to avoid
>
> **Practical outcome:** 40% reduction in human editing time, near-zero style rejections from legal review. The system prompt drops the style section entirely, cutting input token costs.

---

### Key Takeaways — What Fine-Tuning Changes

- Fine-tuning shifts the model's **behaviour and output style** — not its knowledge or architecture.
- It is most valuable for: consistent format/structure, domain tone, task-specific priors, and reducing prompt length.
- It cannot: update knowledge, extend context windows, or reliably teach facts.
- Data quality beats data quantity — 500 excellent examples beat 10,000 mediocre ones.
- Always establish a strong baseline with prompting before investing in fine-tuning.

### Common Mistakes

- Fine-tuning to inject facts or recent events — the model will hallucinate confidently from the fine-tuning data rather than acknowledging uncertainty.
- Using noisy, unreviewed data from production logs without cleaning — the model faithfully reproduces errors, inconsistencies, and bad agent behaviour.
- Skipping evaluation: fine-tuning without a held-out test set means you cannot measure whether the model actually improved.
- Over-fitting on a small dataset — the model memorises examples rather than generalising the pattern.

---

## 2. Fine-Tuning vs RAG vs Prompting

### The Three Tools

Most LLM problems can be solved with one or a combination of three approaches. Choosing the wrong one wastes time, money, or both.

| Approach | What it does | Primary lever |
|---|---|---|
| **Prompting** | Provides instructions, context, and examples at inference time | Runtime behaviour guidance |
| **RAG** (Retrieval-Augmented Generation) | Retrieves relevant documents and injects them into the prompt | Knowledge grounding |
| **Fine-tuning** | Updates model weights on task-specific data | Persistent behaviour change |

### Comparison Table

| Dimension | Prompting | RAG | Fine-Tuning |
|---|---|---|---|
| **Upfront cost** | Near zero | Medium (index, retrieval pipeline) | High (data curation, compute) |
| **Ongoing cost** | Higher per-request token cost | Medium (retrieval + larger prompts) | Lower per-request (shorter prompts) |
| **Latency** | Lowest (no extra step) | Medium (retrieval adds 50–200ms) | Lowest (no extra step) |
| **Knowledge freshness** | Fresh — update the prompt | Fresh — update the index | Stale — requires a new fine-tune run |
| **Consistency of behaviour** | Low — model can drift from instructions | Low — depends on retrieval quality | High — behaviour is in the weights |
| **Factual grounding** | Weak — relies on model memory | Strong — grounded in retrieved docs | Weak — facts are unreliable in weights |
| **Implementation complexity** | Trivial | Medium | High |
| **Maintenance burden** | Low | Medium (index freshness) | High (retraining when behaviour drifts) |
| **Data requirement** | None | Document corpus | Labelled input-output pairs |
| **Best for** | One-off tasks, fast iteration | Dynamic knowledge, factual Q&A | Style, format, task-specific behaviour |

### Decision Flow

```
                    ┌──────────────────────────────────────┐
                    │  Which approach do I need?            │
                    │  Start here                           │
                    └──────────────┬───────────────────────┘
                                   │
                    ┌──────────────▼───────────────────────┐
                    │  Is this a one-off task or quick      │
                    │  experiment? (no production system)   │
                    └──────────┬───────────────┬────────────┘
                               │               │
                             YES               NO
                               │               │
                     ┌─────────▼──┐   ┌────────▼───────────────────────┐
                     │ PROMPTING  │   │ Does the task require up-to-    │
                     │ Start here │   │ date or private knowledge?      │
                     └────────────┘   └────────┬──────────────┬─────────┘
                                               │              │
                                             YES              NO
                                               │              │
                                    ┌──────────▼──┐  ┌────────▼─────────────────────┐
                                    │    RAG      │  │ Does the task need consistent │
                                    │             │  │ style, format, or behaviour   │
                                    └─────────────┘  │ enforced at every call?       │
                                                      └────────┬──────────┬───────────┘
                                                               │          │
                                                             YES          NO
                                                               │          │
                                                    ┌──────────▼──┐  ┌────▼───────┐
                                                    │ FINE-TUNING │  │ PROMPTING  │
                                                    │             │  │ + iteration│
                                                    └─────────────┘  └────────────┘
```

> **Starting rule:** Always try prompting first. Only move to RAG or fine-tuning when you have a clear reason — dynamic knowledge needs, or persistent behaviour requirements that prompting cannot reliably deliver.

### When to Combine Approaches

The three approaches are not mutually exclusive. The most sophisticated production systems often use all three together:

```
User query
    │
    ▼
System prompt (prompting)          ← defines behaviour and constraints
    +
Retrieved documents (RAG)          ← supplies current knowledge
    +
Fine-tuned model weights           ← enforces style, format, task-specific priors
    │
    ▼
Response
```

---

### Scenarios: Fine-Tuning vs RAG vs Prompting

---

> **Scenario A — Medical Q&A Bot**
>
> A healthtech company builds a tool for clinical pharmacists to query drug interaction data. The database of interactions is updated weekly as new research is published.
>
---

> **Scenario B — An automation platform that returns a Strict JSON Schema**
>
> An automation platform uses an LLM to extract structured data from unstructured user messages. Downstream systems break if the JSON schema is violated — wrong field names, missing required fields, or incorrect types cause pipeline failures.
>

---

> **Scenario C — Document Summarisation**
>
> A consultant needs to summarise a 50-page market research report into a 3-paragraph executive summary. This is a one-time task.
>
---

> **Scenario D — Brand-Voice Customer Bot with a Live Product Catalogue**
>
> A retail brand wants a customer-facing chatbot that speaks in their distinctive, warm, slightly playful brand voice and can answer questions about current product availability, pricing, and promotions — all of which change daily.
>
---

> **Scenario E — Code Generation in an Internal Domain-Specific Language (DSL)**
>
> An infrastructure team has built a proprietary YAML-based DSL for defining cloud resource configurations. The DSL has its own keywords, patterns, and conventions. Engineers spend hours writing boilerplate. They want LLM-assisted code generation, but no frontier model has ever seen this DSL.
>
---

### Key Takeaways — Fine-Tuning vs RAG vs Prompting

- **Default to prompting.** It is free, instant to iterate, and sufficient for the majority of tasks.
- **Reach for RAG** when the task requires knowledge that is dynamic, private, or too large for a context window.
- **Reach for fine-tuning** when the task requires consistent behaviour, style, format, or task-specific priors that prompting cannot reliably deliver — and when the knowledge requirement is stable.
- **Combine when needed.** RAG + fine-tuned model is the most powerful production pattern: fine-tuning handles style/format, RAG handles knowledge.
- **Cost sequence:** Prompting < RAG < Fine-tuning, in both upfront and ongoing cost. Move right only when you have a clear, demonstrated need.

### Common Mistakes

- Jumping to fine-tuning before exhausting prompt engineering — most formatting and style problems can be solved with a well-crafted system prompt and few-shot examples.
- Using fine-tuning to inject facts — the model will mix and hallucinate facts from fine-tuning data, especially when queries are out-of-distribution.
- Treating RAG and fine-tuning as alternatives when they address orthogonal problems — one provides knowledge, the other shapes behaviour.
- Building a RAG pipeline for a task with a static, small knowledge base — if the knowledge fits in a context window and rarely changes, just put it in the system prompt.
- Not versioning fine-tuned model checkpoints — when behaviour regresses, you need to roll back.

---

## 3. PEFT & LoRA

- PEFT - Parameter-Efficient Fine-Tuning
- LoRA - Low-Rank Adaptation

### Why This Matters for Engineers

You are unlikely to implement LoRA yourself — that is a machine learning engineer's job. But as a software engineer or engineering lead working with LLMs, you need to know that PEFT methods exist and what they mean, because they will come up in:

- Conversations with ML teams about fine-tuning feasibility
- Vendor and cloud provider pricing discussions
- Infrastructure sizing decisions (how many GPUs, what VRAM)
- Evaluating whether a fine-tuning request is reasonable given your hardware

### The Problem PEFT Solves

Full fine-tuning updates every parameter in the model. For a 70B parameter model, that means:

```
70B parameters × 4 bytes each = 280GB of weight memory
+ optimizer states (Adam: 2× weights) = 560GB additional
+ gradients = another 280GB

Total VRAM needed for full fine-tune of a 70B model ≈ 1TB+
```

This requires a cluster of 8–16 high-end GPUs. Most teams do not have this. PEFT (Parameter-Efficient Fine-Tuning) solves this by training only a tiny fraction of the parameters.

### LoRA — Low-Rank Adaptation

LoRA is the most widely used PEFT method. Instead of updating all weights in the model's attention layers, it injects small, trainable "adapter" matrices alongside the frozen original weights.

https://youtu.be/KEv-F5UkhxU?si=zkk5m38p4BvbEF7L&t=125
https://www.ibm.com/think/topics/lora

```
Without LoRA (full fine-tune):
  Original weight matrix W (e.g. 4096 × 4096 = 16.7M parameters)
  → W gets updated directly during training

With LoRA:
  Original weight matrix W — FROZEN, never updated
  + Two small adapter matrices A and B injected in parallel
    A: 4096 × 8  (rank r = 8)
    B: 8 × 4096
    A × B = 4096 × 4096 equivalent, but only 65,536 parameters
  → Only A and B are trained

Parameters trained: 65,536 instead of 16,777,216
Reduction: ~0.4% of original parameter count
```

The rank `r` controls the tradeoff: lower rank → fewer parameters, faster training, less expressive; higher rank → more parameters, more expressive, approaches full fine-tune.

**At inference time:** The LoRA adapter matrices are added to the original weights. The merged model behaves like a fine-tuned model but was trained at a fraction of the cost.

### QLoRA — Quantised LoRA

QLoRA extends LoRA by also quantising the base model weights to 4-bit precision during training. This dramatically reduces the VRAM requirement further:

```
Full fine-tune of Llama 4 Maverick (17B active params):
  ~80GB+ VRAM across multiple GPUs

LoRA on Llama 4 Maverick:
  ~40–50GB VRAM (fits on 1–2 A100 80GB GPUs)

QLoRA on Llama 4 Maverick:
  ~20–25GB VRAM (fits on a single A100 80GB GPU)
```

QLoRA makes fine-tuning a 70B-class model feasible on a single high-end consumer GPU (e.g. an RTX 4090 with 24GB, with further quantisation).

### What You Need to Know as an Engineer

1. **It exists and makes fine-tuning tractable** — if an ML engineer tells you fine-tuning requires a 16-GPU cluster, ask if they have evaluated LoRA or QLoRA.
2. **LoRA adapters are small and modular** — a LoRA adapter for a 70B model is typically 50–500MB, not 140GB. Multiple adapters can be stored and swapped for the same base model.
3. **Quality is close to full fine-tune** — for most task-adaptation use cases, LoRA achieves 90–98% of the quality of full fine-tuning at 1–5% of the compute cost.
4. **It is supported by standard tooling** — Hugging Face PEFT library, Unsloth, Axolotl, and most fine-tuning platforms (Together AI, Replicate, Modal) support LoRA natively.
5. **Ask the right question** — when scoping a fine-tuning project, ask: "Are we doing full fine-tuning or LoRA? What rank? What is the expected VRAM requirement and training time?"

### Conceptual Comparison

```
┌─────────────────────────────────────────────────────────────────┐
│                     Full Fine-Tuning                            │
│                                                                 │
│  All 70B parameters updated                                     │
│  ████████████████████████████████████████████████████████████  │
│  VRAM: ~1TB+   Time: days   Cost: $$$$$                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     LoRA Fine-Tuning                            │
│                                                                 │
│  Frozen base: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  Trained adapters: ██                                           │
│  ~0.1–1% of parameters trained                                  │
│  VRAM: ~40–80GB   Time: hours   Cost: $                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     QLoRA Fine-Tuning                           │
│                                                                 │
│  4-bit quantised base: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │
│  Trained adapters: ██                                           │
│  ~0.1% of parameters, base in 4-bit                             │
│  VRAM: ~20–25GB   Time: hours   Cost: $                         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Scenarios: PEFT & LoRA in Practice

---

> **Scenario A — Small Team Wants a Domain-Specific Model on Limited Hardware**
>
> A biotech startup has a single NVIDIA A100 80GB GPU available in their cloud account. They want to fine-tune Llama 4 Maverick (17B active parameters) on 3,000 biomedical research abstracts to improve its accuracy on their internal literature Q&A tool.
>

---

> **Scenario B — Enterprise Wants 10 Department-Specific Variants of One Base Model**
>
> A large enterprise wants customised LLM behaviour for 10 different departments: Legal, Finance, HR, Engineering, Sales, Customer Support, Product, Marketing, Compliance, and Operations. Each department has distinct terminology, output formats, and behavioural requirements.

---

### Key Takeaways — PEFT & LoRA

- **PEFT/LoRA makes fine-tuning accessible** — it reduces compute requirements from multi-GPU clusters to a single consumer or professional GPU.
- **LoRA trains ~0.1–1% of parameters** and achieves 90–98% of full fine-tune quality for most task-adaptation use cases.
- **QLoRA goes further** — 4-bit quantisation of the base model reduces VRAM by another 50%, making a 70B model trainable on a single 80GB A100.
- **Adapters are modular** — one base model + N small adapters is far more cost-effective than N separate full fine-tuned models. This is the right pattern for multi-department or multi-tenant deployments.


### Common Mistakes

- Confusing LoRA with full fine-tuning when estimating compute — always clarify which method is being used before sizing infrastructure.
- Assuming LoRA adapters can be combined arbitrarily — mixing two adapters trained on different tasks does not give you both capabilities; it typically degrades both.
- Using too low a rank (r=1 or r=2) for complex tasks — the adapter lacks expressive capacity and under-fits. r=8 to r=64 is typical for most tasks.
- Forgetting to merge adapter weights before serving at high throughput — dynamically applying adapters at inference time adds latency; merge them into the base model for production serving.
- Not saving the base model checkpoint used during training — if you lose it, you cannot reproduce the fine-tuned model even with the adapter weights saved.

---

## Summary: Choosing Your Approach

```
┌────────────────────────────────────────────────────────────────┐
│                  What is the primary problem?                  │
└──────────────────────────────┬─────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
  ┌───────────────┐   ┌────────────────┐   ┌────────────────────┐
  │ Knowledge or  │   │ Consistent     │   │ One-off task or    │
  │ factual       │   │ behaviour,     │   │ fast experiment    │
  │ grounding     │   │ style, format  │   │                    │
  └───────┬───────┘   └───────┬────────┘   └──────────┬─────────┘
          │                   │                       │
          ▼                   ▼                       ▼
   ┌─────────────┐   ┌───────────────────┐    ┌─────────────┐
   │     RAG     │   │   Fine-Tuning     │    │  Prompting  │
   │             │   │   (use LoRA to    │    │             │
   │             │   │   reduce cost)    │    │             │
   └─────────────┘   └───────────────────┘    └─────────────┘
```
