# Underwriting Co-Pilot

A real-time AI assistant that sits alongside loan underwriters during personal discussions (PDs) with borrowers. It transcribes the conversation live, identifies speakers, surfaces data-backed suggestions at the right moments, and ensures no critical question or red flag is missed.

---

## The Problem

Underwriting of large-ticket loans (home loans, LAP, MSME credit) today depends almost entirely on the individual capability of the underwriter conducting the personal discussion.

- **No real-time data access** -- Underwriters don't have live bureau data, banking analytics, or market intelligence at their fingertips while talking to the borrower. They rely on memory or switch between multiple systems mid-conversation.
- **Inconsistent evaluation quality** -- An experienced underwriter catches contradictions (borrower says "no other loans" when bureau shows 5 active accounts) while a junior one might miss them entirely. There is no standardized safety net.
- **Key questions get missed** -- Under time pressure, underwriters skip critical questions or forget to verify specific data points, leading to incomplete case assessments.
- **No institutional learning** -- When a top underwriter leaves, their judgment and questioning patterns leave with them. There's no system to capture and replicate what "good" looks like.

The result: higher delinquency rates, longer turnaround times, inconsistent credit decisions, and revenue leakage across the lending funnel.

---

## The Solution

The Underwriting Co-Pilot listens to the live PD conversation, transcribes it in real time with speaker identification, and generates contextual suggestions for the underwriter -- what question to ask next, what data point to verify, what contradiction to probe.

It follows a structured underwriting methodology (income assessment, margin validation, supplier payment analysis, red flag investigation, FOIR checks) but adapts to each borrower's profile rather than following a static checklist.

The co-pilot does NOT interrupt after every sentence. It uses an **event-driven trigger model** -- only surfacing insights when there's a genuine "stop and pay attention" moment, such as a data contradiction, a risk threshold breach, or a critical question that's been skipped for too long.

---

## Architecture and Agent Flow

The system has three core layers working in real time:

```
┌──────────────┐     ┌────────────────────────────┐     ┌─────────────────┐
│  Audio       │     │  Underwriting Co-Pilot LLM  │     │  Underwriter    │
│  Stream      │────▶│                              │────▶│  Dashboard      │
│              │     │  Processes full transcript    │     │                 │
│  Browser mic │     │  after each utterance.        │     │  - Transcript   │
│  ──▶ PCM     │     │                              │     │  - Inline cards │
│  chunks via  │     │  Self-classifies each turn:  │     │  - Sidebar chat │
│  WebSocket   │     │  • "suggestion" → show card  │     │  - Checklist    │
│              │     │  • "discrepancy" → alert card │     │  - Data points  │
│  ┌─────────┐ │     │  • "gap" → nudge card        │     │                 │
│  │ STT     │ │     │  • "none" → stay silent       │     │                 │
│  │ Whisper │ │     │                              │     │                 │
│  └─────────┘ │     │  Returns structured JSON:    │     │                 │
│  ┌─────────┐ │     │  question, detail, reason,   │     │                 │
│  │ Speaker │ │     │  stage, data_points,         │     │                 │
│  │ ID      │ │     │  checklist_updates           │     │                 │
│  └─────────┘ │     └────────────────────────────┘     └─────────────────┘
└──────────────┘
```

### Component Details

#### 1. Speech-to-Text + Speaker Identification

**STT Engine: faster-whisper (base.en)**
- Runs locally on CPU (int8 quantization) -- no cloud STT costs
- Converts live audio into text transcript segments in real time

**Speaker Diarization: SpeechBrain ECAPA-TDNN**
- Generates voice embeddings for each audio segment
- Maintains an embedding bank per speaker (Underwriter vs Borrower)
- Uses cosine similarity to assign each utterance to the correct speaker
- Falls back to question-detection heuristic (questions → Underwriter, statements → Borrower) when audio segments are too short for reliable embedding

#### 2. Underwriting Co-Pilot LLM

The core intelligence layer. After each transcript segment, the full conversation history is sent to the LLM along with the borrower's profile data.

**Primary model:** Gemini 2.5 Flash
**Fallback model:** GPT-4o-mini (switchable from the UI)

**What the LLM does on every turn:**
- Reads the entire transcript and borrower background
- Self-classifies the current turn into one of four trigger types:
  - `suggestion` -- there's a clear next question to ask per the underwriting sequence (most common)
  - `discrepancy` -- borrower said something that contradicts their profile data
  - `gap` -- important checklist items are still pending and conversation seems to be wrapping up
  - `none` -- nothing to surface right now (underwriter already asked the right thing, or waiting for more info)
- Generates a short question (max 8 words, plain language, no jargon), a detailed explanation, and a reason
- Tracks data points confirmed during the conversation (income figures, margins, obligations)
- Updates the checklist status (which topics have been verified, discussed, or flagged)
- Performs live calculations (FOIR, UPI % of sales, margin validation against industry benchmarks, supplier payment gap analysis)

**Trigger conditions** (when the co-pilot surfaces a card):
1. Borrower answers something → co-pilot suggests the next logical question in the underwriting sequence
2. Red flag detected → discrepancy between borrower claims and profile data
3. Underwriter has missed a key topic for too long → nudge
4. Risk threshold crossed (FOIR breach, unsecured exposure limit)

#### 3. Underwriter Dashboard

A single-page web app with three panels:

- **Left sidebar: Borrower Profile** -- loan details, banking summary, red flags with expandable metadata (EMI breakdowns, discrepancies, recent loans)
- **Center: Live Transcript** -- conversation bubbles (Underwriter on right, Borrower on left) with inline co-pilot suggestion cards that appear between conversation turns
- **Right panel: Checklist + AI Chat**
  - **Question Checklist** -- income assessment items and red flag items, silently updating status (pending → discussed → verified → warning) as conversation progresses
  - **Co-pilot Chat** -- private underwriter-to-AI chat. Click "Ask about this" on any suggestion card to start a contextual conversation. Has quick questions like "What is the FOIR?", "Recalculate without this EMI", "Should I ask for more documents?"

---

## KPIs to Improve

### Credit Quality
| Metric | Description |
|--------|-------------|
| **Delinquency rate (DPD 30+/60+/90+)** | Reduction in early and late delinquencies for cases where the co-pilot was used vs. control group |
| **Credit loss ratio** | Lower net credit losses on co-pilot-assisted cases |

### Underwriter Productivity
| Metric | Description |
|--------|-------------|
| **Cases per underwriter per day** | Throughput improvement due to faster, more structured PDs |
| **Average PD duration** | Time spent per personal discussion (target: same or lower while improving quality) |
| **Turnaround time (TAT)** | End-to-end time from PD to credit decision |

### Process Completeness
| Metric | Description |
|--------|-------------|
| **Question coverage %** | Percentage of critical questions asked during PD (tracked via the dynamic checklist) |
| **Document completeness at first submission** | Reduction in back-and-forth for missing documents |
| **Red flags surfaced per case** | Number of contradictions/risks identified that would have been missed without the co-pilot |

---

## Future Scope

### 1. Co-Pilot Effectiveness Tracking
Track whether the suggestions given by the co-pilot are actually useful and being adopted by underwriters:
- **Suggestion acceptance rate** -- percentage of inline cards/suggestions that the underwriter acted on (asked the suggested question, requested the flagged document)
- **Quick-action usage** -- frequency of underwriters using the chat panel to query the agent (recalculate FOIR, check eligibility, summarize discrepancies)
- **Suggestion-to-outcome correlation** -- whether acting on co-pilot suggestions correlates with better loan performance (lower DPD rates)
- **Ignore vs. dismiss tracking** -- distinguish between suggestions the underwriter saw and ignored vs. ones they actively dismissed (provides signal on relevance)

### 2. Video PD Integration
Integrate with video personal discussion (Video PD) solutions to:
- Fetch the audio stream directly from the Video PD platform -- the rest of the processing flow (STT, speaker identification, co-pilot analysis) remains the same
- Overlay co-pilot suggestions directly on the Video PD portal so underwriters don't need to switch between screens

### 3. Loan Origination System (LOS) Integration
- Pull case details (applicant profile, income documents, property details, existing obligations) directly from the LOS before the PD begins
- Pre-populate the co-pilot checklist based on the specific case type, loan product, and applicant profile
- Push PD summary and flagged items back into the LOS for the credit team to review

### 4. Underwriter Performance Scoring
Score and benchmark underwriter performance over time:
- **Completeness score** -- percentage of important questions asked and key data points verified across all PDs
- **Red flag detection rate** -- how often the underwriter catches issues independently vs. needing the co-pilot to flag them
- **Consistency index** -- variance in evaluation quality across different case types and borrower profiles
- **Skill progression tracking** -- improvement in scores over time, identifying training needs per underwriter
- **Peer benchmarking** -- anonymized comparison against team averages to surface coaching opportunities

### 5. Lightweight Classifier for Cost Optimization
Add a fast, low-cost classifier model before the main LLM to filter utterances that don't need analysis (small talk, acknowledgements, routine confirmations). Currently the full LLM is invoked after every transcript segment. A gate classifier could reduce LLM API calls by 70-80%, significantly lowering cost at scale.

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Python, FastAPI, Uvicorn, WebSocket |
| Speech-to-Text | faster-whisper (base.en, int8 quantized, runs on CPU) |
| Speaker Diarization | SpeechBrain ECAPA-TDNN (cosine similarity on voice embeddings) |
| Co-Pilot LLM | Gemini 2.5 Flash (primary), GPT-4o-mini (alternative) |
| Audio Capture | Browser Web Audio API with AudioWorklet, PCM streaming over WebSocket |
| Frontend | Single-page HTML, vanilla JavaScript, Tailwind CSS |
| ML Libraries | PyTorch, NumPy, scikit-learn |

## Project Structure

```
underwriting-copilot/
├── server.py          # FastAPI backend — WebSocket, STT, diarization, LLM orchestration
├── scenarios.py       # Demo borrower profiles (5 scenarios with red flags, banking data)
├── static/
│   └── index.html     # Frontend (single-page app, ~1000 lines)
├── requirements.txt   # Python dependencies
├── .env               # API keys (not committed)
└── .gitignore
```

## Setup

### 1. Clone and install

```bash
git clone https://github.com/parasavedisweta-collab/meeting-scribe.git
cd meeting-scribe
pip install -r requirements.txt
```

### 2. Add API keys

Create a `.env` file:

```
GEMINI_API_KEY=your_gemini_key_here
OPENAI_API_KEY=your_openai_key_here
```

You only need one -- the UI has a toggle to switch between providers.

### 3. Run

```bash
python server.py
```

Open **http://localhost:8001** in your browser. First run downloads the SpeechBrain model (~100MB) automatically. Runs entirely on CPU -- no GPU required.

## Demo Scenarios

The app comes with 5 pre-loaded borrower profiles for testing:

| Scenario | Profile | Loan | Red Flags |
|----------|---------|------|-----------|
| Vikram Malhotra | Self-employed, Grocery | Business Loan ₹15L | FOIR breach, High unsecured exposure |
| Priya Mehta | Salaried | Personal Loan ₹5L | Bank/Bureau mismatch, FOIR breach |
| Vikram Singh | Salaried | Personal Loan ₹5L | Aggressive borrowing, Recent delinquency, Declining ABB |
| Prakash Reddy | Self-employed, Grocery | Business Loan ₹7L | Business income stopped |
| Priya Sharma | Self-employed, Consulting | Personal Loan ₹8L | High loan velocity, Historical delinquency |
