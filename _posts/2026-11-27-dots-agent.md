---
permalink: /projects/dots-agent/
title: "dots-agent: an LLM video pipeline with human approval gates and an adversarial fact-checker"
description: "A LangGraph pipeline that researches, scripts, fact-checks, voices and renders explainer videos, with two human gates run from Telegram."
type: project
tags: [python, langgraph, llm, human-in-the-loop, telegram, remotion]
abstract: >
  I wanted to make short "connecting the dots" explainer videos on economics and world affairs without letting a language
  model publish unchecked claims. dots-agent is a LangGraph pipeline (research, script, critic, voice, visuals, render) that
  stops twice for a human decision in Telegram, tags every sentence as sourced fact or opinion, and routes doubtful claims
  to me through a claim ledger. It shows how I handle orchestration state, structured LLM output, retries and cost control.
---

## The problem

An explainer video that links a few separate developments ("this rate decision, this commodity price, this election") into one story is mostly research and writing. Language models are good at both and also good at sounding certain about things they made up. In finance and politics a confident wrong number or an opinion stated as fact is the worst possible failure.

So the goal was not "generate a video". It was: let the model do the slow work (finding sources, drafting, cutting Shorts, building charts) while a person keeps control of the two decisions that matter, which story to tell and which claims survive. Everything runs from a Telegram chat on my own machine, so a review takes a few taps on a phone.

## Architecture

<div class="diagram">
<svg viewBox="0 0 740 410" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Pipeline: radar or thesis outline, then a human gate, then research, script, critic, a second human gate for claim review, voice, visuals, render and package. A Telegram bot drives the graph and SQLite stores checkpoints and metrics.">
  <defs><marker id="dots-agent-a1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead"/></marker><marker id="dots-agent-a2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead-accent"/></marker></defs>
  <text x="20" y="26" class="dg-label">Two entry flows</text>
  <rect x="20" y="40" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="64" text-anchor="middle" class="dg-text">RSS radar</text>
  <text x="90" y="82" text-anchor="middle" class="dg-sub">ranked angles</text>
  <rect x="20" y="130" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="154" text-anchor="middle" class="dg-text">Thesis outline</text>
  <text x="90" y="172" text-anchor="middle" class="dg-sub">my notes + clippings</text>
  <rect x="190" y="40" width="160" height="56" rx="8" class="dg-box-accent"/>
  <text x="270" y="64" text-anchor="middle" class="dg-text">Gate: pick angle</text>
  <text x="270" y="82" text-anchor="middle" class="dg-sub">or regenerate</text>
  <rect x="190" y="130" width="160" height="56" rx="8" class="dg-box-accent"/>
  <text x="270" y="154" text-anchor="middle" class="dg-text">Gate: outline</text>
  <text x="270" y="172" text-anchor="middle" class="dg-sub">approve / revise / cancel</text>
  <rect x="380" y="85" width="160" height="56" rx="8" class="dg-box"/>
  <text x="460" y="109" text-anchor="middle" class="dg-text">Research</text>
  <text x="460" y="127" text-anchor="middle" class="dg-sub">web search, FRED, WB</text>
  <rect x="570" y="85" width="150" height="56" rx="8" class="dg-box"/>
  <text x="645" y="109" text-anchor="middle" class="dg-text">Script</text>
  <text x="645" y="127" text-anchor="middle" class="dg-sub">fact / inference tags</text>
  <rect x="570" y="220" width="150" height="56" rx="8" class="dg-box"/>
  <text x="645" y="244" text-anchor="middle" class="dg-text">Critic</text>
  <text x="645" y="262" text-anchor="middle" class="dg-sub">claim ledger</text>
  <rect x="380" y="220" width="160" height="56" rx="8" class="dg-box-accent"/>
  <text x="460" y="244" text-anchor="middle" class="dg-text">Gate: claim review</text>
  <text x="460" y="262" text-anchor="middle" class="dg-sub">keep / suggestion / cut</text>
  <rect x="190" y="220" width="160" height="56" rx="8" class="dg-box"/>
  <text x="270" y="244" text-anchor="middle" class="dg-text">Voice</text>
  <text x="270" y="262" text-anchor="middle" class="dg-sub">TTS + word timings</text>
  <rect x="20" y="220" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="244" text-anchor="middle" class="dg-text">Visuals</text>
  <text x="90" y="262" text-anchor="middle" class="dg-sub">charts, maps, B-roll</text>
  <rect x="20" y="320" width="140" height="56" rx="8" class="dg-box"/>
  <text x="90" y="344" text-anchor="middle" class="dg-text">Render</text>
  <text x="90" y="362" text-anchor="middle" class="dg-sub">Remotion + ffmpeg</text>
  <rect x="190" y="320" width="160" height="56" rx="8" class="dg-box"/>
  <text x="270" y="344" text-anchor="middle" class="dg-text">Package + upload</text>
  <text x="270" y="362" text-anchor="middle" class="dg-sub">metadata, YouTube API</text>
  <rect x="395" y="305" width="335" height="90" rx="10" class="dg-group"/>
  <text x="718" y="322" text-anchor="end" class="dg-label">Control and state</text>
  <rect x="410" y="330" width="140" height="52" rx="8" class="dg-box"/>
  <text x="480" y="352" text-anchor="middle" class="dg-text">Telegram bot</text>
  <text x="480" y="369" text-anchor="middle" class="dg-sub">invoke / resume</text>
  <rect x="570" y="330" width="150" height="52" rx="8" class="dg-box"/>
  <text x="645" y="352" text-anchor="middle" class="dg-text">SQLite</text>
  <text x="645" y="369" text-anchor="middle" class="dg-sub">checkpoints, metrics</text>
  <line x1="160" y1="68" x2="190" y2="68" class="dg-line" marker-end="url(#dots-agent-a1)"/>
  <line x1="160" y1="158" x2="190" y2="158" class="dg-line" marker-end="url(#dots-agent-a1)"/>
  <line x1="350" y1="68" x2="380" y2="100" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="350" y1="158" x2="380" y2="126" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="540" y1="113" x2="570" y2="113" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="645" y1="141" x2="645" y2="220" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="570" y1="248" x2="540" y2="248" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="380" y1="248" x2="350" y2="248" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="190" y1="248" x2="160" y2="248" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="90" y1="276" x2="90" y2="320" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="160" y1="348" x2="190" y2="348" class="dg-line-accent" marker-end="url(#dots-agent-a2)"/>
  <line x1="480" y1="330" x2="460" y2="276" class="dg-line-dashed" marker-end="url(#dots-agent-a1)"/>
</svg>
<p class="diagram-caption">Figure 1. The LangGraph pipeline. Highlighted boxes are the points where the graph pauses for a human decision; the Telegram bot resumes it.</p>
</div>

The pipeline is a LangGraph `StateGraph` with twelve nodes. Every stage lives in its own module under `dots/stages/` and writes a typed pydantic artifact as JSON into `runs/<id>/`, so each step can be inspected and cached on its own.

There are two ways in. The news flow scans a list of RSS feeds from the last 48 hours, deduplicates headlines and asks a model for ranked angles; I pick one in Telegram or ask for new ones. The thesis flow, the one I use most, starts from my own notes plus links I forwarded to the bot during the week (article text extracted with trafilatura). The model turns that into a one-page outline with beats, the strongest counterargument and scenarios with rough odds and signposts. I approve it, send revision notes (which loop back to the outline node) or cancel.

After that the path is shared:

- **Research** returns a fact pack: each fact has a claim, source URL, publisher, date and snippet. Thesis episodes search only a curated list of domains (official data, major outlets, think tanks) from `config.yaml`. The model can *request* chart series by FRED series ID or World Bank indicator, and code fetches the numbers from those APIs.
- **Script** writes sentences, each tagged `fact` (with fact IDs) or `inference`, plus a visual hint and Shorts cut points.
- **Critic** reviews the script against the fact pack and returns a claim ledger: verdict (keep, soften, cut), confidence, reason, a competing explanation, suggested rewrite and risk flags.
- **Claim review** is the second human gate. I get the full script as a file plus one Telegram message per doubtful sentence with Keep / Use suggestion / Cut buttons, or I reply with my own text.
- **Voice** calls ElevenLabs with timestamps, so word timings drive captions and sentence timings drive scene cuts.
- **Visuals** builds a Remotion timeline: FRED/World Bank charts, maps, stat cards, an animated "dots board", and stock B-roll from Pexels/Pixabay with each licence logged.
- **Render** shells out to Remotion for the 16:9 video, 9:16 Shorts and thumbnails, then normalises loudness with ffmpeg. **Package** writes titles, a description with sources and disclosures, and chapters; an optional step uploads to YouTube.

The render side is a small TypeScript/React project (Remotion) that only reads the timeline JSON; it has no logic of its own about content.

## Key design decisions

**LangGraph `interrupt()` with a SQLite checkpointer, instead of a hand-written state machine.** The gates are plain `interrupt()` calls inside nodes, and the bot resumes them with `Command(resume=...)` keyed by run ID as the thread ID. Because state is checkpointed after every node, a crash in rendering does not lose a paid-for research pass: `/retry <run>` re-invokes the graph and it continues from the failed node. I also added `/redo <run> <stage>`, which uses `update_state(..., as_node=...)` to rewind a run to an earlier stage and deletes only the cached files downstream of it. The cost is a dependency on LangGraph's checkpoint semantics, which I covered with offline tests that drive the whole graph through both gates.

**Forced tool calls for structured output, plus a validate-and-repair loop.** Every structured answer is requested through a single `submit` tool whose input schema is the pydantic model (with `$ref`s inlined because one provider rejects them). The alternative, parsing JSON from free text, broke on truncated or fenced output. When the answer is cut off, fails validation, or fails a semantic check such as "fewer than 10 sourced facts", the loop re-asks with the specific error. There are fallbacks for models that reject forced `tool_choice`: auto tool choice with an instruction, then JSON mode.

**Two-phase research so a cheaper model can own a stage.** Phase A uses Anthropic's server-side web search to write cited notes as plain text; phase B has whichever model is configured for that stage structure the notes into the schema. That lets me run outline and script on DeepSeek and keep research and critic on Claude, set per stage in `config.yaml`. Without a DeepSeek key everything falls back to Claude. The search notes are cached per run and per prompt, so a failed structuring pass does not pay for the search again.

**Facts and numbers are separated from the model's prose.** A script sentence marked `fact` that cites no fact ID, or an ID not in the pack, is demoted to `inference`, and with `always_review_inferences` on, the critic's reviewed inferences go to me. Chart data is never written by the model: `ResearchDraft` has no datasets field at all, and before a chart is placed behind a sentence, `chart_matches` checks that any figure the sentence quotes appears in the series within 3%. Otherwise the scene falls back to a stat card or ambient motion.

**A critic from a different model family, told to disagree.** The critic's system prompt rewards catching errors, and the prompt assumes the writer is biased toward the thesis. In the current config the script comes from DeepSeek and the critic is Claude, so the check is not a model grading its own work. Only items that are not a clean "keep", carry a flag, fall under the confidence threshold (0.75) or are inferences reach me, sorted by lowest confidence first; everything else passes without my attention.

## Inside the code

The review gate. The critic's ledger is filtered down to what needs a human, the graph pauses, and the decisions that come back from Telegram are applied to the script:

```python
# dots/stages/critic.py: what reaches the human
for r in ledger.reviews:
    s = by_id.get(r.sentence_id)
    if not s:
        continue
    if (r.verdict != "keep" or r.flags or r.confidence < th
            or (always_inf and s.kind == "inference")):
        out.append({"sentence_id": s.id, "kind": s.kind, ...})
out.sort(key=lambda x: x["confidence"])

# dots/graph.py: the node pauses until the bot resumes it
def n_review(s: State) -> State:
    # … load script and ledger
    items = critic.items_for_review(sc, led)
    resp = (interrupt({"type": "review", "items": items, "script": s["script"]})
            if items else {"decisions": {}})
    decisions = resp.get("decisions", {})
    # … log each decision next to the critic's verdict in metrics.db
    final = critic.apply_decisions(sc, decisions, suggestions)
```

Logging the critic's verdict next to my decision in `metrics.db` gives me data on how often I overrule the critic, which is the obvious way to tune the threshold later.

The repair loop that every LLM call goes through. The budget is checked before each attempt, and each retry carries the concrete reason the last answer was rejected:

```python
for _ in range(retries + 2 if check else retries + 1):
    _budget_check(run_id, stage)
    data, stop = structured(run_id, stage, system, prompt,
                            schema_json, st["model"])
    if stop == "max_tokens":
        prompt = user + "\n\nYour previous answer was cut off ..."
        continue
    try:
        result = schema.model_validate(data)
    except ValidationError as e:
        prompt = (user + "\n\nYour previous answer failed validation:\n"
                  f"{str(e)[:1500]}\nPrevious answer:\n"
                  f"{json.dumps(data)[:6000]}\nFix it.")
        continue
    problem = check(result) if check else None
    if not problem:
        return result
    # … same repair prompt, carrying the semantic problem
raise RuntimeError(f"{stage}: could not get valid output: {last_err}")
```

Around this sit the other reliability pieces: an `httpx` wrapper with exponential backoff on 429/5xx that honours `Retry-After`; B-roll downloads that are rejected when the byte count does not match `Content-Length`; a stage cache keyed by a hash of each stage's input, so voice is only regenerated (and paid for) when the narration text changes; review taps persisted to disk so a bot restart mid-review loses nothing; and a YouTube upload that records each finished part, so tapping the button again skips what is already uploaded. Every LLM call and TTS request is priced from a table in config and written to SQLite, which is what the per-run dollar cap reads.

## Running it

```bash
pip install -r requirements.txt
cd render && npm install && npx remotion browser ensure && cd ..
cp .env.example .env        # fill in API keys and the allowed Telegram chat
python -m dots doctor       # checks env vars and makes live calls to each API
python -m dots sample       # renders a 12 s offline sample, no API calls
python -m pytest -q         # offline tests (28 pass)
python -m dots bot          # long-polling Telegram controller
```

## What I'd do next

- **Give the critic web search.** The README says the critic spot-checks claims on the web, but the code does not: it only checks the script against the research pack. A claim that is wrong in the pack itself can only be caught by me.
- **Enforce critic coverage.** The prompt asks for a review of every inference, but the code only routes sentences the critic actually returned. A sentence the critic skipped is kept without reaching me. The fix is to treat any unreviewed inference as a review item.
- **Gate TTS on the budget.** The cap is checked before LLM calls only. ElevenLabs spend is logged and counted, but voice generation does not check the cap first, and a single LLM call can overshoot it.
- **Score the prediction ledger.** Approved scenarios are saved with odds, signposts and horizons, but nothing scores them yet.
- **Offload rendering.** Rendering is local; the README mentions Cloud Run or Remotion Lambda as a fallback, but that is not built.
- The bot is single-user by design (one allowed chat ID), and the graph runs in a thread from the bot process, so there is no queue for multiple concurrent runs.

## Skills this shows

- **AI engineering:** LangGraph graph with conditional edges, `interrupt()` gates and checkpoint rewind; schema-forced tool output with validate-and-repair; per-stage model routing across two providers; two-phase search-then-structure research.
- **Responsible AI:** fact/inference tagging with automatic demotion of uncited facts, a separate-family adversarial critic, a human claim-review gate, chart numbers only from official APIs, synthetic-voice and disclaimer text added to every description.
- **Backend reliability:** retries with backoff and `Retry-After`, truncation checks, input-hash stage caching, idempotent resumable uploads, error reporting from the bot with full tracebacks to a log file.
- **Cost control and observability:** per-call cost accounting in SQLite, a per-run budget cap, cached paid searches, stage timings and review decisions recorded for later analysis.
- **Integration:** Anthropic, DeepSeek (OpenAI-compatible), FRED, World Bank, ElevenLabs, Pexels/Pixabay, YouTube Data API with OAuth, Telegram, and a TypeScript Remotion renderer driven by JSON.
- **Testing:** 28 offline pytest tests with fake API clients, including a full graph run through both gates and a `/redo` rewind.

The code is private; I'm happy to walk through it in an interview.
