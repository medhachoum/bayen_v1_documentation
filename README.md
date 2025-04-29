# Bayen API – Front-End Integration Guide
Modern, well-typed JSON chat API for accessing **Bayen, the advanced Saudi-law assistant**.  
This document is aimed at **front-end engineers** who need a crystal-clear contract for calling the service from web- or mobile-apps.

---

## Table of Contents
1. [Base URL](#base-url)  
2. [Authentication](#authentication)  
3. [Endpoint Reference](#endpoint-reference)  
4. [Request Schema](#request-schema)  
5. [Response Schema](#response-schema)  
6. [Example Chat Flow](#example-chat-flow)  
7. [Error Handling](#error-handling)  
8. [TypeScript Types](#typescript-types)  
9. [Best Practices](#best-practices)  

---

## Base URL
```
https://bayen-v1.onrender.com
```

All paths below are relative to this root.

---

## Authentication

Every request **must** include an `X-API-Key` header:

```
X-API-Key: <your-project-key>
```

If the header is missing or invalid the server returns **401 Unauthorized**.

> **Never** embed the key in compiled client bundles—pass it from a secure backend, environment variable, or in-app secret store.

---

## Endpoint Reference

| Method | Path   | Description             |
|--------|--------|-------------------------|
| POST   | `/chat` | Submit a chat turn and receive Bayen’s answer. |

Streaming is **not** yet supported—the response arrives as a single JSON payload.

---

## Request Schema
```jsonc
{
  "model": "bayen-pro",        // or "bayen-lite"
  "messages": [                // conversation so far
    { "role": "user", "content": "نص السؤال..." },
    { "role": "assistant", "content": "رد سابق (اختياري)" }
  ],
  "structured_output": true,   // default = true
  "max_tokens": 1024           // optional hard cap
}
```

### Fields

| Name               | Type / Enum                         | Required | Notes |
|--------------------|-------------------------------------|----------|-------|
| `model`            | `"bayen-pro"` \| `"bayen-lite"`     | ✓        | `bayen-pro` is deeper & slower; `bayen-lite` is lighter & faster. |
| `messages`         | `Message[]`                         | ✓        | Chronological chat history. First item **must** have role `"user"`. |
| `structured_output`| `boolean`                           | ✕        | When `true` the answer is returned in a rich JSON container (recommended).|
| `max_tokens`       | `int`                               | ✕        | Soft limit; defaults to the provider’s maximum. |

#### `Message`
```ts
type MessageRole = "system" | "user" | "assistant";

interface Message {
  role: MessageRole;
  content: string;
}
```
* `system` messages are optional and let you override the default assistant behaviour (e.g. localisation).  
* Use `assistant` messages only when you need to replay previous answers to maintain context.

---

## Response Schema

With `structured_output = true` (default):

```jsonc
{
  "think": "تحليل داخلي (قد يكون فارغًا)...",
  "message": "الإجابة النهائية بصيغة Markdown...",
  "citations": [
    "https://laws.boe.gov.sa/.../article-12",
    "https://moj.gov.sa/..."
  ],
  "metadata": {
    "id": "ccd82464-39d2-4c34-a7da-4f0c0c53dbe4",
    "model": "bayen-pro",
    "created": 1714320000,
    "object": "response.completion",
    "title": "عنوان مختصر للاستشارة"
  }
}
```

| Field      | Type            | Description |
|------------|-----------------|-------------|
| `think`    | `string|null`   | Internal reasoning—useful for debugging; hide in production UIs if you wish. |
| `message`  | `string`        | Ready-to-render Markdown answer in Arabic (IRAC-structured). |
| `citations`| `string[]`      | List of authoritative URLs backing the analysis. |
| `metadata` | `object`        | Tracking info (UUID, model, Unix epoch, etc.). |

If you set `structured_output = false` the server returns a **plain Markdown string** instead.

---

## Example Chat Flow

<details>
<summary>Minimal cURL</summary>

```bash
curl -X POST https://bayen-v1.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $BAYEN_KEY" \
  -d @- <<'JSON'
{
  "model": "bayen-lite",
  "messages": [
    { "role": "user", "content": "ما العقوبة على بيع منتج غذائي مغشوش في السعودية؟" }
  ]
}
JSON
```
</details>

<details>
<summary>React + fetch (TypeScript)</summary>

```tsx
import { ChatRequest, AssistantResponse } from "./types";

export async function askBayen(
  payload: ChatRequest,
  apiKey: string
): Promise<AssistantResponse> {
  const res = await fetch("https://bayen-v1.onrender.com/chat", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-API-Key": apiKey,
    },
    body: JSON.stringify(payload),
  });

  if (!res.ok) {
    // Map status codes to UI-friendly errors
    throw new Error(`Bayen API error ${res.status}`);
  }

  return res.json();
}
```
</details>

---

## Error Handling

| Status | Meaning                                | Typical Cause              |
|--------|----------------------------------------|----------------------------|
| 401    | Unauthorized                           | Missing / invalid `X-API-Key`. |
| 502    | Upstream service unavailable           | Temporary provider outage. Retry with back-off. |
| 500    | Invalid structured output              | Rare; file a bug if persistent. |

---

## TypeScript Types
```ts
export type BayenModel = "bayen-pro" | "bayen-lite";

export interface Message {
  role: "system" | "user" | "assistant";
  content: string;
}

export interface ChatRequest {
  model: BayenModel;
  messages: Message[];
  structured_output?: boolean;
  max_tokens?: number;
}

export interface AssistantResponse {
  think?: string | null;
  message: string;
  citations: string[];
  metadata: {
    id: string;
    model: BayenModel;
    created: number;   // Unix epoch (s)
    object: string;    // "response.completion"
    title: string;
  };
}
```

---

## Best Practices

1. **Preserve context.**  Send the full conversation (`messages`) each turn; stateless requests simplify caching and scaling.  
2. **Respect rate limits.**  Implement exponential back-off on 429/502 errors.  
3. **Render Markdown safely.**  Use a sanitizer to avoid XSS when injecting HTML.  
4. **Hide `think` in production.**  It contains raw reasoning, not meant for end-users.  
5. **Display citations.**  They boost user trust—link them at the end of each answer.  
6. **Progressive enhancement.**  Optimistic UI while waiting; fallback to offline message if 502 persists.  
7. **No secret leakage.**  Never expose your `X-API-Key` or internal model names in client logs.

---

### Need Help?

*Open an issue on this repo or ping the maintainers.*  
Happy coding - may your UI and Bayen deliver justice together!
