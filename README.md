# Altibbi AI Medical Assistant

A bilingual (Arabic/English) AI-powered medical assistant built on top of Altibbi's own content and doctor network. It answers user health questions strictly from Altibbi's article library (via RAG), can be talked to by voice, can read and interpret uploaded medical reports and images, and gives the Altibbi team an admin dashboard to monitor usage and cost.

The goal is not a generic medical chatbot — it's a **trustworthy, source-grounded** assistant that only answers from Altibbi's verified content, and that can responsibly point users toward a real Altibbi doctor when appropriate.

---

## What it does today

- **RAG over Altibbi articles** — user questions are answered only using Altibbi's own article database, not the model's general knowledge.
- **Chat with the RAG** — a standard text chat interface backed by the RAG pipeline above.
- **Call the RAG (voice)** — the same RAG pipeline can be triggered by voice, with responses read back to the user via text-to-speech.
- **Medical report / image analysis** — users can upload lab reports, scans, or photos, and Gemini analyzes and explains them.
- **Doctor data collection** — doctor profiles (name, summary, location, contact, insurance accepted, availability) are scraped and stored in Supabase.
- **Admin dashboard** — a read-only dashboard showing top users, total chats, total users, cost per user, and average cost per user.

## What's being built next

- **Two separate UIs**: a end-user chat UI and a separate admin UI.
- **User sign-in** via Google/Gmail (through Supabase Auth).
- **Admin sign-in** via email + password (also through Supabase Auth), gating access to the dashboard.
- **Automatic language matching** — if the user writes in Arabic, the assistant replies in Arabic; if in English, it replies in English.
- **Email chat summaries via Resend** — sent from a dedicated sending domain/address rather than a personal inbox, using the address the user signed in with (no more asking the user to type their email into the chat).
- **A structured doctors database** (name, specialty, contact, location, working hours/دوام, etc.) in Postgres, with the scraping/sync pipeline built in n8n.
- **Natural-language → SQL search** — user prompts about doctors are converted into SQL queries (via an LLM call, not a fixed rules engine) to search the doctors table.
- **Responsible doctor recommendations** — the assistant does not recommend a specific doctor unprompted; it only offers to suggest one as a follow-up question after answering the user's medical question.

---

## Architecture

```
User (chat / voice / file upload)
        │
        ▼
   n8n Webhook  ──────────────────────────────────────────┐
        │                                                  │
        ├─ File attached? ──► Gemini Vision (analyze report/image)
        │                                                  │
        ├─ Text query ──► Embed Query ──► Search Milvus/Zilliz (Altibbi articles)
        │                      │
        │                      ▼
        │              Build RAG Context ──► Gemini (answer, grounded in retrieved articles)
        │                      │
        │                      ├─ Doctor lookup needed? ──► Postgres (doctors DB) ──► Gemini (match specialty)
        │                      │
        │                      ├─ Voice requested? ──► Gemini (rewrite for speech) ──► ElevenLabs TTS
        │                      │
        │                      └─ Save turn to Mem0 (conversation memory)
        │
        └─ Summary requested ──► Mem0 (fetch history) ──► Gemini (summarize) ──► Resend (email)

Frontend (HTML/JS + Supabase JS client)
  ├─ Supabase Auth (Google OAuth + email/password for admin)
  └─ Admin dashboard reads usage/cost data from Supabase
```

## Technologies used

| Layer | Technology |
|---|---|
| Workflow orchestration / backend logic | **n8n** (self-hosted/cloud), triggered via webhook |
| LLM (chat answers, report analysis, speech rewriting, specialty matching) | **Google Gemini** (Flash / Vision) |
| Vector database (RAG over Altibbi articles) | **Milvus**, hosted on **Zilliz Cloud** |
| Structured data (users, usage, doctors) | **Postgres**, via **Supabase** |
| Auth | **Supabase Auth** (Google OAuth for users, email/password for admin) |
| Conversation memory | **Mem0** |
| Text-to-speech | **ElevenLabs** |
| Transactional email (chat summaries) | **Resend** |
| Frontend | Static **HTML/CSS/JavaScript**, `@supabase/supabase-js` client |

## Data flow for a chat message

1. Frontend sends the user's message (and optional file) to the n8n webhook.
2. If a file is attached, it's sent to Gemini Vision for analysis and the explanation is returned directly.
3. Otherwise, the message is embedded and used to search the `altibbi_articles` collection in Milvus/Zilliz.
4. The retrieved article chunks are assembled into a context block and passed to Gemini along with the user's question, so the answer is grounded strictly in Altibbi's content.
5. If the question implies a need for a doctor referral, the workflow checks the Postgres doctors table (matching by specialty via an LLM call) and — per policy — offers this only as a follow-up suggestion, never as the primary answer.
6. If the request came in as voice, the answer is rewritten in a more natural spoken form and sent to ElevenLabs for TTS; the response includes both the original text and the generated audio, with a graceful fallback to the browser's own TTS if ElevenLabs is unavailable.
7. Each turn is logged to Mem0 so a full conversation summary can later be generated and emailed to the user via Resend.

## Environment variables & credentials

None of the API keys, tokens, or secrets used by the workflow are committed to this repository. Every credential is referenced as a placeholder (e.g. `YOUR_GEMINI_API_KEY`) in the workflow JSON and must be supplied via n8n's credential store / environment variables when you import it:

- `YOUR_GEMINI_API_KEY` – Google Gemini (used as a `key` query parameter on every Gemini call — Vision analysis, RAG answer, chat summary, speech rewrite, and specialty matching)
- `YOUR_EMBEDDING_SERVICE_TOKEN` – internal embedding service
- `YOUR_ZILLIZ_API_TOKEN` – Zilliz Cloud (Milvus)
- `YOUR_MEM0_API_KEY` – Mem0
- `YOUR_ELEVENLABS_API_KEY` – ElevenLabs
- `YOUR_RESEND_API_KEY` – Resend

The frontend uses Supabase's public **publishable/anon** key (`SUPABASE_ANON_KEY`), which is safe to expose client-side by design — access to data is controlled by Supabase Row Level Security (RLS) policies, not by keeping this key secret. No `service_role` key is used in the frontend.

## Repository contents

- `index.html` – the frontend chat UI (and, currently, the admin dashboard) with Supabase Auth wired in.
- `N8N_medical_assistant.json` – the exportable n8n workflow implementing the RAG pipeline, file analysis, doctor lookup, TTS, and email summary logic.


