### Scenarios: Fine-Tuning vs RAG vs Prompting

---

> **Scenario A — Medical Q&A Bot with Up-to-Date Drug Interactions**
>
> A healthtech company builds a tool for clinical pharmacists to query drug interaction data. The database of interactions is updated weekly as new research is published.
>
> **Verdict: RAG**
>
> **Why RAG wins:**
> Drug interaction knowledge is factual, structured, and changes frequently. Fine-tuning cannot keep up — every update to the interaction database would require a new training run (expensive, slow). Storing interactions in a vector database and retrieving relevant ones at query time gives the model accurate, current information.
>
> **Why not fine-tuning:** Fine-tuning on drug interaction data causes the model to blend and hallucinate facts. A model that confidently states incorrect drug interactions is dangerous. RAG keeps the model grounded in retrieved, verifiable documents.
>
> **Why not prompting alone:** You cannot stuff an entire drug interaction database into a context window for every query. Retrieval selects only the relevant subset.
>
> **Implementation:** Chunk the interaction database into per-drug-pair records. Embed with a text-embedding model. At query time, retrieve the top-k relevant interactions and inject into the prompt. The model's job is synthesis and explanation, not memory.

---

> **Scenario B — Model Must Always Return a Strict JSON Schema**
>
> An automation platform uses an LLM to extract structured data from unstructured user messages. Downstream systems break if the JSON schema is violated — wrong field names, missing required fields, or incorrect types cause pipeline failures.
>
> **Verdict: Fine-tuning (+ structured output constraints)**
>
> **Why fine-tuning wins:**
> Prompting can produce correct JSON most of the time, but "most of the time" is not good enough for a pipeline that runs 100,000 times per day. Fine-tuning on input-output pairs of unstructured text → valid JSON instils the schema as a learned prior. The model almost never deviates from the format.
>
> **Combine with structured outputs:** Both OpenAI and Anthropic offer constrained decoding / structured output modes that enforce a JSON schema at the token level. Fine-tuning reduces the need for heavy grammar constraints; structured output mode provides the final safety net.
>
> **Why not RAG:** This is a format/behaviour problem, not a knowledge problem. Retrieval does not help enforce output structure.
>
> **Dataset tip:** Include adversarial inputs — messages designed to confuse the schema — in the training set. The model needs to have seen edge cases to handle them reliably.

---

> **Scenario C — One-Off Document Summarisation**
>
> A consultant needs to summarise a 50-page market research report into a 3-paragraph executive summary. This is a one-time task.
>
> **Verdict: Prompting**
>
> **Why prompting wins:**
> The task is well-defined, non-recurring, and within the capability of any current frontier model. A clear prompt with format instructions ("Summarise in 3 paragraphs: key market trends, competitive landscape, and recommendation") is all that is needed. There is no reason to build infrastructure.
>
> **Why not fine-tuning:** Fine-tuning has weeks of lead time for data collection, training, and evaluation. For a one-time task, this is disproportionate.
>
> **Why not RAG:** The document fits in the context window of any modern model (Gemini 2.5 Pro at 1M tokens, Claude Sonnet 4.6 at 1M tokens). No retrieval layer is needed.
>
> **The rule:** If you can paste the problem into the context window and get a good answer, start there. Add complexity only when you have outgrown it.

---

> **Scenario D — Brand-Voice Customer Bot with a Live Product Catalogue**
>
> A retail brand wants a customer-facing chatbot that speaks in their distinctive, warm, slightly playful brand voice and can answer questions about current product availability, pricing, and promotions — all of which change daily.
>
> **Verdict: Fine-tuning + RAG (hybrid)**
>
> **Why fine-tuning for voice:**
> Brand voice is a persistent behaviour requirement. Prompting instructions ("be warm and playful") produce inconsistent results across diverse queries. Fine-tuning on 1,000 curated examples of the brand's desired tone locks in the style regardless of what the user asks.
>
> **Why RAG for catalogue:**
> Product availability, pricing, and promotions change daily. This knowledge cannot be in model weights — it must be retrieved at query time from a live inventory and promotions database. Fine-tuning on the catalogue would be stale within 24 hours.
>
> **How they combine:**
> The fine-tuned model provides the consistent brand voice and conversational behaviour. RAG provides the current product knowledge. The system prompt is kept minimal because tone is in the weights.
>
> **Maintenance split:** The fine-tuned model needs retraining only when the brand voice evolves (rare). The RAG index is updated continuously (daily or in real-time).

---

> **Scenario E — Code Generation in an Internal Domain-Specific Language (DSL)**
>
> An infrastructure team has built a proprietary YAML-based DSL for defining cloud resource configurations. The DSL has its own keywords, patterns, and conventions. Engineers spend hours writing boilerplate. They want LLM-assisted code generation, but no frontier model has ever seen this DSL.
>
> **Verdict: Fine-tuning**
>
> **Why fine-tuning wins:**
> The DSL does not exist in any public training data. Prompting with examples helps at the margins but the model constantly reverts to standard YAML or Terraform syntax. Fine-tuning on 3,000 input-output pairs (natural language description → valid DSL config) teaches the model the DSL grammar as a learned prior.
>
> **Why not RAG:** RAG can supply DSL documentation and examples in context, which helps. But the model still has no innate understanding of the DSL grammar and frequently makes syntax errors that require human correction. Fine-tuning is necessary to achieve reliable, runnable output.
>
> **Why not prompting alone:** Few-shot examples in the prompt (5–10 examples) improve quality but the DSL has hundreds of resource types and configuration patterns. You cannot cover the full distribution in a prompt without consuming the entire context window.
>
> **Dataset construction:** Generate training pairs programmatically. For each existing valid DSL config, write a natural language description. This bootstraps a large labelled dataset cheaply.

---

### Scenarios: PEFT & LoRA in Practice

---

> **Scenario A — Small Team Wants a Domain-Specific Model on Limited Hardware**
>
> A biotech startup has a single NVIDIA A100 80GB GPU available in their cloud account. They want to fine-tune Llama 4 Maverick (17B active parameters) on 3,000 biomedical research abstracts to improve its accuracy on their internal literature Q&A tool.
>
> **Verdict: QLoRA is the right tool**
>
> **Why QLoRA:** Full fine-tuning of Llama 4 Maverick would require multiple GPUs and significant memory. QLoRA brings the requirement down to ~25–30GB VRAM — comfortably within a single A100 80GB.
>
> **Training time estimate:** With QLoRA at rank 16, 3,000 examples, 3 epochs: approximately 2–4 hours on a single A100. Full fine-tuning of a comparable-quality dense model would take 10–20 hours on a multi-GPU setup.
>
> **What to tell your ML engineer:** "We have one A100 80GB. We want to fine-tune on 3,000 examples. Please use QLoRA with rank 16. Target training time under 6 hours. Save checkpoints every epoch."
>
> **What you get:** A LoRA adapter file of ~200MB. The base Llama 4 Maverick weights stay frozen. At inference time, load the base model + merge the adapter. The merged model outperforms the base model on biomedical Q&A tasks with near-zero infrastructure overhead beyond what you already have.

---

> **Scenario B — Enterprise Wants 10 Department-Specific Variants of One Base Model**
>
> A large enterprise wants customised LLM behaviour for 10 different departments: Legal, Finance, HR, Engineering, Sales, Customer Support, Product, Marketing, Compliance, and Operations. Each department has distinct terminology, output formats, and behavioural requirements.
>
> **Verdict: LoRA adapters as modular plugins — one adapter per department, one shared base model**
>
> **Why not 10 separate fine-tuned models:** Full fine-tuning produces 10 separate 70B+ model copies — each requiring its own GPU serving stack. Cost: 10× the infrastructure. Maintenance: 10× the retraining burden when the base model is upgraded.
>
> **Why LoRA adapters win:**
>
> ```
> Shared base model (e.g. Llama 4 Maverick, served once)
>         │
>         ├── Legal adapter (180MB)      → load at request time
>         ├── Finance adapter (200MB)    → load at request time
>         ├── HR adapter (150MB)         → load at request time
>         ├── Engineering adapter (220MB)→ load at request time
>         └── ... 6 more adapters
> ```
>
> The base model is loaded once and shared in memory. Adapters are small enough to swap in per request or per user session. Total storage: ~1.8GB for all 10 adapters vs ~1.4TB for 10 full model copies.
>
> **When the base model is upgraded** (e.g. a new Llama release performs better): retrain 10 small LoRA adapters instead of 10 full fine-tuning runs. Training cost drops by ~95%.
>
> **Infrastructure note:** This pattern is supported natively by vLLM's LoRA serving mode, which can load and unload adapters dynamically without restarting the server.

---