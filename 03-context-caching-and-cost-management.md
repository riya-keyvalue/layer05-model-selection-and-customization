# Context Caching & Cost Management

LLM costs can sneak up on you. A system that costs $50/month in development becomes $50,000/month at production scale — not because the model changed, but because nobody modelled token usage, caching opportunities, or request patterns before going live. This guide covers the mechanics of token economics, how to dramatically cut costs through prompt caching, and how to design request pipelines that stay within rate limits without sacrificing throughput.

---

## Table of Contents

1. [Token Economics](#1-token-economics)
2. [Prompt Caching](#2-prompt-caching)
3. [Rate Limits, Batching & Async Patterns](#3-rate-limits-batching--async-patterns)

---

## 1. Token Economics

### What Is a Token?

Tokens are the unit of measure for LLM inputs and outputs. A token is roughly 3–4 characters of English text, or about ¾ of a word. Tokenisation is done using algorithms like Byte Pair Encoding (BPE) which splits text into sub-word units — not words, not characters.

```
"The quick brown fox"  →  ["The", " quick", " brown", " fox"]  →  4 tokens

"LLM cost optimisation"  →  ["LL", "M", " cost", " optim", "isation"]  →  5 tokens

Code tends to tokenise efficiently:
"def foo():" → ["def", " foo", "():"]  →  3 tokens

JSON and structured data tokenise less efficiently:
'{"key": "value"}' → ['{', '"', 'key', '"', ':', ' "', 'value', '"', '}'] → 9 tokens
```

### Rules of Thumb for Estimating Tokens

| Content type | Rough token estimate |
|---|---|
| 1 page of English prose | ~500 tokens |
| 1,000 words | ~750 tokens |
| 1 line of code | ~10–15 tokens |
| 100 lines of code | ~800–1,200 tokens |
| 1 page of dense JSON | ~700–1,000 tokens |
| 1 average customer support email | ~150–300 tokens |
| 1 legal contract page | ~600–800 tokens |
| 1 minute of transcribed speech | ~150 tokens |
| 1 hour of transcribed speech | ~9,000 tokens |
| 1 high-resolution image (vision models) | ~800–1,500 tokens |

### Input vs Output Token Pricing Asymmetry

This is the most important pricing concept to internalise early. **Output tokens cost significantly more than input tokens** — typically 3–5× more at the same model tier.

```
Why output is more expensive:
  Input tokens: processed in parallel (one forward pass through the transformer)
  Output tokens: generated one at a time, sequentially (autoregressive decoding)
  → Output is computationally more expensive per token
  → Providers pass that cost to you
```

### Current Pricing Reference (March 2026)

*Prices per 1 million tokens. Always verify against provider documentation — pricing evolves. The cost can also vary depending upon context length*

| Model | Input ($/M tokens) | Output ($/M tokens) | Output multiplier |
|---|---|---|---|
| **GPT-5.4** | $5.00 | $22.50 | 4× |
| **GPT-5.3 Instant** | $1.50 | $6.00 | 4× |
| **Claude Sonnet 4.6** | $3.00 | $15.00 | 5× |
| **Claude Opus 4.6** | $5.00 | $25.00 | 5× |
| **Claude Haiku 4.5** | $1.00 | $5.00 | 5× |
| **Gemini 2.5 Pro** | $1.25 | $10.00 | 8× |
| **Gemini 2.5 Flash** | $0.30 | $2.50 | ~8× |
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | 4× |

> **Key implication:** If you are optimising for cost, reducing output length has more impact per token than reducing input length. A response that goes from 500 to 250 tokens saves 2–5× more than removing 250 tokens from your prompt.

### The Cost Equation

```
Cost per request = (input_tokens × input_price) + (output_tokens × output_price)

Example — Claude Sonnet 4.6:
  System prompt: 800 tokens input
  User message:  200 tokens input
  Response:      400 tokens output

  Cost = (1,000 × $3.00/1M) + (400 × $15.00/1M)
       = $0.003 + $0.006
       = $0.009 per request

At 100,000 requests/day:
  Daily cost  = $900
  Monthly cost = $27,000
```

### Cost Modelling at Production Scale

Before building any LLM feature, do this calculation:

```
Monthly cost estimate:
  = requests_per_day × 30
  × (avg_input_tokens × input_price + avg_output_tokens × output_price)

Variables to measure:
  1. requests_per_day    → product analytics or expected traffic
  2. avg_input_tokens    → measure from your actual prompts, not guesses
  3. avg_output_tokens   → measure from pilot runs; don't trust intuition
  4. model price         → check provider pricing page
```

A 10% error in any of these variables compounds. The most common mistake is underestimating output tokens — developers write short test prompts and get short outputs; production prompts are longer and elicit longer responses.

---

### Quick Check ✦

**Q: If Claude Haiku 4.5 costs $1/$5 per million tokens and you send 500 input tokens and get 300 output tokens per request, what does 1 million requests per month cost?**

<details>
<summary>Show answer</summary>

```
Per request:
  Input:  500 tokens × $1.00/1M  = $0.0005
  Output: 300 tokens × $5.00/1M  = $0.0015
  Total per request               = $0.002

1M requests/month:
  1,000,000 × $0.002 = $2,000/month
```

Now compare: if you switched to Gemini 2.5 Flash-Lite ($0.10/$0.40 per M tokens):

```
Per request:
  Input:  500 × $0.10/1M  = $0.00005
  Output: 300 × $0.40/1M  = $0.00012
  Total per request        = $0.00017

1M requests/month: $170/month

Savings: ~$1,830/month (~92% reduction)
```

The quality tradeoff may or may not be acceptable — but the cost difference is worth evaluating.
</details>

---

### Scenarios: Token Economics

---

> **Scenario A — Repeated System Prompt**
>
> A SaaS platform sends the same 2,000-token system prompt on every API call. They make 500,000 requests per day. Engineers assume this is fine because the system prompt is "just a few paragraphs."
---

> **Scenario B — A content moderation pipeline**
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
---

> **Scenario C — Document Processing pipeline**
>
> A document processing pipeline ingests 10,000 legal contracts per day and answers user questions about their content. The team is deciding between two chunking strategies.
>
> **Strategy 1 — Full document in context (no chunking):**
> Each contract averages 50 pages × 600 tokens/page = 30,000 tokens input. Every question re-sends the full document.

>
> **Strategy 2 — RAG with chunked retrieval:**
> Split each contract into 500-token chunks, embed and index. At query time, retrieve top-5 relevant chunks (~2,500 tokens). Add question and system prompt (~500 tokens).
>

---

### Quick Check ✦

**Q: Your pipeline sends a 500-token system prompt + 200-token user message and receives a 150-token response, using Claude Haiku 4.5 ($1/$5 per M tokens). You run 10,000 requests per day. What is your monthly bill, and what single change would have the biggest cost impact?**

<details>
<summary>Show answer</summary>

```
Per request:
  Input:  (500 + 200) = 700 tokens × $1.00/1M = $0.0007
  Output: 150 tokens × $5.00/1M               = $0.00075
  Total:                                         $0.00145

Daily: 10,000 × $0.00145 = $14.50
Monthly: ~$435

Breakdown by token type:
  Input tokens cost:  $210/month (48%)
  Output tokens cost: $225/month (52%)

Biggest single impact:
  Output tokens are already small (150 tokens at 5× price), so they dominate slightly.
  
  Option A: Reduce output length to 75 tokens → saves ~$112/month (26%)
  Option B: Reduce system prompt from 500 to 200 tokens → saves ~$90/month (21%)
  Option C: Switch to Gemini 2.5 Flash-Lite ($0.10/$0.40) → $37/month total (91% savings)
  
  If Flash-Lite performs adequately for the task, Option C dwarfs the others.
```

</details>

---

### Key Takeaways — Token Economics

- Output tokens cost 3–8× more than input tokens. Optimising for shorter responses has higher ROI than trimming prompts.
- Always model cost before building. Measure actual token counts from real prompts — never estimate from simple test cases.
- High-volume classification tasks almost never need a frontier model. Benchmark cheaper tiers; the quality gap is typically 1–3%.
- The right chunking strategy can reduce costs by 80–90% compared to sending full documents on every request.
- When monthly spend approaches $10k+, small per-token savings compound into tens of thousands of dollars.

### Common Mistakes

- Measuring token counts only in development where prompts are short — production prompts are longer, and responses are too.
- Ignoring output token costs because they "seem small" — at scale they dominate the bill.
- Using a frontier model for tasks that a cheap tier handles just as well.
- Not logging token counts per request in production — you cannot optimise what you cannot measure.
- Forgetting that image tokens (800–1,500 per image) are priced as input tokens — vision pipelines at scale can be surprisingly expensive.

---

## 2. Prompt Caching

### What Prompt Caching Does

When you make an API request, the provider computes a key-value (KV) cache for every token in the prompt as it processes it. Normally this cache is discarded after the request. **Prompt caching** saves that KV cache and reuses it on subsequent requests that start with the same prefix — eliminating the compute cost of re-processing those tokens.

```
Without caching (every request):

Request 1:  [System prompt 5,000 tokens][User: "Summarise clause 3"]
            ← compute KV cache for all 5,000+ tokens every time →

Request 2:  [System prompt 5,000 tokens][User: "What is the liability cap?"]
            ← compute KV cache for all 5,000+ tokens again →

Request 3:  [System prompt 5,000 tokens][User: "List the termination conditions"]
            ← compute KV cache for all 5,000+ tokens again →
```

```
With caching:

Request 1:  [System prompt 5,000 tokens][User: "Summarise clause 3"]
            ← compute KV cache, WRITE to cache →  (cache write cost)

Request 2:  [System prompt 5,000 tokens][User: "What is the liability cap?"]
            ← cache HIT for prefix → only process new tokens → (deep discount)

Request 3:  [System prompt 5,000 tokens][User: "List the termination conditions"]
            ← cache HIT for prefix → only process new tokens → (deep discount)
```

### OpenAI vs Anthropic: Caching Mechanics

| Dimension | OpenAI | Anthropic |
|---|---|---|
| **How it works** | Automatic — no code changes needed | Explicit — you mark which parts to cache with `cache_control` |
| **Minimum prompt size** | 1,024 tokens | 1,024 tokens |
| **Cache hit discount** | 50% off input token price | 90% off input token price |
| **Cache write cost** | Included in normal input price | 25% surcharge on cached tokens (one-time) |
| **Cache lifetime** | ~5–10 minutes (sliding window) | 5 minutes (default); up to 1 hour for extended |
| **Granularity** | Full prompt prefix | Up to 4 explicit `cache_control` breakpoints |
| **Model support** | GPT-5.x, GPT-4.1, o3, o4-mini | Claude Sonnet 4.6, Opus 4.6, Haiku 4.5 |

### OpenAI Caching — How It Works

OpenAI's caching is fully automatic. If your prompt has a prefix of 1,024+ tokens that matches a recent request, OpenAI applies a 50% discount to those cached tokens. You do not change your code — you just ensure your static prefix comes first in the prompt.

```python
# No code change needed — just keep static content at the top of the prompt

response = client.chat.completions.create(
    model="gpt-5.4",
    messages=[
        {
            "role": "system",
            "content": LONG_STATIC_SYSTEM_PROMPT  # 3,000 tokens — cached automatically
        },
        {
            "role": "user",
            "content": user_message  # dynamic — always re-processed
        }
    ]
)
# Check if cache was used:
# response.usage.prompt_tokens_details.cached_tokens
```

**Key rule for OpenAI caching:** Static content must be at the very beginning of the prompt (system message first, then static context, then dynamic content). Any change in the prefix breaks the cache match.

### Anthropic Caching — How It Works

Anthropic's caching is explicit. You mark specific blocks with `{"type": "ephemeral"}` to tell the API to cache them. This gives you precise control over what is cached and what is not.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": LONG_STATIC_SYSTEM_PROMPT,   # 5,000 tokens
            "cache_control": {"type": "ephemeral"}  # mark for caching
        }
    ],
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": STATIC_DOCUMENT_CONTEXT,  # 8,000 tokens
                    "cache_control": {"type": "ephemeral"}  # also cached
                },
                {
                    "type": "text",
                    "text": user_question  # dynamic — not cached
                }
            ]
        }
    ]
)
# Check cache usage:
# response.usage.cache_read_input_tokens    → tokens served from cache
# response.usage.cache_creation_input_tokens → tokens written to cache this call
```

**First request (cache miss + write):**
- System prompt (5,000 tokens): billed at 125% of normal input price (write surcharge)
- Document context (8,000 tokens): billed at 125% of normal input price (write surcharge)
- User question: billed at 100% (not cached)

**Subsequent requests (cache hit):**
- System prompt (5,000 tokens): billed at 10% of normal input price (90% discount)
- Document context (8,000 tokens): billed at 10% of normal input price (90% discount)
- User question: billed at 100% (not cached)

### When Caching Saves the Most

```
Break-even analysis for Anthropic caching:

Cache write cost:   5,000 tokens × 125% = 6,250 token-equivalents
Cache read savings: 5,000 tokens × 90%  = 4,500 tokens saved per hit

Break-even point: 6,250 / 4,500 ≈ 1.4 requests
→ By the 2nd request, caching saves money.
→ By the 100th request, total savings = 100 × 4,500 = 450,000 tokens
```

Caching pays off fastest when:
- The same large prefix is reused across many requests (Q&A on one document)
- Session-based interactions where the system prompt and conversation history are sent repeatedly
- Few-shot example sets in the prompt that are identical across requests

### Designing Prompts for Cache Efficiency

The universal rule: **static content first, dynamic content last.**

```
Optimal prompt order for caching:

1. System prompt (static)           ← always cached
2. Few-shot examples (static)       ← cached
3. Reference documents (static)     ← cached
4. Conversation history (semi-static) ← partially cached (grows over session)
5. Current user message (dynamic)   ← never cached

Bad order (breaks cache):
1. Current date/time                ← changes every request → breaks entire cache
2. User-specific personalisation    ← unique per user → breaks entire cache
3. System prompt                    ← never gets cached because prefix already changed
4. Few-shot examples                ← never gets cached
```

---

### Quick Check ✦

**Q: You have a 6,000-token document that you send alongside every question in a Q&A chatbot. You make 1,000 requests per day. Using Anthropic caching on Claude Sonnet 4.6 ($3/M input tokens), how much do you save per month compared to no caching?**

<details>
<summary>Show answer</summary>

```
Without caching:
  1,000 requests/day × 6,000 tokens × $3.00/1M = $18/day = $540/month

With Anthropic caching:
  Day 1, Request 1 (cache write):
    6,000 tokens × 125% × $3.00/1M = $0.0225  (write surcharge)

  All subsequent requests (cache hit):
    6,000 tokens × 10% × $3.00/1M = $0.0018/request

  Monthly cost (30 days × 1,000 req, with 1 write per day):
    Cache writes: 30 × $0.0225 = $0.675
    Cache reads:  30 × 999 × $0.0018 = $53.95
    Total: ~$54.63/month

Savings: $540 - $54.63 = $485.37/month (90% reduction)

Note: The cache write cost ($0.675) is negligible vs the savings.
If you set a 1-hour cache window, you only pay the write surcharge once per hour,
not once per day — further improving savings.
```

</details>

---

### Scenarios: Prompt Caching

---

> **Scenario A — Legal AI with a 10,000-Token Contract Prefix**
>
> A contract review platform sends the full contract text (10,000 tokens) in every message so users can ask multiple questions about it in a session. The platform serves 5,000 sessions per day, with an average of 8 questions per session.
>
> <details>
> <summary><strong>Show cost breakdown & implementation</strong></summary>
>
> **Without caching (Claude Sonnet 4.6, $3/$15 per M tokens):**
>
> ```
> Daily requests: 5,000 sessions × 8 questions = 40,000 requests
> Input per request: 10,000 (contract) + 300 (question) = 10,300 tokens
>
> Daily input cost: 40,000 × 10,300 × $3.00/1M = $1,236/day = ~$37,080/month
> ```
>
> **With Anthropic caching:**
>
> ```
> Cache write per session (first question):
>   5,000 × 10,000 × 125% × $3.00/1M = $187.50/day
>
> Cache reads (questions 2–8 per session):
>   5,000 × 7 questions × 10,000 × 10% × $3.00/1M = $105/day
>
> Question tokens (all 40,000 requests, not cached):
>   40,000 × 300 × $3.00/1M = $36/day
>
> Total daily: $187.50 + $105 + $36 = $328.50/day = ~$9,855/month
> ```
>
> **Monthly saving: $27,225 (73% reduction)** — from a single implementation change.
>
> **Implementation note:** Mark the contract block with `cache_control` in the first message of each session. Subsequent turns in the same session reuse the cache automatically.
>
> </details>

---

> **Scenario B — Few-Shot Examples Shared Across All Requests**
>
> A classification API uses 20 few-shot examples in every prompt to demonstrate the output format. Each example is ~150 tokens. Total: 3,000 tokens of examples, identical across all requests.
>
> <details>
> <summary><strong>Show cost breakdown & tip</strong></summary>
>
> **Setup:** The examples never change. They are the same for every user, every request.
>
> **With OpenAI automatic caching:**
>
> ```
> Cache eligibility: 3,000 tokens > 1,024 minimum ✓
> Cache hit discount: 50%
>
> Without caching:   3,000 tokens × $5.00/1M = $0.015 per request
> With caching:      3,000 tokens × $2.50/1M = $0.0075 per request
>
> At 200,000 requests/day:
>   Saving: 200,000 × $0.0075 = $1,500/day = $45,000/month
> ```
>
> **Zero code changes needed** — OpenAI's automatic caching kicks in as long as the examples are at a fixed position near the start of the prompt and requests come within the ~10-minute cache TTL.
>
> **Tip:** If requests are too infrequent (e.g. < 1 request per 10 minutes), the OpenAI cache may expire before reuse. Anthropic's explicit caching with a 1-hour window is better for lower-frequency applications.
>
> </details>

---

> **Scenario C — Per-User Dynamic Context (When NOT to Rely on Caching)**
>
> A personalisation assistant builds a unique context for each user: their name, preferences, purchase history, and recent activity — all different per user, injected at the top of the system prompt.
>
> <details>
> <summary><strong>Show why caching fails & solutions</strong></summary>
>
> **Why caching fails here:**
>
> ```
> User A's prompt: [Name: Alice, Preferences: outdoor, History: 12 orders...][message]
> User B's prompt: [Name: Bob, Preferences: electronics, History: 3 orders...][message]
> User C's prompt: [Name: Carol, Preferences: cooking, History: 27 orders...][message]
> ```
>
> Every prompt has a unique prefix. There is no shared static content at the start. The cache never hits because the prefix never matches between requests from different users.
>
> **Solutions:**
>
> 1. **Restructure the prompt:** Move user-specific context to the end (after a shared static system prompt). The shared system prompt gets cached; only the dynamic suffix changes.
>
>    ```
>    [Shared static system prompt — 1,500 tokens]  ← cached
>    [User: Alice | Preferences: outdoor | ...]     ← dynamic, not cached
>    [User's message]                               ← dynamic
>    ```
>
> 2. **Per-user session caching:** For Anthropic, if a user sends multiple messages in the same session, cache their personalised context for the duration of that session (1-hour window). The cache only hits within one user's session, not across users — but that is still valuable for multi-turn conversations.
>
> 3. **Accept no caching for this pattern:** If the user context is small (< 500 tokens) and sessions are short (1–2 turns), the caching overhead is not worth the engineering complexity. Optimise elsewhere.
>
> </details>

---

### Quick Check ✦

**Q: Which of the following prompt structures maximises cache efficiency? Explain why.**
- **A:** `[Today's date][User name][System instructions][Few-shot examples][User question]`
- **B:** `[System instructions][Few-shot examples][User name][Today's date][User question]`
- **C:** `[System instructions][Few-shot examples][User question][Today's date][User name]`

<details>
<summary>Show answer</summary>

**B is the best of the three, but C is slightly better for the cache-eligible portion.**

- **A is worst:** Today's date and user name are dynamic — they change every request. Putting them first means the cache prefix changes constantly and no caching occurs for the static instructions and examples.

- **B is better:** Static content (instructions + examples) comes first — those tokens are eligible for caching. But the dynamic tokens (user name, date) are still before the user question, meaning the effective cached prefix ends before them.

- **C is best for maximum caching:** The user question separates the static prefix from the dynamic metadata. However, in practice today's date and user name after the question is unusual structuring.

**Optimal design:**
```
[System instructions (static)]          ← cached
[Few-shot examples (static)]            ← cached
[User question (dynamic)]               ← not cached
[Today's date + user name (dynamic)]    ← not cached (or inject into question)
```

The key rule: all static content before any dynamic content. Dynamic tokens after the cache breakpoint do not affect the cache hit for the static prefix.
</details>

---

### Key Takeaways — Prompt Caching

- **Use caching whenever you have a large static prefix** (1,024+ tokens) repeated across requests. The ROI is almost always positive after 2+ requests.
- **OpenAI caching is automatic** — just structure your prompt with static content first. No code changes needed.
- **Anthropic caching offers 90% discount** but requires explicit `cache_control` markers and has a write surcharge. The break-even is ~2 requests.
- **The single most impactful design rule:** static content first, dynamic content last. Every dynamic token at the top of the prompt breaks the cache for everything below it.
- **Know the TTL:** OpenAI cache lasts ~5–10 minutes (auto-renews on hit). Anthropic default is 5 minutes, extendable to 1 hour. Low-frequency applications should use extended TTL or Anthropic's explicit control.

### Common Mistakes

- Putting timestamps, request IDs, or user-specific data at the top of the system prompt — this destroys cache hit rate.
- Expecting caching to work across different users with unique contexts — caching only works when the prefix is byte-for-byte identical.
- Not monitoring `cache_read_input_tokens` in the API response — you cannot verify caching is working without checking.
- Using Anthropic caching for small prompts (< 1,024 tokens) — the write surcharge exceeds the savings.
- Forgetting that cache TTL means high-latency request spikes can cause cache misses — always handle a degraded (non-cached) cost scenario in your budget.

---

## 3. Rate Limits, Batching & Async Patterns

### Understanding Rate Limits

Every LLM API enforces rate limits to ensure fair usage and system stability. There are two primary limit types:

| Limit type | What it controls | Unit |
|---|---|---|
| **RPM** | Requests Per Minute | Number of API calls |
| **TPM** | Tokens Per Minute | Total tokens (input + output) across all calls |
| **TPD** | Tokens Per Day | Daily token ceiling (some providers) |

You can hit either limit independently. A pipeline with small, frequent requests hits RPM. A pipeline with large prompts hits TPM even at low RPM.

```
Example: You have 60,000 TPM and 500 RPM limits.

Scenario 1 — Small requests:
  300 requests/minute × 100 tokens each = 30,000 TPM
  → Neither limit is hit (300/500 RPM, 30,000/60,000 TPM)
  → RPM is relatively tighter (60% utilized vs 50% for TPM)
  → No throttling — headroom exists on both

Scenario 2 — Large requests:
  50 requests/minute × 2,000 tokens each = 100,000 TPM
  → TPM is the bottleneck (100,000 > 60,000 limit)
  → You would be throttled despite only 50 RPM
```

### Tier Progression

Limits increase as you spend more with a provider or explicitly request upgrades. Tier structure varies by provider but follows this general pattern:

| Tier | Typical profile | OpenAI TPM (GPT-5.x) | Anthropic TPM (Claude Sonnet 4.6) |
|---|---|---|---|
| **Tier 1** | New account / first $100 | 30,000 | 40,000 |
| **Tier 2** | $100–$500 spent | 450,000 | 400,000 |
| **Tier 3** | $500–$5,000 spent | 800,000 | 800,000 |
| **Tier 4** | $5,000–$50,000 spent | 2,000,000 | 4,000,000 |
| **Enterprise** | Negotiated contract | Custom | Custom |

*Always verify current limits on the provider's platform page — tiers and thresholds change.*

### The Batch API

Both OpenAI and Anthropic offer a Batch API for workloads that do not require real-time responses.

| Feature | OpenAI Batch API | Anthropic Message Batches |
|---|---|---|
| **Discount** | 50% off standard pricing | 50% off standard pricing |
| **Turnaround** | Up to 24 hours | Up to 24 hours |
| **Max requests per batch** | 50,000 | 10,000 |
| **Use case** | Offline processing, classification, evaluation | Offline processing, classification, evaluation |
| **Rate limits** | Separate batch quota, much higher | Separate batch quota |
| **Response format** | JSONL file returned | Results available via API poll |

**When to use batch:** Any workload where you can tolerate up to 24-hour latency. Common examples: nightly data enrichment, bulk classification, evaluation runs, content generation pipelines that feed the next day's work.

### Async Patterns for High-Throughput Systems

When you need real-time throughput beyond your rate limits, or want to avoid blocking user-facing requests on LLM calls, async patterns are essential.

#### Pattern 1 — Async Fan-Out

Send multiple requests concurrently rather than sequentially. This maximises throughput within your TPM/RPM budget.

```python
import asyncio
import anthropic

client = anthropic.AsyncAnthropic()

async def process_document(doc: str) -> str:
    response = await client.messages.create(
        model="claude-haiku-4-5",
        max_tokens=500,
        messages=[{"role": "user", "content": f"Summarise: {doc}"}]
    )
    return response.content[0].text

async def process_all(documents: list[str]) -> list[str]:
    # Fan out all requests concurrently, respect rate limits with semaphore
    semaphore = asyncio.Semaphore(20)  # max 20 concurrent requests

    async def bounded(doc):
        async with semaphore:
            return await process_document(doc)

    return await asyncio.gather(*[bounded(doc) for doc in documents])
```

**Throughput gain:** Sequential processing of 100 documents at 1 second/request = 100 seconds. Async fan-out with 20 concurrent = ~5 seconds (20× throughput within rate limits).

#### Pattern 2 — Retry with Exponential Backoff

Rate limit errors (HTTP 429) are transient. Retry with increasing wait intervals to absorb burst throttling without overwhelming the API.

```python
import time
import random

def call_with_backoff(fn, max_retries=5):
    for attempt in range(max_retries):
        try:
            return fn()
        except RateLimitError:
            if attempt == max_retries - 1:
                raise
            wait = (2 ** attempt) + random.uniform(0, 1)  # jitter prevents thundering herd
            time.sleep(wait)
            # Wait times: 1s, 2s, 4s, 8s, 16s (+ jitter)
```

**Why jitter matters:** Without random jitter, all retrying requests hit the API at exactly the same time after the backoff period — creating another burst and another 429. Jitter staggers the retries.

#### Pattern 3 — Token Bucket / Request Queue

For sustained high-throughput pipelines, a token bucket regulates the outflow rate to stay within limits.

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────────┐
│   Incoming   │────▶│   Request Queue     │────▶│   LLM API    │
│   Requests   │     │                     │     │              │
│  (any rate)  │     │  Rate: max 500 RPM  │     │  Rate limit: │
└──────────────┘     │  TPM: max 800K/min  │     │  500 RPM     │
                     │                     │     │  800K TPM    │
                     │  Workers drain      │     └──────────────┘
                     │  queue at safe rate │
                     └─────────────────────┘
```

Implementations: Redis with sorted sets for time-windowed counting, `asyncio.Queue` in Python, or managed services like AWS SQS with a consumer that rate-limits itself.

#### Pattern 4 — Priority Queue with Tiered Models

Route requests by urgency: real-time user-facing requests get a fast/expensive model; background tasks get a cheap/slow model or the Batch API.

```
Incoming requests
        │
        ├─ User-facing (latency < 1s) ──▶ Claude Haiku 4.5 (fast, cheap)
        │
        ├─ Background enrichment      ──▶ Gemini 2.5 Flash (async, cheap)
        │
        └─ Nightly batch jobs         ──▶ Batch API (50% discount, 24hr)
```

---

### Quick Check ✦

**Q: You have a TPM limit of 100,000 tokens/minute on Claude Haiku 4.5. Each request uses 800 tokens input and you expect 400 tokens output (1,200 tokens total per request). How many concurrent requests can you safely run per minute, and what RPM does that translate to?**

<details>
<summary>Show answer</summary>

```
TPM budget: 100,000 tokens/minute
Tokens per request: 800 (input) + 400 (output) = 1,200 tokens

Max requests per minute: 100,000 / 1,200 = 83.3 → 83 requests/minute

RPM = 83 RPM (if each request takes ~1 second end-to-end)

To stay safely under the limit (leave 10% headroom):
  Safe RPM: 83 × 0.9 = ~75 requests/minute

Semaphore for async fan-out:
  If average request takes 1.5 seconds:
  Concurrent requests needed to sustain 75 RPM:
  75 RPM / 60 seconds × 1.5 seconds = ~1.9 → set semaphore to 2

  If average request takes 3 seconds:
  75 RPM / 60 × 3 = ~3.75 → set semaphore to 4
```

</details>

---

### Scenarios: Rate Limits, Batching & Async Patterns

---

> **Scenario A — Nightly Batch Job: Classify 500,000 Support Tickets**
>
> A customer support platform needs to classify all new support tickets into 12 categories overnight — before the morning shift starts at 8 AM. Tickets arrive throughout the day; the batch runs at 2 AM. Latency does not matter; cost does.
>
---

> **Scenario B — Real-Time Chatbot Hitting RPM Limits at Peak**
>
> A B2C chatbot serves 1,000 concurrent users during peak hours. Each conversation turn makes one API call. At peak, this generates ~800 requests/minute. The account's RPM limit is 500.
>
> **Symptoms:** Users see delayed responses or errors during peak; off-peak is fine.
---

> **Scenario C — Async Document Ingestion Fan-Out**
>
> A document intelligence platform ingests 10,000 documents per hour during business hours. Each document needs: (1) a title extracted, (2) a 3-sentence summary, (3) 5 keyword tags. These are three separate LLM calls per document — 30,000 calls/hour.
>
---

> **Scenario D — Multi-Tenant SaaS: Rate Budget Isolation**
>
> A multi-tenant SaaS platform serves 200 enterprise customers on a shared LLM backend. One large customer sends a batch spike that uses 80% of the platform's TPM budget, causing degraded response times for all other customers.
>
> **The problem:** Shared rate limits mean one noisy-neighbour tenant can starve others.

---

### Quick Check ✦

**Q: Your nightly batch job processes 200,000 documents using Claude Haiku 4.5. Each request is 600 input tokens and 200 output tokens. You currently use the standard API. How much would you save per month by switching to the Batch API, and what is the maximum time your batch can take to still complete before 8 AM if it starts at 11 PM?**

<details>
<summary>Show answer</summary>

```
Per request:
  Input:  600 tokens × $1.00/1M = $0.0006
  Output: 200 tokens × $5.00/1M = $0.001
  Total:                          $0.0016 per request

Standard API, 200,000 requests:
  $0.0016 × 200,000 = $320/night = $9,600/month

Batch API (50% discount):
  $160/night = $4,800/month

Monthly saving: $4,800/month

Maximum allowed time:
  Start: 11 PM
  Deadline: 8 AM next day
  Window: 9 hours = 540 minutes

  Anthropic Batch API guarantees 24-hour turnaround, so 9 hours is within SLA.
  
  Note: Submit in batches of 10,000 (Anthropic limit):
  200,000 / 10,000 = 20 batches
  Submit all 20 in parallel → results available within the 24hr window.
  In practice, most batches complete in 1–4 hours, well within the 9-hour window.
```

</details>

---

### Key Takeaways — Rate Limits, Batching & Async Patterns

- **Know your two limits:** RPM and TPM are independent ceilings. Large prompts hit TPM; frequent small requests hit RPM. Measure both.
- **Async fan-out is the highest-leverage throughput improvement** — 20 concurrent workers can achieve 20× the throughput of sequential processing within the same rate limits.
- **The Batch API gives a 50% cost reduction** for any workload that tolerates up to 24-hour latency. Nightly jobs, classification pipelines, and eval runs should all use it.
- **Retry with exponential backoff + jitter** is non-negotiable. Without jitter, all retrying clients synchronise and create a thundering herd that prolongs the throttle.
- **Multi-tenant systems need per-tenant budgets.** A shared rate limit without isolation turns one noisy tenant into a platform-wide outage.

### Common Mistakes

- Treating rate limit errors as permanent failures and surfacing them to users — they are transient and should be retried with backoff.
- Ignoring the TPM limit and only thinking about RPM — at large prompt sizes, TPM is almost always the binding constraint.
- Not using the Batch API for nightly jobs — paying standard prices for work that has no latency requirement is leaving 50% cost savings on the table.
- Building sequential LLM pipelines when the subtasks are independent — fan-out concurrency is almost always applicable and easy to implement.
- Sharing one API key across all tenants without isolation — one spike brings everyone down.
- Not monitoring rate limit headroom in production — set up alerts when TPM/RPM usage exceeds 70% of the limit so you can tier-up before it causes user impact.

---

## Summary: Cost Optimisation Checklist

Work through this list before shipping any LLM feature to production:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Token Economics                                                      │
├──────────────────────────────────────────────────────────────────────┤
│  ☐ Measured actual input/output token counts from real prompts       │
│  ☐ Built monthly cost model (requests/day × token costs × 30)        │
│  ☐ Benchmarked cheaper model tiers for the task                      │
│  ☐ Audited system prompt — removed redundant instructions            │
│  ☐ Set up token count logging in production                          │
├──────────────────────────────────────────────────────────────────────┤
│  Prompt Caching                                                       │
├──────────────────────────────────────────────────────────────────────┤
│  ☐ Identified static prefix (system prompt, few-shot examples, docs) │
│  ☐ Moved all static content before dynamic content in prompt order   │
│  ☐ Enabled caching (OpenAI: automatic; Anthropic: cache_control)     │
│  ☐ Monitoring cache_read_input_tokens in API responses               │
├──────────────────────────────────────────────────────────────────────┤
│  Rate Limits & Batching                                               │
├──────────────────────────────────────────────────────────────────────┤
│  ☐ Know current TPM and RPM limits for each model used               │
│  ☐ Non-real-time workloads use Batch API (50% discount)              │
│  ☐ Async fan-out implemented for parallel processing                 │
│  ☐ Retry logic with exponential backoff + jitter in place            │
│  ☐ Per-tenant rate budgets for multi-tenant systems                  │
│  ☐ Alerts set for TPM/RPM usage > 70% of limit                       │
└──────────────────────────────────────────────────────────────────────┘
```
