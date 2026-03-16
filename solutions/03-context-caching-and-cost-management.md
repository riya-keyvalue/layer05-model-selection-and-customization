### Scenarios: Token Economics

---

> **Scenario A — Repeated System Prompt**
>
> A SaaS platform sends the same 2,000-token system prompt on every API call. They make 500,000 requests per day. Engineers assume this is fine because the system prompt is "just a few paragraphs."
>
> **Cost impact calculation (Claude Sonnet 4.6 at $3/M input tokens):**
>
> ```
> System prompt tokens per request: 2,000
> Daily requests:                   500,000
> Daily system prompt tokens:       1,000,000,000 (1B tokens)
>
> Daily cost (input only, system prompt):
>   1B tokens × $3.00/1M = $3,000/day
>
> Monthly cost from system prompt alone: ~$90,000/month
> ```
>
> **What to do:**
> - **Trim aggressively:** audit the system prompt. Remove redundant instructions, verbose examples, and anything that can be inferred from context. A 2,000-token prompt often compresses to 600–800 tokens without loss of behaviour.
> - **Cache it:** if you cannot trim, cache it (covered in Section 2). Caching reduces per-request cost for the cached prefix by 50–90%.
> - **Move static content to fine-tuning:** if the system prompt mostly encodes a behaviour the model needs to exhibit at every call, that behaviour is a fine-tuning candidate — not a runtime prompt candidate.
>
> **Trimmed outcome:** Reducing from 2,000 to 700 tokens saves 65% of system prompt cost — $58,500/month on this example.

---

> **Scenario B — Choosing the Right Model Tier for High-Volume Classification**
>
> A content moderation pipeline classifies 2 million user-generated posts per day into 8 categories (spam, hate speech, violence, etc.). Each classification requires a short prompt (~300 tokens in) and returns a single label (~20 tokens out). The team's first instinct is to use GPT-5.4.
>
> **Cost comparison across model tiers:**
>
> ```
> Daily volume:          2,000,000 requests
> Avg input tokens:      300
> Avg output tokens:     20
>
> ┌─────────────────────────┬────────────┬─────────────┬──────────────────┐
> │ Model                   │ Input $/M  │ Output $/M  │ Monthly Cost     │
> ├─────────────────────────┼────────────┼─────────────┼──────────────────┤
> │ GPT-5.4                 │ $5.00      │ $20.00      │ ~$96,000         │
> │ Claude Sonnet 4.6       │ $3.00      │ $15.00      │ ~$60,000         │
> │ Claude Haiku 4.5        │ $1.00      │ $5.00       │ ~$20,000         │
> │ Gemini 2.5 Flash        │ $0.30      │ $2.50       │ ~$7,200          │
> │ Gemini 2.5 Flash-Lite   │ $0.10      │ $0.40       │ ~$2,400          │
> └─────────────────────────┴────────────┴─────────────┴──────────────────┘
> ```
>
> **The right answer:** Classification is one of the tasks where smaller, cheaper models perform closest to frontier models. The difference between GPT-5.4 and Gemini 2.5 Flash-Lite on an 8-category classification task is typically 1–3% accuracy.
>
> **Decision process:**
> 1. Benchmark all tiers on a labelled sample of 500 real posts
> 2. If Flash-Lite achieves >95% agreement with GPT-5.4 on your categories, use it
> 3. For edge cases and ambiguous content (typically 3–5%), route to a stronger model
>
> **Cost with routing:**
> - 97% of requests → Gemini 2.5 Flash-Lite: $2,328/month
> - 3% of requests  → Claude Sonnet 4.6: $1,800/month
> - Total: ~$4,128/month vs $96,000/month for GPT-5.4 across the board
>
> **Lesson:** Never run high-volume classification on a frontier reasoning model without first benchmarking cheaper tiers.

---

> **Scenario C — Chunking Strategy Differences in Document Processing**
>
> A document processing pipeline ingests 10,000 legal contracts per day and answers user questions about their content. The team is deciding between two chunking strategies.
>
> **Strategy 1 — Full document in context (no chunking):**
> Each contract averages 50 pages × 600 tokens/page = 30,000 tokens input. Every question re-sends the full document.
>
> ```
> 10,000 contracts/day × 5 questions each = 50,000 requests/day
> Input per request: 30,000 (document) + 200 (question) = 30,200 tokens
> Output per request: ~300 tokens
>
> Daily input cost (Claude Sonnet 4.6 at $3/M):
>   50,000 × 30,200 × $3/1M = $4,530/day = ~$135,900/month
> ```
>
> **Strategy 2 — RAG with chunked retrieval:**
> Split each contract into 500-token chunks, embed and index. At query time, retrieve top-5 relevant chunks (~2,500 tokens). Add question and system prompt (~500 tokens).
>
> ```
> 50,000 requests/day
> Input per request: 2,500 (retrieved chunks) + 500 (system + question) = 3,000 tokens
> Output per request: ~300 tokens
>
> Daily input cost (Claude Sonnet 4.6):
>   50,000 × 3,000 × $3/1M = $450/day = ~$13,500/month
>
> Plus embedding cost (one-time per contract):
>   10,000 contracts × 30,000 tokens = 300M tokens
>   At $0.10/1M for text-embedding-3-small: $30 one-time
> ```
>
> **Cost comparison:** $135,900/month vs $13,500/month — a 90% reduction.
>
> **Quality consideration:** Full-document context has higher recall on questions that require reasoning across distant parts of the contract. RAG can miss if the relevant chunk is not retrieved. For contracts where cross-document reasoning is critical, use full context on premium contracts and RAG on the rest.

---

### Scenarios: Rate Limits, Batching & Async Patterns

---

> **Scenario A — Nightly Batch Job: Classify 500,000 Support Tickets**
>
> A customer support platform needs to classify all new support tickets into 12 categories overnight — before the morning shift starts at 8 AM. Tickets arrive throughout the day; the batch runs at 2 AM. Latency does not matter; cost does.
>
> **Verdict: OpenAI Batch API or Anthropic Message Batches**
>
> **Cost comparison (Claude Haiku 4.5 at $1/$5 per M tokens):**
>
> ```
> Per ticket:
>   Input:  400 tokens (ticket + system prompt)
>   Output: 15 tokens (category label)
>
> Standard API cost (500,000 tickets):
>   Input:  500,000 × 400 × $1.00/1M  = $200
>   Output: 500,000 × 15  × $5.00/1M  = $37.50
>   Total:  $237.50/night
>
> Batch API cost (50% discount):
>   Total: $118.75/night
>   Monthly saving: ~$3,562.50
> ```
>
> **Implementation:**
>
> ```python
> # Prepare batch input (JSONL format)
> requests = [
>     {
>         "custom_id": f"ticket-{ticket['id']}",
>         "method": "POST",
>         "url": "/v1/messages",
>         "body": {
>             "model": "claude-haiku-4-5",
>             "max_tokens": 20,
>             "messages": [
>                 {"role": "user", "content": build_prompt(ticket)}
>             ]
>         }
>     }
>     for ticket in tickets
> ]
>
> # Submit batch at 2 AM, poll for results, load into DB before 8 AM
> ```
>
> **Throughput note:** Anthropic Message Batches handle up to 10,000 requests per batch. For 500,000 tickets, submit 50 batches of 10,000 in sequence, or run them in parallel if your account has sufficient batch quota.

---

> **Scenario B — Real-Time Chatbot Hitting RPM Limits at Peak**
>
> A B2C chatbot serves 1,000 concurrent users during peak hours. Each conversation turn makes one API call. At peak, this generates ~800 requests/minute. The account's RPM limit is 500.
>
> **Symptoms:** Users see delayed responses or errors during peak; off-peak is fine.
>
> **Solutions in order of complexity:**
>
> **1. Upgrade tier (fastest fix):** Request a tier upgrade. Moving from Tier 2 to Tier 3 often 1.5–2× the RPM limit. Takes 1–2 business days.
>
> **2. Distribute across models:** Split traffic between two models (e.g., Claude Haiku 4.5 and Gemini 2.5 Flash). Each has its own rate limit. 400 RPM to each provider = 800 RPM total effective throughput.
>
> **3. Queue with graceful degradation:**
>
> ```
> Incoming requests (800 RPM peak)
>          │
>          ▼
>     Request Queue
>          │
>     Rate limiter (500 RPM)
>          │
>     ├── Process immediately (below limit) ──▶ API call
>     └── Queue overflow ──▶ show typing indicator + serve from queue as capacity frees
> ```
>
> **4. Reduce RPM by compressing turns:** Instead of one API call per user message, batch a user's last 3 messages into one call using a sliding window — reducing API calls per conversation without degrading quality.
>
> **5. Response caching for common queries:** Log the most frequent questions. For exact or near-exact matches, return cached responses without an API call. Common in FAQ-style chatbots.

---

> **Scenario C — Async Document Ingestion Fan-Out**
>
> A document intelligence platform ingests 10,000 documents per hour during business hours. Each document needs: (1) a title extracted, (2) a 3-sentence summary, (3) 5 keyword tags. These are three separate LLM calls per document — 30,000 calls/hour.
>
> **Sequential processing time:**
>
> ```
> 30,000 calls × 1.5 sec/call = 45,000 seconds = 12.5 hours
> This is way too slow for real-time ingestion.
> ```
>
> **Async fan-out design:**
>
> ```
> Document arrives
>      │
>      ▼
> Task queue (e.g. Redis or AWS SQS)
>      │
>      ├──▶ Worker pool (50 workers)
>      │         │
>      │         ├──▶ Extract title   (async, Gemini 2.5 Flash-Lite)
>      │         ├──▶ Generate summary (async, Gemini 2.5 Flash)
>      │         └──▶ Extract keywords (async, Gemini 2.5 Flash-Lite)
>      │                    │
>      └──────────────────  ▼
>                     Aggregate results → store to DB
> ```
>
> **Throughput with 50 workers:**
>
> ```
> 50 concurrent requests × (1 / 1.5 sec) = ~33 requests/sec = ~2,000 requests/min
>
> 30,000 calls / 2,000 per minute = 15 minutes total
> Down from 12.5 hours → 50× speedup
> ```
>
> **Rate limit check:**
>
> ```
> At 2,000 RPM across 3 call types:
>   Avg tokens/call: 400 input + 100 output = 500 tokens
>   TPM: 2,000 × 500 = 1,000,000 TPM
>
> Gemini 2.5 Flash TPM limit (Tier 3): 4,000,000 TPM ✓
> RPM limit: typically well above 2,000 RPM at Tier 3 ✓
> ```
>
> **Cost (Gemini 2.5 Flash at $0.30/$2.50 per M):**
>
> ```
> 30,000 calls/hour × 8 working hours = 240,000 calls/day
> Input:  240,000 × 400 × $0.30/1M = $28.80/day
> Output: 240,000 × 100 × $2.50/1M = $60/day
> Total: ~$88.80/day = ~$2,664/month
> ```

---

> **Scenario D — Multi-Tenant SaaS: Rate Budget Isolation**
>
> A multi-tenant SaaS platform serves 200 enterprise customers on a shared LLM backend. One large customer sends a batch spike that uses 80% of the platform's TPM budget, causing degraded response times for all other customers.
>
> **The problem:** Shared rate limits mean one noisy-neighbour tenant can starve others.
>
> **Solutions:**
>
> **1. Per-tenant budget allocation:**
>
> ```
> Total platform TPM: 4,000,000/minute
>
> Allocation tiers:
>   Enterprise customers (20):  100,000 TPM each  → 2,000,000 TPM total
>   Business customers (80):     20,000 TPM each  → 1,600,000 TPM total
>   Starter customers (100):      4,000 TPM each  →   400,000 TPM total
>   Reserved buffer:                              →   none (use for burst)
> ```
>
> **2. Tenant-level token bucket in middleware:**
>
> ```python
> class TenantRateLimiter:
>     def __init__(self, tenant_id: str, tpm_budget: int):
>         self.tenant_id = tenant_id
>         self.tpm_budget = tpm_budget
>         self.redis = get_redis()
>
>     def check_and_consume(self, tokens: int) -> bool:
>         key = f"tpm:{self.tenant_id}:{current_minute()}"
>         used = self.redis.incrby(key, tokens)
>         self.redis.expire(key, 60)
>         return used <= self.tpm_budget
>         # If False: queue request or return 429 to tenant
> ```
>
> **3. Burst allowance for premium tenants:**
> Allow tenants to "borrow" from the next minute's budget up to 2× their allocation — smooths spikes without starvation.
>
> **4. Separate API keys per tenant tier:** Use different API keys mapped to different OpenAI/Anthropic projects, each with their own rate limits. Enterprise tenants get a dedicated key; starter tenants share a pooled key.

---