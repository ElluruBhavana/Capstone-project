# Agentic incident response

Two LLM agents investigate and repair incidents on a monitored service, built on the tools from the project slide.

| Tool | Used for |
| --- | --- |
| **Prometheus** | Scrapes the service. Alert rules start incidents. The Fixer verifies recovery against it. |
| **OpenSearch (ELK)** | Stores and searches the service logs. |
| **Ansible** | Idempotent remediation playbooks, dry-run first. |
| **Model Context Protocol** | Four MCP servers expose every tool to the agents. |
| **LangChain** | Tool-calling agent loop, prompts, chat models. |
| **ChromaDB** | RAG over incident runbooks and past resolutions. Outcomes are written back. |
| **Llama 3 / DeepSeek-R1** | Local LLMs in Ollama: tool use, root-cause analysis, fix planning. |

```
Prometheus alert -> Investigator (MCP tools, RAG, LLM) -> confidence -> policy -> Fixer (LLM plan, Ansible) -> verify in Prometheus
```

- **Agent 1, the Investigator**, collects evidence through MCP tools (metrics, logs, runbooks, similar incidents) and writes a root-cause analysis. Confidence is `stated × agreement × grounding`, so a model that wavers or skipped the logs scores low.
- **Agent 2, the Fixer**, plans a playbook from the diagnosis and the retrieved runbooks, dry-runs it, runs it, and checks that the alert cleared. If it did not, it plans again.
- A **deterministic policy** between them decides whether the Fixer may act, must ask a human, or must abstain. The model cannot override it.

Details: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/SAFETY.md](docs/SAFETY.md).

## What you need

1. **Docker Desktop**, running. It hosts Prometheus, OpenSearch, the monitored service, and the agents (Ansible only runs on Linux).
2. **Ollama** on the host, with these models:

```powershell
ollama pull llama3.1:8b
ollama pull nomic-embed-text
ollama pull deepseek-r1:8b      # optional. With REASONING_MODEL=auto it is used for root-cause analysis when present
```

About 8 GB of free RAM is a comfortable minimum for OpenSearch plus the models.

## Run it

From the repository root, in PowerShell:

```powershell
docker compose up --build
```

The first build downloads images and takes a while. When the logs settle, open:

| URL | What |
| --- | --- |
| http://localhost:8000 | The operations console |
| http://localhost:8000/docs | API docs |
| http://localhost:9090 | Prometheus |
| http://localhost:9200/app-logs/_search | OpenSearch logs |

To browse logs in OpenSearch Dashboards as well: `docker compose --profile dashboards up --build`, then http://localhost:5601.

## Demo

1. Open **Simulation Lab** and press **Memory leak** (or any fault).
2. Within about 25 seconds Prometheus fires an alert. With auto-monitor on, an incident opens by itself.
3. Open the incident. Watch the **Agent trace** fill in: MCP tool calls, three root-cause samples, the confidence formula, the policy decision, the Ansible dry run and real run, and the Prometheus verification.
4. Try **Disk pressure**. Its only fix deletes files, so the policy asks for human approval. Press **Approve** (needs the operator token, default `demo-operator`).
5. Open **Stack & Tools** to see every tool on the slide with its live status.

If a tool shows "not ready", the console says which one and why.

## Configuration

Copy `.env.example` to `.env`. Everything is optional. Notable settings:

| Variable | Default | Meaning |
| --- | --- | --- |
| `TOOL_MODEL` | `llama3.1:8b` | Drives MCP tool calls and plans the fix |
| `REASONING_MODEL` | `auto` | `auto` uses `deepseek-r1:8b` if pulled, otherwise `llama3.1:8b` |
| `RCA_SAMPLES` | `3` | Independent root-cause samples used for the agreement term |
| `HIGH_THRESHOLD` / `LOW_THRESHOLD` | `0.80` / `0.50` | Act on its own at or above high. Abstain below low. |
| `AUTO_MONITOR` | `true` in Docker | Open an incident automatically for each new alert |

If the agents container cannot reach Ollama on Windows, set `OLLAMA_HOST=0.0.0.0` for Ollama and restart it.

## Tests

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m pytest
```

The tests use fake LLMs and a fake toolbox, so they need no Ollama, Docker or network. They cover the fault model, the confidence maths, the policy, the Ansible executor (argument lists only, no shell), the Prometheus and OpenSearch clients, the whole Investigator to Fixer pipeline including replanning, and the API.

## Layout

- `backend/agents/` Investigator, Fixer, orchestrator
- `backend/mcp_servers/` the four MCP servers
- `backend/rag/` ChromaDB knowledge base
- `backend/policy.py`, `backend/confidence.py` the deterministic gate and the confidence maths
- `ansible/` playbooks and the allowlist catalogue
- `monitoring/` Prometheus config and alert rules
- `target/` the monitored, fault-injectable service
- `knowledge/` runbooks and past incidents for RAG
- `frontend/` React console
- `_legacy_ml/` the previous scikit-learn prototype, kept for reference. Safe to delete.

## What has been verified

| Verified | How |
| --- | --- |
| 109 automated tests pass | `pytest`: fault model, confidence, policy, Ansible executor, clients, MCP result parsing, RAG on real ChromaDB, the whole agent pipeline, the API |
| Four MCP servers over stdio, 12 tools | Started for real; the allowlist refuses `drop_database`; outages return errors as data |
| ChromaDB with real `nomic-embed-text` embeddings | 33 runbook sections and 12 past incidents ingested and retrieved correctly |
| Whole pipeline with real Llama 3.1 8B | Cache stampede injected into the real target service. 3 of 3 samples agreed, confidence 0.80, policy allowed it, dry run, real run, Prometheus confirmed recovery. Resolved in 75 s |
| Wrong diagnosis is contained | An earlier run misdiagnosed (2 of 3 samples said CPU, confidence 0.55). The cheap playbook changed nothing, verification caught it, and the risky restart was held for human approval |
| React console builds under strict TypeScript | `npm run build` |

**Not verified:** the Docker Compose stack itself (Docker was not installed here), real Prometheus, real OpenSearch and real Ansible. In the live runs, small stand-ins for those three read the live target service and performed what the playbooks do. Expect small fixes on your first `docker compose up`.

**Known behaviour:** Llama 3.1 8B writes tool calls as text instead of making them, so the Investigator's fallback fetches the evidence directly. The trace says so. A stronger tool-calling model would use the loop as intended. Results vary run to run, and the confidence formula is what keeps a wavering run from acting.
