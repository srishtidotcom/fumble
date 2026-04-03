# 🩷 Fumble — Build Log

> **Status:** 🔨 Active Development
> **Co-builders:** Siya & Srishti
> **Project:** AI Copilot for Decoding Mixed Signals in Text Conversations
> **Last Updated:** April 2026

---

## 📌 What We're Building

Fumble is an AI copilot that reads your text message conversations and tells you exactly what the mixed signals mean — no sugarcoating, no bias, just clarity.

You paste a chat thread. Fumble analyzes response timing, message length patterns, initiation imbalance, tone shifts, topic avoidance, and other behavioral signals across the full conversation. It then gives you a plain-English breakdown of what's going on, a scored vibe meter, and optional reply suggestions based on what you're actually trying to accomplish.

We're currently in the **build phase**, implementing the full feature set described below. This README tracks our progress, setup steps, and build process end-to-end.

> **AI Strategy:** We're using the **Anthropic API (Claude Sonnet 4)** as the core inference layer throughout all phases. The signal detection logic, vibe scoring, and reply generation all run through Claude with structured prompts and JSON outputs.

---

## 👩‍💻 Team

| Name | Role |
|------|------|
| Siya | Co-builder |
| Srishti | Co-builder |

---

## 🗂️ Table of Contents

1. [Project Structure](#project-structure)
2. [Prerequisites](#prerequisites)
3. [Environment Setup](#environment-setup)
4. [Installation](#installation)
5. [Build Process — Phase by Phase](#build-process)
6. [Running the System](#running-the-system)
7. [Testing & Evaluation](#testing--evaluation)
8. [Current Progress](#current-progress)
9. [Known Issues](#known-issues)

---

## 📁 Project Structure

```
fumble/
├── app/
│   ├── page.tsx                  # Landing page
│   ├── analyze/
│   │   └── page.tsx              # Main analysis interface
│   └── api/
│       └── analyze/
│           └── route.ts          # Anthropic API handler
├── components/
│   ├── ChatInput.tsx             # Paste / screenshot upload interface
│   ├── ContextForm.tsx           # Optional context fields
│   ├── SignalDebrief.tsx         # Signal analysis output cards
│   ├── VibeScore.tsx             # Scored meter component
│   └── ReplyOptions.tsx          # Suggested reply cards
├── lib/
│   ├── prompts.ts                # Fumble system prompts
│   ├── signals.ts                # Signal detection & scoring logic
│   └── parser.ts                 # Chat thread parser & normalizer
├── types/
│   └── index.ts                  # TypeScript types
├── .env.example
├── testing.md                    # Full testing guide
└── README.md                     # ← You are here
```

---

## ✅ Prerequisites

Make sure the following are installed and configured before proceeding.

### System Requirements

- OS: Ubuntu 22.04+ / Windows 11 with WSL2 / macOS 13+
- RAM: 8 GB minimum
- Node.js: 18+
- Disk: 2 GB free space

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | 18+ | Frontend + API routes |
| npm | 9+ | Package management |
| Git | Any | Version control |
| Tesseract OCR | Latest | Screenshot import (Phase 3) |

### Installing Tesseract OCR

Required for the screenshot import feature (Phase 3). The `tesseract.js` npm package handles this in-browser, but for server-side processing install the binary:

```bash
# Ubuntu / Debian
sudo apt-get install tesseract-ocr

# macOS
brew install tesseract

# Windows
# Download the installer from:
# https://github.com/UB-Mannheim/tesseract/wiki
```

---

## 🛠️ Environment Setup

### Step 1 — Clone the Repository

```bash
git clone https://github.com/Siya1202/fumble.git
cd fumble
```

### Step 2 — Copy Environment Variables

```bash
cp .env.example .env.local
```

Open `.env.local` and fill in the required fields:

```env
# Anthropic API
ANTHROPIC_API_KEY=your_api_key_here

# Supabase (optional — for session persistence in Phase 5)
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key

# App config
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

> **Important:** The `ANTHROPIC_API_KEY` is required from Phase 1 onwards. Get one at [console.anthropic.com](https://console.anthropic.com). Sessions are ephemeral by default — Supabase is only needed if enabling opt-in conversation history in Phase 5.

---

## 📦 Installation

### Step 3 — Install Dependencies

```bash
npm install
```

Key packages being installed:

```
next react react-dom typescript
tailwindcss
@anthropic-ai/sdk
tesseract.js              # In-browser OCR for screenshot imports
zod                       # Schema validation for AI outputs
```

---

## 🏗️ Build Process

Here's the full step-by-step build sequence we're following, phase by phase.

---

### Phase 0 — Project Setup & API Connection ✅ Complete

**Goal:** Scaffold the Next.js app and verify Anthropic API connection.

```bash
npx create-next-app@latest fumble --typescript --tailwind --app
cd fumble
npm install @anthropic-ai/sdk
```

Test the API connection:

```typescript
// app/api/analyze/route.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function POST(req: Request) {
  const { thread } = await req.json();
  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1000,
    messages: [{ role: "user", content: thread }],
  });
  return Response.json(response);
}
```

Verify:

```bash
npm run dev
curl -X POST http://localhost:3000/api/analyze \
  -H "Content-Type: application/json" \
  -d '{"thread": "test message"}'
```

-------

### Phase 1 — Chat Thread Parser 🔨 In Progress

**Goal:** Accept pasted text, normalize it into a structured message array with timestamps and sender labels.

Input formats to handle:
- Raw paste (iMessage copy-paste format)
- WhatsApp export `.txt`
- Manual entry

Parser output schema:

```typescript
type ParsedThread = {
  messages: {
    sender: "you" | "them";
    content: string;
    timestamp?: string;
    readAt?: string;
  }[];
  metadata: {
    totalMessages: number;
    initiationRatio: number;   // % of conversations started by each party
    avgResponseGap?: number;   // in minutes
  };
};
```

Build the parser:

```bash
# lib/parser.ts
# Handles: iMessage, WhatsApp, raw paste
# Normalizes sender labels to "you" / "them"
# Extracts timestamps where available
```

Test the parser with sample threads:

```bash
npm run test:parser
```

---

### Phase 2 — Signal Detection Engine 🔨 In Progress

**Goal:** Build the core Claude-powered signal analysis. Takes a parsed thread and returns a structured signal report.

Signal categories being detected:

| Signal | What Fumble looks for |
|--------|----------------------|
| Response time patterns | Average reply latency, sudden changes, time-of-day shifts |
| Message length shifts | Compression over time, effort asymmetry |
| Initiation imbalance | Who starts conversations, how consistently |
| Hot/cold cycles | Engagement spikes followed by withdrawal |
| Tone and energy mismatch | Enthusiasm calibration between both parties |
| Topic avoidance | Subjects deflected or left unacknowledged |
| Future-plan language | Use of "we should," "sometime," "maybe" vs. concrete plans |
| Breadcrumbing | Just enough engagement to maintain presence without real intent |
| Orbiting signals | Passive engagement without active conversation |
| Ghosting indicators | Pre-fade patterns before a communication drop |

System prompt structure:

```typescript
// lib/prompts.ts
export const SIGNAL_DETECTION_PROMPT = `
You are Fumble — a brutally honest, emotionally intelligent signal reader.
Analyze the provided conversation thread and return a structured JSON report.

You are NOT a therapist. You do NOT soften outputs. You read patterns, not people.

Return ONLY valid JSON. No preamble. No markdown. Schema:
{
  "signals": [{ "type": string, "severity": "low"|"medium"|"high", "explanation": string }],
  "vibeScore": { "interested": number, "ambivalent": number, "pullingAway": number },
  "summary": string,
  "replySuggestions": [{ "label": string, "message": string }]
}
`;
```

Validate AI output schema with Zod:

```typescript
import { z } from "zod";

const SignalReportSchema = z.object({
  signals: z.array(z.object({
    type: z.string(),
    severity: z.enum(["low", "medium", "high"]),
    explanation: z.string(),
  })),
  vibeScore: z.object({
    interested: z.number().min(0).max(100),
    ambivalent: z.number().min(0).max(100),
    pullingAway: z.number().min(0).max(100),
  }),
  summary: z.string(),
  replySuggestions: z.array(z.object({
    label: z.string(),
    message: z.string(),
  })),
});
```

---

### Phase 3 — Screenshot Import (OCR) ⬜ Not Started

**Goal:** Allow users to upload a screenshot of a conversation instead of pasting text.

```bash
npm install tesseract.js
```

OCR pipeline:

```typescript
// lib/ocr.ts
import Tesseract from "tesseract.js";

export async function extractTextFromImage(imageFile: File): Promise<string> {
  const { data: { text } } = await Tesseract.recognize(imageFile, "eng", {
    logger: (m) => console.log(m),
  });
  return text;
}
```

After extraction, feed the raw OCR text through the same parser from Phase 1. Test against common screenshot formats:

```bash
# Test fixtures
tests/fixtures/imessage_screenshot.png
tests/fixtures/whatsapp_screenshot.png
tests/fixtures/instagram_dm_screenshot.png
```

---

### Phase 4 — UI — Analysis Interface ⬜ Not Started

**Goal:** Build the main analysis interface — paste input, context form, and debrief output.

Components to build:

- **ChatInput** — Large textarea for pasting threads, drag-and-drop for screenshots
- **ContextForm** — Optional fields: relationship type, timeline, what's confusing you, your goal
- **SignalDebrief** — Signal cards with severity badges and explanations
- **VibeScore** — Three-bar confidence meter (Interested / Ambivalent / Pulling Away)
- **ReplyOptions** — Labeled reply suggestion cards with copy button

```bash
cd app
# Build components in /components
# Hook into /api/analyze route
npm run dev
# Test at http://localhost:3000/analyze
```

VibeScore component logic:

```typescript
// components/VibeScore.tsx
// Three scores must sum to 100
// Animate meter fills on mount
// Color: interested = green, ambivalent = amber, pullingAway = red
```

---

### Phase 5 — Multi-turn Follow-up Conversation ⬜ Not Started

**Goal:** Let users ask follow-up questions about their thread within the same session. Fumble holds full context.

Conversation state:

```typescript
type ConversationState = {
  thread: ParsedThread;
  context: UserContext;
  history: { role: "user" | "assistant"; content: string }[];
  signalReport: SignalReport;
};
```

Each follow-up message appends to `history` and is sent back to Claude with the original thread and report as context:

```typescript
messages: [
  { role: "user", content: INITIAL_ANALYSIS_PROMPT },
  { role: "assistant", content: JSON.stringify(signalReport) },
  ...conversationHistory,
  { role: "user", content: userFollowUp },
]
```

Example follow-ups Fumble handles:

```
"Why did things go cold after Tuesday?"
"Is this a breadcrumbing pattern or are they just busy?"
"Should I even bother responding at this point?"
"What does it mean that they keep opening my messages but not replying?"
```

---

### Phase 6 — Privacy & Session Management ⬜ Not Started

**Goal:** Enforce ephemeral sessions by default. No conversation data persisted unless user explicitly opts in.

Default behavior:
- All thread data lives in React state only — never sent to a database
- Sessions cleared on tab close
- No logging of conversation content on the backend

Opt-in persistence (requires Supabase):

```typescript
// Only activated if user toggles "Remember this conversation"
// Stores encrypted session blob in Supabase
// User can delete at any time from settings
```

Privacy audit checklist:

```
[ ] No raw thread text logged in API route
[ ] No PII fields in analytics events
[ ] Session state cleared on unmount
[ ] Supabase RLS policies enforce user-scoped access
[ ] .env.local excluded from version control
```

---

### Phase 7 — Pattern-Over-Time Insights ⬜ Not Started

**Goal:** For returning users who opt in to session history, surface trends across multiple conversations with the same person.

```typescript
type PatternInsight = {
  person: string;           // user-assigned label, no real names stored
  sessions: number;
  trendDirection: "improving" | "declining" | "stagnant";
  recurringSignals: string[];
  recommendation: string;
};
```

Insight examples:

```
"Over 4 sessions, their response times have shortened by ~40 min on average — a consistent positive shift."
"Hot/cold cycles have appeared in 3 of your last 4 conversations with this person."
```

---

### Phase 8 — Deployment ⬜ Not Started

**Goal:** Deploy Fumble to production on Vercel.

```bash
# Install Vercel CLI
npm install -g vercel

# Deploy
vercel --prod
```

Set production environment variables in Vercel dashboard:

```
ANTHROPIC_API_KEY
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
NEXT_PUBLIC_APP_URL
```

Build check before deploy:

```bash
npm run build
npm run lint
```

---

## ▶️ Running the System

Once Phases 0–4 are complete:

```bash
# Start the development server
npm run dev

# Visit
http://localhost:3000
```

For production build:

```bash
npm run build
npm run start
```

---

## 🧪 Testing & Evaluation

See [testing.md](./testing.md) for the full testing guide.

```bash
# Run parser unit tests
npm run test:parser

# Run signal detection tests (requires ANTHROPIC_API_KEY)
npm run test:signals

# End-to-end test with sample thread
curl -X POST http://localhost:3000/api/analyze \
  -H "Content-Type: application/json" \
  -d @tests/fixtures/sample_thread.json
```

Sample thread fixtures:

```
tests/fixtures/sample_thread_ambivalent.json    # Classic mixed signals
tests/fixtures/sample_thread_fading.json        # Pre-ghosting pattern
tests/fixtures/sample_thread_interested.json    # Positive engagement
tests/fixtures/sample_thread_breadcrumbing.json # Low-intent re-engagement
```

Vibe score accuracy target: **≥ 85% match with human-labeled ground truth** on test fixtures before v1.0.

---

## 📊 Current Progress

| Phase | Status | Notes |
|-------|--------|-------|
| Phase 0 — Project Setup & API Connection | ✅ Complete | Scaffolded, API verified |
| Phase 1 — Chat Thread Parser | 🔨 In Progress | iMessage format working, WhatsApp in testing |
| Phase 2 — Signal Detection Engine | 🔨 In Progress | Prompts drafted, Zod schema validation in progress |
| Phase 3 — Screenshot Import (OCR) | ⬜ Not Started | |
| Phase 4 — UI — Analysis Interface | ⬜ Not Started | |
| Phase 5 — Multi-turn Follow-up | ⬜ Not Started | |
| Phase 6 — Privacy & Session Management | ⬜ Not Started | |
| Phase 7 — Pattern-Over-Time Insights | ⬜ Not Started | |
| Phase 8 — Deployment | ⬜ Not Started | |

---

## 🗺️ Roadmap — Post v1.0

Once all phases are complete and stable:

- **Voice message analysis** — Transcribe and analyze voice note threads
- **Anonymous community signals** — Aggregated pattern stats across all sessions (opt-in, anonymized): "72% of threads with this pattern resulted in a fade within 2 weeks"
- **Fumble API** — Public API for third-party integrations
- **Multi-platform parsing** — Native parsers for Instagram DMs, Hinge, Bumble chat exports

---

## 🐛 Known Issues

- OCR accuracy on dark-mode screenshots is lower — Tesseract performs better on light backgrounds. Preprocessing (contrast boost) planned for Phase 3.
- Claude occasionally returns vibe scores that don't sum to 100 — Zod validation catches this and renormalizes. A stricter prompt fix is planned.
- WhatsApp group chat exports include multiple senders — parser currently only handles 1:1 threads. Group support is post-v1.0.
- No rate limiting on the `/api/analyze` route yet — to be added in Phase 6.

---

## 📎 References

- [Anthropic API Docs](https://docs.anthropic.com)
- [Claude Sonnet 4](https://www.anthropic.com/claude)
- [Next.js 14 Docs](https://nextjs.org/docs)
- [Tesseract.js](https://github.com/naptha/tesseract.js)
- [Zod](https://zod.dev)
- [Supabase](https://supabase.com/docs)
- [Vercel](https://vercel.com/docs)
