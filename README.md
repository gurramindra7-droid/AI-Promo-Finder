# AI Promo Finder

**Ranked, ready-to-cut promos from microdrama scripts.**

AI Promo Finder reads a microdrama's scripts and gives the promo editor a ranked list of
ready-to-cut promos: the episode and exact start/end time of every segment, the hook,
the cliffhanger, one line on why it works, and why it is ranked where it is.

Built for the Kuku AI Generalist assignment, Problem 1 ("Find the Promos").

---

## What a promo is (from the brief)

A 20-60 second clip from the show itself that:

1. **opens on a hook**: something in the first 3 seconds that stops a scrolling viewer
2. **ends on a cliffhanger**: it cuts at an unresolved moment
3. **makes sense on its own** to someone who has never seen the show

The tool only finds moments and cut points. No editing, music or overlays.

---

## How it works

```
script(s) ──► 1. PARSE (code) ──► 2. TAG (AI) ──► 3. CANDIDATES (code) ──► 4. STRANGER TEST (AI) ──► 5. RANK (code) ──► editor (app)
```

| Step | Who | What it does | Why this way |
|---|---|---|---|
| 1. Parse | code | Reads any supported format into one timeline: each line gets an id, episode, start/end time, type (dialogue / action / scene / sound / camera), speaker | Timestamps come **only** from the script, never from the AI |
| 2. Tag | AI | Scores every line: `hook` 0-5, `cliff` 0-5, `needs_context` 0-5, moment type, short reason. Answers with **line ids only** | Small, checkable judgements; the AI cannot invent a time |
| 3. Candidates | code | Builds 20-60s clips that **start on a strong hook line** and **end on a strong cliffhanger line**, cut on whole lines, never across episodes; removes near-duplicates | Rules from the brief, fully explainable |
| 4. Stranger test | AI | A **fresh, blind** AI call per promo sees only that promo's lines (no show name, no story): what happens? what question is left? what is confusing? `standalone` 0-5 | Makes "makes sense on its own" measurable |
| 5. Rank | code | `score = (hook+1)(cliff+1)(standalone+1)`, then rules: early-episode bonus, spoiler guard, preferred 25-45s, variety in the top 10 | Every rank comes with a plain-English "why this rank" |
| App | human | Editor watches/reads each promo and clicks Approve / Reject (saved) | The AI suggests, the editor decides |

"Strong" hook/cliff lines are the **top X% of each show** (not a fixed score), because
different models use the 0-5 scale differently. That keeps results stable when the model is swapped.

---

## Quick start

Python 3.10+ (tested on 3.14, Windows 11).

```bash
python -m venv .venv
.venv/Scripts/python -m pip install -r requirements.txt        # macOS/Linux: .venv/bin/python
cp .env.example .env                                            # then paste your API key(s) into .env
.venv/Scripts/python src/llm.py --test                          # checks the key works
```

Free keys: Groq (https://console.groq.com/keys) and/or Gemini (https://aistudio.google.com/apikey).

### Run on a script, a folder of episodes, or many shows

```bash
.venv/Scripts/python src/run.py --input samples/kuku_style_show          # folder of episode files = one show
.venv/Scripts/python src/run.py --input path/to/script.txt               # one file = one show
.venv/Scripts/python src/run.py --input path/to/folder_of_shows          # each file / sub-folder = one show
```

Results land in `outputs/`:

| File | Content |
|---|---|
| `ranked_<show>.md` / `.csv` / `.json` | **The answer**: ranked promos with cuts, hook, cliffhanger, why it works, why this rank |
| `run_log_<show>.json` | model used, which model actually answered, AI calls, tokens, time per step, settings |
| `timeline_`, `beats_`, `candidates_`, `cold_<show>.json` | raw output of every step (unedited) |

### Swap the model (one line)

In `config.yaml` change `llm.model`, or override on the command line:

```bash
.venv/Scripts/python src/run.py --input samples/kuku_style_show --model gemini/gemini-3.8-flash
.venv/Scripts/python src/run.py --input samples/kuku_style_show --model openai/gpt-4.1          # needs OPENAI_API_KEY
.venv/Scripts/python src/run.py --input samples/kuku_style_show --model anthropic/claude-sonnet-5-5   # needs ANTHROPIC_API_KEY
```

Any [LiteLLM](https://docs.litellm.ai/docs/providers) model name works. All AI calls go through
`src/llm.py`, which validates JSON, retries, waits on rate limits, falls back to a backup model,
caches answers, and logs which model really answered.

### Editor app

```bash
.venv/Scripts/python -m streamlit run app.py
```

- **Review results**: promo cards in rank order; for YouTube sources the clip plays at the exact
  start/end; Approve / Reject + note, saved to `outputs/feedback_<show>.json`.
- **Run on a new script**: upload a script, pick a model, get ranked promos.

---

## Supported input formats (auto-detected)

| Format | Example |
|---|---|
| **Kuku screenplay** (the assignment's input pack) | `**MAYA (Desperate):**` / `Start: (00:13.50) \| End: (00:14.30)` / `"Why me?"`, plus `**CAMERA:**`, `**ACTION:**`, `**SOUND FX:**`, scene headings, episode logline header |
| Timestamped screenplay | `[00:12] BELLA: line`, `[00:16] (Victoria slaps Maya)`, `SCENE 2 - INT. ...`, `NAME (O.S.)` on its own line, ranges `[00:04.0 - 00:07.5]` |
| YouTube transcript copy | `1:031 minute, 3 secondsHey loser.` (as pasted from "Show transcript") |
| Plain timestamps / subtitles | `1:03 Hey loser.`, SRT / VTT |

Episodes: one file per episode (`ep01.txt`, `episode-001-...txt`) or `EPISODE n` lines inside one
file. Times may restart at 0:00 every episode; output cuts are always **episode + time in that
episode's script**.

`samples/` contains a small **synthetic** 3-episode screenplay (written for testing, not a real
show) so the tool can be tried without any private data.

---

## Checking quality

```bash
.venv/Scripts/python eval/verify_outputs.py <input>          # independent check of every timestamp vs the raw script
.venv/Scripts/python eval/baseline_keywords.py <input>       # no-AI keyword baseline, same candidate rules
.venv/Scripts/python eval/evaluate.py <input>                # precision@10, recall, cut-point error vs an answer key
```

- **`verify_outputs.py`** re-reads the original files (without the parser) and checks that every
  start time, hook and cliffhanger is really in the script, lengths are 20-60s, cuts don't spill
  into the next line, and the stranger never saw timestamps or the show name. It was itself tested
  by planting errors (all caught).
- **`evaluate.py`** compares the ranked list with an answer key (`data/my_picks.txt`, format in the
  file's docstring) and with the keyword baseline, and lists misses and false alarms.

---

## Key settings (`config.yaml`)

| Setting | Value | Reason |
|---|---|---|
| `llm.model` | `groq/openai/gpt-oss-120b` | free tier with ~1,000 requests/day (Gemini Flash free = 20/day) |
| `promo.min/max_seconds` | 20 / 60 | the brief |
| `promo.selection` | `relative`, top 10% (widens to 25% if too few) | model-independent |
| `promo.allow_joins` | `false` | a held-out spot check showed joined clips mostly failed (1 good of 13) |
| `ranking.*` | early bonus x1.05, spoiler guard x0.9, 25-45s preferred | posting strategy; all editable |

---

## Known limitations

- The AI reads **text**: a moment that is only visual and not described in the script can be missed.
  Kuku screenplays describe actions and shots, so this matters far less than on auto-captions.
- AI scores vary slightly run to run; the cache makes a saved run reproducible.
- Free tiers are slow (rate-limit waits); a paid key removes the waiting.
- The stranger test is still an AI imitating a viewer, not a real audience.

---

## Project layout

```
src/        adapter.py (formats) · parser.py (timeline, episodes) · tagger.py (AI scores)
            candidates.py (clips) · cold_viewer.py (stranger test) · ranker.py · llm.py (model wrapper) · run.py
eval/       verify_outputs.py · baseline_keywords.py · evaluate.py · heldout_config.yaml · heldout_spotcheck.txt
samples/    synthetic screenplay for trying the tool
app.py      Streamlit editor app
config.yaml settings (model in one line)
ai_notes.txt  AI tools used + places where AI (and my design) got it wrong
```

**Data policy:** scripts, transcripts, the Kuku input pack, AI caches and outputs are **not** in this
repository (`data/`, `cache/`, `outputs/` are git-ignored). The Kuku material is confidential.
