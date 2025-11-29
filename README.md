# AgentsVille Trip Planner

**Technical Architecture & Agent Reasoning System**

## Overview

The **AgentsVille Trip Planner** is a multi-agent LLM system that builds, evaluates, and optimizes travel itineraries using structured Pydantic models, ReAct reasoning, and tool-assisted execution.

It integrates:

* Structured data modeling
* Multi-step LLM reasoning
* Deterministic tool invocation
* Mocked API retrieval (weather, activities)
* Iterative refinement loops
* Schema-validated outputs

Helper utilities: `project_lib.py` 
Project instructions and rubric:

* The Scenario 
* Rubric 
* Assignment 
* Implementation Notes 
* Evaluation Summary 

---

# 1. System Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    AgentsVille Trip Planner                │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│                    User Input Layer                        │
│  - Travelers, dates, budget, interests                     │
│  - Parsed & validated via Pydantic                         │
│    (Traveler, VacationInfo models)                         │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│              Data Acquisition Layer                        │
│  (from project_lib.py)                                     │
│   - call_weather_api_mocked()                              │
│   - call_activities_api_mocked()                           │
│   - call_activity_by_id_api_mocked()                       │
│  Provides contextual constraints for the itinerary          │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│                ItineraryAgent (LLM #1)                     │
│  - Role-based system prompt                                │
│  - Chain-of-Thought reasoning                              │
│  - Generates initial TravelPlan JSON                       │
│  - Must conform to TravelPlan schema                       │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│                  Evaluation Layer                          │
│   - Budget accuracy                                        │
│   - Date alignment                                         │
│   - No hallucinated activities                             │
│   - Weather compatibility check using                      │
│     ACTIVITY_AND_WEATHER_ARE_COMPATIBLE prompt             │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│            ItineraryRevisionAgent (LLM #2)                 │
│  - ReAct cycle: THOUGHT → ACTION → OBSERVATION             │
│  - Uses tools:                                             │
│        • calculator_tool                                   │
│        • get_activities_by_date_tool                       │
│        • run_evals_tool (must run twice)                   │
│        • final_answer_tool                                 │
│  - Revises plan until all criteria pass                    │
└────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│                   Final Output Layer                       │
│  - Final validated TravelPlan (structured JSON)            │
│  - Optional natural-language narrative (TTS)               │
└────────────────────────────────────────────────────────────┘
```

---

# 2. Agent Lifecycle Diagrams

## 2.1 ItineraryAgent Lifecycle (LLM #1)

```
┌────────────────────────────────────────┐
│ ItineraryAgent SYSTEM PROMPT          │
│ - Role: Expert Travel Planner          │
│ - Input: VacationInfo, weather, events │
│ - Output: TravelPlan JSON              │
└────────────────────────────────────────┘
                │
                ▼
    ┌────────────────────────────┐
    │     ANALYSIS STAGE         │
    │  - Extract dates           │
    │  - Filter activities       │
    │  - Check weather           │
    │  - Respect interests       │
    │  - Track cost              │
    └────────────────────────────┘
                │
                ▼
    ┌────────────────────────────┐
    │  JSON GENERATION STAGE     │
    │  - Must match schema       │
    │  - No invented IDs         │
    │  - Valid daily schedule    │
    └────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────┐
│    INITIAL ITINERARY (TravelPlan)      │
└────────────────────────────────────────┘
```

---

## 2.2 ReAct Agent Lifecycle (LLM #2)

Reference: Itinerary Revision requirements in project assignment  and rubric .

```
               ┌───────────────────────────────┐
               │ ItineraryRevisionAgent Prompt │
               │ - Role: Refinement Agent      │
               │ - Explicit ReAct instructions │
               │ - Tools list + descriptions   │
               └───────────────────────────────┘
                               │
                               ▼
       ┌─────────────────────────────────────────────┐
       │ THOUGHT: internal reasoning                 │
       │ Examples:                                    │
       │  - “Need to verify weather constraints”      │
       │  - “Budget mismatch detected”                │
       │  - “Insufficient daily activities”           │
       └─────────────────────────────────────────────┘
                               │
                               ▼
       ┌─────────────────────────────────────────────┐
       │ ACTION: tool call                           │
       │ {"tool_name": "...", "arguments": {...}}     │
       └─────────────────────────────────────────────┘
                               │
                               ▼
       ┌─────────────────────────────────────────────┐
       │ OBSERVATION: Python returns tool output      │
       │ (weather, activity list, eval results)       │
       └─────────────────────────────────────────────┘
                               │
                               ▼
                   (loop repeats N times)
                               │
                               ▼
       ┌─────────────────────────────────────────────┐
       │ Must run run_evals_tool AGAIN before exit   │
       └─────────────────────────────────────────────┘
                               │
                               ▼
       ┌─────────────────────────────────────────────┐
       │ ACTION: final_answer_tool                   │
       │ Returns FINAL TravelPlan JSON               │
       └─────────────────────────────────────────────┘
```

---

# 3. Tools & Execution Model

### Tool definitions (in `project_lib.py` ):

| Tool                          | Purpose                                       | Typical Use                                    |
| ----------------------------- | --------------------------------------------- | ---------------------------------------------- |
| `calculator_tool`             | Summation & basic arithmetic                  | Ensure total trip cost accuracy                |
| `get_activities_by_date_tool` | Search valid activities for a given date/city | Fill missing days, avoid invalid IDs           |
| `run_evals_tool`              | Execute all evaluation functions              | Must be run at start *and before final answer* |
| `final_answer_tool`           | System exit point                             | Outputs validated itinerary to Python          |

Tool calls are always JSON of the form:

```
{"tool_name": "get_activities_by_date_tool",
 "arguments": {"date": "2025-06-11", "city": "AgentsVille"}}
```

---

# 4. Data Flows

```
Traveler Input
    │
    ▼
VacationInfo (Pydantic)
    │
    ▼
Weather + Activities APIs (Mocked)
    │
    ▼
ItineraryAgent (LLM #1)
    │ → Generates TravelPlan JSON
    ▼
Evaluation Layer (LLM + tools)
    │ → Flags errors, mismatches, weather issues
    ▼
ItineraryRevisionAgent (LLM #2)
    │ → ReAct cycles, tool usage
    ▼
Final Answer Tool
    │
    ▼
Final Validated Itinerary
```

---

# 5. Compliance with Project Rubric

Project Evaluation confirms full compliance: **MEETS SPECIFICATIONS** 

Key validated components:

* Correct role-based prompts
* Detailed Chain-of-Thought reasoning
* Comprehensive ReAct design
* Strict JSON schema validation
* Complete tooling layer with accurate docstrings
* Effective weather/event compatibility agent
* Successful end-to-end workflow generation

---

# 6. Running the System

1. Open `project_starter.ipynb`
2. Insert your OpenAI key
3. Execute sequentially:

   * Define traveler data
   * Retrieve weather & activities
   * Build ItineraryAgent prompt
   * Run ItineraryAgent
   * Evaluate
   * Run ItineraryRevisionAgent
   * Generate final narrative (optional)

---

# 7. Optional: Narrative Generation

Using `narrate_my_trip()` from `project_lib.py` ,
the system produces a natural-language overview plus an audio narration via TTS.
