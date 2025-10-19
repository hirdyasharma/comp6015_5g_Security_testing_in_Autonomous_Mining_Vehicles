# 5G Security Testing in Autonomous Mining Vehicles

A reproducible 5G edge testbed for detecting security events that impact autonomous mining vehicles.
The stack combines **Open5GS** (AMF, SMF, UPF), a **MEC Edge API** hosting a PyTorch model with **federated learning (FedAvg)** utilities, a **telemetry + RNIS generator**, and full **observability** via **Prometheus** and **Grafana**.

> Synthetic but realistic vehicle and network features are streamed to the MEC. The MEC exposes prediction and RNIS metrics for Prometheus; Grafana correlates model verdicts, RNIS and host/container health with controlled attack bursts.

---

## Authors

Curtin University — COMP6015 team

* Hitesh Pankhania — 22471264
* Deepika Sharma — 21952195
* Hirdya Sharma — 21749180
* Gurleen Kaur — 22131597
* Achani Bandara — 21741102

The same author list applies to the release artifact **`5g-observability.zip`**.

---

## Directory layout

```
.
├─ docker-compose.yml
├─ prometheus/
│  └─ prometheus.yml
├─ grafana/
│  └─ provisioning/
│     ├─ datasources/             # Prometheus data source YAML
│     └─ dashboards/              # Optional: dashboard JSON(s)
├─ mec-edge-api/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  ├─ data/                       # PERSISTED MODEL + DATA
│  │  ├─ UL-ECE-5G-AV-DDoS2025.csv
│  │  ├─ 5G_V2N_Communication_Dataset.csv
│  │  ├─ scaler.pkl
│  │  ├─ model_meta.json
│  │  ├─ model.pkl                # persisted model file used by the API
│  │  └─ global_model.pt          # (optional) torch state_dict if present
│  └─ src/
│     ├─ main.py                  # FastAPI: RNIS, FL, Predict, /metrics
│     └─ ai/
│        └─ model.py              # MLP + FedAvg + scaler/meta I/O
├─ telemetrics/
│  ├─ traffic.py                  # generator v1
│  └─ telemetry_gen.py            # generator v2 (burst/steady, RNIS shaping)
└─ (optional) other docs/scripts
```

> The **compose** file mounts `./mec-edge-api/data:/app/data` so `scaler.pkl`, `model_meta.json` and `model.pkl` survive container restarts.

---

## Prerequisites

* Docker and Docker Compose v2
* Python 3.11 only if you run the generators locally
* Open5GS core reachable from the host where this stack runs
* Host IP defaults to `192.168.56.105` in examples; adjust if yours differs

---

## Launch

### 1) Clone and configure

```bash
git clone https://github.com/hirdyasharma/comp6015_5g_Security_testing_in_Autonomous_Mining_Vehicles.git
cd comp6015_5g_Security_testing_in_Autonomous_Mining_Vehicles
```

Make sure the CSVs are present under `mec-edge-api/data/` exactly as shown above.

### 2) Bring up the stack

```bash
docker compose up -d
```

* MEC Edge API: `http://192.168.56.105:8080`  (Swagger: `/docs`, Metrics: `/metrics`)
* Prometheus: `http://192.168.56.105:29090`  (Targets: `/targets`)
* Grafana: `http://192.168.56.105:3030`  (default `admin/admin` unless changed)
* cAdvisor: `http://192.168.56.105:8082`
* node_exporter: `:9100` (scraped by Prometheus)

### 3) Bootstrap the model once

Create the scaler and metadata from a dataset so inference and FL have a consistent shape.

```bash
curl -X POST http://192.168.56.105:8080/ai/bootstrap \
  -H "Content-Type: application/json" \
  -d '{"csv_path":"/app/data/UL-ECE-5G-AV-DDoS2025.csv","label_col":"Attack_Type"}'
```

Confirm:

```bash
curl http://192.168.56.105:8080/ai/info
```

> The MEC API persists the model to **`/app/data/model.pkl`** and the scaler to `scaler.pkl`. If `global_model.pt` is also used by your build, it will live in the same folder.

### 4) Optional: local training and FedAvg demo

```bash
# Local client update
curl -X POST http://192.168.56.105:8080/ai/fl/train_local \
  -H "Content-Type: application/json" \
  -d '{"csv_path":"/app/data/UL-ECE-5G-AV-DDoS2025.csv","label_col":"Attack_Type"}'
```

Take the returned `{weights, samples, meta}` and aggregate:

```bash
curl -X POST http://192.168.56.105:8080/ai/fl/aggregate \
  -H "Content-Type: application/json" \
  -d '[{"client_id":"c1","weights":[...],"samples":123}]'
```

Grafana should show `mec_fl_rounds_total` increasing.

### 5) Drive telemetry and RNIS

You have **two generator versions** under `telemetrics/`. Use either.

**Option A — `telemetrics/telemetry_gen.py`**
Burst mode with RNIS shaping and optional location updates:

```bash
python3 telemetrics/telemetry_gen.py --base http://192.168.56.105:8080 \
  --vehicles veh-A,veh-B --ues UE1,UE2 --cells CELL-1,CELL-2 \
  --pattern burst --atype ddos --intensity 0.9 --rate 1 \
  --window 30 --pause 30 --loc
```

**Option B — `telemetrics/traffic.py`**
If this version takes different flags, run `-h` first:

```bash
python3 telemetrics/traffic.py -h
```

Then run a comparable DDoS burst targeting the same `--base` URL.

Console output will show status codes, predicted class and top probability. Use these timestamps to align with Grafana.

### 6) Observe

Open Grafana and view your dashboard:

* Winner probability: `mec_ai_attack_level{vehicle="veh-A"}`
* Per-class probabilities: `mec_ai_prob{vehicle="veh-A"}`
* RNIS: `mec_rnis_rsrp{ue_id="UE1",cell_id="CELL-1"}`
* FL rounds: `mec_fl_rounds_total`
* Host/container health via node_exporter and cAdvisor

---

## MEC Edge API endpoints

* `GET /health`
* `POST /rnis/reports`
* `GET /rnis/reports/latest`
* `POST /location/devices`, `GET /location/devices/{ue_id}`
* `POST /ai/bootstrap`, `POST /ai/fl/train_local`, `POST /ai/fl/aggregate`, `GET /ai/info`
* `POST /ai/predict` → `{ predicted_class, probs{class: p} }`
* `GET /metrics` → Prometheus exposition of:

  * `mec_ai_predictions_total`
  * `mec_ai_attack_level{vehicle,predicted_class}`
  * `mec_ai_prob{vehicle,class_name}`
  * `mec_fl_rounds_total`
  * `mec_rnis_samples_total{ue_id,cell_id}`, `mec_rnis_rsrp{ue_id,cell_id}`

---

## Troubleshooting

* **UE not registering or no PDU**
  Check AMF, SMF, UPF bind addresses and PLMN/TAC. Ensure reachability from `192.168.56.105` or your host IP.

* **Prometheus target DOWN**
  Verify `http://192.168.56.105:8080/metrics`. Keep both scrape targets in `prometheus.yml`: `mec-api:8080` and `192.168.56.105:8080`. Restart Prometheus and MEC API.

* **Prediction 400 errors**
  Ensure generator keys match `feature_order` from `GET /ai/info`. Prefer `features_map` with exact key names.

* **Only DDoS vs Normal classes**
  Baseline training may be binary. To extend, retrain with labels such as `Normal, UDP, TCP_SYN, ICMP, DDoS`, aggregate with FedAvg, redeploy.

---

## Security and ethics

Use flood tools only inside an isolated lab environment that you own or control.

---

## Citation

If you use this work in academic contexts, please cite:

H. Pankhania, D. Sharma, H. Sharma, G. Kaur, A. Bandara. 5G Security Testing in Autonomous Mining Vehicles. Curtin University, 2025.

---

## Maintainer

For repo questions or issues, open a GitHub Issue and tag @hirdyasharma.
