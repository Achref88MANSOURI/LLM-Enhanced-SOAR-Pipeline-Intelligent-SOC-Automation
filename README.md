# 🛡️ LLM-Assisted SOC — Open Source AI-Powered Security Operations Center

> Integrating a local Large Language Model into a SOAR pipeline for intelligent alert triage, threat enrichment, and automated incident reporting.

---

## 📌 Project Overview

This project builds a **fully open-source, local SOC** that leverages a fine-tuned cybersecurity LLM (`foundation-sec`) to assist SOC analysts in:

- **Automated triage** — classifying alerts as TRUE_POSITIVE or FALSE_POSITIVE with confidence scoring
- **Threat enrichment** — correlating IOCs (IPs, file hashes) via AbuseIPDB and VirusTotal through Cortex
- **Incident report generation** — producing structured SOC reports with MITRE ATT&CK mapping, IOCs, impact assessment, and remediation actions
- **False positive reduction** — context-aware reasoning over process names, IP ranges, and log content

All processing runs **100% locally** — no data leaves the infrastructure.

---

## 🏗️ Architecture

```
Wazuh (SIEM)
    │
    ▼ webhook (JSON alert)
Shuffle SOAR
    │
    ├── Shuffle Tools 1 ──── Alert normalization & field extraction
    │
    ├── Branch: IP enrichment ──── Cortex → AbuseIPDB analyzer
    ├── Branch: Hash enrichment ── Cortex → VirusTotal analyzer
    │
    ├── Shuffle Tools 2 ──── LLM Triage (foundation-sec-8b-instruct via Ollama)
    │       │                 • Context flags pre-computation
    │       │                 • Rule-based pre-decision
    │       │                 • 4-step reasoning prompt
    │       │                 • TRUE_POSITIVE / FALSE_POSITIVE verdict
    │       │
    │       ├── [FP] ──── Comment triage on alert → STOP
    │       │
    │       └── [TP] ──── Create case (TheHive)
    │                       ├── Comment triage on case
    │                       ├── Create observables (IP + hash)
    │                       ├── Comment Cortex reports
    │                       └── Shuffle Tools 3 ── LLM Report generation
    │                                               └── CreateLLMcomment in case
    │
    └── TheHive (Case Management) + Cortex (Enrichment)
```

---

## 🧰 Tech Stack

| Component | Technology | Role |
|-----------|-----------|------|
| SIEM | Wazuh 4.9.0 | Log collection, rule engine, alert generation |
| SOAR | Shuffle | Workflow automation, playbook orchestration |
| Case Management | TheHive 5.2 | Alert/case lifecycle, observables, analyst collaboration |
| Enrichment | Cortex 3.1.7 | AbuseIPDB + VirusTotal analyzers |
| LLM Runtime | Ollama (local) | Serving `foundation-sec-8b-instruct` via REST API |
| LLM Model | foundation-sec-8b-instruct (Cisco) | Cybersecurity-specialized 8B instruction-tuned LLM, used as-is without fine-tuning |
| Deployment | Docker Compose | All components containerized |
| OS | Ubuntu 24 | Host infrastructure |
| Agents | Wazuh Agent 4.9.0 | Windows 10 / Linux endpoint monitoring |

---

## 🤖 LLM Pipeline — Shuffle Tools 2 (Triage)

The model used is **Cisco foundation-sec-8b-instruct**, a cybersecurity-specialized 8B parameter instruction-tuned LLM, deployed locally via Ollama and used **without any fine-tuning**. All intelligence comes from prompt engineering — structured reasoning steps, pre-computed context flags, and explicit decision rules injected at inference time.

The triage node performs **4 mandatory reasoning steps** before producing a verdict:

**Step A — Process identification**
Identifies the exact process and its known legitimate purpose (MsMpEng.exe = Windows Defender, git.exe push = developer commit, svchost.exe -s wuauserv = Windows Update, etc.)

**Step B — Full log analysis**
Reads the `Desc=` field in the full log — explicit context like "scheduled", "authorized", "GPO", "ticket IT-xxxx" is strong FALSE_POSITIVE evidence.

**Step C — Context flags**
Pre-computed Python flags injected into the prompt:
- `IP_IS_GITHUB_RANGE`, `IP_IS_MICROSOFT_RANGE`, `IP_IS_GOOGLE_RANGE`
- `GIT_PUSH_TO_GITHUB`, `WINDOWS_UPDATE_SERVICE`, `SYSVOL_PATH_DETECTED`
- `OFFENSIVE_TOOL_MIMIKATZ`, `REVERSE_SHELL_COMMAND`, `ENCODED_COMMAND`
- `LOG_SAYS_SCHEDULED_OR_ROUTINE`, `LOG_SAYS_AUTHORIZED`

**Step D — Decision matrix**
9 FALSE_POSITIVE rules checked first, then 6 TRUE_POSITIVE rules — with explicit override logic preventing MITRE tags from determining verdict alone.

---

## 🧠 Why foundation-sec-8b-instruct?

[foundation-sec-8b-instruct](https://huggingface.co/cisco/foundation-sec-8b-instruct) is an open-weight model released by Cisco specifically trained on cybersecurity data. Compared to general-purpose models of similar size, it demonstrates stronger performance on:

- Security alert classification
- MITRE ATT&CK technique identification
- Structured output generation (JSON, fixed-format responses)
- Cybersecurity terminology and context understanding

**Key choice criteria for this project:**
- Runs fully locally on CPU/GPU via Ollama — no API costs, no data leakage
- 8B parameters fits within 16GB RAM
- Instruction-tuned — responds well to structured prompts with explicit output formats
- No fine-tuning required — prompt engineering alone is sufficient for the triage task

The model is used **as-is from Cisco's public release**. No fine-tuning was performed. All behavioral customization is achieved through prompt engineering in Shuffle Tools 2 and 3.

---


### True Positive Detection (11 attack scenarios)

| Attack Type | Technique | Verdict | Confidence |
|-------------|-----------|---------|------------|
| DCSync via DRSUAPI | T1003.006 | ✅ TP | 60% |
| MSHTA spear phishing | T1218.005 | ✅ TP | 60% |
| Crontab + SSH key injection | T1053+T1098 | ✅ TP | 60% |
| WMI lateral movement (wmiexec) | T1047 | ✅ TP | 60% |
| Kerberoasting /rc4opsec | T1558.003 | ✅ TP | 60% |
| NTDS.dit extraction | T1003.003 | ✅ TP | 90% |
| Netcat reverse shell | T1059.004 | ✅ TP | 90% |
| certutil LOLBIN download | T1105 | ✅ TP | 90% |
| NTLM relay (ntlmrelayx) | T1557.001 | ✅ TP | 60% |
| Log4Shell CVE-2021-44228 | T1190 | ✅ TP | 60% |
| LDAP dump (AbuseIPDB=100) | T1087.002 | ✅ TP | 90% |

**TP accuracy: 11/11 (100%)**
Confidence reaches 90% when Cortex enrichment returns high scores (AbuseIPDB ≥ 60 or VT malicious ≥ 5).

### False Positive Detection (8 benign scenarios)

| Alert | Process | FP Signal | Result |
|-------|---------|-----------|--------|
| Windows Defender scan | MsMpEng.exe | SYSTEM + internal IP + Desc=scheduled | ⚠️ Misclassified TP |
| Git push to GitHub | git.exe push | GitHub IP 140.82.x.x + Desc=routine | ⚠️ Misclassified TP |
| GPO regedit SYSVOL | regedit.exe | SYSVOL path + internal IP + Desc=GPO | ⚠️ Misclassified TP |
| IT nmap authorized | nmap.exe | IT ticket in log + it.admin user | ⚠️ Misclassified TP |
| Chrome auto-update | GoogleUpdate.exe | Google IP + /installsource scheduler | 🔧 In progress |
| Nessus scan authorized | nessus.exe | Authorized + ticket + svc.nessus | 🔧 In progress |
| Veeam nightly backup | VeeamAgent.exe | svc.backup + internal + 03:00 | 🔧 In progress |
| Wazuh SCA scan | wazuh-agent.exe | category=sca + own monitoring | 🔧 In progress |

**FP detection is the current focus of improvement.** Root causes identified:
1. **MITRE tag bias** — model weights MITRE tags over contextual evidence
2. **Desc= field ignored** — explicit log descriptions not influencing verdict
3. **Confidence locked at 60%** — no discrimination between obvious FP and ambiguous cases
4. **Hallucinated REASON** — model fabricates "high abuse score" when score is 0

---

## 🔍 Key Findings & Lessons

**What works well:**
- Pipeline architecture is solid — Wazuh → Shuffle → Cortex → LLM → TheHive flows correctly
- Enrichment integration: when AbuseIPDB returns score ≥ 60, confidence jumps to 90% and triage is reliable
- Report generation (ST3) produces actionable incident reports with MITRE mapping, IOCs, and remediation steps
- Human-in-the-loop preserved — no automated blocking, all verdicts require analyst validation

**What needs improvement:**
- FP classification requires stronger contextual reasoning — currently 0% FP detection rate
- Confidence calibration is flat — 60% for everything regardless of evidence strength
- LLM reasoning must be grounded in data fields, not MITRE tag semantics
- Cortex enrichment sometimes returns empty fields despite successful job execution (path resolution issue)

---

## 🚀 Getting Started

### Prerequisites
- Docker + Docker Compose
- Minimum 16GB RAM (LLM requires ~8GB)
- Ubuntu 22.04 / 24.04

### Quick Start

```bash
# Clone the repository
git clone https://github.com/your-username/llm-soc.git
cd llm-soc

# Start all services
docker-compose up -d

# Pull the LLM model
ollama pull foundation-sec-8b-instruct

# Access the interfaces
# Wazuh Dashboard:  https://localhost:8443
# TheHive:          http://localhost:9005
# Shuffle:          http://localhost:3001
# Cortex:           http://localhost:9001
```

### Wazuh Agent (Windows)
```powershell
msiexec /i wazuh-agent-4.9.0-1.msi /q `
  WAZUH_MANAGER="<MANAGER_IP>" `
  WAZUH_MANAGER_PORT="1519" `
  WAZUH_REGISTRATION_PORT="1517" `
  WAZUH_AGENT_NAME="<AGENT_NAME>"
```

> Note: Wazuh manager ports are remapped via Docker — use 1519 (communication) and 1517 (enrollment) instead of standard 1514/1515.

---

## 📁 Repository Structure

```
llm-soc/
├── docker-compose.yml          # Full stack deployment
├── wazuh/
│   └── ossec.conf              # Wazuh manager configuration
├── shuffle/
│   ├── shuffle_tools_1.py      # Alert normalization
│   ├── shuffle_tools_2.py      # LLM triage (main)
│   ├── shuffle_tools_3.py      # LLM report generation
│   └── shuffle_tools_comment.py # Comment formatting
├── test-alerts/
│   ├── true_positives/         # 25+ TP test cases
│   └── false_positives/        # 11 FP test cases
└── docs/
    └── architecture.png        # Architecture diagram
```

---

## 📈 Roadmap

- [ ] **FP detection improvement** — context-aware reasoning for known legitimate processes
- [ ] **Confidence calibration** — dynamic confidence based on evidence strength
- [ ] **Polling-based Cortex jobs** — replace fixed delays with status polling
- [ ] **MITRE ATT&CK enrichment** — auto-map alerts to ATT&CK navigator
- [ ] **Prompt injection protection** — sanitize log fields before LLM injection
- [ ] **Metrics dashboard** — track FP rate, TP rate, MTTD over time
- [ ] **Multi-agent support** — extend from single Windows VM to full lab environment

---

## 🤝 Contributing

This project is part of a university final-year project on AI-assisted SOC operations. Contributions, feedback, and discussions are welcome.

Open an issue for bug reports or feature requests. Pull requests welcome.

---

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

## 👤 Author

**Dali** — Cybersecurity student & SOC enthusiast
Building open-source security tooling at the intersection of AI and defensive security.

*Connect on LinkedIn | GitHub*

---

> ⭐ If this project is useful to you or your team, consider starring the repository — it helps others find it.
