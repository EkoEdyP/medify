a. Pengecekan yang Dilakukan

### 1. Cek Level Infrastruktur (Resource Server)
```bash
# CPU, Memory, Load Average
top -c
htop
uptime

# Disk I/O
iostat -x 1 5
iotop

# Disk space (space penuh bisa bikin lambat/error)
df -h
du -sh /var/log/* 

# Network
netstat -tulnp
ss -s
```

### 2. Cek Level Aplikasi Web Server (Nginx/Apache)
```bash
# Cek jumlah koneksi aktif
netstat -an | grep :80 | wc -l

# Cek error log
tail -f /var/log/nginx/error.log
tail -f /var/log/apache2/error.log

# Cek access log, cari request yang response time-nya tinggi
tail -f /var/log/nginx/access.log
```

### 3. Cek Level PHP-FPM (kalau stack pakai PHP)
```bash
# Status pool PHP-FPM
systemctl status php-fpm
tail -f /var/log/php-fpm/www-error.log

# Cek slow log kalau diaktifkan
tail -f /var/log/php-fpm/slow.log
```

### 4. Cek Level Database
```sql
-- Proses yang sedang berjalan, cari query yang lama
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- Slow query log
SHOW VARIABLES LIKE 'slow_query_log%';
```
```bash
tail -f /var/log/mysql/slow-query.log
```

### 5. Cek Level Aplikasi
- Cek APM/monitoring tools (New Relic, Datadog) atau **Grafana + Prometheus** kalau sudah ada
- Cek apakah ada deployment/perubahan kode terbaru yang bertepatan dengan mulai lambatnya sistem
- Cek cache (Redis/Memcached) — apakah hit rate turun atau service down

### 6. Cek dari Sisi Network/Eksternal
```bash
# Test response time langsung
curl -w "@curl-format.txt" -o /dev/null -s https://domain.com

# Cek DNS resolution
dig domain.com
```

### Ringkasan Alur Pengecekan
```
User complain → Cek resource server (CPU/RAM/Disk/Network)
             → Cek web server (Nginx/Apache error & access log)
             → Cek PHP-FPM (pool status, error log)
             → Cek database (slow query, process list)
             → Cek aplikasi (recent deployment, cache)
```

---
