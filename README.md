# SENTINEL SOC — Defensive Security Operations Console

A single-file, browser-based **blue-team / SOC analyst console**. It runs directly in the browser with no build step, no dependencies, and no backend — open `index.html` and it works. All data shown is **synthetic** and generated locally for demonstration; nothing touches a real system.

> **This is a demonstration UI, not a production security product.** It contains no real detection engine. It exists to prototype the *analyst experience* — how alerts, AI triage reasoning, guardrails and human-approval gates should be presented.

---

## 1. What's in the console

| # | Panel | Purpose |
|---|-------|---------|
| 01 | **Global Attack Surface** | Animated wireframe globe with live inbound attack arcs, origins and block/leak status |
| 02 | **Threat Posture** | 5-level threat indicator (Low → Severe) + posture score with detection/patch/MFA/EDR sub-metrics |
| 03 | **Detection Rules** | Sigma-style rules with enabled/disabled toggles and live hit counters |
| 04 | **Alert Triage Queue** | Live queue with severity badges, source (endpoint/network/cloud/identity), MITRE tags |
| 05 | **AI Triage Agent** | Animated reasoning chain, enrichment, ATT&CK mapping, confidence score, next steps |
| 06 | **Investigation Timeline** | Correlated event timeline, raw telemetry and suggested next queries |
| 07 | **Response Playbooks (SOAR)** | Containment actions — read-only until an analyst approves each one |
| 08 | **Immutable Audit Log** | Every AI recommendation, approval, denial and action, with actor identity |

Top KPI tiles cover **active alerts, critical incidents, MTTD, MTTR and assets monitored**, all drifting live.

---

## 2. Architecture (the design this UI models)

The console is the *presentation layer* of a pipeline that looks like this:

```
 ┌────────────┐   ┌─────────────┐   ┌──────────────┐   ┌──────────┐
 │ COLLECTORS │──▶│ MESSAGE BUS │──▶│ NORMALIZATION│──▶│ STORAGE  │
 │ EDR/NDR/   │   │  Kafka /    │   │  ECS / OCSF  │   │ OpenSearch│
 │ Cloud/IdP  │   │  Redpanda   │   │  enrichment  │   │ + object │
 └────────────┘   └─────────────┘   └──────────────┘   └────┬─────┘
                                                             │
                        ┌────────────────────────────────────┘
                        ▼
                 ┌─────────────┐        ┌──────────────────┐
                 │  DETECTION  │        │ AI ORCHESTRATOR  │
                 │ Sigma rules │───────▶│ triage · enrich  │
                 │ + ML models │        │ map · score      │
                 └─────────────┘        └────────┬─────────┘
                                                 │  (proposes only)
                                                 ▼
                                        ┌──────────────────┐
                                        │   SOAR / SOAR-   │
                                        │   GATED ACTIONS  │◀── HUMAN APPROVAL
                                        └────────┬─────────┘
                                                 ▼
                                        ┌──────────────────┐
                                        │ ANALYST CONSOLE  │
                                        │  (this UI)       │
                                        └──────────────────┘
```

**Stage by stage**

1. **Collectors** — endpoint (EDR/Sysmon), network (Zeek, Suricata), cloud (CloudTrail / audit logs), identity (IdP sign-in + directory), email, and FIM canaries stream telemetry out.
2. **Message bus** — high-throughput append-only transport (Kafka / Redpanda) decouples ingestion from processing.
3. **Normalization** — events are parsed into a common schema (ECS / OCSF), deduplicated, timestamp-synchronised and enriched with asset ownership.
4. **Storage** — hot search index for recent data (OpenSearch) plus cheap object storage for long-term/raw retention.
5. **Detection** — Sigma-format rules compiled to the query backend, plus statistical/ML anomaly models.
6. **AI orchestrator** — takes a fired detection and produces the *triage package*: correlation across sources, threat-intel + GeoIP + asset-criticality enrichment, ATT&CK mapping, severity, confidence.
7. **SOAR** — holds the response playbooks. Critically, the AI **only proposes**; execution is gated behind explicit human approval.
8. **Analyst console** — this UI: the single pane where triage, investigation and approval happen.

---

## 3. Defensive AI agents

| Agent | Responsibility | Output |
|-------|----------------|--------|
| **Triage agent** (ARIA-CHIEF) | Correlate, enrich, map to ATT&CK, assign severity + confidence | The AI Triage panel |
| **Investigation agent** | Build the correlated timeline; propose next queries | Timeline + suggested queries |
| **Detection-engineering agent** | Suggest rule tuning / false-positive reduction from feedback | Rule hit drift |
| **SOAR planner** | Draft containment options with blast-radius estimates | Recommended next steps |
| **Explainer agent** | Write human-readable rationale for every decision | Reasoning chain in-panel |

All agents are **advisory**. They write to the audit log and the review queue — never to the environment.

---

## 4. Guardrails (enforced by design, not by policy text)

- **Read-only by default** — no containment, blocking or disabling action runs without explicit analyst approval.
- **No autonomous destructive actions** — the AI cannot isolate a host, disable an account or block an IP on its own.
- **Human-approval gate** — every state-changing action opens a modal requiring a written justification before execution.
- **Blast-radius caps** — actions are scoped to 1 host / 1 account / 1 IP per approval, and are time-boxed (4h default) and reversible.
- **RBAC** — the console renders the caller's role; only roles holding `responder` may approve.
- **Immutable audit log** — every AI recommendation, approval, denial and executed action is appended with actor identity and timestamp. Denials are logged too.
- **Policy pre-check** — each action is validated against blast-radius policy *before* it is offered for approval (shown as `PASSED` in the modal).

---

## 5. Recommended open-source stack

| Layer | Tool | Role |
|-------|------|------|
| SIEM / XDR | **Wazuh** | Host + log detection, FIM, compliance, agent fleet |
| Network | **Zeek** | Rich connection/protocol telemetry (conn.log, dns.log, smb_files) |
| Network IDS | **Suricata** | Signature + protocol-anomaly detection, EVE JSON output |
| Detection rules | **Sigma** | Vendor-neutral detection-rule format, compiled to the query backend |
| Threat intel | **MISP** | Indicator sharing, correlation, intel enrichment source |
| SOAR | **Shuffle** | Playbook automation, approval workflows, action connectors |
| Case management | **TheHive** | Incident cases, tasks, observables, analyst collaboration |
| Local LLM | **Ollama** | On-prem model serving so telemetry never leaves the network |
| Vector store | **Qdrant** | Embeddings for intel retrieval / RAG over past incidents |
| Search | OpenSearch | Hot storage + query backend for normalized events |
| Bus | Kafka / Redpanda | Ingestion transport and replay |

**Why local-first:** a SOC's telemetry contains credentials, PII and crown-jewel topology. Running the LLM (Ollama) and vector store (Qdrant) on-prem keeps sensitive data inside the trust boundary while still enabling AI-assisted triage.

---

## 6. How to open it

**Option A — just open the file**
```bash
# clone or download, then:
open index.html          # macOS
xdg-open index.html      # Linux
start index.html         # Windows
```

**Option B — serve locally** (recommended if you extend it)
```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

**Option C — live** — see the URL in the project's delivery message.

### Interacting with the demo
- Click any **alert** in the triage queue → the AI Triage and Investigation panels re-run for that case.
- Toggle a **detection rule** → it's written to the audit log as a human action.
- Click **AI PROPOSE** on a playbook → the AI recommends it (logged, nothing executes).
- Click **REQUEST APPROVAL** → the human-approval modal opens. Type a justification (8+ chars) → **Approve & Execute**, or **Deny**. Both outcomes are audited.

---

## 7. Files

```
sentinel-soc/
├── index.html   # the entire console — HTML + CSS + vanilla JS, self-contained
└── README.md    # this file
```

No frameworks, no bundler, no network calls. Roughly 1,500 lines, one file.

---

*Synthetic demonstration data only. Designed as a defensive/blue-team reference interface.*
