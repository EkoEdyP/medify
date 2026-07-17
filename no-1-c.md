C. Setup Replikasi Master-Master MySQL

Asumsi 2 server: **Server A** (IP A) dan **Server B** (IP B).

## 1. Konfigurasi `my.cnf`

**Server A:**
```ini
[mysqld]
server-id = 1
log_bin = mysql-bin
binlog_format = ROW
auto_increment_increment = 2
auto_increment_offset = 1
log-slave-updates = 1
```

**Server B:**
```ini
[mysqld]
server-id = 2
log_bin = mysql-bin
binlog_format = ROW
auto_increment_increment = 2
auto_increment_offset = 2
log-slave-updates = 1
```

> Restart MySQL setelah edit konfigurasi di kedua server.

| Parameter | Fungsi |
|---|---|
| `server-id` | ID unik tiap server, wajib beda antar node |
| `log_bin` | Mengaktifkan binary log, dasar dari replikasi |
| `binlog_format = ROW` | Mencatat perubahan per-baris data, lebih akurat dan konsisten |
| `auto_increment_increment` | Interval lompatan nilai auto-increment (2 = lompat 2 tiap insert) |
| `auto_increment_offset` | Titik awal beda tiap server, mencegah bentrok PK saat insert bersamaan |
| `log-slave-updates` | Perubahan yang diterima dari replikasi ikut dicatat lagi ke binlog sendiri |

---

## 2. Buat User Replikasi (di masing-masing server)

```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'repl_password_kuat';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

---

## 3. Ambil Posisi Binlog

**Di Server A:**
```sql
SHOW MASTER STATUS;
```
Catat `File` dan `Position`, misal `mysql-bin.000001`, `154`.

**Di Server B:**
```sql
SHOW MASTER STATUS;
```
Catat juga `File` dan `Position` milik Server B.

---

## 4. Hubungkan B sebagai Slave dari A

**Di Server B:**
```sql
CHANGE MASTER TO
  MASTER_HOST='IP_A',
  MASTER_USER='repl',
  MASTER_PASSWORD='repl_password_kuat',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;
START SLAVE;
```

---

## 5. Hubungkan A sebagai Slave dari B

**Di Server A:**
```sql
CHANGE MASTER TO
  MASTER_HOST='IP_B',
  MASTER_USER='repl',
  MASTER_PASSWORD='repl_password_kuat',
  MASTER_LOG_FILE='mysql-bin.<file_B>',
  MASTER_LOG_POS=<position_B>;
START SLAVE;
```

> Dua `CHANGE MASTER TO` di atas (langkah 4 & 5) adalah **inti dari master-master** — masing-masing server dijadikan slave bagi server lainnya, sehingga replikasi jalan dua arah.

---

## 6. Verifikasi di Kedua Sisi

```sql
SHOW SLAVE STATUS\G
```

Pastikan:
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0`

---

## Catatan Penting

- **Auto-increment offset** wajib beda di tiap server (1 dan 2), supaya PK auto-increment tidak bentrok kalau insert terjadi bersamaan di kedua sisi.
- **Write conflict** adalah risiko utama master-master — kalau row yang sama diubah di kedua sisi secara bersamaan, MySQL native replication tidak punya conflict resolution otomatis. Idealnya salah satu node tetap jadi primary write, node lain untuk failover/read saja.
- Untuk database `xyz_%` yang sudah ada isinya, lakukan `mysqldump` full dari master awal lalu restore ke server kedua **sebelum** `START SLAVE`, supaya data konsisten sebelum replikasi berjalan.
