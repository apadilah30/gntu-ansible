# GNTU — Ansible Deployment & Observability

Ansible untuk men-deploy dan memantau sistem GNTU (FE + BE) pada **satu server Ubuntu**, secara **native** (systemd/binary, tanpa Docker), dengan **alerting Telegram**.

Repo aplikasi yang dikelola:
- **FE:** [`apadilah30/gntu-cms`](https://github.com/apadilah30/gntu-cms) — Laravel 11 + Inertia/React, dijalankan via **Octane (FrankenPHP)**.
- **BE:** [`apadilah30/gntu-be-microservices`](https://github.com/apadilah30/gntu-be-microservices) — Django microservices via **Gunicorn + UvicornWorker (ASGI)**, Celery, RabbitMQ, Redis, + websocket Node/Socket.IO.

Proxy: **nginx** → domain FE, dengan `/api-gateway` di-proxy ke BE.

---

## 1. Struktur

```
ansible/
├── ansible.cfg
├── inventory/hosts.yml          # 1 server, group: frontend/backend/websocket/monitoring
├── group_vars/all.yml           # SEMUA variabel terpusat (ports, paths, versi, dll)
├── group_vars/vault.example.yml # contoh secret (encrypt jadi vault.yml)
├── site.yml                     # orchestrator penuh (pakai tags)
├── playbooks/
│   ├── deploy-frontend.yml
│   ├── deploy-backend.yml
│   └── deploy-monitoring.yml
└── roles/
    ├── common/      # timezone, paket dasar, ufw (tidak auto-enable)
    ├── nginx/       # stub_status utk exporter
    ├── backend/     # gunicorn+uvicorn systemd (timeout & workers benar)
    ├── frontend/    # Laravel Octane systemd + build
    ├── websocket/   # Node socket systemd (pengganti PM2)
    └── monitoring/  # Prometheus, Grafana, Loki/Promtail, Alertmanager, exporters
```

## 2. Prasyarat

- Kontrol node dengan `ansible` (>=2.15) + collection: `ansible-galaxy collection install community.general`.
- Akses SSH (sudo) ke server. Isi `inventory/hosts.yml` (IP, user, key).
- Edit `group_vars/all.yml` — minimal ganti semua nilai `CHANGE_ME`, path, dan repo branch.
- Simpan secret di `group_vars/vault.yml` (encrypt dengan `ansible-vault`).

## 3. Cara pakai

```bash
# Semua sekaligus (base + FE + BE + monitoring)
ansible-playbook site.yml --ask-vault-pass

# Hanya sebagian (via tag)
ansible-playbook site.yml --tags monitoring
ansible-playbook site.yml --tags frontend
ansible-playbook site.yml --tags backend

# Atau via playbook terpisah
ansible-playbook playbooks/deploy-monitoring.yml
ansible-playbook playbooks/deploy-frontend.yml
ansible-playbook playbooks/deploy-backend.yml

# Cek dulu tanpa mengubah apa pun
ansible-playbook site.yml --check --diff
```

## 4. Peta port (semua monitoring bind ke 127.0.0.1)

| Komponen | Port | Komponen | Port |
|---|---|---|---|
| Grafana | 3001 | node_exporter | 9100 |
| Prometheus | 9090 | nginx_exporter | 9113 |
| Alertmanager | 9093 | redis_exporter | 9121 |
| Loki | 3100 | postgres_exporter | 9187 |
| Promtail | 9080 | blackbox_exporter | 9115 |
| nginx stub_status | 8080 | statsd_exporter | 9125(udp)/9102 |

**Akses Grafana dengan aman:** jangan buka port 3001 ke internet. Tambahkan reverse proxy di nginx (mis. `monitoring.domainanda`) dengan auth/VPN, lalu `proxy_pass http://127.0.0.1:3001`.

---

## 5. Menjawab masalah 504 intermiten Anda

Gejala: hang sewaktu-waktu → 504, **tanpa jejak di API gateway** (request belum sampai BE), padahal resource/traffik normal. Dari pembacaan kode, penyebab yang paling cocok ada di **lapisan sebelum backend (nginx + Laravel/Octane)** dan di **gateway** untuk kasus lain. Stack ini dipasang untuk **membuktikan dan menangkapnya**.

### 5.1 Perbaikan yang sangat disarankan (di repo aplikasi, bukan otomatis oleh Ansible)

Karena "hang belum masuk backend", fokus utama di **FE (Laravel)**:

1. **Pindahkan session, cache, dan queue dari database ke Redis.** Di `.env` FE saat ini `SESSION_DRIVER=database`, `CACHE_STORE=database`, `QUEUE_CONNECTION=database`. Session database memakai **lock per-session**; di Octane (worker long-running) request untuk user yang sama saling menunggu lock → hang tanpa error, resource rendah. Ganti:
   ```
   SESSION_DRIVER=redis
   CACHE_STORE=redis
   QUEUE_CONNECTION=redis
   ```
   lalu `php artisan config:cache && systemctl restart gntu-cms-octane`.

2. **Beri timeout ketat pada SEMUA panggilan `Http::` ke backend.** Beberapa call tidak punya timeout, dan yang ada memakai `timeout(120)` (120 detik). Jika gateway lambat, worker Octane ketahan sampai 120 detik sambil memegang lock session → worker habis → request baru hang → nginx 504, semua **sebelum** ada log gateway. Ubah pola menjadi:
   ```php
   Http::timeout(8)->connectTimeout(3)-> ... ;
   ```
   Terutama di: `HandleInertiaRequests` (dipakai tiap halaman), `ProfileController`, `NewPasswordController`, `PasswordResetLinkController`, `RegisteredUserController`, `ContactController`, dan magic-login.

3. **Tune worker Octane** (`frontend_octane_workers` di `all.yml`) ~ jumlah CPU core. Worker terlalu sedikit = antrean di dalam Octane.

4. **Sisi BE (gateway):** panggilan keluar `requests.get(obj.base_url)` di gateway **tanpa `timeout`**. Tambahkan `timeout=(3, 8)` pada semua `requests`/`aiohttp` agar satu downstream lambat tidak menyandera worker async. Playbook sudah menambahkan `--timeout 30` pada Gunicorn sebagai jaring pengaman terakhir.

### 5.2 Cara memakai monitoring untuk debug saat 504 terjadi

Buka Grafana → dashboard **"GNTU - Server & Services Overview"**:

- **Endpoint Up (blackbox)** & **nginx 504 rate** → konfirmasi kapan persisnya down.
- **Gunicorn busy workers vs request rate** → jika worker "busy" tapi request rate ~0 = **worker stuck** (blocking call tanpa timeout). Ini bukti akar masalah.
- **PostgreSQL locks** → jika `ExclusiveLock` melonjak saat hang = **kontensi lock session database** (perbaikan 5.1 #1).
- **CPU/Mem/Disk** → memastikan memang bukan resource (sesuai laporan Anda).
- **Panel Logs (Loki)** → filter `{job="nginx"}` cari `504`, lalu lihat log unit `gntu-cms-octane`/`gntu-api-gateway` pada detik yang sama.

### 5.3 Alert yang akan masuk Telegram

`EndpointDown`, `SlowEndpoint`, `Nginx504Spike`, `GunicornWorkersSaturated`, `PostgresLockWaits`, `RedisDown`, `PostgresDown`, `HostHighCPU`, `HostLowMemory`, `HostDiskAlmostFull`.

Setup bot: buat via **@BotFather** → dapat `bot_token`; dapatkan `chat_id` (mis. via @userinfobot atau `getUpdates`). Isi di `group_vars/vault.yml`.

---

## 6. Prasyarat instrumentasi aplikasi (opsional tapi disarankan)

Agar metrik per-endpoint Django muncul (job `django` di Prometheus), pasang [`django-prometheus`](https://github.com/korfuri/django-prometheus) di tiap service dan ekspos `/metrics`. Tanpa ini, worker-saturation tetap terlihat via job `gunicorn` (statsd). Untuk Celery, tambahkan `celery-exporter` bila diperlukan (belum diaktifkan default).

## 7. Catatan keamanan

- Semua exporter & komponen monitoring bind ke `127.0.0.1`. Jangan expose langsung.
- UFW diinstal tapi **tidak** di-enable otomatis (menghindari lockout). Aktifkan manual: izinkan 22/80/443, tutup sisanya.
- Secret (Grafana password, token Telegram, DSN Postgres) simpan di `group_vars/vault.yml` terenkripsi, bukan di `all.yml`.
