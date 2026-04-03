# 🩷 Fumble — Build Log

> **Status:** 🔨 Active Development
> **Co-builders:** Siya & Srishti
> **Project:** AI Copilot for Decoding Mixed Signals in Text Conversations
> **Last Updated:** April 2026

---

## 📌 What We're Building

Fumble is an AI copilot iOS app that reads your text message conversations and tells you exactly what the mixed signals mean — no sugarcoating, no bias, just clarity.

You paste a chat thread. Fumble analyzes response timing, message length patterns, initiation imbalance, tone shifts, topic avoidance, and other behavioral signals across the full conversation. It then gives you a plain-English breakdown of what's going on, a scored vibe meter, and optional reply suggestions based on what you're actually trying to accomplish.

We're currently in the **build phase**, implementing the full feature set described below. This README tracks our progress, setup steps, and build process end-to-end.

---

## 👩‍💻 Team

| Name    | Role       |
| ------- | ---------- |
| Siya    | Co-builder |
| Srishti | Co-builder |

---

## 🗂️ Table of Contents

1. [Project Structure](#project-structure)
2. [Prerequisites](#prerequisites)
3. [Environment Setup](#environment-setup)
4. [Installation](#installation)
5. [Build Process — Phase by Phase](#build-process)
6. [Running the App](#running-the-app)
7. [Testing & Evaluation](#testing--evaluation)
8. [Current Progress](#current-progress)
9. [Known Issues](#known-issues)

---

## 📁 Project Structure

```
Fumble/
├── Fumble.xcodeproj
├── Fumble/
│   ├── App/
│   │   └── FumbleApp.swift              # App entry point
│   ├── Views/
│   │   ├── LandingView.swift            # Landing / onboarding screen
│   │   ├── AnalyzeView.swift            # Main analysis interface
│   │   ├── ChatInputView.swift          # Paste / photo import interface
│   │   ├── ContextFormView.swift        # Optional context fields
│   │   ├── SignalDebriefView.swift      # Signal analysis output cards
│   │   ├── VibeScoreView.swift          # Scored meter component
│   │   └── ReplyOptionsView.swift       # Suggested reply cards
│   ├── Models/
│   │   └── Models.swift                 # Swift data models & types
│   ├── Services/
│   │   ├── AIService.swift              # AI API client & request handling
│   │   ├── ParserService.swift          # Chat thread parser & normalizer
│   │   ├── SignalService.swift          # Signal detection & scoring logic
│   │   └── OCRService.swift             # Vision framework OCR pipeline
│   ├── Prompts/
│   │   └── Prompts.swift                # System prompts & structured output schemas
│   ├── Utils/
│   │   └── Extensions.swift             # Swift extensions & helpers
│   └── Resources/
│       └── Assets.xcassets
├── FumbleTests/
│   ├── ParserTests.swift
│   ├── SignalTests.swift
│   └── Fixtures/                        # Sample thread JSON fixtures
│       ├── sample_thread_ambivalent.json
│       ├── sample_thread_fading.json
│       ├── sample_thread_interested.json
│       └── sample_thread_breadcrumbing.json
└── README.md                            # ← You are here
```

---

## ✅ Prerequisites

Make sure the following are installed and configured before proceeding.

### System Requirements

- macOS: 14.0 (Sonoma) or later
- Xcode: 15.0+
- iOS Deployment Target: iOS 17.0+
- Device / Simulator: iPhone 14 or later recommended

### Required Tools

| Tool             | Version | Purpose                      |
| ---------------- | ------- | ---------------------------- |
| Xcode            | 15+     | IDE, build system, simulator |
| Swift            | 5.9+    | Primary language             |
| CocoaPods or SPM | Latest  | Dependency management        |
| Git              | Any     | Version control              |

### Apple Developer Account

Required to run on a physical device. A free account works for development; a paid account ($99/year) is needed for TestFlight and App Store distribution.

---

## 🛠️ Environment Setup

### Step 1 — Clone the Repository

```bash
git clone https://github.com/Siya1202/fumble.git
cd fumble
open Fumble.xcodeproj
```

### Step 2 — Configure API Keys

Create a `Secrets.plist` file in the `Fumble/Resources/` directory (this file is gitignored):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "...">
<plist version="1.0">
<dict>
  <key>AI_API_KEY</key>
  <string>your_api_key_here</string>
  <key>AI_BASE_URL</key>
  <string>https://your-ai-provider-endpoint.com</string>
</dict>
</plist>
```

Load secrets at runtime via a `SecretsManager` utility — never hardcode keys in source files.

> **Important:** Add `Secrets.plist` to `.gitignore` before your first commit.

---

## 📦 Installation

### Step 3 — Install Dependencies (Swift Package Manager)

Open `Fumble.xcodeproj` in Xcode, then go to:

**File → Add Package Dependencies**

Add the following packages:

| Package                                             | Purpose         |
| --------------------------------------------------- | --------------- |
| [Alamofire](https://github.com/Alamofire/Alamofire) | HTTP networking |

All other functionality (OCR, JSON parsing, async/await) is handled natively by Apple frameworks:

- **Vision** — On-device OCR for screenshot imports
- **Foundation** — JSON decoding & networking primitives
- **SwiftUI** — UI framework
- **Combine / async-await** — Reactive state & async calls

---

## 🏗️ Build Process

Here's the full step-by-step build sequence we're following, phase by phase.

---

### Phase 0 — Project Setup & API Connection ✅ Complete

**Goal:** Scaffold the Xcode project and verify AI API connection.

```swift
// Services/AIService.swift
import Foundation

struct AIService {
    private let baseURL: String
    private let apiKey: String

    func analyze(prompt: String) async throws -> String {
        var request = URLRequest(url: URL(string: "\(baseURL)/v1/messages")!)
        request.httpMethod = "POST"
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body: [String: Any] = [
            "model": "your-model-id",
            "max_tokens": 1000,
            "messages": [["role": "user", "content": prompt]]
        ]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)

        let (data, _) = try await URLSession.shared.data(for: request)
        // Parse and return response
        return String(data: data, encoding: .utf8) ?? ""
    }
}
```

Verify by running the app in the simulator and triggering a test API call from `AnalyzeView`.

---

### Phase 1 — Chat Thread Parser 🔨 In Progress

**Goal:** Accept pasted text, normalize it into a structured message array with timestamps and sender labels.

Input formats to handle:

- Raw paste (iMessage copy-paste format)
- WhatsApp export `.txt`
- Manual entry

Parser output schema:

```swift
// Models/Models.swift
struct ParsedThread: Codable {
    struct Message: Codable {
        enum Sender: String, Codable { case you, them }
        let sender: Sender
        let content: String
        let timestamp: String?
        let readAt: String?
    }
    struct Metadata: Codable {
        let totalMessages: Int
        let initiationRatio: Double   // % of conversations started by each party
        let avgResponseGap: Double?   // in minutes
    }
    let messages: [Message]
    let metadata: Metadata
}
```

Build the parser in `Services/ParserService.swift`:

- Handles iMessage, WhatsApp, raw paste formats
- Normalizes sender labels to `.you` / `.them`
- Extracts timestamps where available

Test with unit tests in `FumbleTests/ParserTests.swift`.

---

### Phase 2 — Signal Detection Engine 🔨 In Progress

**Goal:** Build the core AI-powered signal analysis. Takes a parsed thread and returns a structured signal report.

Signal categories being detected:

| Signal                   | What Fumble looks for                                           |
| ------------------------ | --------------------------------------------------------------- |
| Response time patterns   | Average reply latency, sudden changes, time-of-day shifts       |
| Message length shifts    | Compression over time, effort asymmetry                         |
| Initiation imbalance     | Who starts conversations, how consistently                      |
| Hot/cold cycles          | Engagement spikes followed by withdrawal                        |
| Tone and energy mismatch | Enthusiasm calibration between both parties                     |
| Topic avoidance          | Subjects deflected or left unacknowledged                       |
| Future-plan language     | Use of "we should," "sometime," "maybe" vs. concrete plans      |
| Breadcrumbing            | Just enough engagement to maintain presence without real intent |
| Orbiting signals         | Passive engagement without active conversation                  |
| Ghosting indicators      | Pre-fade patterns before a communication drop                   |

System prompt structure:

```swift
// Prompts/Prompts.swift
let SIGNAL_DETECTION_PROMPT = """
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
"""
```

Output model with validation:

```swift
// Models/Models.swift
struct SignalReport: Codable {
    struct Signal: Codable {
        enum Severity: String, Codable { case low, medium, high }
        let type: String
        let severity: Severity
        let explanation: String
    }
    struct VibeScore: Codable {
        let interested: Double      // 0–100, must sum to 100 with others
        let ambivalent: Double
        let pullingAway: Double
    }
    let signals: [Signal]
    let vibeScore: VibeScore
    let summary: String
    let replySuggestions: [ReplySuggestion]
}
```

---

### Phase 3 — Screenshot Import (OCR) ⬜ Not Started

**Goal:** Allow users to upload a screenshot of a conversation instead of pasting text. Uses Apple's on-device Vision framework — no third-party OCR needed.

```swift
// Services/OCRService.swift
import Vision
import UIKit

struct OCRService {
    func extractText(from image: UIImage) async throws -> String {
        guard let cgImage = image.cgImage else { throw OCRError.invalidImage }

        return try await withCheckedThrowingContinuation { continuation in
            let request = VNRecognizeTextRequest { request, error in
                if let error { continuation.resume(throwing: error); return }
                let text = (request.results as? [VNRecognizedTextObservation])?
                    .compactMap { $0.topCandidates(1).first?.string }
                    .joined(separator: "\n") ?? ""
                continuation.resume(returning: text)
            }
            request.recognitionLevel = .accurate
            request.usesLanguageCorrection = true
            try? VNImageRequestHandler(cgImage: cgImage).perform([request])
        }
    }
}
```

After extraction, feed the raw OCR text through the same parser from Phase 1. Test against common screenshot formats using fixtures in `FumbleTests/Fixtures/`.

---

### Phase 4 — UI — Analysis Interface ⬜ Not Started

**Goal:** Build the main analysis interface in SwiftUI — paste input, context form, and debrief output.

Views to build:

- **ChatInputView** — Large text editor for pasting threads, photo picker for screenshots
- **ContextFormView** — Optional fields: relationship type, timeline, what's confusing you, your goal
- **SignalDebriefView** — Signal cards with severity badges and explanations
- **VibeScoreView** — Three-bar animated meter (Interested / Ambivalent / Pulling Away)
- **ReplyOptionsView** — Labeled reply suggestion cards with copy-to-clipboard button

```swift
// Views/VibeScoreView.swift
// Three scores must sum to 100
// Animate bar fills on appear using withAnimation
// Color: interested = .green, ambivalent = .orange, pullingAway = .red
struct VibeScoreView: View {
    let score: SignalReport.VibeScore
    // ...
}
```

Test on iPhone 15 Pro simulator and physical device.

---

### Phase 5 — Multi-turn Follow-up Conversation ⬜ Not Started

**Goal:** Let users ask follow-up questions about their thread within the same session. Fumble holds full context.

Conversation state:

```swift
// Models/Models.swift
struct ConversationState {
    let thread: ParsedThread
    let context: UserContext
    var history: [(role: String, content: String)]
    let signalReport: SignalReport
}
```

Each follow-up message appends to `history` and is sent back to the AI with the original thread and report as context:

```swift
messages: [
    ["role": "user", "content": INITIAL_ANALYSIS_PROMPT],
    ["role": "assistant", "content": encodedSignalReport],
    ...conversationHistory,
    ["role": "user", "content": userFollowUp]
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

- All thread data lives in SwiftUI `@State` / `@StateObject` only — never written to disk
- Sessions cleared when app is backgrounded or closed
- No logging of conversation content in API requests

Opt-in persistence (requires iCloud or local encrypted store):

```swift
// Only activated if user toggles "Remember this conversation"
// Store encrypted session data in Keychain or CloudKit
// User can delete at any time from Settings
```

Privacy audit checklist:

```
[ ] No raw thread text sent to analytics
[ ] No PII in crash reports (use redacted summaries only)
[ ] Session state cleared on scene disconnect
[ ] Keychain entries scoped to app identifier
[ ] Secrets.plist excluded from version control
[ ] App Privacy Manifest (PrivacyInfo.xcprivacy) filled out accurately
```

---

### Phase 7 — Pattern-Over-Time Insights ⬜ Not Started

**Goal:** For returning users who opt in to session history, surface trends across multiple conversations with the same person.

```swift
// Models/Models.swift
struct PatternInsight: Codable {
    enum TrendDirection: String, Codable {
        case improving, declining, stagnant
    }
    let person: String           // user-assigned label, no real names stored
    let sessions: Int
    let trendDirection: TrendDirection
    let recurringSignals: [String]
    let recommendation: String
}
```

Insight examples:

```
"Over 4 sessions, their response times have shortened by ~40 min on average — a consistent positive shift."
"Hot/cold cycles have appeared in 3 of your last 4 conversations with this person."
```

---

### Phase 8 — App Store Deployment ⬜ Not Started

**Goal:** Submit Fumble to the App Store via TestFlight then production.

```bash
# Archive in Xcode
Product → Archive → Distribute App → App Store Connect
```

Pre-submission checklist:

```
[ ] App icon complete (all required sizes via Assets.xcassets)
[ ] Launch screen configured
[ ] Privacy manifest (PrivacyInfo.xcprivacy) submitted
[ ] App Store Connect listing: screenshots, description, keywords
[ ] TestFlight beta tested on real devices
[ ] All App Store Review Guidelines reviewed
[ ] Age rating configured
[ ] Data privacy questionnaire filled in App Store Connect
```

---

## ▶️ Running the App

Once Phases 0–4 are complete:

```bash
# Open in Xcode
open Fumble.xcodeproj

# Select target: iPhone 15 Pro simulator or your connected device
# Press ▶ or Cmd+R to build and run
```

For device testing, ensure your Apple ID is added under **Xcode → Settings → Accounts** and the signing team is set in **Target → Signing & Capabilities**.

---

## 🧪 Testing & Evaluation

Unit tests live in `FumbleTests/`. Run them via:

```bash
# In Xcode
Cmd+U

# Or via command line
xcodebuild test -scheme Fumble -destination 'platform=iOS Simulator,name=iPhone 15 Pro'
```

Test targets:

| Test File           | Coverage                                     |
| ------------------- | -------------------------------------------- |
| `ParserTests.swift` | iMessage, WhatsApp, raw paste parsing        |
| `SignalTests.swift` | Signal detection accuracy on fixture threads |

Sample thread fixtures:

```
FumbleTests/Fixtures/sample_thread_ambivalent.json    # Classic mixed signals
FumbleTests/Fixtures/sample_thread_fading.json        # Pre-ghosting pattern
FumbleTests/Fixtures/sample_thread_interested.json    # Positive engagement
FumbleTests/Fixtures/sample_thread_breadcrumbing.json # Low-intent re-engagement
```

Vibe score accuracy target: **≥ 85% match with human-labeled ground truth** on test fixtures before v1.0.

---

## 📊 Current Progress

| Phase                                    | Status         | Notes                                               |
| ---------------------------------------- | -------------- | --------------------------------------------------- |
| Phase 0 — Project Setup & API Connection | ✅ Complete    | Scaffolded, API verified                            |
| Phase 1 — Chat Thread Parser             | 🔨 In Progress | iMessage format working, WhatsApp in testing        |
| Phase 2 — Signal Detection Engine        | 🔨 In Progress | Prompts drafted, JSON schema validation in progress |
| Phase 3 — Screenshot Import (OCR)        | ⬜ Not Started |                                                     |
| Phase 4 — UI — Analysis Interface        | ⬜ Not Started |                                                     |
| Phase 5 — Multi-turn Follow-up           | ⬜ Not Started |                                                     |
| Phase 6 — Privacy & Session Management   | ⬜ Not Started |                                                     |
| Phase 7 — Pattern-Over-Time Insights     | ⬜ Not Started |                                                     |
| Phase 8 — App Store Deployment           | ⬜ Not Started |                                                     |

---

## 🗺️ Roadmap — Post v1.0

Once all phases are complete and stable:

- **Voice message analysis** — Transcribe and analyze voice note threads via Apple's Speech framework
- **Anonymous community signals** — Aggregated pattern stats across all sessions (opt-in, anonymized): "72% of threads with this pattern resulted in a fade within 2 weeks"
- **Fumble API** — Public API for third-party integrations
- **Multi-platform parsing** — Native parsers for Instagram DMs, Hinge, Bumble chat exports
- **iPad support** — Expanded layout for larger screens
- **Widgets** — Quick-access vibe summaries on the home screen

---

## 🐛 Known Issues

- OCR accuracy on dark-mode screenshots is lower — Vision framework performs better on light backgrounds. Preprocessing (contrast boost) planned for Phase 3.
- AI occasionally returns vibe scores that don't sum to 100 — JSON validation catches this and renormalizes. A stricter prompt fix is planned.
- WhatsApp group chat exports include multiple senders — parser currently only handles 1:1 threads. Group support is post-v1.0.
- No request throttling on the AI service layer yet — to be added in Phase 6.

---

## 📎 References

- [Apple Developer Documentation](https://developer.apple.com/documentation/)
- [SwiftUI](https://developer.apple.com/xcode/swiftui/)
- [Vision Framework (OCR)](https://developer.apple.com/documentation/vision)
- [Swift Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [TestFlight](https://developer.apple.com/testflight/)
