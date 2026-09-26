# Agent Simulation for Regression Testing

Can multi-turn agent simulations catch regressions that ordinary smoke tests miss?

This repository contains a small experiment that compares a **healthy AI agent implementation** with a deliberately **regressed implementation** using Gemini on Vertex AI.

The experiment demonstrates a simple failure mode:

> The LLM correctly understands the user's constraint, but downstream business logic accidentally ignores it.

In the test run included in this project:

```text
Manual smoke tests with same output: 5/5
v1 refundable-violation scenarios:   0
v2 refundable-violation scenarios:   6
Paired regressions caught:            6
```

The goal is not to benchmark Gemini or claim that simulation catches every possible regression.

The goal is to demonstrate how **paired, multi-turn user simulations can exercise behavioral paths that basic smoke tests may never reach**.

---

## The Experiment

The example uses a simple AI travel agent.

A user can specify:

- origin
- destination
- maximum price
- whether the flight must be refundable

The system follows this flow:

```text
User conversation
        ↓
Gemini
        ↓
Structured constraints
        ↓
Flight-selection logic
        ↓
Recommendation
```

Gemini extracts structured intent such as:

```json
{
  "from_city": "Bengaluru",
  "to_city": "Delhi",
  "max_price": 160,
  "refundable_required": true
}
```

That structured state is then passed to deterministic business logic.

---

## v1 — Correct Implementation

The healthy version respects the refundable requirement.

```python
if refundable_required:
    matches = [
        flight
        for flight in matches
        if flight["refundable"]
    ]
```

If the user asks for:

```text
Bengaluru → Delhi
Budget: $160
Refundable: required
```

the healthy implementation selects:

```text
F100
$140
Refundable
```

---

## v2 — Regressed Implementation

The second implementation deliberately contains a bug:

```python
# Regression:
# refundable_required is accidentally ignored.
```

Route and budget are still enforced, but refundability is no longer applied before selecting the cheapest flight.

For the same request, v2 can select:

```text
F101
$95
Non-refundable
```

The LLM did not misunderstand the user.

The structured constraint was still correct:

```python
refundable_required = True
```

The regression occurred inside the downstream decision logic.

---

## Why Use Agent Simulation?

A normal smoke test might check:

```text
Find the cheapest Bengaluru → Delhi flight
Find Delhi flights under $200
Find Mumbai flights under $150
```

Both v1 and v2 produce the same output for these tests.

In this experiment:

```text
5/5 smoke tests produced identical results.
```

The regression remains invisible because none of those tests exercise refundability.

The simulated conversations introduce constraints across multiple turns.

For example:

```text
User:
I need a flight from Bengaluru to Delhi,
ideally under $160.

User:
And it must be refundable.
What's the best option you found?
```

This produces:

```text
v1 → F100 → refundable
v2 → F101 → non-refundable
```

---

## Paired Simulation

One important design choice in the experiment is that the synthetic user conversation is generated **once**.

The exact same messages are then replayed against both agent versions.

```text
                   Synthetic user
                         │
               ┌─────────┴─────────┐
               │                   │
               ▼                   ▼
             v1 Agent            v2 Agent
               │                   │
               ▼                   ▼
       choose_flight_v1()  choose_flight_v2()
               │                   │
               └─────────┬─────────┘
                         ▼
                       Compare
```

This reduces noise.

If separate simulated conversations were generated for each version, differences could come from the user simulator rather than the implementation change.

---

## Test Scenarios

The notebook contains eight user scenarios.

Six intentionally exercise the refundable requirement.

Examples include:

```text
Ask for Bengaluru → Delhi under $160.
Later clarify that the flight must be refundable.
```

```text
Ask for the cheapest Bengaluru → Mumbai flight.
Later request a refundable alternative under $130.
```

```text
Ask for Bengaluru → Delhi under $180.
Later reject all non-refundable options.
```

The experiment then compares v1 and v2 on the exact same conversations.

---

## Results

The experiment produced:

| Metric | Result |
|---|---:|
| Manual smoke tests | 5 |
| Smoke tests with same output | 5/5 |
| v1 refundable violations | 0 |
| v2 refundable violations | 6 |
| Paired regressions caught | 6 |
| Expected regressions missed | 0 |
| Unexpected regressions | 0 |

The affected scenarios were:

```text
S1
S3
S4
S5
S7
S8
```

---

## Why the Evaluator Is Deterministic

The experiment does not use an LLM judge to determine whether the refundable constraint was violated.

Instead, it checks the actual selected flight:

```python
if (
    constraints["refundable_required"]
    and selected_flight
    and not selected_flight["refundable"]
):
    violation = True
```

This separates responsibilities:

```text
Gemini
  ↓
Understand the user

Application code
  ↓
Make the decision

Evaluator
  ↓
Verify the invariant
```

For deterministic business rules, deterministic evaluation is preferable whenever possible.

LLM judges can still be useful for subjective properties such as:

- helpfulness
- tone
- completeness
- reasoning quality

But a property such as:

```text
refundable == true
```

does not require another model to evaluate.

---

## Architecture

```text
Scenario specification
        │
        ▼
Gemini generates synthetic user script
        │
        ▼
Conversation
        │
        ▼
Gemini extracts structured constraints
        │
        ├─────────────────────────┐
        ▼                         ▼
choose_flight_v1()         choose_flight_v2()
        │                         │
        ▼                         ▼
Correct business logic     Regressed business logic
        │                         │
        └────────────┬────────────┘
                     ▼
          Deterministic evaluator
                     │
                     ▼
              Paired comparison
```

---

## Tech Stack

- Python
- Google Colab
- Gemini 2.5 Flash
- Vertex AI
- Google Gen AI SDK
- Pandas

---

## Running the Notebook

Open:

```text
agent_simulation_implementation_regression.ipynb
```

in Google Colab.

### 1. Configure your Google Cloud project

Replace:

```python
PROJECT_ID = "YOUR_PROJECT_ID"
```

with your own Google Cloud project ID.

The notebook uses:

```python
LOCATION = "global"
MODEL_NAME = "gemini-2.5-flash"
```

### 2. Authenticate

The notebook uses Colab authentication:

```python
from google.colab import auth

auth.authenticate_user()
```

### 3. Ensure Vertex AI is available

Your Google Cloud project must have access to Vertex AI and the Gemini model used by the notebook.

### 4. Run all cells

Run the notebook from top to bottom.

The final section prints:

```text
=== EXPERIMENT FACTS ===
```

including:

```text
Manual smoke tests with same output
v1 refundable-violation scenarios
v2 refundable-violation scenarios
Paired regressions caught
Scenario IDs
```

---

## Repository Structure

```text
agent_simulation_implementation_regression/
│
├── agent_simulation_implementation_regression.ipynb
└── README.md
```

---

## What This Experiment Does Not Prove

This is intentionally a small controlled experiment.

It uses:

- one travel-agent example
- a fixed flight dataset
- eight synthetic scenarios
- five manual smoke tests
- one deliberately introduced implementation regression

The result should therefore not be interpreted as:

```text
Agent simulation catches every regression.
```

Instead, the experiment demonstrates a narrower point:

> Multi-turn simulation can exercise behavioral paths that a small set of ordinary smoke tests may not cover.

---

## Potential Extensions

The same approach could be used to inject and evaluate other agent regressions:

```text
Constraint dropped between turns
Incorrect tool argument
Permission check bypassed
Stale memory value
Wrong currency conversion
Fallback path ignoring a policy
Tool timeout triggering an unsafe default
Incorrect ranking after filtering
```

A larger experiment could compare which failures are caught by:

```text
Unit tests
vs
Smoke tests
vs
Fixed end-to-end scenarios
vs
Agent simulations
```

---

## Key Takeaway

The interesting result is not simply that a deliberately broken function failed.

The important difference was:

```text
5 ordinary smoke tests
→ no visible difference

6 affected multi-turn scenarios
→ 6 regressions detected
```

The LLM understood the user's intent correctly.

The failure happened later in the system.

That is why evaluating an agent should go beyond checking isolated model responses.

As agentic systems combine models, state, tools, APIs, and business logic, regression testing increasingly needs to evaluate the **behavior of the complete system**.

---

## Related Article

A detailed explanation of the experiment, design decisions, failure mode, and results is available in the accompanying article:

**Can Agent Simulation Catch Regressions That Smoke Tests Miss?**

---

