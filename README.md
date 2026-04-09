# OpenFlo

**An open-source web automation framework powered by large multimodal models — with built-in UX evaluation.**

*Developed under [Nexus Labs](https://github.com/UCL-Nexus-Labs)*

> Built upon [Avenir-Web](https://github.com/Princeton-AI2-Lab/Avenir-Web) by the Princeton AI² Lab.

OpenFlo is a research-oriented framework for autonomous web agents. Beyond executing tasks in a browser, it evaluates the *quality of the interaction* at every step — scoring usability with established HCI metrics (SEQ and SUS), surfacing friction points, and optionally adopting a configurable user persona to simulate how different users would experience the same workflow.

---

## How It Works

At a high level, OpenFlo runs a **predict → execute → evaluate** loop:

1. **Predict** — A vision-language model observes the current browser screenshot (with optional labeled element overlays) and predicts the next action (click, type, scroll, etc.).
2. **Execute** — The predicted action is dispatched to Playwright, with coordinate mapping, element grounding, and automatic recovery on failure.
3. **Evaluate** — After each action, a UX scorer rates the step across four dimensions using a multi-metric extension of the Single Ease Question (SEQ). At session end, these step-level scores are synthesized into a System Usability Scale (SUS) report.

A `ChecklistManager` tracks intermediate goals to ensure sub-tasks are not silently skipped. Termination uses LLM-based semantic analysis (not simple string matching) and only fires after a minimum of 25 actions to avoid premature exits.

---

## UX Evaluation Design

OpenFlo's evaluation layer is the primary research contribution on top of Avenir-Web. It implements two established HCI instruments in a fully automated pipeline.

### Single Ease Question (SEQ) — Step-Level Scoring

The [SEQ](https://measuringu.com/seq10/) is a single 7-point item ("How easy was this step?") used widely in usability research for micro-task evaluation. OpenFlo extends it with **three additional dimensions**, scored on the same 1–7 scale after each action:

| Metric | What it measures |
|---|---|
| **SEQ** (overall ease) | Perceived difficulty of the action |
| **Efficiency** | Whether the action was direct and fast |
| **Clarity** | How understandable the UI element/response was |
| **Confidence** | How certain the user felt about the action/outcome |

Each dimension also produces a 1–2 sentence qualitative assessment. When a persona is active, scoring bias offsets are applied after the LLM response to simulate persona-specific tolerance levels.

### System Usability Scale (SUS) — Session-Level Scoring

The [SUS](https://www.usability.gov/how-to-and-tools/methods/system-usability-scale.html) is a standardized 10-item questionnaire (Brooke, 1996) that produces a single 0–100 usability score. OpenFlo's `SUSCalculator` maps accumulated step-level SEQ data onto the SUS framework using LLM-driven synthesis:

```
Final SUS Score = (X + Y) × 2.5
  X = Σ (score − 1) for positive items  {1, 3, 5, 7, 9}
  Y = Σ (5 − score) for negative items  {2, 4, 6, 8, 10}
```

Grades follow the [Sauro-Lewis curved grading scale](https://measuringu.com/sus/) (A+ ≥ 84.1 → F < 51.7). A statistical heuristic fallback is used if the LLM call fails, using volatility detection and learning-curve analysis across the four multi-metric dimensions.

### Friction Point Analysis

Steps where SEQ ≤ 3 are flagged as friction points and classified by severity. The report surfaces:
- Which actions caused friction
- The qualitative reasoning behind low scores
- A breakdown across efficiency, clarity, and confidence dimensions
- Persona-specific friction labels (e.g. `waiting`, `confusion`, `searching`, `error`, `ambiguity`)

---

## Persona-Based Evaluation

Any UX session can be evaluated through the lens of a specific user persona. When a persona is injected (via `-p persona.toml`), the LLM embodies that user's profile when scoring each action — adjusting judgements based on their:

- **Digital literacy**: `expert` | `intermediate` | `beginner` | `very_low`
- **Primary device**: `desktop_keyboard` | `desktop_mouse` | `tablet_touch` | `mobile_touch`
- **Reading speed**: `fast` | `normal` | `slow`
- **Tolerance for friction**: `high` | `medium` | `low` | `very_low`
- **Prior experience**: free-text description fed directly into prompts
- **Description**: 3–4 sentence narrative the LLM embodies during scoring
- **Scoring bias**: integer offsets applied post-LLM per metric (e.g. `seq_modifier = -1` to simulate a more critical user)

This makes it possible to compare how the same workflow scores for an expert desktop user vs. a low-literacy mobile user — without re-running the task.

---

## Repository Layout

```
OpenFlo/
├── src/
│   ├── openflo/
│   │   ├── agent/
│   │   │   ├── agent.py          # Central orchestrator: predict → execute → evaluate loop
│   │   │   ├── config.py         # Config loading and validation (TOML + env)
│   │   │   ├── executor.py       # Action dispatch (click, type, scroll, drag, …)
│   │   │   ├── predictor.py      # LLM interaction, action prediction, history compression
│   │   │   ├── evaluation.py     # Task completion verification and termination logic
│   │   │   └── reporting.py      # Result serialisation and action summary generation
│   │   ├── browser/              # Playwright integration and browser state management
│   │   ├── llm/                  # LLM engine abstraction (via LiteLLM)
│   │   ├── managers/
│   │   │   └── ux_synthesis.py   # SEQ-to-SUS orchestration (UXSynthesisManager)
│   │   ├── ux/
│   │   │   ├── seq_scorer.py     # Multi-metric SEQ evaluator (SEQ, Efficiency, Clarity, Confidence)
│   │   │   ├── sus_calculator.py # SUS scoring with Sauro-Lewis grading and heuristic fallback
│   │   │   └── report_generator.py # Markdown + JSON UX report output
│   │   ├── personas/
│   │   │   └── profile.py        # PersonaProfile dataclass
│   │   ├── prompts/              # Prompt templates and builders
│   │   └── utils/                # Image processing, reasoning utilities
│   ├── run_agent.py              # Entry point: single-task demo + batch runner
│   └── config/
│       ├── auto_mode.toml        # Primary config (model, playwright, UX, experiment)
│       └── persona.toml          # Example persona config with inline docs
├── data/                         # Example task JSON files
└── .env.example                  # API key template
```

---

## Requirements

- Python `>=3.9`
- Playwright-compatible browser (Chromium recommended)
- An API key for your chosen LLM provider (OpenRouter preferred)

**Key dependencies:**

| Package | Purpose |
|---|---|
| `playwright` | Browser automation |
| `litellm` | Unified LLM provider abstraction |
| `torch` + `sentence_transformers` | Embedding-based grounding and evaluation |
| `Pillow` | Screenshot processing |
| `backoff` | Exponential retry for LLM calls |
| `python-dotenv` | `.env` key loading |
| `BeautifulSoup4` / `lxml` | HTML parsing for element extraction |

---

## Setup

```bash
# Create a conda environment
conda create -n openflo python=3.11
conda activate openflo

# Install in editable mode
pip install -e src

# Install Playwright browser kernels
playwright install
```

Set your API key:

```bash
cp .env.example .env
# edit .env and fill in OPENROUTER_API_KEY
```

Environment variables take precedence over any `[api_keys]` values in the config TOML.

---

## Running

All scripts are run from `src/` (config paths are relative to `src/`):

```bash
cd src
```

### Batch Mode

Set `experiment.task_file_path` in your config, then:

```bash
uv run run_agent.py -c config/auto_mode.toml
```

### With a Persona

```bash
uv run run_agent.py -c config/auto_mode.toml -p config/persona.toml
```

The persona is injected into SEQ scoring prompts and surfaced in the final `sus_report.json`. See [`src/config/persona.toml`](src/config/persona.toml) for all fields with inline documentation.

---

## Task JSON Format

```json
[
  {
    "task_id": "task_001",
    "confirmed_task": "Find the official API docs for X",
    "website": "https://example.com/"
  }
]
```

---

## Configuration Reference

Configs are TOML files. See `src/config/auto_mode.toml` for a fully annotated example.

### `[basic]`
- `save_file_dir` — output root directory
- `default_task`, `default_website` — used when no batch file is provided

### `[model]`
- `name` — model identifier (e.g. `openrouter/anthropic/claude-sonnet-4-5`)
- `temperature`, `rate_limit`
- `reasoning_model` — separate model for termination/evaluation reasoning
- `checklist_model` — model for checklist management
- `completion_eval_model` — model for final task success evaluation

### `[experiment]`
- `task_file_path` — JSON task list for batch mode
- `overwrite` — skip or overwrite existing task output folders
- `max_op`, `max_continuous_no_op` — execution limits
- `highlight` — draw labeled overlays on screenshots

### `[playwright]`
- `headless`, `viewport`, `tracing`, `save_video`
- `locale`, `geolocation` — for locale/region-sensitive tasks

### `[ux]`
- `enable_synthesis` — enable SEQ/SUS evaluation (default `false`)
- `generate_report` — write `sus_report.json` at session end (default `true`)
- `ux_model` — model for SEQ scoring; falls back to main model if omitted
- `seq_screenshot_context` — include screenshots in step evaluation (default `true`)

### `[persona]`
All fields are optional; or pass a separate file with `-p persona.toml`.
- `id`, `display_name`, `age_range`
- `digital_literacy`: `"expert"` | `"intermediate"` | `"beginner"` | `"very_low"`
- `primary_device`: `"desktop_keyboard"` | `"desktop_mouse"` | `"tablet_touch"` | `"mobile_touch"`
- `reading_speed`: `"fast"` | `"normal"` | `"slow"`
- `tolerance_for_friction`: `"high"` | `"medium"` | `"low"` | `"very_low"`
- `prior_experience` — free text fed into scoring prompts
- `description` — 3–4 sentence narrative the LLM embodies
- `common_friction_types` — friction labels surfaced in the report
- `[persona.scoring_bias]` — integer offsets per metric (e.g. `seq_modifier = -1`)

---

## Outputs

Each task writes to `<save_file_dir>/<task_id>/`:

| File | Contents |
|---|---|
| `agent.log` | Per-task execution log |
| `result.json` | Final summary (task, actions, outcome, timing) |
| `config.toml` | Resolved config snapshot |
| `all_predictions.json` | Full LLM I/O trace for the task |
| `screenshots/` | `screen_<step>.png` and `screen_<step>_labeled.png` |
| `sus_report.json` | UX evaluation report — SEQ scores per step, SUS score and grade, friction analysis, persona context |

Run-level logs are written to `src/logs/`.

---

## Troubleshooting

- **Missing API key** — fill in `OPENROUTER_API_KEY` in `.env` (copy from `.env.example`)
- **Playwright browser not found** — run `uv run playwright install chromium`
- **Want to watch the browser** — set `playwright.headless = false`
- **Config paths look wrong** — run from `src/` or pass an absolute path with `-c`

---

## References

- Brooke, J. (1996). *SUS: A quick and dirty usability scale.* In P. Jordan, B. Thomas, B. Weerdmeester & I. McClelland (Eds.), Usability evaluation in industry. Taylor & Francis. — foundational SUS paper
- Sauro, J. & Lewis, J. R. (2011). *Correlations among prototypical usability metrics: evidence for the construct of usability.* CHI '11. — basis for the Sauro-Lewis grading scale used in `sus_calculator.py`
- McGee, M. (2004). *Master usability scaling: Magnitude estimation and master scaling applied to usability measurement.* CHI '04. — background on magnitude-estimation approaches to usability scoring
- [measuringu.com/seq10/](https://measuringu.com/seq10/) — SEQ methodology and benchmarks
- [measuringu.com/sus/](https://measuringu.com/sus/) — SUS grading and percentile lookup

---

## Attribution

OpenFlo is built upon [Avenir-Web](https://github.com/Princeton-AI2-Lab/Avenir-Web) by the Princeton AI² Lab. The UX evaluation layer (SEQ multi-metric extension, SUS synthesis, persona framework) is original work developed at UCL Nexus Labs.

## License

This project maintains the same license as the original Avenir-Web framework (OpenRAIL-S). See the [LICENSE](LICENSE) file for details.
