# Soccer Club — AI Customer Support Automation

> Production AI agent that handles multilingual customer support emails for a Montreal community soccer club. Incoming emails are classified, processed through a multi-tool RAG agent, and returned as ready-to-review Gmail drafts — in the same language the customer wrote in.

---

## Problem

Soccer Club manages customer inquiries in French, English, and Spanish across registration, refunds, schedules, tax receipts (Relevé 24), and activity questions. Volume peaks seasonally, and the club operates on a non-standard two-season calendar (Winter: October–April, Summer: May–September) that frequently causes confusion. All of this was handled manually.

---

## Solution Architecture

```
Unread Gmail
      │
      ▼
┌─────────────────────┐
│   Gmail Trigger     │  Polls every minute for unread emails
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Set Content       │  Extracts: emailBody, threadID, from address
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Get Thread        │  Fetches full Gmail thread history
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Build History     │  Labels each message: "Soccer Club" or "Customer"
│   (JavaScript)      │  based on sender domain (club-domain.com)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   PII Redaction     │  Strips emails, phones, names, addresses,
│   (JavaScript)      │  postal codes, SIN, credit cards before any LLM call
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Customer Support   │  GPT-4o-mini classifier — returns JSON:
│  Eval               │  { "customerSupport": true | false }
└──────────┬──────────┘
           │
      true │  false → stop
           ▼
┌──────────────────────────────────────────────────────┐
│            Customer Support AI Agent                 │
│                   (GPT-5-MINI)                       │
│                                                      │
│  Tools:                                              │
│  ┌──────────────────────────────────────────────┐    │
│  │ customerSupportDocs                          │    │
│  │ Pinecone index: scrape / ns: web-scraper│    │    │  
│  │ Website content, pricing, registration info  │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ clubDocuments                                │    │
│  │ Pinecone index: club-email / ns: club-docs   │    │
│  │ Schedules, calendars, official documents     │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ previousEmailResponses                       │    │
│  │ Pinecone index: scrape / ns: email-history   │    │
│  │ Past email exchanges — used for Relevé 24    │    │
│  │ and any gap not covered by other tools       │    │
│  └──────────────────────────────────────────────┘    │
│  ┌──────────────────────────────────────────────┐    │
│  │ createDraft (Gmail Tool)                     │    │
│  │ Creates draft reply in the original thread   │    │
│  │ subject + body injected by the agent         │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│  Mark as Read       │  Removes UNREAD label from thread
└─────────────────────┘
```

---

## Key Implementation Details

### PII Redaction (before any LLM call)

All identifying information is stripped from the email body and thread history before being passed to any model. The original `from` address is kept separately for routing — it never reaches the LLM.

```javascript
function redactPII(text) {
  return text
    .replace(/[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}/g, '[EMAIL]')
    .replace(/(\+?1[\s\-.]?)?\(?\d{3}\)?[\s\-.]?\d{3}[\s\-.]?\d{4}/g, '[PHONE]')
    .replace(/\b(Mr\.|Mrs\.|Ms\.|Dr\.|M\.|Mme\.)\s+[A-ZÀ-Ö][a-zà-ö]+(\s+[A-ZÀ-Ö][a-zà-ö]+)?/g, '[NAME]')
    .replace(/[A-Z]\d[A-Z]\s?\d[A-Z]\d/gi, '[POSTAL_CODE]')
    .replace(/\b\d{4}[\s\-]?\d{4}[\s\-]?\d{4}[\s\-]?\d{4}\b/g, '[CARD_NUMBER]')
    .replace(/\b\d{3}[\s\-]\d{3}[\s\-]\d{3}\b/g, '[SIN]');
}
```

### Thread History Builder

Each message in the thread is labelled by role before being passed to the classifier and agent. This gives both models clean conversational context.

```javascript
const history = messages.map(msg => {
  const from = getHeader(msg, 'from');
  const snippet = msg.snippet || '';
  const role = from.includes('club-domain.com') ? 'Soccer club' : 'Customer';
  return `${role}: ${snippet}`;
}).join('\n\n');
```

### Customer Support Classifier

A lightweight GPT-4o-mini call classifies the email before routing it to the full agent. Topics that always return `true` include Relevé 24 / RL-24 / tax receipt requests in any of the three supported languages.

```
Topics: activities, uniforms, teams, refunds, Relevé 24, tax receipts,
        registration changes, coach issues, weather/cancellations
Languages: French, English, Spanish
Output: { "customerSupport": true | false }
```

### Agent — Season Logic

The most critical prompt constraint. Soccer Club's calendar doesn't match the conventional calendar, and the agent is instructed to remap customer language before every search:

```
Fall / Autumn  →  Winter season (October–April)
Spring         →  Summer season (May–September)
Winter         →  Winter season (October–April)
Summer         →  Summer season (May–September)
```

The agent always includes the correct season in its Pinecone queries (e.g. `"winter season U8"`, `"summer camp 2026 U12"`) and is blocked from referencing activities from previous years.

### Agent — Mandatory Tool Usage

The agent is instructed to search **all three tools** before drafting — not just the first one that returns results. The `createDraft` Gmail tool call is mandatory regardless of whether the retrieved information is complete.

### createDraft Tool

The agent injects subject and emailBody via `$fromAI()`. The draft is created in the original thread and addressed to the original sender. The reply language always matches the incoming email.

---

## Stack

| Component | Technology |
|---|---|
| Orchestration | n8n (self-hosted on Hostinger) |
| LLM — Classifier | GPT-5-MINI (OpenAI) |
| LLM — Agent | GPT-5-MINI (OpenAI) |
| Embeddings | text-embedding-3-large (OpenAI) |
| Vector DB | Pinecone — 2 indexes, 3 namespaces |
| Email | Gmail API (OAuth2) |
| Languages supported | French, English, Spanish |

---

## Pinecone Index Structure

| Tool name | Index | Namespace | Content |
|---|---|---|---|
| customerSupportDocs | `scrape` | `test-web-scraper` | Scraped website — activities, pricing, registration |
| previousEmailResponses | `scrape` | `email-history` | Past email exchanges |
| clubDocuments | `Club-email` | `club-documents` | Schedules, calendars, official documents |

---

## Outcome

- Customer emails are classified, processed, and drafted without manual intervention
- Drafts are created directly in the Gmail thread, ready for staff review before sending
- Language detection handles French, English, and Spanish automatically
- Season mapping prevents the most common category of incorrect responses
- PII never reaches the LLM layer

---

## Note on Code

This repository contains architecture documentation and representative code snippets. Full source is not published to protect client configuration and operational details.

---

## Related

- [AlterEco Publishing Pipeline](https://github.com/daniszwarc/altereco-pipeline) — document intelligence pipeline using Claude Vision API
- [WorkflowSynth](https://github.com/daniszwarc/workflowsynth) — MSc research on automated AI workflow discovery
