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

## Pencil
## Text Generation Models

GPT-4o, OpenAI o4 mini, OpenAI o3 mini, OpenAI o3, OpenAI GPT-4.1, Google Gemini 2.5 Pro, Google Gemini 2.5 Flash, Google Gemini 2.0 Flash, OpenAI GPT-5, OpenAI GPT-5 mini, OpenAI GPT-5 Nano, GPT 5.1 - Thinking, GPT-5.4 Instant, GPT-5.4 Thinking, Gemini 3 flash thinking, Gemini 3 flash instant, Gemini 3 pro instant, Gpt 5.1 instant, Gpt 5.2 Thinking, Gpt 5.2 Instant, Gemini 3.1 Pro Instant, Gemini 3.1 Pro Thinking, Gemini 3.1 Flash Lite Instant, Gemini 3.1 Flash Lite Thinking

## Image Generation Models

Stability SD, Stable Diffusion 3.5, Google Imagen 3, Getty Images, OpenAI DALLE 3, Bria Base v2.3, Adobe Firefly v4, GPT image 1.5, Imagen 4 Ultra, Nano Banana - Gemini 2.5 Flash Image, Nano Banana Pro - Gemini 3, Nano Banana 2 - Gemini 3.1, Kling AI IMAGE 2 (China Only), Kling AI IMAGE 2.1 (China Only)

## Video Generation Models

Runway Gen3 Alpha Turbo, Google Veo 2, Runway Gen4 Alpha Turbo, Google Veo 3, Google Veo 3.1, Adobe Firefly Video, Sora 2, Sora 2 Pro, Kling AI VIDEO 2.5 Turbo (China Only), Kling AI VIDEO 2.1 (China Only)

## Audio & Enhancement Models

Google Gemini 2.5 Pro TTS, Google Chirp 3 HD, Topaz High Fidelity v2, Topaz Prob4