# Bayen API SDK – Front-End Integration Guide  
**Advanced Saudi Legal Assistant · v1.2.1**

Modern JSON chat API that lets you embed _Bayen_’s Saudi-law reasoning into any web or mobile front-end.  
This document is written **for front-end engineers**—no backend secrets are exposed.

---

## Table of Contents
1. [Base URL](#base-url)  
2. [Authentication](#authentication)  
3. [Endpoint Reference](#endpoint-reference)  
4. [Request Schema](#request-schema)  
5. [Response Schema](#response-schema)  
6. [Minimal Request Example](#minimal-request-example)  
7. [TypeScript Types](#typescript-types)  
8. [Error Handling & Troubleshooting](#error-handling--troubleshooting)  
9. [Best Practices](#best-practices)  

---

## Base URL
```
https://bayen-v1.onrender.com
```
All paths below are relative to this root.

---

## Authentication
Every call **must** include an `X-API-Key` header with your project key.
```http
X-API-Key: <your-project-key>
```
If the key is missing or wrong you will receive **401 Unauthorized**.

> **Never** ship the key inside a public bundle—store it in a secure vault, environment variable, or native secret store.

---

## Endpoint Reference

| Method | Path   | Description |
|--------|--------|-------------|
| POST   | `/chat` | Submit a user/assistant turn and get Bayen’s answer. |

Responses are delivered as one JSON object (no streaming—yet).

---

## Request Schema

| Field                | Type / Enum                     | Required | Notes |
|----------------------|---------------------------------|----------|-------|
| `model`              | `"bayen-pro"` &#124; `"bayen-lite"` | ✓ | `bayen-pro` = deeper, slower. `bayen-lite` = faster, lighter. |
| `messages`           | `Message[]`                     | ✓ | Chronological chat history (see below). |
| `structured_output`  | `boolean`                       | ✕ | **Default = true** → Bayen returns a rich JSON envelope. |
| `stream`             | `boolean`                       | ✕ | Currently ignored—kept for future streaming support. |
| `max_tokens`         | `int`                           | ✕ | Soft cap; omit for default limit. |

```ts
// Message object
type MessageRole = 'system' | 'user' | 'assistant';
interface Message {
  role: MessageRole;
  content: string;
}
```
*   You normally send only `user` and (optionally) previous `assistant` messages.  
*   **Do not** repeat the built-in system prompt—Bayen injects it server-side.

---

## Response Schema (`structured_output = true`)

```jsonc
{
  "think": "internal Arabic reasoning…",
  "message": "Markdown answer in Arabic (IRAC-formatted)…",
  "citations": [
    "https://laws.boe.gov.sa/.../article-12",
    "https://moj.gov.sa/..."
  ],
  "metadata": {
    "id": "ccd82464-39d2-4c34-a7da-4f0c0c53dbe4",
    "model": "bayen-pro",
    "created": 1714320000,
    "object": "response.completion",
    "title": "مختصر الاستشارة"
  }
}
```

| Field      | Type            | Description |
|------------|-----------------|-------------|
| `think`    | `string|null`   | Internal chain-of-thought (hide in production UIs). |
| `message`  | `string`        | Ready-to-render Markdown answer. |
| `citations`| `string[]`      | Authoritative URLs used as sources. |
| `metadata` | `object`        | Tracking & provenance info. |

If you set `structured_output = false`, the body is returned as **plain Markdown**.

---

## Minimal Request Example

### HTTP ( cURL )

```bash
curl -X POST https://bayen-v1.onrender.com/chat \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $BAYEN_KEY" \
  -d '{
    "model": "bayen-pro",
    "messages": [
      {
        "role": "user",
        "content": "تم بيعي قمح مغشوش"
      }
    ],
    "structured_output": true,
    "max_tokens": 1024
  }'
```

### React / fetch (TypeScript)

```tsx
import type { ChatRequest, AssistantResponse } from './types';

export async function askBayen(
  payload: ChatRequest,
  apiKey: string
): Promise<AssistantResponse> {
  const res = await fetch('https://bayen-v1.onrender.com/chat', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': apiKey
    },
    body: JSON.stringify(payload)
  });

  if (!res.ok) throw new Error(`Bayen error ${res.status}`);
  return res.json();
}
```

---

## TypeScript Types

```ts
export type BayenModel = 'bayen-pro' | 'bayen-lite';

export interface Message {
  role: 'system' | 'user' | 'assistant';
  content: string;
}

export interface ChatRequest {
  model: BayenModel;
  messages: Message[];
  structured_output?: boolean;
  stream?: boolean;
  max_tokens?: number;
}

export interface AssistantResponse {
  think?: string | null;
  message: string;
  citations: string[];
  metadata: {
    id: string;
    model: BayenModel;
    created: number; // Unix epoch seconds
    object: string;  // "response.completion"
    title: string;
  };
}
```

---

## Error Handling & Troubleshooting

| HTTP Status | Meaning                                | Common Cause                               |
|-------------|----------------------------------------|-------------------------------------------|
| 401         | Unauthorized                           | Missing/invalid `X-API-Key`.              |
| 502         | Bad Gateway (upstream error)           | Bayen’s NLP provider returned ≥400; often due to **invalid JSON**. |
| 500         | Invalid structured output              | Rare—open an issue if persistent.         |

### Typical 502 root cause  
If you see `Perplexity API failed: 400 Bad Request` inside the 502 body, your JSON payload is malformed (trailing commas, comments) or headers are missing. Validate with [jsonlint.com](https://jsonlint.com) and ensure `Content-Type: application/json`.

---

## Best Practices

1. **Preserve context** – send the entire `messages` array each turn; the API is stateless.  
2. **Render Markdown safely** – sanitise HTML to prevent XSS.  
3. **Hide `think`** in production; expose it only for internal QA.  
4. **Back-off & retry** on 429/502 with exponential delays.  
5. **Never leak the API key** in logs or client code.  
6. **Show citations** – linking them boosts user trust and legal traceability.

---

### Need Help?  
Open an issue on this repo or ping the maintainers.  
_© 2025 Bayen – Saudi Legal Assistant_
