# agent-compliance-mapper

![Status](https://img.shields.io/badge/status-active--development-orange)
![License](https://img.shields.io/badge/license-Apache%202.0-blue)
![Stack](https://img.shields.io/badge/stack-Python%20CLI-informational)
![Regulatory](https://img.shields.io/badge/regulatory-OSFI%20E--23%20%7C%20EU%20AI%20Act%20%7C%20NIST%20AI%20RMF-darkgreen)

> Give it a LangChain agent config. Get back a regulatory gap report mapped to OSFI E-23, EU AI Act, and NIST AI RMF — in under a minute.

> **Part of the [agentic-ai-security-audit-framework](https://github.com/sumitgiri/agentic-ai-security-audit-framework) — a structured methodology for auditing enterprise agentic AI deployments.**

---

## Table of Contents

1. [The Problem This Solves](#1-the-problem-this-solves)
2. [What the Tool Does](#2-what-the-tool-does)
3. [CLI Usage](#3-cli-usage)
4. [Output Format](#4-output-format)
5. [Frameworks Covered](#5-frameworks-covered)
6. [Installation](#6-installation)
7. [Repository Structure](#7-repository-structure)
8. [Current Status](#8-current-status)
9. [Related Repos](#9-related-repos)
10. [References](#10-references)
11. [Author](#11-author)

---

## 1. The Problem This Solves

A compliance officer or CISO who receives a LangChain agent configuration file faces a specific problem: the configuration contains technical decisions — which tools are declared, what memory type is used, which model is called, how the system prompt is structured — but provides no visibility into what those decisions mean for regulatory compliance.

Is this agent subject to OSFI E-23 model risk management requirements? Which EU AI Act obligations apply? Does the memory configuration create a data retention issue under Article 12? Does the tool access scope create a human oversight problem under Article 14? Is there enough documentation to satisfy NIST AI RMF's Measure function?

None of these questions can be answered by reading the config file without cross-referencing it against the regulatory text. That cross-referencing is time-consuming, requires both technical and regulatory expertise simultaneously, and produces inconsistent results when done manually across different reviewers.

`agent-compliance-mapper` automates the cross-reference. It parses the agent configuration, maps each component against the applicable regulatory requirements, and produces a structured gap report that tells a compliance officer exactly where the deployment stands — and what evidence would be needed to close each gap.

It does not replace a human audit. It produces the baseline analysis that makes a human audit faster, more consistent, and more defensible.

---

## 2. What the Tool Does

**Input:** A LangChain or LangGraph agent configuration file (YAML or JSON), or a directory containing multiple agent configs.

**Process:**

1. Parses the configuration to extract: tools declared, tool access scope, memory type and persistence settings, model identifier, system prompt structure, output handlers, logging configuration
2. Maps each extracted component against requirement-level entries in three regulatory frameworks
3. Assigns each requirement a status: `MET` | `PARTIAL` | `GAP` | `CANNOT_ASSESS`
4. For `GAP` and `CANNOT_ASSESS` items, specifies what evidence or configuration change would be required to satisfy the requirement
5. Produces a risk-ranked report in both JSON and Markdown

**What `CANNOT_ASSESS` means:** Some requirements — primarily those concerning runtime behaviour, actual data processed, or human oversight in practice — cannot be evaluated from a static configuration file alone. The tool marks these explicitly rather than guessing. This is the honest answer: a static analysis tool has limits, and a compliance officer needs to know where they are.

---

## 3. CLI Usage

> **Note:** The CLI is under active development. Check [Current Status](#8-current-status) before running. OSFI E-23 mapping is the current build focus — EU AI Act and NIST mappings are planned.

**Basic usage:**

```bash
python mapper.py --config agent_config.yaml --frameworks osfi_e23 eu_ai_act nist_ai_rmf
```

**With output directory:**

```bash
python mapper.py --config agent_config.yaml --frameworks all --output ./reports/
```

**Multiple configs (batch mode):**

```bash
python mapper.py --config-dir ./agent_configs/ --frameworks all --output ./reports/
```

**Output formats:**

```bash
# Default: both JSON and Markdown
python mapper.py --config agent_config.yaml --frameworks all

# Markdown only (for human review)
python mapper.py --config agent_config.yaml --frameworks all --format markdown

# JSON only (for pipeline integration)
python mapper.py --config agent_config.yaml --frameworks all --format json
```

One command in. One report out. The report is named `{agent_name}_{timestamp}_gap_report`.

---

## 4. Output Format

**Markdown report (excerpt):**

```
# Compliance Gap Report — customer-service-agent
Generated: 2025-03-14T10:42:01Z
Frameworks assessed: OSFI E-23, EU AI Act Articles 9-15, NIST AI RMF

## Executive Summary
Requirements assessed: 47
Met: 12 (26%)
Partial: 8 (17%)
Gap: 14 (30%)
Cannot assess from config: 13 (28%)

---

## OSFI E-23 — Model Risk Management

| Requirement | Section | Status | Evidence Needed |
|---|---|---|---|
| Model inventory documentation | E-23 §4.1 | GAP | Agent not registered in model inventory. No inventory file found in config. |
| Independent model validation | E-23 §4.3 | CANNOT_ASSESS | Requires runtime evidence. Document validation process and validator independence. |
| System prompt documentation | E-23 §5.2 | PARTIAL | System prompt present but undocumented. Add description field to config. |
| Tool access scope documented | E-23 §5.2 | GAP | 6 tools declared. No access justification documented for any tool. |
| Logging sufficient for post-hoc review | E-23 §6.1 | GAP | No audit logging handler detected in config. See llm-audit-logger. |
| Model version pinned | E-23 §5.1 | MET | Model version string present: gpt-4o-2024-08-06 |

## EU AI Act — Articles 9-15

| Requirement | Article | Status | Evidence Needed |
|---|---|---|---|
| Risk management system documented | Art. 9 | GAP | No risk management documentation linked in config. |
| Logging of events for post-hoc assessment | Art. 12 | GAP | No audit logging handler detected. |
| Human oversight mechanism specified | Art. 14 | CANNOT_ASSESS | Cannot determine from config. Document human-in-the-loop procedure. |
| Accuracy and robustness testing documented | Art. 15 | CANNOT_ASSESS | Requires runtime evidence. Document adversarial testing results. |

## NIST AI RMF

| Requirement | Function | Status | Evidence Needed |
|---|---|---|---|
| AI system mapped to organisational risk | MAP 1.1 | CANNOT_ASSESS | No organisational context in config. |
| Bias and fairness evaluation documented | MEASURE 2.2 | GAP | No evaluation documentation found. |
| Incident response procedure defined | MANAGE 4.1 | GAP | No incident response reference in config. |
```

**JSON report (excerpt):**

```json
{
  "agent_name": "customer-service-agent",
  "generated_utc": "2025-03-14T10:42:01Z",
  "summary": {
    "total_requirements": 47,
    "met": 12,
    "partial": 8,
    "gap": 14,
    "cannot_assess": 13
  },
  "findings": [
    {
      "framework": "OSFI_E23",
      "requirement": "Logging sufficient for post-hoc review",
      "section": "E-23 §6.1",
      "status": "GAP",
      "severity": "HIGH",
      "evidence_needed": "No audit logging handler detected in config.",
      "remediation": "Attach AuditCallbackHandler from llm-audit-logger to agent executor."
    }
  ]
}
```

---

## 5. Frameworks Covered

### OSFI Guideline E-23 — Model Risk Management

**Why it matters for Canadian enterprises:** E-23 applies to all federally regulated financial institutions — the Big Five banks, major insurers, and federally regulated pension funds. It explicitly covers AI and ML models under its model risk management obligations. The compliance deadline is **May 2027.** Independent validation, ongoing monitoring, and documentation requirements apply directly to agentic AI deployments. A bank that cannot produce structured documentation of its agent configurations and behaviour is not in compliance.

**What the mapper assesses against E-23:** Model inventory status, system prompt documentation, tool access justification, logging sufficiency, model version pinning, validation documentation, change management evidence.

### EU AI Act — Articles 9-15

**Why it matters for Canadian enterprises:** The EU AI Act applies extraterritorially. Canadian enterprises with EU operations, EU-facing products, or EU data subjects are subject to its high-risk AI system obligations. Articles 9-15 cover risk management systems, data governance, technical documentation, record-keeping, transparency, human oversight, and cybersecurity. Article 15 explicitly requires adversarial robustness. The enforcement window for high-risk system obligations runs through 2025-2026.

**What the mapper assesses against Articles 9-15:** Risk management documentation (Art. 9), data governance procedures (Art. 10), technical documentation completeness (Art. 11), logging configuration (Art. 12), transparency mechanisms (Art. 13), human oversight specification (Art. 14), robustness testing evidence (Art. 15).

### NIST AI Risk Management Framework (AI RMF 1.0)

**Why it matters for Canadian enterprises:** OSFI explicitly references NIST frameworks in its model risk management guidance. While NIST AI RMF compliance is not itself mandatory, Canadian regulated institutions are expected to demonstrate awareness of it, and aligning documentation to NIST provides a defensible baseline for regulatory conversations. It also maps to the vocabulary that most enterprise risk functions already use.

**What the mapper assesses against NIST AI RMF:** Organisational risk mapping (MAP), bias and fairness evaluation (MEASURE), incident response documentation (MANAGE), AI system categorisation (GOVERN).

---

## 6. Installation

```bash
git clone https://github.com/sumitgiri/agent-compliance-mapper
cd agent-compliance-mapper
pip install -r requirements.txt
```

**Dependencies:** `pyyaml`, `pydantic >= 2.0`, `rich` (terminal output formatting)

No external services required. Framework mapping files are bundled as YAML in `mappings/`.

---

## 7. Repository Structure

**Current:** CLI core and OSFI E-23 mapping in progress.  
**Planned:** EU AI Act and NIST mappings, batch mode, JSON output format.

See [Current Status](#8-current-status) below.

---

## 8. Current Status

| Component | Status |
|---|---|
| Config parser (LangChain YAML/JSON) | 🔄 In progress |
| OSFI E-23 mapping file | 🔄 In progress |
| Markdown report output | 🔄 In progress |
| EU AI Act Articles 9-15 mapping | 📅 Planned |
| NIST AI RMF mapping | 📅 Planned |
| JSON report output | 📅 Planned |
| LangGraph config support | 📅 Planned |
| Batch mode (config directory) | 📅 Planned |

---

## 9. Related Repos

| Repository | Description |
|---|---|
| [agentic-ai-security-audit-framework](https://github.com/sumitgiri/agentic-ai-security-audit-framework) | Flagship repo — full audit methodology, compliance mapper, evidence templates |
| [agentic-rag-security-lab](https://github.com/sumitgiri/agentic-rag-security-lab) | Vulnerable-by-design RAG pipeline for attack research |
| [llm-audit-logger](https://github.com/sumitgiri/llm-audit-logger) | Drop-in middleware for structured agent action logging |
| agent-compliance-mapper | **This repo** |

---

## 10. References

- OSFI Guideline E-23 — *Model Risk Management* (revised, effective May 2027). Office of the Superintendent of Financial Institutions, Canada.
- EU Artificial Intelligence Act (Regulation 2024/1689) — Articles 9-15. European Parliament, August 2024.
- NIST AI Risk Management Framework (AI RMF 1.0, January 2023). National Institute of Standards and Technology.
- NIST SP 800-53 Rev 5 — Security and Privacy Controls for Information Systems.

---

## 11. Author

**Sumit Giri**  
Security Engineer · AI Red Teamer · PhD Mathematics (Cryptography)  
Toronto, Ontario, Canada

AI red teaming at Mindrift. Cybersecurity consulting at CyStack. Building an independent AI agent security audit practice for Canadian regulated enterprises.

[LinkedIn](https://linkedin.com/in/sumitgiri) · [GitHub](https://github.com/sumitgiri)

---

*This is independent research. No vendor relationship. No affiliation with any of the frameworks or organisations referenced.*

