# Tools Monitoring Server (CPU, Memory, Storage)

## Tools yang Biasa Digunakan

| Kategori | Tools |
|---|---|
| **Metrics collection & time-series DB** | Prometheus |
| **Visualisasi/dashboard** | Grafana |
| **Exporter (agent di tiap server)** | Node Exporter |
| **Alerting** | Alertmanager |
| **Command-line cepat (real-time check)** | `htop`, `df -h`, `iostat`, `vmstat`, `netdata` |
| **Alternatif all-in-one (kalau butuh lebih ringan/cepat setup)** | Netdata, Zabbix |
| **Log-based monitoring (tambahan)** | Loki (dipasangkan dengan Grafana) |
| **Cloud-native (kalau di AWS)** | CloudWatch |

Saya biasa pakai kombinasi **Prometheus + Node Exporter + Grafana + Alertmanager** sebagai stack utama karena open-source, fleksibel, dan sudah terbiasa dipakai di project-project sebelumnya (termasuk untuk monitoring Dumb Merch App deployment).

---

## Tahapan Persiapan

### 1. Install & Konfigurasi Prometheus (Server Monitoring Pusat)

```bash
# Download & extract Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz
tar xvf prometheus-2.53.0.linux-amd64.tar.gz
sudo mv prometheus-2.53.0.linux-amd64 /opt/prometheus

# Buat user khusus (best practice, jangan run pakai root)
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /opt/prometheus
```

Konfigurasi `/opt/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100', '10.0.0.2:9100', '10.0.0.3:9100']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
```

Buat systemd service (`/etc/systemd/system/prometheus.service`):
```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/data \
  --storage.tsdb.retention.time=30d

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

---

### 2. Install Node Exporter (di Setiap Server yang Ingin Dimonitor)

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/

sudo useradd --no-create-home --shell /bin/false node_exporter
```

Systemd service:
```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Node Exporter otomatis expose metrics CPU, memory, disk, network di `http://server:9100/metrics` — inilah yang di-scrape Prometheus secara berkala.

**Buka port firewall** (kalau Prometheus server terpisah dari target):
```bash
sudo ufw allow from <IP_PROMETHEUS_SERVER> to any port 9100
```

---

### 3. Install & Konfigurasi Grafana (Visualisasi)

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y

sudo systemctl enable --now grafana-server
```

**Tambahkan Prometheus sebagai Data Source:**
- Login ke Grafana (`http://server:3000`, default user/pass `admin/admin`)
- Configuration → Data Sources → Add → Prometheus → masukkan URL `http://localhost:9090`

**Import Dashboard siap pakai** (hemat waktu, tidak perlu bikin dari nol):
- Dashboard ID `1860` (Node Exporter Full) — dashboard komunitas yang sudah lengkap untuk CPU, memory, disk, network

---

### 4. Setup Alerting dengan Alertmanager

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64 /opt/alertmanager
```

Buat rule alert (`/opt/prometheus/alert_rules.yml`):
```yaml
groups:
  - name: server_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage tinggi di {{ $labels.instance }}"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage tinggi di {{ $labels.instance }}"

      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space kritis di {{ $labels.instance }}, tersisa < 15%"
```

Konfigurasi notifikasi Alertmanager (`/opt/alertmanager/alertmanager.yml`) — biasanya diarahkan ke Telegram/Slack:
```yaml
route:
  receiver: 'telegram-notifications'

receivers:
  - name: 'telegram-notifications'
    telegram_configs:
      - bot_token: '<BOT_TOKEN>'
        chat_id: <CHAT_ID>
        api_url: 'https://api.telegram.org'
```
```bash
sudo systemctl enable --now alertmanager
```

---

### 5. Validasi & Testing

```bash
# Cek Prometheus berhasil scrape target
curl http://localhost:9090/api/v1/targets

# Cek metrics langsung dari node_exporter
curl http://localhost:9100/metrics | grep node_cpu

# Test alert dengan simulasi (misal stress test CPU)
sudo apt install stress -y
stress --cpu 4 --timeout 300
```
Pastikan alert muncul di Grafana/Alertmanager dan notifikasi terkirim.

---

### 6. Maintenance & Review Berkala

- Set **retention period** data Prometheus sesuai kapasitas storage (`--storage.tsdb.retention.time=30d`)
- Review threshold alert secara berkala — sesuaikan dengan baseline normal traffic aplikasi (hindari alert fatigue karena threshold terlalu sensitif)
- Backup konfigurasi Grafana dashboard (export JSON) dan simpan di repository/version control

---

## Ringkasan Alur Setup

```
Node Exporter (tiap server) → expose metrics port 9100
        ↓
Prometheus (server pusat) → scrape metrics tiap 15s, simpan time-series
        ↓
Grafana → query Prometheus, tampilkan dashboard visual
        ↓
Alertmanager → terima alert dari Prometheus rule, kirim notifikasi (Telegram/Slack)
```
