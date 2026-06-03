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
git clone https://github.com/parasavedisweta-collab/underwriting-copilot.git
cd underwriting-copilot
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
