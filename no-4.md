# Mengubah Timezone Server dari UTC ke WIB (Asia/Jakarta)

## 1. Cek Timezone Saat Ini
```bash
timedatectl
# atau
date
cat /etc/timezone
```

## 2. Set Timezone ke WIB

**Cara paling umum (Linux modern, pakai `timedatectl`):**
```bash
sudo timedatectl set-timezone Asia/Jakarta
```

**Verifikasi:**
```bash
timedatectl
date
```
Pastikan output menunjukkan `Asia/Jakarta (WIB, +0700)`.

**Cara manual (kalau `timedatectl` tidak tersedia, misal di sistem lama/container minimal):**
```bash
sudo ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime

# Debian/Ubuntu, update juga file /etc/timezone
echo "Asia/Jakarta" | sudo tee /etc/timezone
sudo dpkg-reconfigure -f noninteractive tzdata
```

## 3. Sinkronisasi Waktu (NTP)

Pastikan waktu server tetap akurat setelah ganti timezone, aktifkan NTP sync:
```bash
sudo timedatectl set-ntp true
timedatectl status
```

## 4. Sesuaikan Konfigurasi Aplikasi/Service

Timezone OS berubah, tapi beberapa service **punya konfigurasi timezone sendiri** yang perlu disamakan manual:

**PHP** (`php.ini`):
```ini
date.timezone = "Asia/Jakarta"
```
```bash
sudo systemctl restart php-fpm
```

**MySQL:**
```sql
SET GLOBAL time_zone = '+07:00';
-- atau kalau timezone tables sudah di-load:
SET GLOBAL time_zone = 'Asia/Jakarta';
```
Permanen di `my.cnf`:
```ini
[mysqld]
default-time-zone = '+07:00'
```

**Aplikasi/Framework** (contoh Laravel `.env`/`config/app.php`):
```php
'timezone' => 'Asia/Jakarta',
```

**Cron jobs** — cek ulang jadwal cron, karena kalau sebelumnya jadwal di-set berdasarkan waktu UTC, waktunya perlu disesuaikan (mis. backup jam 2 pagi WIB = jam 19:00 UTC, kalau sebelumnya crontab di-set manual berdasarkan asumsi UTC).

**Docker container** (kalau aplikasi jalan di container terpisah, timezone host tidak otomatis kepakai):
```dockerfile
ENV TZ=Asia/Jakarta
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```
Atau mount volume:
```bash
docker run -v /etc/localtime:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro ...
```

## 5. Restart Service Terkait

```bash
sudo systemctl restart nginx
sudo systemctl restart php-fpm
sudo systemctl restart mysql
sudo systemctl restart cron
```

## Catatan Penting

- **Database dengan data timestamp existing**: kalau kolom timestamp di database disimpan sebagai UTC (praktik umum & disarankan), **jangan ubah datanya**. Cukup pastikan aplikasi yang melakukan konversi ke WIB saat display ke user. Mengubah timezone server tidak otomatis mengubah data yang sudah tersimpan.
- **Log aplikasi**: setelah timezone berubah, timestamp di log baru akan pakai WIB, tapi log lama masih UTC — perlu diperhatikan saat analisis log lintas waktu perubahan ini, terutama untuk correlate insiden.
- **Best practice sebenarnya**: banyak yang tetap menyimpan waktu di database & internal system pakai UTC (universal, tidak ambigu, aman dari isu daylight saving/perubahan zona), dan konversi ke WIB hanya dilakukan di **layer presentasi** (saat ditampilkan ke user). Kalau requirement-nya memang "server harus WIB" secara OS-level (misal untuk kebutuhan cron/log lokal yang lebih mudah dibaca tim), langkah di atas sudah cukup.
