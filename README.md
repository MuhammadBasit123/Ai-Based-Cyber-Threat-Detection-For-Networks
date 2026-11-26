# 🛡️ AI-Based Cyber Threat Detection for Networks (Explainable Hybrid NIDS/IPS)

An end-to-end, real-time **Network Intrusion Detection and Prevention System (NIDS/IPS)** trained on the **CIC-IDS2017** dataset. This project bridges the gap between offline machine learning and live network defense by combining calibrated supervised classifiers (**Random Forest** & **XGBoost**) with unsupervised sequence-based anomaly detection (**LSTM-Autoencoder**), real-time **SHAP** explainability, **MITRE ATT&CK** mapping, and a controlled firewall response layer.

---

## 🌟 Key Features

* **Hybrid Detection Engine:** Fuses supervised models (for high-precision classification of known attacks like DoS, PortScan, Brute Force, and Web Attacks) with an LSTM-Autoencoder trained strictly on benign traffic (to flag zero-day anomalies via reconstruction error).
* **Training-Serving Feature Parity:** Uses a single-source-of-truth specification (`features.json`) and a custom bidirectional 5-tuple flow builder (`flow_builder.py`) over **Scapy** to ensure live packet streams match CICFlowMeter statistical features without data leakage.
* **Actionable Explainability (XAI):** Integrates **SHAP (`TreeExplainer`)** and per-feature autoencoder reconstruction error to attach plain-language explanations, top contributing features, and **MITRE ATT&CK** / Cyber Kill Chain tags to every alert.
* **Controlled Response Layer (IPS Mode):** Tiered, reversible containment using Linux `nftables`/`iptables` (or Windows `netsh`) with strict allowlisting, automatic rule expiration, dry-run mode, human-in-the-loop approval workflows, and a full audit log to prevent self-inflicted denial-of-service from IP spoofing.
* **Real-Time Analyst Dashboard:** Built with **React**, **Vite**, and **WebSockets** over a **FastAPI** and **PostgreSQL** backend for sub-30-second triage, SHAP visualisations, model drift monitoring, and incident approval.

---

## 🏗️ System Architecture

```text
[Live Traffic / PCAP] 
       │
       ▼
1. Packet Capture (Scapy + BPF Filter, least-privilege CAP_NET_RAW)
       │
       ▼
2. Live Flow Builder (Canonical 5-Tuple, Welford Running Stats, Timeouts)
       │
       ▼
3. Feature Extractor (20–30 header-derived features synced via features.json)
       │
       ▼
4. Hybrid AI Engine ──► [Random Forest + XGBoost (Calibrated)] ──┐
                    └──► [LSTM-Autoencoder (Benign Sequence)] ───┼──► 5. Decision Fusion & Severity (0–100)
                                                                 │
       ┌─────────────────────────────────────────────────────────┘
       ▼
6. Explainability Worker (SHAP TreeExplainer + Per-Feature AE Error + ATT&CK Mapping)
       │
       ▼
7. FastAPI + PostgreSQL + WebSocket ──► 8. Controlled Response Layer (Allowlist + Firewall + Audit)
       │
       ▼
9. React Analyst Dashboard (Live Alerts, SHAP Panel, Model Drift, Approvals)
```

---

## 🛠️ Tech Stack

* **Machine Learning & XAI:** Python 3.10+, Scikit-Learn, XGBoost, PyTorch / TensorFlow, SHAP, NumPy, Pandas, Joblib
* **Network Capture & Flow Processing:** Scapy, Npcap (Windows) / libpcap (Linux)
* **Backend & Database:** FastAPI, Uvicorn, Pydantic, SQLAlchemy, Alembic, PostgreSQL, WebSockets
* **Frontend Dashboard:** React, Vite, TanStack Query, Recharts, Tailwind CSS
* **Active Response & DevOps:** `nftables` / `iptables`, Docker, Docker Compose, Nginx, Pytest

---

## 📂 Repository Structure

```text
├── configs/
│   ├── features.json          # Single source of truth for feature definitions & formulas
│   ├── thresholds.yaml        # Calibrated decision thresholds & severity weights
│   └── policy.yaml            # Response rules, allowlists, and firewall timeouts
├── notebooks/
│   ├── 01_eda_and_supervised.ipynb   # CIC-IDS2017 cleaning, RF & XGBoost training
│   └── 02_lstm_autoencoder.ipynb     # Benign sequence generation & LSTM-AE training
├── src/
│   ├── capture/
│   │   ├── sniffer.py         # Multi-process Scapy packet capture
│   │   └── flow_builder.py    # Bidirectional 5-tuple flow table & feature extractor
│   ├── models/
│   │   ├── decision_engine.py # Hybrid fusion, severity scoring & alert deduplication
│   │   └── explain.py         # Async SHAP & reconstruction error explainer
│   ├── response/
│   │   └── firewall.py        # Dry-run & active firewall blocking with auto-expiry
│   └── api/
│       ├── main.py            # FastAPI REST & WebSocket server
│       └── models.py          # SQLAlchemy ORM schemas (alerts, explanations, audit_log)
├── frontend/                  # React + Vite analyst dashboard
├── tests/                     # Unit tests, synthetic packet tests & PCAP replay validation
├── docker-compose.yml
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites
* **OS:** Linux VM (recommended for raw socket capture via `libpcap` and `nftables`/`iptables`) or Windows with Npcap.
* **Runtime:** Docker & Docker Compose, or Python 3.10+ and Node.js 18+.
* **Privileges:** Root or `CAP_NET_RAW` / `CAP_NET_ADMIN` capabilities for live interface sniffing.

### 1. Clone the Repository
```bash
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>
```

### 2. Configure Environment Variables
Copy the example environment file and update your credentials (never commit `.env` to version control):
```bash
cp .env.example .env
```
Key variables in `.env`:
* `DATABASE_URL=postgresql://nids_user:strong_password@db:5432/nids_db`
* `CAPTURE_INTERFACE=eth0`
* `RESPONSE_DRY_RUN=true` *(Keep `true` until testing in an isolated lab)*
* `JWT_SECRET_KEY=your_generated_secret_key`

### 3. Launch with Docker Compose
```bash
docker compose up --build -d
```
* **Dashboard:** `http://localhost:3000`
* **FastAPI Docs (Swagger):** `http://localhost:8000/docs`

### 4. Running the Live Capture Sensor (Linux Host)
Because raw packet capture requires host network access, run the sensor worker directly with least-privilege capabilities:
```bash
pip install -r requirements.txt
sudo setcap cap_net_raw,cap_net_admin=eip $(which python3)
python3 src/capture/sniffer.py --interface eth0
```

---

## 🧪 Validation & Testing

1. **Synthetic Flow Unit Tests:** Verify TCP flag counters, Welford running statistics, and flow timeouts:
   ```bash
   pytest tests/test_flow_builder.py
   ```
2. **PCAP Replay Parity Check:** Replay CIC-IDS2017 PCAP files through `flow_builder.py` to validate live feature parity against offline dataset labels:
   ```bash
   python3 tests/pcap_replay.py --pcap data/Wednesday-WorkingHours.pcap
   ```
3. **Zero-Day Leave-One-Attack-Out Evaluation:** Evaluate the LSTM-Autoencoder's ability to flag unseen attack families withheld during supervised training.

---

## ⚠️ Scope, Limitations & Ethical Use

* **Supported Network Scope:** Validated on IPv4 TCP, UDP, and ICMP traffic over Ethernet/Wi-Fi LAN segments. Encrypted payloads are analyzed strictly via flow metadata (privacy by design; no payload storage).
* **Dataset Caveats:** Trained on **CIC-IDS2017** (Sharafaldin et al., 2018), acknowledging documented flow-construction and labeling artifacts (Engelen et al., 2021) as well as cross-day benign traffic drift.
* **Authorized Lab Use Only:** This tool is developed strictly for academic research and authorized defensive monitoring. **Never** run active blocking or red-team attack simulations (`nmap`, `hping3`, `hydra`, `slowloris`) on public, enterprise, or university networks. Always test inside an isolated host-only virtual lab.

---

## 📚 References & Acknowledgments

* Sharafaldin, I., Lashkari, A. H., & Ghorbani, A. A. (2018). *Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization* (CIC-IDS2017). ICISSP.
* Engelen, G., Rimmer, V., & Joosen, W. (2021). *Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study*. IEEE S&P Workshops.
* Lundberg, S. M., & Lee, S. I. (2017). *A Unified Approach to Interpreting Model Predictions* (SHAP). NeurIPS.
