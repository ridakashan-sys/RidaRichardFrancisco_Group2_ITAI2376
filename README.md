# RidaRichardFrancisco_Group2_ITAI2376
Final AI Agent Team Project - Francisco | Richard | Rida

# 🗼 Paris Travel Planner Agent

> A conversational AI agent that builds personalized, constraint-aware multi-day Paris itineraries through a continuous chat interface — handling hotels, attractions, events, budgets, and walking routes in a single conversation.

---

## Team Members

 **Francisco**
 **Richard** 
 **Rida** 

**Course:** ITAI 2376 — Date: May 4, 2026

---

## The Problem & Target User

Planning an international trip means juggling hotel sites, mapping apps, ticketing platforms, opening hours, and a budget spreadsheet — all at the same time. A single wrong assumption (a museum is closed on Mondays, a day's schedule is physically impossible) can unravel an entire itinerary.

**This agent solves that problem for first-time and returning Paris travelers** who want a complete, feasible, optimized itinerary produced through a simple conversation — without switching between ten different tabs. The user provides their travel dates, nightly budget, interests, and mobility preferences. The agent does the rest: it checks opening hours, clusters locations geographically, routes each day efficiently, warns when a day is over-scheduled, and suggests seasonal events — all while remembering everything said earlier in the conversation.

---

## Architecture Choice

**Option A — Single Agent System**

> ⚠️ **Architectural Pivot from Midterm Plan:** The Midterm blueprint planned to use a standard LangChain `AgentExecutor`. During implementation, pip resolved LangChain v0.3.0+ in Google Colab, which silently broke all `AgentExecutor` import paths. Rather than pin deprecated dependencies, the team modernized the architecture to **LangGraph** (`create_react_agent`). This was not a switch from Single to Multi-Agent — the single-agent philosophy was preserved — but the underlying framework was fully replaced. The pivot produced a more reliable, stateful application than the original plan would have.

---

## Architecture Overview

The agent is organized as five layers stacked vertically, with data flowing top-to-bottom through each on every user message.

The **Gradio ChatInterface** captures freeform user input across multiple turns. Each message is passed to the **LangGraph ReAct Core**, which maintains full conversation history via `MemorySaver` pinned to a stable thread ID (`"session_2"`). The core runs a 12-rule system prompt that enforces real-world planning constraints — geographic clustering, daily pacing limits, mandatory meal breaks — before deciding which tool to call. LLM requests are routed through a **Multi-Model Fallback Chain** (OpenCode → Groq → Gemini 2.0 → Gemini 1.5 Pro) so the agent continues working even if one provider hits a rate limit. The chosen model invokes one or more of the **six custom LangChain tools**, each of which queries the **local JSON knowledge base** (hotels, locations, events) rather than live external APIs — making every tool call deterministic and near-instant.

### Architecture Diagram

## Frameworks, Tools & Libraries

| Category | Technology |
|---|---|
| **Agent Framework** | LangGraph `create_react_agent`, LangGraph MemorySaver |
| **LLM Orchestration** | LangChain Core, LangChain Tools (`@tool`) |
| **Primary LLM** | OpenCode — gpt-5.5 / claude-opus-4-7 (via OpenCode router) |
| **Fallback LLM 1** | Groq — llama-3.3-70b-versatile |
| **Fallback LLM 2** | Google Gemini 2.0 Flash |
| **Fallback LLM 3** | Google Gemini 1.5 Pro |
| **LLM Adapters** | `langchain-openai`, `langchain-groq`, `langchain-google-genai` |
| **Conversational UI** | Gradio `ChatInterface` |
| **Knowledge Base** | Local JSON files (no vector store; RAG planned for v2) |
| **Route Optimization** | Custom Python — Nearest-Neighbor Greedy + 2-opt algorithm |
| **Distance Calculation** | Haversine approximation (Paris-latitude-corrected) |
| **Runtime Environment** | Google Colab (Python 3.10+) |

---

## Installation

### Prerequisites

- Python **3.10 or higher**
- A Google account (the agent runs in Google Colab and reads data from Google Drive)
- At least **one** API key from the providers listed below

### Step 1 — Clone the Repository

```bash
git clone https://github.com/your-team/paris-travel-planner.git
cd paris-travel-planner
```

### Step 2 — Upload Data Files to Google Drive

Create a folder called `ParisTravelPlanner` in the root of your Google Drive and upload the three knowledge base files from this repo:

```
MyDrive/
└── ParisTravelPlanner/
    ├── hotels.json
    ├── locations.json
    └── events.json
```

### Step 3 — Set Up Environment Variables (API Keys)

The agent is designed to run in **Google Colab Secrets** (the 🔑 key icon in the left sidebar). Add each key you have as a secret — the agent will load whichever keys exist and build its fallback chain automatically.

| Secret Name | Where to Get It | Required? |
|---|---|---|
| `OPENCODE_API_KEY` | [opencode.ai](https://opencode.ai) | Optional (Primary) |
| `GROQ_API_KEY` | [console.groq.com](https://console.groq.com) | Optional (Fallback 1) |
| `GEMINI_API_KEY` | [aistudio.google.com](https://aistudio.google.com/app/apikey) | Optional (Fallback 2) |
| `GOOGLE_API_KEY` | Same as above | Optional (Fallback 3) |

> **You need at least one key.** The agent checks all four and uses whatever is available.

If running **locally** instead of Colab, copy `.env.example` to `.env` and fill in your keys:

```bash
cp .env.example .env
# then open .env and add your keys
```

### Step 4 — Install Dependencies

**In Google Colab** (Cell 1 handles this automatically):

```python
!pip install -q langchain langchain-core langchain-openai langchain-google-genai langchain-groq langgraph gradio
```

**Locally:**

```bash
pip install -r requirements.txt
```

---

## How to Run

### In Google Colab (Recommended)

Open `Paris_Travel_Planner_V10.ipynb` in Google Colab and run all cells in order:

```
Cell 1 → API keys + package installation
Cell 2 → Imports, data loading, tool definitions
Cell 3 → Route optimizer
Cell 4 → LLM fallback chain + agent assembly
Cell 5 → Launch Gradio interface
```

Cell 5 will print a public shareable URL that looks like:

```
Running on public URL: https://xxxxxx.gradio.live
```

Open that URL in any browser. The chat interface loads immediately.

### Locally (Optional)

```bash
python app.py
```

Then open `http://localhost:7860` in your browser.

---
# Example Usage

The screenshots  show the live agent running in the Gradio interface.

---

### Example — 5-Day Art & Wine Trip Planning

When the interface loads, the agent surfaces three suggested starter prompts to guide new users:

![Gradio chat interface on launch showing three example prompts]

**User input:**
```
Plan a 5-day trip for me this May. My budget is 250 euros per night. I love art and wine!
```

The agent calls `search_hotels`, `search_locations_by_tags`, `search_events`, and `optimize_route` before generating a full response. Below is the live output for Day 1:

![Agent output showing Day 1 itinerary table for Louvre, Tuileries and wine tasting](docs/screenshots/day1_output.jpeg)

**What the agent produced for Day 1:**

- **Theme:** Masterpieces + central Paris wine
- **Closure warning:** Avoid Tuesday — the Louvre is closed Tuesdays
- **Optimized route:** Louvre Museum → Tuileries Garden → Musée de l'Orangerie → Les Caves du Louvre

| Time | Plan | Fee |
|---|---|---|
| 09:30–12:30 | **Louvre Museum** — Mona Lisa, Winged Victory, French & Italian masters | ~€22 |
| 12:30–14:00 | Lunch near Palais Royal / rue Saint-Honoré | — |
| 14:00–15:00 | **Tuileries Garden** stroll | Free |
| 15:00–16:30 | **Musée de l'Orangerie** — Monet's Water Lilies | ~€12.50 |

Days 2–5 follow in the same timed-table format, each with its own theme, route order, closure checks, and fee totals. The agent ends every response with three contextual follow-up questions — visible in the interface — so the user can immediately refine any part of the plan.



## Known Limitations

**Static knowledge base, not live data.** The hotels, locations, and events datasets were curated manually and are frozen at the time of project completion. Real pricing, seasonal closures, and newly opened venues are not reflected. The agent cannot check real-time availability or confirm whether a specific hotel room is bookable tonight.

**Paris only.** The JSON knowledge base, the route optimizer's Haversine correction factor, and the system prompt's planning rules are all calibrated specifically for Paris. The agent is not designed to plan trips to other cities and would produce unreliable results if asked.

**No actual booking capability.** The agent recommends and plans but cannot complete a reservation. At the end of a conversation the user still needs to visit Booking.com, the attraction's ticketing site, or another platform to confirm and pay.

**Free-tier API rate limits.** Groq's free tier caps usage at 100,000 tokens per day. Gemini's free tier also has per-minute limits. Heavy testing or long multi-turn sessions can exhaust these limits within a single day, degrading response quality if all providers are saturated simultaneously.

**No persistent memory between sessions.** LangGraph's `MemorySaver` holds conversation state in RAM for the duration of a Colab session. When the notebook is closed or the runtime resets, all conversation history is lost. A returning user must re-state their preferences from scratch.

**Route optimizer is walking-only.** The `optimize_route` tool minimizes walking distance using a Haversine approximation. It does not account for the Paris Métro, RER trains, or bus routes. For cross-city days (e.g., a trip that includes both Montmartre and Versailles), transit time is not factored into the schedule.

**No image or map output.** The agent produces text itineraries only. It does not render a map, generate a PDF, or produce any visual output beyond what Gradio displays in the chat window.

---

## Demo Video



## Repository Structure

```
paris-travel-planner/
├── Paris_Travel_Planner_V10.ipynb   
├── README.md                      
├── REFLECTION.md                    
├── requirements.txt                 
├── .env.example                     
├── .gitignore                    
├── docs/
│   └── architecture.png           
└── data/
    ├── hotels.json                  
    ├── locations.json              
    └── events.json                
```

---

## License

This project was created for academic purposes as part of ITAI 2376. All data in the `data/` folder is manually curated for educational use.
