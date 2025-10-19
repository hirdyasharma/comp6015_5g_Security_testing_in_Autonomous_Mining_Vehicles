
# 5G Security Testing in Autonomous Mining Vehicles

A reproducible 5G edge testbed for detecting security events that impact autonomous mining vehicles.
The stack combines **Open5GS** (AMF/SMF/UPF), a **MEC Edge API** hosting a PyTorch model with **federated learning (FedAvg)** utilities, a **telemetry+RNIS generator**, and full **observability** via **Prometheus** and **Grafana**.

> **Key idea**
> Synthetic but realistic vehicle and network features are streamed to the MEC. The MEC exposes prediction and RNIS metrics for Prometheus. Grafana dashboards correlate model verdicts, RNIS, and system health with controlled attack bursts.

---

## Authors

**TEAM — Curtin University (COMP6015)**

* Hitesh Pankhania — 22471264
* Deepika Sharma — 21952195
* Hirdya Sharma — 21749180
* Gurleen Kaur — 22131597
* Achani Bandara — 21741102

> The same author list applies to the release artifact **`5g-observability.zip`**.

---

## Repository structure

```
.
├─ docker-compose.yml                  # Orchestrates MEC API, Prometheus, Grafana, node_exporter, cAdvisor
├─ prometheus/
│  └─ prometheus.yml                   # 5s scrape interval; targets for MEC (service + host IP), node_exporter, cAdvisor, Open5GS
├─ grafana/
│  └─ provisioning/
│     ├─ datasources/                  # Prometheus data source (YAML)
│     └─ dashboards/                   # Optional: prebuilt dashboard JSON(s)
├─ mec-edge-api/
│  ├─ Dockerfile                       # Python 3.11 slim; runs FastAPI + Uvicorn
│  ├─ requirements.txt
│  ├─ data/                            # Persisted: scaler.pkl, model_meta.json, global_model.pt
│  └─ src/
│     ├─ main.py                       # FastAPI: RNIS, FL, Predict, /metrics
│     └─ ai/
│        └─ model.py                   # MLP, scaler/meta persistence, FedAvg, inference
├─ telemetry_gen.py                    # Synthetic telemetry + RNIS; steady/burst attacks
└─ data/                               # Optional: CSVs (UL-ECE-5G-AV-DDoS2025.csv, 5G_V2N_Communication_Dataset.csv)
```

---

## Prerequisites

* **Docker** and **Docker Compose v2**
* **Python 3.11** (only needed if you plan to run `telemetry_gen.py` outside of a container)
* An **Open5GS** core (AMF/SMF/UPF) reachable from the host running this stack
* Host network accessible at `192.168.56.105` (or update the IPs accordingly in configs)

> If your host IP is different, search-and-replace `192.168.56.105` in:
>
> * `prometheus/prometheus.yml` targets
> * any local notes/commands in your runbook

---

## Quick start

### 1) Clone and configure

```bash
git clone https://github.com/hirdyasharma/comp6015_5g_Security_testing_in_Autonomous_Mining_Vehicles.git
cd comp6015_5g_Security_testing_in_Autonomous_Mining_Vehicles
```

Add or verify CSVs under `./mec-edge-api/data/` or `./data/`, for example:

* `UL-ECE-5G-AV-DDoS2025.csv`
* `5G_V2N_Communication_Dataset.csv`

### 2) Bring up the stack

```bash
docker compose up -d
```

Services and default URLs:

* MEC Edge API: `http://192.168.56.105:8080`

  * Health: `/health`
  * Swagger UI: `/docs`
  * Prometheus metrics: `/metrics`
* Prometheus: `http://192.168.56.105:29090`

  * Targets: `/targets`
* Grafana: `http://192.168.56.105:3030`

  * Default login: `admin / admin` (unless you changed `GF_SECURITY_ADMIN_PASSWORD`)
* node_exporter: `:9100` (scraped by Prometheus)
* cAdvisor: `http://192.168.56.105:8082`

### 3) One-time bootstrap of the model

Create scaler and meta so inference and FL can run:

```bash
curl -X POST http://192.168.56.105:8080/ai/bootstrap \
  -H "Content-Type: application/json" \
  -d '{"csv_path":"./mec-edge-api/data/UL-ECE-5G-AV-DDoS2025.csv","label_col":"Attack_Type"}'
```

Check model info:

```bash
curl http://192.168.56.105:8080/ai/info
```

### 4) Optional: local client training + FedAvg

Train locally on a CSV to simulate a client update:

```bash
curl -X POST http://192.168.56.105:8080/ai/fl/train_local \
  -H "Content-Type: application/json" \
  -d '{"csv_path":"./mec-edge-api/data/UL-ECE-5G-AV-DDoS2025.csv","label_col":"Attack_Type"}'
```

This returns `{ "weights": [...], "samples": N, "meta": {...} }`.

Aggregate one or more updates:

```bash
curl -X POST http://192.168.56.105:8080/ai/fl/aggregate \
  -H "Content-Type: application/json" \
  -d '[{"client_id":"c1","weights":[...],"samples":N}]'
```

Grafana should show `mec_fl_rounds_total` increasing.

### 5) Drive the system with telemetry and RNIS

Install Python dependencies if running locally:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install requests
```

Run the generator in **burst** mode to simulate an attack cycle:

```bash
python3 telemetry_gen.py --base http://192.168.56.105:8080 \
  --vehicles veh-A,veh-B --ues UE1,UE2 --cells CELL-1,CELL-2 \
  --pattern burst --atype ddos --intensity 0.9 --rate 1 --window 30 --pause 30 --loc
```

You should see console lines like:

```
[09:12:03] UE=UE1 CELL=CELL-1 atk=ddos rnis=200 ai=200 class=DDOS p=0.86
```

### 6) Observe in Grafana

Open `http://192.168.56.105:3030` and select the **5G Security** dashboard (or import the JSON under `grafana/provisioning/dashboards`). Panels of interest:

* Winner probability: `mec_ai_attack_level{vehicle="veh-A"}`
* Per-class probabilities: `mec_ai_prob{vehicle="veh-A"}`
* RNIS: `mec_rnis_rsrp{ue_id="UE1",cell_id="CELL-1"}`
* FL rounds: `mec_fl_rounds_total`
* Host and container health: node_exporter and cAdvisor panels

> Tip: set the time window to 6–12 minutes to include a full burst cycle. Use annotations to mark attack start and end.

---

## MEC Edge API — endpoints

* `GET /health`
* `POST /rnis/reports` → `{ ue_id, cell_id, rsrp, rsrq, cqi, sinr, cell_load, ts_ms? }`
* `GET /rnis/reports/latest`
* `POST /location/devices` and `GET /location/devices/{ue_id}`
* `POST /traffic/rules` (stub for policy experimentation)
* `POST /ai/bootstrap`
* `POST /ai/fl/train_local`
* `POST /ai/fl/aggregate`
* `GET /ai/info`
* `POST /ai/predict` → `{ predicted_class, probs{class: p} }`
* `GET /metrics` → Prometheus exposition with:

  * `mec_ai_predictions_total`
  * `mec_ai_attack_level{vehicle,predicted_class}`
  * `mec_ai_prob{vehicle,class_name}`
  * `mec_fl_rounds_total`
  * `mec_rnis_samples_total{ue_id,cell_id}`, `mec_rnis_rsrp{ue_id,cell_id}`

---

## Launching the whole project from the ZIP

If you are using the release artifact **`5g-observability.zip`**:

1. Unzip to a working directory on the host.
2. Ensure Docker and Docker Compose are installed.
3. Update host IPs if not `192.168.56.105` in `prometheus/prometheus.yml`.
4. Run:

   ```bash
   docker compose up -d
   ```
5. Follow **Quick start** steps 3–6 above (bootstrap → optional FL → generator → Grafana).

> **Authors for the ZIP:** Hitesh Pankhania, Deepika Sharma, Hirdya Sharma, Gurleen Kaur, Achani Bandara.

---

## Attacks and validation (optional but recommended)

* Launch controlled floods from a Kali VM:

  ```bash
  # UDP flood
  sudo hping3 --flood --udp -p 80 192.168.56.105
  # SYN flood
  sudo hping3 --flood -S -p 443 192.168.56.105
  # ICMP flood
  sudo hping3 --flood --icmp 192.168.56.105
  ```
* Align **telemetry_gen** burst windows with these floods.
* Confirm packet spikes in **Wireshark** and compare timestamps with Grafana.

---

## Troubleshooting

* **UE not registering / no PDU session**

  * Check AMF/SMF/UPF bind addresses and PLMN/TAC in Open5GS. Ensure reachability from the host network.

* **Prometheus shows MEC target DOWN**

  * Visit `http://192.168.56.105:8080/metrics`.
  * In `prometheus.yml`, keep both targets for the MEC API: `mec-api:8080` and `192.168.56.105:8080`.
  * Restart Prometheus and the MEC API.

* **`/ai/predict` returns HTTP 400**

  * Ensure your JSON keys match the model `feature_order` from `GET /ai/info`.
  * Prefer `features_map` with exactly those keys.

* **No class other than DDoS/Normal**

  * Current baseline was trained on two classes. To extend, retrain and aggregate with labels such as `Normal, UDP, TCP_SYN, ICMP, DDoS`, then redeploy.

---

## Security and ethics

This repository is for **research and education**. Do not run flood tools on networks you do not own or control. Keep all testing inside isolated lab environments.

---


## Citation

If you use this work in academic contexts, please cite:

> H. Pankhania, D. Sharma, H. Sharma, G. Kaur, A. Bandara. *5G Security Testing in Autonomous Mining Vehicles.* Curtin University, 2025.

---

### Maintainer

For repo questions or issues, open a GitHub Issue and tag **@hirdyasharma**.


