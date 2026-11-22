# Pitchwise Agentic Pitch Clinic

A prototype multi-agent system that takes a founder's rough startup pitch and returns a polished version, graded against the accelerator's own playbook. It combines three agentic patterns in one pipeline: **routing**, **evaluator-optimizer**, and **MCP** (Model Context Protocol).

Built as Lab 9 using the OpenAI Agents SDK.

## Background 

Pitchwise is a (fictional) startup accelerator whose partners are overwhelmed with practice pitch drafts before demo day. They want a first-pass screen that reflects *their* standards rather than generic AI coaching, so partners can spend review time on pitches that have already reached a baseline quality.

## How It Works

```
 founder draft
      |
      v
 [Classifier]  --> consumer | b2b_saas | deep_tech
      |
      v
 [Matched Coach]  --> draft v1
      |
      v
 +------------------------------------------+
 |  Evaluator-Optimizer loop (max 3 iters)  |
 |                                          |
 |  [Evaluator] <--get_rubric-- [MCP Server]|
 |      |                                   |
 |   passes? --yes--> done                  |
 |      |no                                 |
 |  [Optimizer] --> revised draft ----------+
 +------------------------------------------+
      |
      v
 final pitch + per-criterion feedback
```

1. **Routing**: A classifier agent returns a Pydantic-constrained `PitchClassification` (`Literal["consumer", "b2b_saas", "deep_tech"]` plus reasoning). The result picks the matching coach.
2. **Coaching**: One of three specialist coaches rewrites the draft to address that pitch type's rubric criteria. Coaches never invent facts; they insert bracketed placeholders (e.g. `[FIRST CUSTOMER NEEDED: ...]`) where information is missing.
3. **Evaluation**: The evaluator calls the `get_rubric` MCP tool at run time, scores the pitch on three criteria, and returns a structured `PitchEvaluation` (`passes`, `per_criterion_feedback`, `summary`).
4. **Optimization**: If the pitch fails, the optimizer revises it using the feedback. The loop stops on `passes=True` or after `MAX_ITERATIONS = 3`.

## The Playbook (Rubrics)

| Pitch type | Criteria |
|---|---|
| Consumer | Hook, Why now, Distribution |
| B2B SaaS | Customer pain, Market sizing, Traction |
| Deep tech | Technical edge, Defensibility, First customer |

## Project Structure

| Component | Description |
|---|---|
| `pitchwise_server.py` | FastMCP server exposing `get_rubric(pitch_type)`. Written out by the notebook (Part 1). |
| Lab 9 notebook | Defines the classifier, coaches, evaluator, optimizer, and `run_pitch_pipeline`. |
| `openai_setup` | Local helper module the notebook imports for API configuration. |

### Key Objects

- `PitchClassification`, `PitchEvaluation`: Pydantic output schemas
- `classifier_agent`: routes pitches
- `consumer_coach`, `b2b_saas_coach`, `deep_tech_coach`: specialist rewriters
- `EVALUATOR_INSTRUCTIONS`: evaluator prompt; the agent is built inside an `async with MCPServerStdio(...)` block so it can use the live server
- `optimizer_agent`: revises drafts from evaluator feedback
- `run_pitch_pipeline(pitch_id, pitch_text)`: runs the full flow and returns `pitch_type`, `final_draft`, `evaluation`, and `iterations`

## MCP Server

```python
@mcp.tool()
def get_rubric(pitch_type: str) -> dict:
    """Return the Pitchwise scoring rubric for a given pitch type."""
    return RUBRICS.get(pitch_type, {"name": "Unknown", "criteria": []})
```

## Setup

**Requirements**

- Python 3.10+ (developed on 3.13)
- An OpenAI API key (agents use `gpt-4.1`)
- Jupyter

**Install**

```bash
pip install openai-agents mcp pydantic ipython jupyter
```

Provide your API key via the environment or the `openai_setup` module.

## Usage

1. Open the notebook and run the cells in order.
2. Part 1 writes `pitchwise_server.py` to disk. The evaluator launches it as a subprocess via `MCPServerStdio` (`python pitchwise_server.py`).
3. Run the pipeline on a pitch:

```python
result = await run_pitch_pipeline("my_pitch", pitch_text)
print(result["final_draft"])
print(result["evaluation"].summary)
```

The notebook runs three sample founder pitches end to end (about 30-60 seconds total).

## Sample Results

| Pitch | Type | Iterations | Passed |
|---|---|---|---|
| A: SnackPocket | Consumer | 1 | Yes |
| B: ShiftRoster | B2B SaaS | 3 (cap) | No |
| C: KitchenSense | Deep tech | 3 (cap) | No |

Pitches B and C never converged because the original drafts lacked key facts (a traction signal and a named first customer). Since the coaches and optimizer are forbidden from inventing facts, the loop can sharpen wording but cannot supply missing business information. More iterations would not help; the founder must fill in the placeholders.

## Design Notes

- **Why MCP for the rubric?** The playbook lives in one place. Updating it updates every evaluation with no prompt changes. The tradeoff is an extra tool call per evaluation (slightly slower, more moving parts).
- **Why a hard iteration cap?** It bounds API cost and stops the loop on pitches that cannot improve without new information.
- **Why placeholders instead of invention?** Fabricated customers or technical claims would make the output untrustworthy. Placeholders show the founder exactly what to fill in.

## Possible Next Steps

- Surface unresolved placeholders as an explicit founder to-do list instead of looping on them
- Stop early when feedback stops changing between iterations
- Add an "unclear/other" category to the classifier
- Log per-iteration scores so partners can see how each pitch improved
