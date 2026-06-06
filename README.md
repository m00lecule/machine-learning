# AIOps for Containerized Scientific Computing

A research reference for a doctoral thesis on continuous operational intelligence for containerized scientific workflows — using agentic AI to surface evidence-backed improvement suggestions to operators.

---

## Table of Contents

1. [What is AIOps](#what-is-aiops)
2. [Thesis Concept](#thesis-concept)
3. [Architecture](#architecture)
4. [Anomaly Detection](#anomaly-detection)
5. [Diagnosis Agent & RAG](#diagnosis-agent--rag)
6. [Recommendation Engine](#recommendation-engine)
7. [Self-Improvement Loop](#self-improvement-loop)
8. [Observability Stack](#observability-stack)
9. [PhD Research Questions](#phd-research-questions)
10. [Key Tools & Frameworks](#key-tools--frameworks)
11. [Further Reading](#further-reading)

---

## What is AIOps

AIOps (AI for IT Operations) applies machine learning and AI to automate the detection, diagnosis, and remediation of operational issues. Current tools (Dynatrace, Datadog Watchdog, New Relic AI) detect anomalies well but produce shallow alerts — a threshold breach triggers a notification with no reasoning behind it.

The gap this thesis addresses: **no existing AIOps system continuously reasons across heterogeneous HPC/container signals, explains the root cause in context, and surfaces prioritized, evidence-backed improvement suggestions to the operator.**

| Existing AIOps | This Thesis |
|---|---|
| Rule-based thresholds | LLM reasoning over multi-signal context |
| Cloud-native only | HPC-aware (Slurm, PBS, MPI, Singularity) |
| Static runbooks | RAG over runbooks + growing incident memory |
| Alerts what happened | Explains why and suggests what to do next |
| No learning | Suggestion quality improves over time |

---

## Thesis Concept

> *A continuous operational intelligence framework for containerized scientific workflows that ingests monitoring telemetry, diagnoses anomalies via RAG over operational knowledge, and surfaces prioritized, evidence-backed improvement suggestions to operators — eliminating the cognitive load of manual dashboard watching without removing human control.*

The key distinction from full AIOps autonomy: **the agent never touches the infrastructure**. It observes, reasons, and recommends. The operator decides and acts. This keeps humans in control of critical HPC resources while removing the burden of interpreting raw signals.

The key distinction from existing alerting: recommendations are grounded in specific evidence (which spans, which metrics, which past incidents), ranked by urgency, and include a concrete suggested action — not just "something is wrong."

---

## Architecture

```
Monitoring Stack
(Prometheus, Loki, Jaeger, cAdvisor, DCGM)
        │  metrics, logs, traces, GPU stats
        ▼
  [Ingestion Agent] ──── continuous telemetry stream
        │
        ▼
  [Anomaly Detector] ─── statistical + LLM-based detection
        │  anomaly confirmed
        ▼
  [Diagnosis Agent] ──── RAG over runbooks + incident history
        │                + container configs + recent deployments
        ▼
  [Recommendation Engine]
        │
        └──► Operator Dashboard / Notification
             ┌─────────────────────────────────────────────┐
             │ [HIGH] Job 847 — likely OOM pressure         │
             │ Evidence: CPU throttle + mem at 94%          │
             │ Similar incidents: 3 (last: 2025-11-14)      │
             │ Suggested action: increase mem limit 4→8 GB  │
             └─────────────────────────────────────────────┘
                          ↑
                   human decides and acts
```

---

## Anomaly Detection

The ingestion agent correlates across multiple signal types simultaneously. A single metric spike is noise; the same spike co-occurring with a container restart and a recent config change is a diagnosis.

| Signal | Source | Example Anomaly |
|---|---|---|
| CPU throttling | cAdvisor / cgroups | Job running at 10% expected speed |
| Memory pressure | cgroups `memory.stat` | OOM killer invoked, silent job death |
| Long-running queries | DB slow query log / `pg_stat` | Query >10× baseline duration |
| Network errors | node_exporter, `ss` | Packet loss between MPI ranks |
| Filesystem quota | `df`, `quota` | Job fails writing output silently |
| GPU utilization drop | DCGM exporter | GPU stall mid-training |
| Job wall-time risk | Slurm accounting | 80% wall-time consumed, <20% progress |
| Container exit codes | OCI runtime hooks | Non-zero exits, segfaults |

---

## Diagnosis Agent & RAG

Once an anomaly is confirmed, the diagnosis agent builds a context window from the live signal, recent telemetry history, and retrieved operational knowledge, then reasons toward a root cause and recommendation.

```
anomaly signal
      +
recent telemetry window   →  [Retriever] ──► Vector DB
                                               ├── internal runbooks
                                               ├── past incident reports
                                               ├── container / job configs
                                               └── software changelogs
      +
retrieved context
      │
      ▼
[LLM] ──► {
  "root_cause": "memory limit reduced below working set size",
  "evidence": ["CPU throttle spike at 14:32", "mem at 94%", "config changed 2h ago"],
  "confidence": 0.91,
  "suggested_action": "increase container mem limit from 4GB to 8GB",
  "similar_incidents": ["incident_203", "incident_187"]
}
```

### Example Integration

```python
import anthropic
from chromadb import Client
from chromadb.utils.embedding_functions import OllamaEmbeddingFunction

embed_fn = OllamaEmbeddingFunction(model_name="nomic-embed-text")
db = Client()
runbooks = db.get_or_create_collection("runbooks", embedding_function=embed_fn)

def diagnose(anomaly_summary: str) -> dict:
    results = runbooks.query(query_texts=[anomaly_summary], n_results=3)
    context = "\n\n".join(results["documents"][0])

    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": (
                f"Anomaly detected:\n{anomaly_summary}\n\n"
                f"Relevant runbooks:\n{context}\n\n"
                "Return JSON: {root_cause, evidence, confidence (0-1), suggested_action}"
            ),
        }],
    )
    return message.content[0].text
```

---

## Recommendation Engine

Recommendations are ranked by urgency and presented with full evidence chains so the operator can make an informed decision quickly.

```
Priority  Job / Service     Root Cause                  Suggested Action
────────  ────────────────  ──────────────────────────  ──────────────────────────
HIGH      job_847           OOM pressure (conf: 0.91)   Increase mem limit 4→8 GB
MEDIUM    postgres-replica  Slow query, missing index    Run EXPLAIN, add index on user_id
LOW       job_831           Approaching wall-time        Checkpoint now or request extension
```

Each recommendation links to the specific spans, metric windows, and past incidents that support it — the operator sees the reasoning, not just the conclusion. This is what distinguishes the system from a smarter alert: it is transparent and auditable.

---

## Self-Improvement Loop

The framework improves over time as operators act on recommendations and outcomes are recorded.

```
operator acts on recommendation
        │
        ▼
outcome recorded:
{
  "symptom": "CPU throttle + memory pressure on job_847",
  "root_cause": "memory limit too low",
  "suggested_action": "increase mem limit 4→8 GB",
  "operator_action": "accepted — applied change",
  "outcome": "CPU utilization recovered to 98% within 2 min",
  "confidence_was": 0.91
}
        │
        ▼
written back to RAG corpus as a new incident document
        │
        ▼
future similar anomalies retrieve this as evidence
→ higher confidence, faster diagnosis, better suggestions
```

Operator feedback (accepted / modified / rejected) is also a training signal for DSPy prompt optimization — as labeled outcomes accumulate, the diagnosis prompts can be automatically refined without manual rewriting.

---

## Observability Stack

The monitoring stack is both the **primary input** to the agent and the **source of verification** that a recommendation was effective after the operator acts.

| Layer | Tool | What It Provides |
|---|---|---|
| Metrics | Prometheus + node_exporter + cAdvisor | CPU, memory, network, container stats |
| GPU metrics | DCGM Exporter | GPU utilization, memory, temperature |
| Logs | Loki + Promtail | Structured container and application logs |
| Traces | Jaeger / Tempo | Distributed traces across microservices |
| HPC jobs | Slurm REST API / `sacct` | Job state, resource usage, exit codes |
| Agent traces | Langfuse (self-hosted) | Full LLM reasoning chains, tool calls, latency |
| Dashboards | Grafana | Unified view; recommendation overlay |

Langfuse traces every agent reasoning chain, making each recommendation fully auditable — critical for scientific workflows where operators need to understand and trust the system before acting on its suggestions.

---

## PhD Research Questions

1. Can an LLM agent, given only monitoring telemetry and a RAG corpus of runbooks, match the diagnostic accuracy of an experienced HPC sysadmin?
2. Does suggestion quality improve measurably over time as the incident memory store grows with operator feedback?
3. What evidence presentation format most effectively supports operator decision-making — ranked list, natural language explanation, evidence chain, or a combination?
4. How do you formally evaluate a recommendation system for HPC operations — what does a benchmark dataset look like for this domain?

---

## Key Tools & Frameworks

### LLM / Agent
- **Anthropic Claude API** (`anthropic` Python SDK) — `claude-opus-4-8`, `claude-sonnet-4-6`
- **Ollama** — local inference for air-gapped HPC environments
- **LangGraph** — agent state machine for the detect → diagnose → recommend loop
- **DSPy** — prompt optimization from accumulated operator feedback labels

### RAG & Memory
- **LlamaIndex** — ingestion pipeline for runbooks and incident reports
- **Qdrant** — vector store with hybrid dense+sparse retrieval
- **BGE-M3** — embedding model; runs locally, hybrid retrieval

### Observability
- **Prometheus + Grafana** — metrics and dashboards
- **Loki** — log aggregation
- **Langfuse** — self-hosted LLM tracing and agent observability
- **OpenTelemetry** — vendor-neutral instrumentation

### HPC / Container Runtime
- **Singularity / Apptainer** — HPC container runtime
- **cAdvisor** — container resource usage metrics
- **Slurm REST API** — job management programmatic interface

---

## Further Reading

### AIOps
- Dang et al. (2019) — *AIOps: Real-World Challenges and Research Innovations* (Microsoft)
- Chen et al. (2020) — *Towards Intelligent Incident Management: Why We Need It and How We Make It*
- Ahmed et al. (2023) — *Recommending Root-Cause and Mitigation Steps for Cloud Incidents using Large Language Models*

### Agentic Foundations
- Yao et al. (2022) — *ReAct: Synergizing Reasoning and Acting in Language Models*
- Shinn et al. (2023) — *Reflexion: Language Agents with Verbal Reinforcement Learning*
- Khattab et al. (2023) — *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines*

### RAG
- Lewis et al. (2020) — *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*
- Asai et al. (2023) — *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection*

---

*Doctoral thesis: containerization in scientific computation, focused on continuous operational intelligence via agentic AI.*
