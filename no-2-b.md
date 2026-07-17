b. Analisis Log: `max_children` PHP-FPM

```
WARNING: [pool www] server reached max_children setting (5), consider raising it
```

### Apa Artinya
PHP-FPM sudah mencapai batas maksimum **child process** yang boleh jalan bersamaan (di sini cuma di-set **5**). Request baru yang masuk setelah itu harus **antre** (queue) sampai ada child process yang selesai/free — inilah yang bikin user merasa akses lambat, bahkan bisa timeout kalau antreannya panjang.

### Langkah yang Dilakukan

**1. Konfirmasi dulu, jangan asal naikkan angka**
```bash
# Cek berapa banyak request yang sebenarnya nyangkut/pending
systemctl status php-fpm

# Cek status pool detail (aktifkan pm.status_path dulu kalau belum)
curl http://localhost/status?full
```

**2. Cek resource server tersedia berapa**
```bash
free -h
nproc
```
Karena kalau langsung naikkan `max_children` tanpa cek RAM, malah bisa bikin server **OOM** (Out of Memory) — child process baru butuh RAM, kalau RAM habis server bisa crash/swap berat, makin lambat.

**3. Hitung nilai `max_children` yang aman**

Rumus kasar:
```
max_children = (Total RAM tersedia untuk PHP-FPM) / (Rata-rata memory per proses PHP)
```
Contoh: RAM tersedia 2GB, rata-rata 1 proses PHP pakai 40MB →
```
max_children ≈ 2000MB / 40MB ≈ 50
```

Cek rata-rata pemakaian memory per proses:
```bash
ps aux | grep php-fpm | awk '{sum+=$6; count++} END {print sum/count/1024 " MB"}'
```

**4. Update konfigurasi** (`/etc/php/x.x/fpm/pool.d/www.conf`)
```ini
pm = dynamic
pm.max_children = 50
pm.start_servers = 10
pm.min_spare_servers = 5
pm.max_spare_servers = 15
pm.max_requests = 500
```

**5. Restart PHP-FPM**
```bash
systemctl restart php-fpm
```

**6. Cari root cause, bukan cuma naikkan angka**
- Kenapa child process cepat penuh? Cek apakah ada query lambat/proses PHP yang nyangkut lama (harusnya idealnya request PHP cepat selesai lalu child langsung free lagi)
- Pasang `pm.status_path` dan monitoring (Prometheus php-fpm exporter + Grafana) supaya ada alert sebelum kejadian lagi
- Pertimbangkan juga **PHP-OPcache** untuk mempercepat eksekusi PHP, sehingga proses lebih cepat selesai dan child lebih cepat free

---
