# Gita Oracle — HTTP & Credentials, In Depth
### Day 4 of the n8n + AI learning series

> A single webhook that answers questions about the Bhagavad Gita using the Groq API — built **six different ways**, so you can see, side by side, how HTTP requests actually get authenticated in the real world.

---

## Table of Contents

1. [What This Project Actually Teaches](#1-what-this-project-actually-teaches)
2. [Core Concepts (the Wikipedia part)](#2-core-concepts-the-wikipedia-part)
3. [System Architecture](#3-system-architecture)
4. [The Six Methods, Explained](#4-the-six-methods-explained)
5. [File Manifest](#5-file-manifest)
6. [Setup Guide](#6-setup-guide)
7. [How to Extend This — Swap the AI Provider](#7-how-to-extend-this--swap-the-ai-provider)
8. [Common Pitfalls & Fixes](#8-common-pitfalls--fixes)
9. [Glossary](#9-glossary)
10. [Practice Exercises — Master This by Modifying It](#10-practice-exercises--master-this-by-modifying-it)

---

## 1. What This Project Actually Teaches

Most tutorials show you *one* way to call an API and stop there. That hides the actual decision a developer makes every time they wire up an integration: **where does the secret key live, and who — or what — gets to see it?**

This project asks the same question (*"What does Krishna say about duty?"*) through **six different pipes**, so the differences are visible rather than theoretical:

| Layer | What varies between the six methods |
|---|---|
| **Abstraction** | Does a framework handle the HTTP call for you, or do you build it by hand? |
| **Credential storage** | Is the API key in a provider-specific vault, a generic vault, or typed manually? |
| **Auth mechanism** | Custom header, generic Header Auth, or dedicated Bearer Auth? |
| **Prompt construction** | Written inline in an expression, or assembled programmatically in code? |

Once you can answer those four questions for *any* API integration, you've learned the actual transferable skill — not just "how to call Groq."

---

## 2. Core Concepts (the Wikipedia part)

### 2.1 What is a Webhook?
A **webhook** is a URL that waits for something to be sent to it (usually an HTTP `POST` request with a JSON body) and then runs code in response. Instead of your app *polling* a server every few seconds asking "anything new?", the other system pushes data to you the moment something happens. In this project, the webhook is the front door: the HTML page `POST`s a question to it, and n8n takes it from there.

### 2.2 What is an HTTP Request, Structurally?
Every HTTP request has four parts, and understanding them is the entire point of "HTTP in depth":

| Part | What it is | Example in this project |
|---|---|---|
| **Method** | The verb — what you want to do | `POST` |
| **URL** | Where the request goes | `https://api.groq.com/openai/v1/chat/completions` |
| **Headers** | Metadata *about* the request — including auth | `Authorization: Bearer gsk_xxx`, `Content-Type: application/json` |
| **Body** | The actual payload | `{ "model": "...", "messages": [...] }` |

Authentication almost always lives in the **headers**, not the body — that's a deliberate convention so that logging tools, caches, and proxies (which often log bodies but not sensitive headers) don't accidentally leak your key.

### 2.3 Credential Storage: Three Philosophies
| Approach | How it works | Used in this project |
|---|---|---|
| **Hardcoded** | The key is typed directly into the request (❌ never do this — used only for illustration) | Nowhere — deliberately absent |
| **Provider-specific credential** | The tool (n8n) has a dedicated, pre-built connector for a specific API, which knows exactly what header/format that API expects | Method 2 (`predefined`) |
| **Generic credential** | The tool provides a general-purpose vault (e.g. "store any header," "store any bearer token") that works for *any* API, even ones without dedicated support | Methods 3, 4, 5, 6 |

The generic approach is the more important skill to learn — it's what you fall back on for the thousands of APIs that don't have a pre-built connector.

### 2.4 Header Auth vs. Bearer Auth
These are easy to confuse because they produce an *identical* HTTP header, but they're configured differently:

- **Generic Header Auth** — you manually type both the header **name** (`Authorization`) and the header **value** (`Bearer gsk_xxx`), including the word "Bearer" yourself. Maximum flexibility — works for APIs with unusual auth headers (`X-API-Key`, `Api-Token`, etc.).
- **Bearer Auth** — a dedicated credential type built specifically for the extremely common `Authorization: Bearer <token>` pattern. You paste just the token; the tool adds the header name and the word "Bearer" for you. Less flexible, less typing, fewer mistakes.

### 2.5 Why a "Fallback" / Validation Path Matters
Any time you route execution based on a value someone else controls (a `method` field from a webhook body, a status code, a dropdown selection), that value can be missing, misspelled, or malicious. A validation step that catches "none of the above" and returns a clear error is a basic reliability pattern — the alternative is a silent hang or a cryptic crash with no explanation, which is far worse to debug.

---

## 3. System Architecture

```
┌─────────────────┐        POST /webhook/gita-oracle
│  HTML Console    │ ─────────────────────────────────┐
│  (browser)        │   { question, method, ts }        │
└─────────────────┘                                    ▼
                                          ┌───────────────────────┐
                                          │   Gita Question         │
                                          │   (Webhook node)        │
                                          └───────────┬─────────────┘
                                                        ▼
                                          ┌───────────────────────┐
                                          │   Validate Method       │
                                          │   (IF node)              │
                                          └─────┬───────────┬───────┘
                                       valid ────┘           └──── invalid
                                          ▼                          ▼
                              ┌───────────────────┐    ┌─────────────────────┐
                              │  Route by Method    │    │ Respond – Unknown   │
                              │  (Switch node)       │    │ Method                │
                              └──┬──┬──┬──┬──┬──┬───┘    └─────────────────────┘
                                 │  │  │  │  │  │
                     1: agent ───┘  │  │  │  │  └─── 6: code_prompt
                  2: predefined ────┘  │  │  └─────── 5: header_auth_persona
                     3: header_auth ───┘  └────────── 4: bearer_auth

     Each branch → calls Groq its own way → shapes { reply, method } → Respond to Webhook
```

---

## 4. The Six Methods, Explained

### Method 1 — `agent`: AI Agent Node
n8n's **AI Agent** node, paired with a **Groq Chat Model** sub-node, handles the entire HTTP call internally. You never see a request being built — you configure a system prompt and a model, and the framework does the rest.
- **Best for:** getting something working fast, or when you want built-in features like tool-calling and memory.
- **Trade-off:** you can't see or control the exact request being sent.

### Method 2 — `predefined`: Dedicated Credential
An **HTTP Request** node, built by hand, authenticated using n8n's purpose-built `groqApi` credential type. The key is stored in n8n's credential vault and injected automatically — it never appears in the node's parameters or in an exported workflow JSON.
- **Best for:** APIs that n8n has explicit, first-class support for.

### Method 3 — `header_auth`: Generic Header Auth
Same manual HTTP Request node, but Groq has no dedicated credential type in this branch on purpose — so the key goes into n8n's **Generic Header Auth** credential instead. You type the header name (`Authorization`) and value (`Bearer <key>`) yourself.
- **Best for:** any API without a pre-built connector — the most broadly reusable pattern you'll learn here.

### Method 4 — `bearer_auth`: Generic Bearer Auth
Identical hiding guarantee to Method 3, but using n8n's **dedicated Bearer Auth** credential type instead of the fully generic Header Auth. You paste only the raw token — n8n constructs the `Authorization: Bearer <token>` header for you.
- **Best for:** the extremely common case where an API just wants a bearer token — less to configure, fewer typos.

### Method 5 — `header_auth_persona`: Hidden Key + Custom System Prompt
Same credential mechanism as Method 3, plus a `Set` node that defines a named persona (a system prompt) before the request is built.
- **Teaches:** hiding the key doesn't limit how much you can customize the request body.

### Method 6 — `code_prompt`: Prompt Built in Code
Same credential mechanism as Method 3, but a **Code node** runs first — trimming the input, enforcing a length guard, and assembling the `messages[]` array in plain JavaScript instead of an inline expression.
- **Teaches:** once your prompt-building logic gets more complex than a one-liner (validation, truncation, multi-turn history), a Code node is more maintainable than a giant expression.

---

## 5. File Manifest

| File | Purpose |
|---|---|
| `gita-oracle-console.html` | The frontend — question input, six-method selector, live request-trace visualizer, response display |
| `gita-oracle-workflow-v3.json` | The n8n workflow — all six branches, importable directly |
| `sticky-notes.json` | Two documentation sticky notes, pasteable onto the n8n canvas |
| `mock-test-payloads.json` | Sample JSON bodies + curl commands for testing each branch without the HTML frontend |
| `README.md` | This file |

---

## 6. Setup Guide

1. **Import the workflow** — n8n → Workflows → Import from File → `gita-oracle-workflow-v3.json`.
2. **Create three credentials:**
   - `groqApi` (n8n's dedicated Groq credential type) — used by Methods 1 & 2.
   - **Generic Header Auth** — Name: `Authorization`, Value: `Bearer <your-groq-key>` — used by Methods 3, 5, 6.
   - **Generic Bearer Auth** — just the raw key, no `Bearer ` prefix — used by Method 4.
3. **Assign each credential** to its corresponding nodes (they show a red warning triangle until you do).
4. **Paste in the sticky notes** (`sticky-notes.json`) for in-canvas documentation — copy the file contents, click the canvas, `Ctrl+V`.
5. **Activate the workflow**, copy the production webhook URL.
6. **Open `gita-oracle-console.html`**, paste the URL into the settings panel, and test each of the six radio options.
7. Optionally, test the raw webhook first using `mock-test-payloads.json` (via curl or Postman) before touching the HTML at all — this isolates workflow bugs from frontend bugs.

---

## 7. How to Extend This — Swap the AI Provider

The entire point of this architecture is that **only the HTTP Request node's URL, headers, and body shape change** — the webhook, routing, and response-shaping logic stay identical. Try rebuilding Methods 2–6 against a different provider to prove this to yourself:

### OpenAI (ChatGPT)
- **Endpoint:** `https://api.openai.com/v1/chat/completions`
- **Auth header:** `Authorization: Bearer <key>` — identical shape to Groq, so Methods 3–6 need almost no changes, just a new credential.
- **Body:** same `messages[]` array shape as Groq (Groq's API is OpenAI-compatible by design).
- **Dedicated credential type:** n8n has a built-in `openAiApi` credential — a direct parallel to Method 2's `groqApi`.

### Google Gemini
- **Endpoint:** `https://generativelanguage.googleapis.com/v1beta/models/<model>:generateContent`
- **Auth:** commonly passed as a **query parameter** (`?key=<key>`) rather than a header — this is a genuinely different pattern worth building as a *seventh* method (`query_auth`), since n8n's Generic **Query Auth** credential type exists specifically for this case.
- **Body shape:** different from OpenAI/Groq — uses a `contents[]` array with `parts[]` instead of `messages[]` with `role`/`content`. Rebuilding this forces you to actually read a new API's docs instead of copy-pasting.

### Anthropic (Claude)
- **Endpoint:** `https://api.anthropic.com/v1/messages`
- **Auth header:** `x-api-key: <key>` (**not** `Authorization: Bearer`) — a good example of an API that genuinely needs Generic Header Auth rather than Bearer Auth, since the header name itself is non-standard.
- **Additional required header:** `anthropic-version: <date>`.

### Suggested exercise
Duplicate the `header_auth` branch three times, point each at a different provider, and add a `provider` field to the webhook payload alongside `method`. You'll end up with a matrix (provider × auth-method) instead of a single row — genuinely useful for comparing cost, latency, and answer quality across models on the same question.

---

## 8. Common Pitfalls & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Red warning triangle on an HTTP node | No credential assigned | Open the node, select/create the credential |
| Workflow hangs with no response | No fallback for an unmatched `method` value | Confirm the `Validate Method` → `Respond – Unknown Method` path is connected |
| Switch node keeps resetting output numbers | UI auto-renumbering bug in some n8n versions | Use an IF-node chain instead of relying on manual output indices, or switch to Expression mode |
| `401 Unauthorized` from the API | Wrong header format (e.g. missing `Bearer ` prefix in Header Auth, or including it in Bearer Auth) | Recheck Section 2.4 above — the two auth types expect different input formats |
| Frontend shows "Request failed" | Webhook is on the **test** URL, not **production**, or workflow isn't activated | Activate the workflow; use the production URL |

---

## 9. Glossary

- **API (Application Programming Interface):** a defined way for one program to ask another program to do something.
- **REST:** an architectural style for APIs built on standard HTTP methods (`GET`, `POST`, etc.) and resource-based URLs.
- **Credential / Secret:** any value (API key, password, token) that grants access and must be kept private.
- **Environment variable:** a way to store a secret outside of source code, injected at runtime.
- **Principle of least privilege:** give each component only the access it strictly needs — relevant when deciding which nodes get which credentials.
- **Idempotency:** whether calling the same request twice produces the same result — worth knowing before you retry a failed API call.
- **Rate limit:** a cap on how many requests you can make in a given time window, enforced by most APIs including Groq, OpenAI, and Gemini.

---

## 10. Practice Exercises — Master This by Modifying It

The fastest way to actually own this concept is small, deliberate changes to what already works — not new projects from scratch. Each exercise below touches **one thing at a time**, in rising order of difficulty. Do them in order; each one assumes the previous ones still work.

### 🟢 Beginner — same mechanism, different provider

**Exercise 1: Swap Groq for OpenAI, change nothing else**
Duplicate the `header_auth` branch. Point the URL at `https://api.openai.com/v1/chat/completions`, create a new Header Auth credential with your OpenAI key, change the `model` field. Everything else — the auth *mechanism*, the `messages[]` shape — stays identical, because OpenAI and Groq speak the same API dialect.
> **What this proves:** the auth pattern you learned isn't Groq-specific. It's the standard `Bearer` pattern used by dozens of APIs.

**Exercise 2: Break it on purpose**
Temporarily put a wrong/expired key into one credential and send a request. Read the exact error n8n surfaces, then look at what the HTML console shows the user.
> **What this proves:** you understand what a `401 Unauthorized` looks like *before* you hit one for real, and whether your error-handling (the "Unknown Method" pattern) actually degrades gracefully or just breaks.

**Exercise 3: Add a seventh branch — model comparison**
Add `method: "compare"` that calls **two** providers (e.g. Groq and OpenAI) for the same question and returns both replies side by side in one JSON response.
> **What this proves:** you can compose multiple HTTP Request nodes in one branch, not just route to a single one.

### 🟡 Intermediate — new auth shapes

**Exercise 4: Build the Gemini branch (Query Auth)**
Add `method: "query_auth"`. Point it at Google's Gemini endpoint, and instead of Header Auth or Bearer Auth, use n8n's **Generic Query Auth** credential type — Gemini expects the key as a URL parameter (`?key=...`), not a header at all.
> **What this proves:** not every API puts the secret in a header. This is the auth shape you'll hit most often *after* Bearer tokens.

**Exercise 5: Rebuild the Anthropic branch (custom header name)**
Add `method: "claude"`. Anthropic uses `x-api-key: <key>` instead of `Authorization: Bearer`. Generic Header Auth is the *only* one of your four hidden-key mechanisms that can express this, since Bearer Auth hardcodes the header name.
> **What this proves:** you now know exactly when Bearer Auth is enough, and when you're forced back to fully-generic Header Auth.

**Exercise 6: Secure the webhook itself**
Right now, anyone with your webhook URL can call it. Add a `Header Auth` check on the **incoming** webhook — require a shared secret header, and reject requests without it before they reach `Validate Method`.
> **What this proves:** everything you learned about *outgoing* auth (to Groq) applies symmetrically to *incoming* auth (to your own webhook). This is the same concept from the other side.

### 🔴 Advanced — structural changes

**Exercise 7: Multi-turn conversation**
Modify the `code_prompt` Code node to accept an array of `{role, content}` messages from the frontend instead of a single `question` string, so a user can ask a follow-up that remembers the previous answer.
> **What this proves:** you can extend the Code-node pattern from Method 6 to handle real conversation state, not just single-shot Q&A.

**Exercise 8: Add response caching**
Before calling any provider, check (via a `Code` node or n8n's built-in cache) whether the exact same `{question, method}` pair was answered in the last hour; if so, skip the HTTP call and return the cached reply.
> **What this proves:** you understand that hitting a paid API on every request is a choice, not a requirement — and where in the pipeline caching belongs (before the auth/HTTP step, not after).

**Exercise 9: Add latency logging**
In every branch's `Set` node, add a field capturing how long the HTTP call took (n8n's `$now` before and after, or the HTTP node's built-in timing if available). Log it somewhere simple — even just returning it in the response — and compare the six methods.
> **What this proves:** abstraction (Method 1, AI Agent) usually costs a little latency compared to a raw HTTP call (Methods 2–6) — now you can measure it instead of assuming it.

### Suggested order to actually do these
Exercises 1 → 2 → 4 → 5 are the highest-value four if you're short on time — they cover every auth shape you're likely to meet in real work. Save 6–9 for once the basics feel automatic.

