# Case Studies

## Wisely

- Generates an AI Twin for users
- Uses a Llama model hosted on AWS Bedrock + RAG for summarization

---

## Axari

- Managed platform that reads through Slack, calendar, emails, etc.
- Earlier used Claude; switched to Gemini 2.5 Pro by leveraging prompting
- Cost-effective approach through prompt engineering rather than fine-tuning

---

## Ally

- Assists counsellors and psychologists
- Uses an STT → TTT → TTS pipeline
- STS (Speech-to-Speech) was expensive and lacked the reasoning capability of a TTT model
- Known downside of the STT → TTT → TTS pipeline: **latency**

---

## Vakil

- Uses **Llama 3.2 3B** model; evaluated models in the 1B–7B range, focusing on Small Language Models (SLMs)
- There were 3 phases of model training in Vakil - continued pretraining, fine tuning and alignment
- Fine tuning is done via LORA technique
- Model selection rationale:
  - Ran model selection experiments using metrics like **perplexity on Indian Legal tokens**
  - Applied **scaling laws of LLMs** to guide the selection
  - [Base Model Evaluation — detailed doc](https://ai-legal-slm.atlassian.net/wiki/spaces/VS/pages/26443777/Base+Model+Evaluation)
