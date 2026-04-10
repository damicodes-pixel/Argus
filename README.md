# Argus
Ci/CD pipeline fo rmy detections, and my scripts.

# Argus

**A multi-node security data platform built around one research question: can ML-based detection systems be made resilient to adversarial manipulation?**

Argus is not a homelab tutorial. It is an active research platform combining detection engineering, security data engineering, and adversarial ML across three physical machines with no cloud dependency.

---

## The research question

Most ML-based detection systems have a fundamental vulnerability — an intelligent adversary who studies the system can manipulate inputs to evade detection. Slow normalization attacks gradually shift behavioral baselines to accept malicious activity as normal. Feature-level evasion crafts telemetry that scores in the suppress band despite representing a real attack.

Argus is designed to study and improve resilience against exactly this threat model.

---

## Platform components

### Themis — autonomous SOC analyst

An end-to-end autonomous triage system with a three-tier decision architecture.

Wazuh alerts are validated against a versioned data contract, transformed into feature vectors enriched with per-entity behavioral baseline deltas, and scored by a custom-trained XGBoost classifier.

- **Below 0.4** → suppressed immediately
- **Above 0.85** → escalated immediately  
- **Between 0.4 and 0.85** → routed to Qwen 2.5 via Ollama for LLM triage reasoning, augmented by ChromaDB semantic memory

Every decision is written to a time-partitioned Parquet data lake with full lineage. A pipeline observability layer monitors event rates, latency, schema failures, and log source health continuously. The classifier is trained in Jupyter and deployed to the inference tower via SCP. Human overrides feed back as labeled training data.

### Dolos — adversarial stress-tester

An adversarial model designed to induce model drift and simulate modern attacker strategies.

Dolos operates at two layers:
- **Classifier layer** — crafts feature vectors designed to score in the suppress band despite representing malicious activity
- **Baseline layer** — executes slow normalization attacks that gradually poison per-entity behavioral profiles to make attack patterns appear normal over time

The research loop between Themis and Dolos is the core contribution. Dolos finds weaknesses. Retraining closes them. Dolos finds new ones. The system improves through adversarial pressure, not just labeled data.

### Aletheia — autonomous threat hunter

An on-demand hunting model running on the laptop that translates natural language questions into DuckDB queries against the live data lake. Surfacing anomalies, behavioral patterns, and historical decision context without writing SQL. Session-based: VMs paused, hunting model started, investigation runs, VMs resume.

### Detection as code

All Wazuh rules, active response configuration, and schema contract definitions are version-controlled in this repository. GitHub Actions validates, tests, and deploys changes automatically on merge to main. MITRE ATT&CK coverage gaps are tracked and updated on each deployment.

---

## System topology

```
┌─────────────────────────────────────────────────────┐
│  LAPTOP                                             │
│  Wazuh (Docker) · Kali VM · Windows VM · Jupyter   │
│  Feature factory · Schema validator · CI/CD         │
│  Aletheia (on-demand hunting model)                 │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP (feature vectors)
                       │ SSH (model deployment via SCP)
┌──────────────────────▼──────────────────────────────┐
│  TOWER (inference node)                             │
│  FastAPI · XGBoost classifier                      │
│  Qwen 2.5 via Ollama · ChromaDB                    │
│  Behavioral baseline engine                         │
│  Parquet data lake (USB) · DuckDB · Pipeline monitor│
└──────────────────────┬──────────────────────────────┘
                       │ WebSocket (real-time decisions)
┌──────────────────────▼──────────────────────────────┐
│  SURFACE PRO (analyst interface)                    │
│  Live alert dashboard · Escalation queue            │
│  Override capture · MITRE coverage map              │
└─────────────────────────────────────────────────────┘
```

---

## Three-tier decision architecture

| Tier | Component | Handles | Latency |
|------|-----------|---------|---------|
| 1 | XGBoost classifier | Bulk triage — suppress or escalate | Milliseconds |
| 2 | Qwen 2.5 via Ollama | Ambiguous cases — LLM reasoning with semantic memory | Seconds |
| 3 | Human analyst | Escalated alerts — final judgment, override capture | Human |

Each tier only handles what the tier below it could not confidently resolve. Most alerts never leave tier 1.

---

## Data infrastructure

- **Schema validation** — versioned data contract enforced at pipeline entry. Malformed alerts logged to a separate error partition, never reach the classifier.
- **Data lake** — Parquet files time-partitioned by year/month/day on a 1TB USB. Every record carries full lineage: rule fired, features extracted, baseline delta, classifier score, LLM reasoning if invoked, human override if applicable.
- **DuckDB** — SQL analytics interface over the Parquet lake. Serves both Aletheia's natural language query translation and Jupyter retraining analysis.
- **ChromaDB** — vector database storing LLM reasoning as embeddings. Enables semantic similarity search across historical decisions, retrieved as context for new triage prompts.
- **Pipeline observability** — continuous monitoring of event rates, latency, schema failure rates, and log source health. A silent pipeline failure is indistinguishable from an attacker who stopped generating noise.

---

## Behavioral baseline engine

Per-entity rolling profiles built exclusively from a synthetic curated dataset generated on the Windows VM. Attack traffic never touches baseline profiles.

The synthetic traffic generator runs continuously on the Windows VM — file reads, directory listings, application launches, network requests, authentication events on a randomized schedule. This defines normal. Everything outside the synthetic envelope is anomalous by definition.

Delta scores representing deviation from entity-specific historical profiles are included in the classifier feature vector alongside raw alert features. The same action from different entities scores differently based on their individual history.

**Baseline poisoning is a known threat model and a specific Dolos research target.** A patient adversary who establishes presence before the baseline is set becomes effectively invisible to per-entity anomaly systems.

---

## Detection as code

```
argus/
├── rules/              # Wazuh detection rules (XML)
├── decoders/           # Custom Wazuh decoders
├── .github/
│   └── workflows/      # GitHub Actions CI/CD
└── README.md
```

Rules are version-controlled, reviewed, and deployed automatically. Every merge to main triggers validation and deployment to the Wazuh manager. MITRE ATT&CK technique coverage is tracked per deployment.

**Detection rules are public. Implementation code is not.**

---

## Training data strategy

Three sources, weighted by quality:

1. **Attack simulation data** — MITRE ATT&CK techniques run via Atomic Red Team with timestamps recorded. Alerts in attack windows labeled malicious.
2. **Synthetic baseline data** — continuous output of the Windows VM traffic generator. Automatically labeled benign. Builds without manual effort.
3. **Human override data** — every analyst correction flagged as human-corrected in the data lake. Weighted most heavily in retraining. Captures expertise that automated labeling cannot.

---

## Threat hunting query library

A documented collection of DuckDB queries answering analytical questions over accumulated alert history — not detection rules, but investigative queries:

- First contact analysis — hosts that communicated with IPs they have never contacted before
- Temporal anomaly detection — accounts authenticating outside their historical hour baseline
- Process lineage anomalies — processes that spawned children they have never spawned before
- Volume anomalies — entities generating event volumes in the top percentile of their own history
- Lateral movement indicators — internal hosts appearing as both source and destination in short windows

Each query is documented with the question it answers, the SQL, example output, and what a positive result means for investigation.

---

## Research contribution

The Dolos adversarial loop is what distinguishes this platform from a detection engineering implementation.

Most detection systems are evaluated against static attack datasets. Dolos evaluates Themis against an intelligent adversary that adapts — studying the classifier's scoring behavior, crafting evasion strategies, executing baseline poisoning attacks over time. The question of what architectural properties make a detection system resilient to this kind of pressure is the research thread that makes this publishable.

---

## Status

Active development. Building toward full pipeline integration (P5 on the roadmap) — Wazuh active response connected to FastAPI, classifier scoring live alerts, decisions flowing to the dashboard in real time.

---

## About

Built by Damilola Ogunsuyi — CS @ UMBC, Undergraduate Security Researcher at UMBC's DAMS Lab.

[LinkedIn](https://linkedin.com/in/damilola-ogunsuyi-49a52630a)

---

*Argus watches everything. Themis judges it. Dolos attacks it. Aletheia reveals what's hidden.*
