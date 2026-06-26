# n00dles 🍜

> **Multi-agent AI orchestration for engineers who ship.**  
> Your LLM pipelines are spaghetti. n00dles fixes that.

[![Official Website](https://i.ibb.co/WvsxSx4T/n00dles-Multi-Agent-AI-Orchestration-Untangle-Your-LLM-Pipelines-06-26-2026-09-54-AM.png)](https://www.n00dles.com)
[![PyPI](https://img.shields.io/pypi/v/get-n00dles.svg)](https://pypi.org/project/get-n00dles/)

```python
from n00dles import agent, pipeline, run

@agent(model="claude-sonnet-4-6")
def researcher(topic: str) -> str:
    """Research the given topic and return key facts."""

@agent(model="claude-sonnet-4-6")
def writer(research: str) -> str:
    """Write a compelling article from the research."""

@agent(model="claude-sonnet-4-6")
def editor(draft: str) -> str:
    """Review and improve the article for clarity."""

# Chain agents with >>
content_pipeline = pipeline(researcher >> writer >> editor, retry=3)

# Run it
result = run(content_pipeline, topic="AI orchestration in 2026")
print(result.output)
```

---

## Why n00dles?

Building multi-agent AI systems should not require reading 800 lines of framework source code to understand why your pipeline silently returned `None` at 2am.

We built n00dles because every tool we tried made us fight the framework instead of shipping the product. It started as a weekend fix and turned into the orchestration layer we wished had existed.

**n00dles is:**
- **Simple** — first agent pipeline in 10 lines, not 120
- **Reliable** — retry, timeout, and circuit breaking built in at every node
- **Transparent** — no magic, no hidden state, full traces on every run
- **Production-ready** — persistent state, parallel execution, one-command deploy
- **Provider-agnostic** — Anthropic, OpenAI, Mistral, Gemini, Ollama, any OpenAI-compatible endpoint

---

## Install

```bash
pip install n00dles
```

**Requirements:** Python 3.10+ · No mandatory cloud dependencies · Self-hostable

Set your provider key:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
# or
export OPENAI_API_KEY=sk-...
```

---

## Core concepts

### Agents

An agent is any Python function decorated with `@agent`. The docstring becomes the system prompt. Type annotations define the I/O contract.

```python
from n00dles import agent
from pydantic import BaseModel

class ResearchResult(BaseModel):
    summary: str
    sources: list[str]
    confidence: float

@agent(model="claude-sonnet-4-6", temperature=0.3)
def researcher(topic: str) -> ResearchResult:
    """
    Research the given topic thoroughly.
    Return a summary, your sources, and a confidence score 0–1.
    """
```

### Pipelines

Chain agents with `>>` for sequential flow, `|` for parallel, and `branch()` for conditional routing.

```python
from n00dles import agent, pipeline, branch, run

# Sequential
sequential = pipeline(fetch >> clean >> analyze >> report)

# Parallel — runs fetch_news and fetch_prices simultaneously, merges results
parallel = pipeline(
    (fetch_news | fetch_prices) >> merge >> analyze
)

# Conditional routing
routed = pipeline(
    classify >> branch(
        positive=sentiment_writer,
        negative=crisis_responder,
        neutral=standard_writer
    )
)
```

### Retry and resilience

Every node gets retry logic by default. Configure per-agent or per-pipeline:

```python
@agent(
    model="claude-sonnet-4-6",
    retry=5,                        # max attempts
    timeout=30,                     # seconds per attempt
    backoff="exponential",          # linear | exponential | jitter
    fallback=simple_summarizer      # agent to call if all retries fail
)
def analyst(data: str) -> str:
    """Analyze the data and return key insights."""
```

### Persistent state

Pipelines survive restarts. State is checkpointed after every node.

```python
from n00dles import pipeline, run, StateBackend

# SQLite for local dev
p = pipeline(fetch >> process >> store, state=StateBackend.sqlite("./runs.db"))

# Redis for production
p = pipeline(fetch >> process >> store, state=StateBackend.redis("redis://localhost:6379"))

# Resume any previous run from where it stopped
result = run(p, run_id="run_abc123", resume=True)
```

### Parallel execution

Run independent agents simultaneously with `|`. n00dles handles fan-out, fan-in, and result merging:

```python
@agent(model="claude-sonnet-4-6")
def merge_reports(news: str, prices: str, sentiment: str) -> str:
    """Combine the three reports into one executive summary."""

market_pipeline = pipeline(
    (fetch_news | fetch_prices | fetch_sentiment) >> merge_reports
)
```

---

## Deployment

### Local

```bash
noodles run pipeline.py --input '{"topic": "AI trends"}'
```

### As a serverless function

```bash
noodles deploy pipeline.py --target aws-lambda
noodles deploy pipeline.py --target gcp-functions
noodles deploy pipeline.py --target fly
```

### As a long-running worker

```bash
noodles serve pipeline.py --port 8080
```

### Docker

```bash
noodles build pipeline.py --output Dockerfile
docker build -t my-pipeline .
docker run -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY my-pipeline
```

---

## Provider support

| Provider | Model strings | Status |
|---|---|---|
| Anthropic | `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5` | ✅ Supported |
| OpenAI | `gpt-4o`, `gpt-4o-mini`, `o3` | ✅ Supported |
| Mistral | `mistral-large`, `mistral-small` | ✅ Supported |
| Google | `gemini-2.0-flash`, `gemini-2.0-pro` | ✅ Supported |
| Ollama | Any local model | ✅ Supported |
| OpenAI-compatible | Custom `base_url` | ✅ Supported |

Switch providers with one line:

```python
@agent(model="gpt-4o")           # OpenAI
@agent(model="mistral-large")    # Mistral
@agent(model="ollama/llama3.2")  # Local via Ollama
```

---

## Observability

Every run produces structured traces. Export to any OpenTelemetry backend, or use our built-in dashboard:

```bash
noodles dashboard
# Opens http://localhost:4000 — live pipeline view, per-agent latency, token costs
```

Integrate with your existing stack:

```python
from n00dles import pipeline
from n00dles.telemetry import LangfuseExporter, HeliconeExporter

p = pipeline(
    researcher >> writer >> editor,
    telemetry=LangfuseExporter(public_key="...", secret_key="...")
)
```

---

## Testing pipelines without spending tokens

Mock any agent or the entire LLM layer in tests:

```python
from n00dles.testing import mock_agent, MockLLM

def test_content_pipeline():
    with mock_agent(researcher, returns="AI is transforming software development."):
        with mock_agent(writer, returns="Here is a well-written article..."):
            result = run(content_pipeline, topic="AI")
            assert result.output is not None
            assert len(result.output) > 100

# Or mock the entire LLM layer
def test_with_mock_llm():
    with MockLLM(response="Mocked response"):
        result = run(content_pipeline, topic="AI")
        assert result.success
```

---

## Comparison

|  | n00dles | LangChain | CrewAI | LangGraph |
|---|:---:|:---:|:---:|:---:|
| Lines to first pipeline | **~10** | ~120 | ~70 | ~90 |
| Built-in retry / timeout | ✅ | ❌ | ❌ | ⚠️ |
| Persistent state | ✅ | ❌ | ❌ | ✅ |
| Parallel execution | ✅ | ⚠️ | ✅ | ✅ |
| Type-safe I/O contracts | ✅ | ❌ | ⚠️ | ⚠️ |
| Visual pipeline debugger | ✅ | ❌ | ❌ | ❌ |
| One-command deploy | ✅ | ❌ | ❌ | ❌ |
| Zero-token pipeline testing | ✅ | ⚠️ | ❌ | ⚠️ |
| Provider agnostic | ✅ | ✅ | ⚠️ | ✅ |
| MIT licensed | ✅ | ✅ | ✅ | ✅ |

---

## Real-world use cases

<details>
<summary><b>🔍 Deep research pipeline</b></summary>

```python
from n00dles import agent, pipeline, run

@agent(model="claude-sonnet-4-6")
def web_scraper(query: str) -> str:
    """Search the web and return raw content for the query."""

@agent(model="claude-sonnet-4-6")
def summarizer(content: str) -> str:
    """Distill the content into the 5 most important facts."""

@agent(model="claude-sonnet-4-6")
def analyst(summary: str) -> str:
    """Analyze the facts and identify strategic implications."""

@agent(model="claude-sonnet-4-6")
def report_writer(analysis: str) -> str:
    """Write a concise executive report from the analysis."""

research_pipeline = pipeline(
    web_scraper >> summarizer >> analyst >> report_writer,
    retry=3,
    timeout=45
)

result = run(research_pipeline, query="DeFi lending market Q2 2026")
print(result.output)
```
</details>

<details>
<summary><b>📄 Document intelligence at scale</b></summary>

```python
from n00dles import agent, pipeline, run
from pydantic import BaseModel

class KYCResult(BaseModel):
    name: str
    dob: str
    id_number: str
    risk_score: float
    flags: list[str]

@agent(model="claude-sonnet-4-6")
def document_reader(pdf_path: str) -> str:
    """Extract all text and data from the identity document."""

@agent(model="claude-sonnet-4-6")
def kyc_extractor(document_text: str) -> KYCResult:
    """Extract structured KYC data from the document text."""

@agent(model="claude-sonnet-4-6")
def risk_assessor(kyc: KYCResult) -> KYCResult:
    """Assess risk and add flags based on KYC data."""

kyc_pipeline = pipeline(
    document_reader >> kyc_extractor >> risk_assessor,
    state=StateBackend.redis("redis://localhost:6379")
)

# Process 1000 documents in parallel
import asyncio
results = asyncio.run(
    run.batch(kyc_pipeline, inputs=[{"pdf_path": p} for p in pdf_paths])
)
```
</details>

<details>
<summary><b>🔮 Multi-agent prediction system</b></summary>

```python
from n00dles import agent, pipeline, branch, run

@agent(model="claude-sonnet-4-6")
def news_scraper(market: str) -> str:
    """Scrape latest news for the given market."""

@agent(model="claude-sonnet-4-6")
def sentiment_analyzer(news: str) -> str:
    """Score market sentiment from -1.0 (bearish) to 1.0 (bullish)."""

@agent(model="claude-sonnet-4-6")
def price_fetcher(market: str) -> str:
    """Fetch current price data and recent trend for the market."""

@agent(model="claude-sonnet-4-6")
def signal_aggregator(sentiment: str, prices: str) -> str:
    """Combine sentiment and price signals into a trade recommendation."""

prediction_pipeline = pipeline(
    (
        (news_scraper >> sentiment_analyzer)
        |
        price_fetcher
    ) >> signal_aggregator
)

result = run(prediction_pipeline, market="BTC/USDT")
```
</details>

---

## Architecture

```
┌─────────────────────────────────────────────┐
│              YOUR CODE                       │
│    @agent functions · pipeline() · run()     │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│            ORCHESTRATION LAYER               │
│  Pipeline executor · State machine           │
│  Retry engine · Branch router · Scheduler    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│               RUNTIME                        │
│  Async worker pool · State store             │
│  Trace collector · Token budget manager      │
└──────────┬───────────────────┬──────────────┘
           │                   │
┌──────────▼──────┐   ┌────────▼──────────────┐
│   LLM LAYER     │   │    DEPLOY TARGETS       │
│  Anthropic      │   │  AWS Lambda · Docker    │
│  OpenAI         │   │  Fly.io · Railway       │
│  Mistral        │   │  Self-hosted worker     │
│  Ollama         │   └───────────────────────┘
└─────────────────┘
```

---

## Project structure

```
n00dles/
├── n00dles/
│   ├── __init__.py          # Public API: agent, pipeline, run, branch
│   ├── core/
│   │   ├── agent.py         # @agent decorator and AgentNode class
│   │   ├── pipeline.py      # Pipeline definition and >> / | operators
│   │   ├── executor.py      # Async pipeline runner
│   │   ├── state.py         # State store backends (SQLite, Redis)
│   │   └── retry.py         # Retry, timeout, circuit breaker
│   ├── providers/
│   │   ├── anthropic.py
│   │   ├── openai.py
│   │   ├── mistral.py
│   │   └── ollama.py
│   ├── telemetry/
│   │   ├── tracer.py        # OpenTelemetry integration
│   │   ├── langfuse.py
│   │   └── helicone.py
│   ├── deploy/
│   │   ├── lambda_.py
│   │   ├── docker.py
│   │   └── worker.py
│   └── testing/
│       ├── mock_agent.py
│       └── mock_llm.py
├── tests/
├── examples/
│   ├── research_pipeline.py
│   ├── kyc_pipeline.py
│   ├── content_factory.py
│   └── prediction_system.py
├── docs/
├── pyproject.toml
├── LICENSE
└── README.md
```

---

## Contributing

We welcome contributions from everyone. The codebase is intentionally approachable — if you can read one file and understand what it does without reading three others first, we've done our job.

```bash
git clone https://github.com/n00dleshq/n00dles
cd n00dles
pip install -e ".[dev]"
pytest
```

**Before you open a PR:**
- Run `pytest` — all tests must pass
- Run `ruff check .` and `mypy n00dles/` — no new errors
- Add tests for new behavior
- Update docs if you change public API surface

---

## Roadmap

- [x] Sequential pipelines with `>>`
- [x] Parallel execution with `|`
- [x] Persistent state (SQLite + Redis)
- [x] Built-in retry / timeout / circuit breaker
- [x] Provider abstraction (Anthropic, OpenAI, Mistral, Ollama)
- [x] Structured traces + OpenTelemetry export
- [x] Zero-token pipeline testing
- [x] One-command deploy (Lambda, Docker, Fly)
- [ ] Visual pipeline debugger (web UI) — `v0.2`
- [ ] Streaming output from any agent — `v0.2`
- [ ] Human-in-the-loop approval gates — `v0.3`
- [ ] Pipeline versioning and A/B testing — `v0.3`
- [ ] Native RAG integration — `v0.4`
- [ ] Multi-modal agents (vision + audio) — `v0.4`
- [ ] n00dles Cloud (managed hosting) — `future`

---

## Community

| | |
|---|---|
| 📖 **Docs** | [docs.n00dles.com](https://n00dles.com/docs) |
| 🌐 **Website** | [n00dles.com](https://n00dles.com) |
| 🐛 **Issues** | [github.com/n00dlehouse](https://github.com/n00dlehouse) |
| 🔒 **Security** | contact@n00dles.com — we respond within 24 hours |

---

## The team

Built by [Al Dente](https://n00dles.com/about), [Ramen Dass](https://n00dles.com/about), and [13 more noodles](https://n00dles.com/about) across 9 countries.  
Everyone on the team has a noodle type. This is non-negotiable.

---

## License

MIT — do whatever you want with it. If you build something cool, tell us about it.

```
Copyright (c) 2026 n00dles contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
```

---

<div align="center">

**Made with 🍜 by the n00dles team**

[Website](https://n00dles.com) · [Docs](https://docs.n00dles.com) · [Discord](https://discord.gg/n00dles) · [Twitter](https://x.com/n00dles_dev)

*If this saved you from a 2am debugging session, consider starring the repo.*

[![Star History Chart](https://api.star-history.com/svg?repos=n00dleshq/n00dles&type=Date)](https://star-history.com/#n00dleshq/n00dles)

</div>
