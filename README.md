[README.md](https://github.com/user-attachments/files/29973449/README.md)
<div align="center">

# 📬 Inbox Triage Assistant

**An n8n automation that reads your inbox, classifies emails with an LLM, and drafts replies — so you don't have to open every message just to know what matters.**

![n8n](https://img.shields.io/badge/n8n-EA4B71?style=flat&logo=n8n&logoColor=white)
![Groq](https://img.shields.io/badge/Groq-Llama%203.3%2070B-F55036?style=flat)
![Gmail API](https://img.shields.io/badge/Gmail-API-EA4335?style=flat&logo=gmail&logoColor=white)
![Status](https://img.shields.io/badge/status-working%20prototype-brightgreen)

</div>

---

## What it does

Every new email that lands in the inbox is automatically:

- 🔍 **Classified** — `urgent`, `spam`, or `needs-reply`, by an LLM reasoning over sender, subject, and body
- 🏷️ **Labeled** — sorted into Gmail labels so the inbox is scannable at a glance
- ✍️ **Drafted a reply** (for `needs-reply` emails only) — a context-aware response is prepared and saved to Gmail Drafts, ready for human review

**Nothing is ever auto-sent.** The AI prepares; a human decides. That boundary is intentional — see [Design Philosophy](#design-philosophy).

---

## Architecture

\`\`\`mermaid
flowchart TD
    A[📥 Gmail Trigger<br/>polls inbox every minute] --> B[🤖 Groq LLM<br/>classifies: urgent / spam / needs-reply]
    B --> C[Parse response]
    C --> D{Switch on category}
    D -->|urgent| E[🏷️ Label: AI/Urgent]
    D -->|spam| F[🏷️ Label: AI/Spam]
    D -->|needs-reply| G[🏷️ Label: AI/NeedsReply]
    G --> H[📄 Fetch full message<br/>decode body + sender]
    H --> I[🤖 Groq LLM<br/>drafts a reply]
    I --> J[📝 Gmail: Create Draft<br/>same thread, awaiting review]
\`\`\`

---

## Tech stack

| Layer | Tool |
|---|---|
| Orchestration | [n8n](https://n8n.io) (cloud) |
| Classification + drafting | [Groq API](https://groq.com) — Llama 3.3 70B Versatile |
| Email platform | Gmail API (trigger, labeling, message retrieval, draft creation) |
| Auth | n8n Header Auth credentials (no keys stored in workflow JSON) |

---

## Design philosophy

This isn't built to fully automate email — it's built to **remove the triage overhead** while keeping a human in the loop for anything that actually goes out. Two decisions reflect that:

1. **Drafts, never sends.** Every generated reply sits in Gmail Drafts. The AI's job ends at "here's a reasonable starting point."
2. **A second-layer classifier, not a spam filter replacement.** Testing showed that clearly promotional emails are often intercepted by Gmail's own spam filter before they ever reach this workflow. That's fine — this system is designed to handle the *ambiguous* cases Gmail's filter doesn't resolve (e.g. a terse, impersonal-sounding email that's actually a legitimate message), not to duplicate spam detection Gmail already does well.

---

## Key technical challenges solved

<details>
<summary><b>1. Tone vs. intent in spam classification</b></summary>
<br>

An early prompt version classified a generic-sounding "Congratulations on your new role" email as spam — the model was reading impersonal *tone* as a spam signal. Fixed by rewriting the system prompt to explicitly separate tone from intent, then validated against a 5-email test set (clear spam, clear urgent, casual needs-reply, and the original borderline case).
</details>

<details>
<summary><b>2. Decoding Gmail's nested MIME body structure</b></summary>
<br>

Gmail's API returns message bodies as base64url-encoded MIME parts, often nested (\`multipart/alternative\` → \`parts[]\`). Wrote a recursive part-finder to locate the \`text/plain\` part at any nesting depth, with a snippet fallback if decoding fails.
</details>

<details>
<summary><b>3. Field-name drift across Gmail node operations</b></summary>
<br>

Different Gmail operations (Trigger, Get, Add Label) return differently shaped JSON for the same data — e.g. message ID appears as \`id\` in some outputs and \`messageId\` in others, and \`headers\` is sometimes an array of \`{name, value}\` pairs and sometimes a flat keyed object. Required inspecting each node's raw output individually rather than assuming a consistent schema.
</details>

<details>
<summary><b>4. Building valid JSON with multi-line email content</b></summary>
<br>

Injecting raw email text (with real line breaks) into a fixed-field JSON body broke the JSON parser. Solved by switching the HTTP Request body to Expression mode and building the payload with \`JSON.stringify()\`, which handles escaping automatically.
</details>

<details>
<summary><b>5. Keeping API keys out of the exported workflow</b></summary>
<br>

Groq API keys were initially typed directly into request headers — which would have leaked into the exported workflow JSON. Moved authentication into n8n's Header Auth credential store, so the key never appears in the workflow file itself (verified by searching the exported JSON for the key prefix before publishing this repo).
</details>

---

## Known limitations

- No retry/error handling yet around the Groq API calls — a failed call fails the run rather than retrying gracefully
- Classification tested on a small (5-email) hand-built set, not large-scale real inbox traffic
- Draft replies are generated from the current email in isolation, with no memory of prior thread history
- Only processes mail that reaches the Gmail inbox — anything Gmail's native spam filter intercepts upstream never reaches this system

## Possible next steps

- [ ] Add retry/error handling around LLM calls
- [ ] Expand and track accuracy against a larger, more varied test set
- [ ] Pull thread history into the reply-drafting prompt for better continuity on ongoing conversations

---

<div align="center">

Built with n8n + Groq · [Report an issue](../../issues)

</div>
