# Edge-AI Secure Surveillance System

Capstone project yang mengintegrasikan **Computer Vision (Proyek 1: Smart Parking Detection)** dan **Network Security Monitoring (Proyek 2)** ke dalam satu sistem surveillance terdistribusi. Ide utamanya: edge node menjalankan deteksi objek secara real-time, sementara node terpisah mengawasi *jalur komunikasi* antar node itu sendiri — jadi sistem tidak hanya "melihat" dunia fisik lewat CV, tapi juga mengamankan jaringan yang menghubungkan komponen-komponennya.

## Latar Belakang & Framing

Proyek 1 dan Proyek 2 dikembangkan sebagai dua sistem terpisah:
- **Proyek 1** — model YOLOv8n untuk deteksi parkir (98.5% mAP50), di-deploy lewat FastAPI + Streamlit.
- **Proyek 2** — `network_monitor.py` (port scan & threat intel detection via Scapy) dan `tls_inspector.py` (TLS/SSL ClientHello & certificate inspection).

Capstone ini menyatukan keduanya dengan pembagian tanggung jawab yang jelas: **edge node** menjalankan CV, **central server** menerima hasil deteksi sekaligus menjadi titik observasi bagi **security monitor** yang mengawasi trafik jaringan antar node.

## Arsitektur

```
┌─────────────────────┐         HTTPS (self-signed cert)        ┌──────────────────────────┐
│   Edge Node          │ ────────────────────────────────────▶ │   Central Server          │
│   (Docker container)│   POST /ingest  →  :8443                │   (Docker container)      │
│                      │                                          │                            │
│  - YOLOv8n detection │                                          │  - Menerima alert deteksi  │
│  - Input: video file │                                          │  - Streamlit dashboard     │
│    (looping, simulasi│                                          │    (port 8501)             │
│    CCTV)              │                                          │  - Live YOLO detection +   │
└─────────────────────┘                                          │    security alerts          │
                                                                    └──────────────────────────┘
                                                                              ▲
                                                                              │ sniffing live traffic
                                                                    ┌──────────────────────────┐
                                                                    │   Security Monitor         │
                                                                    │   (Docker container)      │
                                                                    │  - network_monitor.py      │
                                                                    │    (port scan + threat     │
                                                                    │    intel / Feodo Tracker)  │
                                                                    │  - tls_inspector.py         │
                                                                    │    (ClientHello inspection) │
                                                                    │  - Heartbeat log tiap 200   │
                                                                    │    paket                    │
                                                                    └──────────────────────────┘
```

**Keputusan desain kunci:**
- Topologi disimulasikan sebagai **2+ mesin terpisah** (edge node, central server, security monitor) via Docker network — bukan localhost — supaya trafik jaringan antar node realistis dan bisa benar-benar diawasi.
- Edge node memakai **video file sample (looping)** sebagai input, mensimulasikan feed CCTV, bukan webcam/gambar statis.
- Komunikasi edge → central di-upgrade dari HTTP ke **HTTPS dengan self-signed certificate**.

## Komponen

| Komponen | Deskripsi | Asal |
|---|---|---|
| `yolo_detector` (edge node) | Menjalankan YOLOv8n pada video loop, kirim hasil deteksi ke central server | Proyek 1 |
| `central-server` | Menerima alert via HTTPS `/ingest` (port 8443), serving Streamlit dashboard (port 8501) | Baru (capstone) |
| `security-monitor` | `network_monitor.py` + `tls_inspector.py` — port scan detection, threat intel cross-reference, TLS inspection | Proyek 2 |

## Cara Menjalankan

```bash
# Clone repo
git clone https://github.com/fale2323/<repo-name>.git
cd <repo-name>

# Jalankan seluruh stack
docker compose up --build

# Dashboard bisa diakses di:
# http://localhost:8501
```

Environment variable opsional:
- `VERBOSE=1` pada container `security-monitor` — mengaktifkan logging detail per-paket (default: hanya heartbeat tiap 200 paket + alert).

## Fitur Dashboard

Dashboard Streamlit di central-server menyatukan dua sumber data secara live, auto-refresh setiap 3 detik:
1. **Live YOLO detection** — hasil deteksi objek dari edge node.
2. **Security alerts** — port scan attempts, threat intel matches (Feodo Tracker), dan temuan TLS/SSL dari security-monitor.

## Known Limitations & Root Cause Analysis

Konsisten dengan pendekatan error analysis di Proyek 1 & 2, berikut limitasi yang sudah teridentifikasi dan didokumentasikan (bukan disembunyikan):

1. **Certificate inspection gagal pada sertifikat besar** — sertifikat yang fragmented across multiple TCP packets tidak berhasil di-parse oleh `tls_inspector.py`.
   - **Root cause:** Scapy tidak melakukan TCP stream reassembly secara built-in, sehingga inspector hanya membaca payload per-paket, bukan payload yang sudah digabung ulang.
   - **Dampak:** ClientHello inspection (TLS version, cipher suite, SNI) tetap 100% andal karena umumnya muat dalam satu paket; certificate inspection untuk sertifikat besar tidak.

2. **Deteksi objek gagal pada wide-shot dengan banyak kendaraan kecil** (diwarisi dari Proyek 1).
   - **Root cause:** Skala objek relatif terhadap frame — bukan sudut kamera — yang membuat model kesulitan mendeteksi objek kecil pada frame lebar.

## Riwayat Pengembangan

Capstone ini dibangun bertahap (Step 1–5):
1. Framing arsitektur & pembagian tanggung jawab CV vs security.
2. Setup topologi Docker multi-container (edge node vs central server terpisah).
3. Upgrade komunikasi edge → central dari HTTP ke HTTPS (self-signed cert), diverifikasi lewat status 200 konsisten di `/ingest`.
4. Integrasi `network_monitor.py` dan `tls_inspector.py` ke dalam container `security-monitor`, dengan fix `libpcap-dev` di Dockerfile agar Scapy bisa live sniffing.
5. Dashboard Streamlit yang menyatukan live YOLO detection dan security alerts, auto-refresh 3 detik, plus heartbeat log untuk indikasi sistem tetap aktif.

## Proyek Terkait

- Proyek 1 — Smart Parking Detection: [github.com/fale2323/smart-parking-detection](https://github.com/fale2323/smart-parking-detection)
- Proyek 2 — Network Security Monitor: [github.com/fale2323/network-security-monitor](https://github.com/fale2323/network-security-monitor)
