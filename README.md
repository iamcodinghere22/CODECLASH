# Green Building Digital Twin: GenAI Strategy Agent

Code Clash 2.0 (DSU Hackathon) | Problem Statement 04: Green Building Digital Twin

This module is **Part A (GenAI Intelligence and Optimization Agent)**. It receives
live telemetry from the Digital Twin (N), detects energy waste, and returns
**deterministic control commands** (for example, "set Zone_B lighting to 0%")
with projected power, cost and carbon savings. It also has a chat/audit interface
("Why is Zone_B flagged red?").

## Team
- S (Computer Vision): <name>
- N (Digital Twin and Dashboard): <name>
- A (GenAI Agent): <name>

## Architecture

```
[S: CV] --frames--> [N: Digital Twin] --telemetry--> [A: GenAI Agent]
                          ^                                |
                          +-------- control actions -------+
```

### Design: code decides, AI explains
- `analytics.py` (plain Python) detects waste and calculates every command,
  target value and watt saving, so results are reproducible.
- The LLM only writes the summary and the rationale for each action, and answers
  chat questions. It cannot change a command or a number.
- If the AI service is unavailable, the agent still returns the same commands with
  templated rationales (`generated_by: "fallback"`).

## Setup

```
python -m venv venv
venv\Scripts\activate          # Mac/Linux: source venv/bin/activate
python -m pip install -r requirements.txt
copy .env.example .env         # Mac/Linux: cp .env.example .env
# edit .env and add your API key
python -m uvicorn main:app --reload
```

Open http://localhost:8000/docs for interactive API docs.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/health` | Health check |
| POST | `/api/agent/recommend` | Twin telemetry in, control actions out |
| POST | `/api/agent/chat` | Natural-language audit of the telemetry |

Query options for `/api/agent/recommend`:
- `?minimal=true` returns only the contract fields (no extras)
- `?language=Hindi` writes the summary and rationales in another language

### Input: Twin Telemetry (N to A)

```json
{
  "timestamp": "2026-10-15T14:30:00+05:30",
  "room_metadata": {"room_id": "Lab_302", "capacity": 40, "area_sqft": 1200},
  "zone_states": {
    "Zone_B": {
      "occupancy": 0,
      "light_status": {"active_fixtures": 10, "dimming_level": 100},
      "hvac_setpoint_c": 22,
      "measured_temp_c": 23.5,
      "power_draw_w": 1900,
      "wasted_power_w": 1900,
      "vacant_minutes": 7
    }
  },
  "metrics": {"total_power_kw": 4.4, "cumulative_wasted_kwh": 1.8, "carbon_rate_kgco2_per_hr": 3.6}
}
```

| Field | Required | Notes |
|---|---|---|
| `room_metadata.room_id` | Yes | |
| `zone_states` | Yes | Dictionary keyed by zone id |
| `occupancy` | Yes in practice | 0 means vacant |
| `light_status` | For lighting actions | Active fixture count and dimming level (0-100). Key names accepted: `active_fixtures`/`fixture_count`/`fixtures`, `dimming_level`/`dimming_pct`/`dim_level` |
| `hvac_setpoint_c` | For HVAC actions | |
| `power_draw_w`, `wasted_power_w` | Recommended | Caps and sanity-checks the savings |
| `measured_temp_c` | Optional | Extra evidence |
| `vacant_minutes` | Optional | Enables the "empty longer than 3 minutes" rule |
| `light_status.watts_per_fixture` | Optional | Defaults to 40 W if missing |

Extra fields are accepted and ignored.

### Output: GenAI Recommendation (A to N)

Contract fields: `timestamp`, `summary`, `actions[]` with
`zone_id`, `component` (`LIGHTING` or `HVAC`), `current_value`, `target_value`,
`projected_power_saving_w`, `rationale`.

Optional extras: per-action `unit`, `priority`, `projected_cost_inr_per_hr`,
`projected_co2_kg_per_hr`; and top-level `totals`, `zone_status`
(`optimal` = green, `sub_optimal` = yellow, `energy_waste` = red),
`assumptions`, `generated_by` (`ai`, `rules` or `fallback`).

```json
{
  "timestamp": "2026-10-15T14:30:00+05:30",
  "summary": "...",
  "actions": [
    {
      "zone_id": "Zone_B", "component": "LIGHTING",
      "current_value": 100.0, "target_value": 0.0,
      "projected_power_saving_w": 400.0,
      "rationale": "...",
      "unit": "percent", "priority": "high",
      "projected_cost_inr_per_hr": 3.2, "projected_co2_kg_per_hr": 0.328
    }
  ],
  "totals": {"power_saving_w": 1105.6, "cost_inr_per_hr": 8.85, "co2_kg_per_hr": 0.907},
  "zone_status": [{"zone_id": "Zone_B", "status": "energy_waste", "issues": ["..."]}],
  "assumptions": ["Lighting power assumed 40 W per fixture (not provided)"],
  "generated_by": "ai"
}
```

### Chat

```json
POST /api/agent/chat
{"telemetry": { ...same as above... }, "question": "Why is Zone_B flagged red?"}
```
returns `{"answer": "...", "zones_referenced": ["Zone_B"], "generated_by": "ai"}`.

## Calling the agent from the Twin

```python
import requests
r = requests.post("http://localhost:8000/api/agent/recommend", json=telemetry)
for action in r.json()["actions"]:
    apply_to_twin(action["zone_id"], action["component"], action["target_value"])
```

After the twin applies the actions and sends fresh telemetry, the agent does not
repeat them, because that waste no longer exists.

## Quick test (no other modules needed)

```
python test_agent.py
```
It sends `mock_telemetry.json` to both endpoints and prints the results.

## Files

| File | Role |
|---|---|
| `main.py` | FastAPI app and endpoints |
| `schemas.py` | Pydantic models for the input and output contract |
| `analytics.py` | Deterministic engine: waste detection, commands, savings |
| `genai_service.py` | Orchestration: calls the engine, then the LLM for wording; chat |
| `llm.py` | LLM client (any OpenAI-compatible provider, configured in `.env`) |
| `mock_telemetry.json` | Sample twin output for testing |
| `test_agent.py` | Sends the sample to the API |

## Assumptions (tunable in `analytics.py` or `.env`)

Savings are **estimates**, not meter readings.

- Lighting power: 40 W per fixture when not provided
- HVAC saving: 6% of HVAC power per 1 C of setpoint change, capped at 50%
- Targets: 24 C setpoint when occupied, 26 C when vacant
- Vacancy grace period: 3 minutes (when `vacant_minutes` is sent)
- Tariff: INR 8 per kWh (assumed); carbon factor: 0.82 kg CO2 per kWh (team contract)
- HVAC power is estimated as zone power minus estimated lighting power

The assumptions actually used are returned in the `assumptions` field of each response.

## Security
API keys live in `.env`, which is git-ignored. Never commit it.
