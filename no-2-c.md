c. Analisis Log: PHP Fatal Error - Memory Exhausted

```
PHP Fatal error: Allowed memory size of 134217728 bytes exhausted 
(tried to allocate 20480 bytes) in .../PDOStatement.php on line 262
```

### Apa Artinya
- `134217728 bytes` = **128 MB** → ini adalah nilai `memory_limit` PHP saat ini
- Error terjadi di `PDOStatement.php` — artinya proses ini kehabisan memory pas lagi **fetch data dari database** (kemungkinan query yang hasilnya terlalu besar dimuat semua ke memory sekaligus)
- Ini penyebab lain dari "akses lambat" — request yang fatal error biasanya bikin user nunggu lama duluan sebelum akhirnya error/gagal, dan proses PHP yang crash juga bisa berkontribusi ke munculnya masalah `max_children` di poin b

### Langkah yang Dilakukan

**1. Jangan langsung asal naikkan `memory_limit`**

Naikkan `memory_limit` itu solusi **sementara/simptomatis**, bukan solusi akar masalah — karena kalau query-nya memang boros memory, masalah cuma tertunda dan bisa muncul lagi dengan data yang lebih besar, plus makin banyak RAM server yang terpakai per request.

**2. Investigasi dulu, cari query/kode penyebabnya**
```bash
# Cari full stack trace di log, biasanya PHP fatal error tercatat lengkap di sini
grep -B 5 -A 5 "Allowed memory size" /var/log/php-fpm/www-error.log
```

Cek kode di sekitar baris yang disebut (`PDOStatement.php` line 262 dipanggil dari kode aplikasi mana):
```bash
grep -rn "fetchAll\|->fetch(" /var/www/apps/src/ | grep -i "repository\|query"
```

Kemungkinan besar penyebabnya:
- Query pakai `fetchAll()` untuk data dalam jumlah besar (ratusan ribu row) dimuat sekaligus ke array PHP
- Query tanpa `LIMIT`/pagination
- Ada `JOIN` yang menghasilkan cartesian product besar
- Loop yang nge-fetch data berulang tanpa `unset()`/pembersihan memory

**3. Perbaikan di level kode (root cause fix)**
- Gunakan **pagination/limit** pada query besar, jangan `fetchAll()` semua data
- Gunakan **cursor/generator** (`yield`, `fetch()` per baris) untuk data besar, bukan load semua ke array sekaligus
- Untuk Doctrine DBAL khususnya, pertimbangkan pakai `iterateResult()` atau batch processing

**4. Cek dulu apakah ini query yang memang wajar butuh resource besar**

Kalau memang by design butuh proses data besar (misal export/report), sebaiknya:
- Pisahkan proses tersebut ke **job/queue terpisah** (background job, misal pakai queue worker), jangan lewat HTTP request biasa yang punya batas memory ketat
- Beri `memory_limit` khusus lebih tinggi hanya untuk script tersebut, bukan menaikkan global `memory_limit` semua request

```ini
; hanya untuk script tertentu, via php_admin_value di pool config khusus
php_admin_value[memory_limit] = 512M
```

**5. Kalau memang diperlukan, naikkan `memory_limit` sebagai mitigasi sementara**

`/etc/php/x.x/fpm/php.ini`:
```ini
memory_limit = 256M
```
```bash
systemctl restart php-fpm
```
Tapi ini harus dibarengi rencana fix kode di kemudian hari, bukan solusi permanen.

**6. Tambahkan monitoring/alerting**
- Pasang alert kalau ada PHP fatal error muncul di log (misal via Grafana Loki + Alertmanager, atau ELK stack)
- Cek **APM** untuk lihat query/endpoint mana yang paling boros memory & waktu eksekusi
