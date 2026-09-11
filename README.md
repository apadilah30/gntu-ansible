# GNTU — Ansible Deployment & Observability

Ansible untuk men-deploy dan memantau sistem GNTU (FE + BE) pada **satu server Ubuntu**, secara **native** (systemd/binary, tanpa Docker), dengan **alerting Telegram**.

Repo aplikasi yang dikelola:
- **FE:** [`apadilah30/gntu-cms`](https://github.com/apadilah30/gntu-cms) — Laravel 11 + Inertia/React, disajikan oleh **nginx** dari `public/` dengan **PHP-FPM 8.3** (unix socket). Bukan Octane.
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
| | | phpfpm_exporter | 9253 |

**Akses Grafana dengan aman:** jangan buka port 3001 ke internet. Tambahkan reverse proxy di nginx (mis. `monitoring.domainanda`) dengan auth/VPN, lalu `proxy_pass http://127.0.0.1:3001`.

---

## 5. Menjawab masalah 504 intermiten Anda

Gejala: hang sewaktu-waktu → 504, **tanpa jejak di API gateway** (request belum sampai BE), padahal resource/traffik normal. Dari pembacaan kode + konfigurasi nginx/FPM, penyebab yang paling cocok ada di **lapisan FE (nginx + PHP-FPM)**, dan stack ini dipasang untuk **membuktikan dan menangkapnya**.

### 5.1 Mekanisme hang yang terjadi

```
user -> nginx -> PHP-FPM child -> Laravel Http::timeout(120)-> (menunggu gateway) -> ... 504
```

1. Request masuk ke **PHP-FPM** (via nginx fastcgi_pass).
2. Laravel memanggil backend lewat `Http::` (banyak tanpa timeout, sebagian `timeout(120)`).
3. Kalau gateway/downstream lambat, child FPM **tahan 120–300 detik** (= `fastcgi_read_timeout`).
4. Child lain juga ikut tertahan oleh request lain. Semua child habis (`pm.max_children` tercapai).
5. Request baru **antre di listen queue FPM** → akhirnya nginx 504 — **sebelum** request sempat jalan di Laravel, apalagi sampai ke API gateway.
6. CPU & memory tetap rendah karena child hanya menunggu I/O, bukan memproses.

Diperparah oleh: **`SESSION_DRIVER=database`** — session PostgreSQL pakai lock per-session, jadi request dari user yang sama saling menunggu lock, memperlambat semua child.

### 5.2 Perbaikan yang sangat disarankan (di repo aplikasi, bukan otomatis oleh Ansible)

Fokus utama di **FE (Laravel + nginx)**:

1. **Beri timeout ketat pada SEMUA panggilan `Http::` ke backend.** Ubah pola menjadi:
   ```php
   Http::timeout(8)->connectTimeout(3)-> ... ;
   ```
   Terutama di: `ProfileController`, `NewPasswordController`, `PasswordResetLinkController`, `RegisteredUserController`, `ContactController`, dan magic-login di `AuthenticatedSessionController`. Yang sekarang `timeout(120)` juga harus diturunkan.

2. **Pindahkan session, cache, dan queue dari database ke Redis.** Di `.env` FE:
   ```
   SESSION_DRIVER=redis
   CACHE_STORE=redis
   QUEUE_CONNECTION=redis
   ```
   lalu `php artisan config:cache && systemctl reload php8.3-fpm`.

3. **Turunkan `fastcgi_read_timeout` dari 300s ke 30–60s.** Ini jaring pengaman: kalau child macet, nginx langsung 504 dan child dibebaskan — bukan menyandera selama 5 menit.

4. **Tune FPM `pm.max_children`** sesuai RAM server. Rumus kasar: `Free RAM MB / avg child MB` (cek `ps --no-headers -o rss -C php-fpm8.3 | awk '{sum+=$1} END {print sum/NR/1024 " MB avg"}'`). Playbook menyediakan variabel `fpm_max_children` di `group_vars/all.yml`. Juga `request_terminate_timeout: 60s` agar child yang macet dibunuh otomatis.

5. **Sisi BE (gateway):** panggilan keluar `requests.get(obj.base_url)` di gateway **tanpa `timeout`**. Tambahkan `timeout=(3, 8)` pada semua `requests`/`aiohttp` agar satu downstream lambat tidak menyandera worker async. Playbook sudah menambahkan `--timeout 30` pada Gunicorn sebagai jaring pengaman terakhir.

### 5.3 Cara memakai monitoring untuk debug saat 504 terjadi

Buka Grafana → dashboard **"GNTU - Server & Services Overview"**:

- **Endpoint Up (blackbox)** & **nginx 504 rate** → konfirmasi kapan persisnya down.
- **PHP-FPM children: active vs total** → jika `active == total` dan `listen_queue > 0` saat hang = **child exhaustion** (ini bukti definitif). Ini panel paling penting untuk kasus Anda.
- **Gunicorn busy workers vs request rate** → jika worker "busy" tapi request rate ~0 = **worker stuck** (blocking call tanpa timeout) di sisi BE.
- **PostgreSQL locks** → jika `ExclusiveLock` melonjak saat hang = **kontensi lock session database** (perbaikan 5.2 #2).
- **CPU/Mem/Disk** → memastikan memang bukan resource (sesuai laporan Anda).
- **Panel Logs (Loki)** → filter `{job="nginx"}` cari `504`, lalu lihat journal log php-fpm/`gntu-api-gateway` pada detik yang sama.
- **PHP-FPM slowlog** (`/var/log/php8.3-fpm/gntu-cms-slow.log`) → stack trace dari request yang melebihi `request_slowlog_timeout`; ini menunjukkan **fungsi Laravel mana** yang sedang blocking.

### 5.3 Alert yang akan masuk Telegram

`EndpointDown`, `SlowEndpoint`, `Nginx504Spike`, `GunicornWorkersSaturated`, `PostgresLockWaits`, `RedisDown`, `PostgresDown`, `HostHighCPU`, `HostLowMemory`, `HostDiskAlmostFull`.

Setup bot: buat via **@BotFather** → dapat `bot_token`; dapatkan `chat_id` (mis. via @userinfobot atau `getUpdates`). Isi di `group_vars/vault.yml`.

---

### 5.4 Bila PHP-FPM tetap bermasalah — alternatif runtime yang lebih cepat & proper

PHP-FPM aman sebagai baseline (perbaikan 5.2 sudah cukup untuk mayoritas kasus). Tapi model "1 request = 1 child yang boot ulang framework tiap kali" memang boros dan rawan child-exhaustion saat ada I/O lambat. Jika setelah tuning 5.2 masih sering saturasi, pertimbangkan pindah ke runtime **persistent worker** berikut (urut dari paling direkomendasikan untuk kasus Anda):

| Opsi | Kelebihan | Catatan |
|---|---|---|
| **FrankenPHP (worker mode)** ⭐ | App di-boot sekali lalu tetap di memori → jauh lebih cepat & hemat, HTTP/2+3, bisa jadi web server sekaligus. Composer FE Anda **sudah** punya `laravel/octane`. | Ganti eksekusi PHP-FPM dengan FrankenPHP di depan/di belakang nginx. Wajib jalankan lewat **systemd** + `--max-requests` untuk cegah memory leak. |
| **Laravel Octane + RoadRunner** | Sama cepatnya, worker Go yang matang, mudah di-scale via jumlah worker. | Perlu jaga kode agar "stateless" (hindari state statis antar-request). |
| **Swoole/OpenSwoole** | Performa tertinggi + coroutine. | Ekstensi PECL, kurva belajar lebih tinggi, paling rawan bug state. |

**Rekomendasi:** **FrankenPHP worker mode** — karena dependency Octane sudah ada, transisinya paling kecil dan langsung menghilangkan overhead boot per-request.

**Penting apa pun runtime-nya:** kelebihan performa **tidak menghapus** akar 504 Anda. Dengan persistent worker, panggilan `Http::` yang lambat tanpa timeout justru **lebih berbahaya** (worker yang tertahan lebih sedikit jumlahnya). Jadi perbaikan **5.2 #1 (timeout Http::) dan #2 (session Redis) tetap wajib lebih dulu**, baru migrasi runtime.

**Kalau mau saya siapkan role-nya:** saya bisa tambahkan `frontend_runtime: fpm|frankenphp|roadrunner` di `group_vars/all.yml` dan role `frontend` yang mem-provision runtime terpilih via systemd (unit + `--workers`/`--max-requests` + reload tanpa downtime), plus exporter metrik worker-nya. Cukup beri tahu pilihan Anda. Metrik Grafana untuk mode ini memakai panel "busy workers vs request rate" yang sama.

## 6. Prasyarat instrumentasi aplikasi (opsional tapi disarankan)

Agar metrik per-endpoint Django muncul (job `django` di Prometheus), pasang [`django-prometheus`](https://github.com/korfuri/django-prometheus) di tiap service dan ekspos `/metrics`. Tanpa ini, worker-saturation tetap terlihat via job `gunicorn` (statsd). Untuk Celery, tambahkan `celery-exporter` bila diperlukan (belum diaktifkan default).

## 7. Catatan keamanan

- Semua exporter & komponen monitoring bind ke `127.0.0.1`. Jangan expose langsung.
- UFW diinstal tapi **tidak** di-enable otomatis (menghindari lockout). Aktifkan manual: izinkan 22/80/443, tutup sisanya.
- Secret (Grafana password, token Telegram, DSN Postgres) simpan di `group_vars/vault.yml` terenkripsi, bukan di `all.yml`.
