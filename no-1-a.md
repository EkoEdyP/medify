a. Grant Akses User joni ke Semua Database xyz_*
Karena prefix xyz_ mengandung underscore, ingat bahwa _ dan % adalah wildcard di MySQL GRANT statement. Kalau mau match literal underscore, escape pakai backslash:

```sql
-- Membuat user MySQL baru bernama 'joni'
-- '%' artinya joni bisa login dari host mana saja (semua IP)
-- Kalau mau lebih aman, ganti '%' dengan IP spesifik, misal 'joni'@'192.168.1.10'
CREATE USER 'joni'@'%' IDENTIFIED BY 'password_yang_kuat';

-- Memberikan hak akses PENUH (SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, dll)
-- ke semua database yang namanya diawali 'xyz_'
-- Tanda backslash (\) sebelum underscore WAJIB dipakai,
-- karena underscore (_) di MySQL adalah wildcard (artinya "1 karakter apa saja")
-- Kalau tidak di-escape, 'xyz_%' bisa match ke nama lain juga, misal 'xyzA_test'
-- Tanda * setelah nama db artinya semua tabel di dalam database tersebut
GRANT ALL PRIVILEGES ON `xyz\_%`.* TO 'joni'@'%';

-- Me-reload tabel privilege di memory MySQL
-- supaya perubahan hak akses langsung berlaku tanpa perlu restart service
FLUSH PRIVILEGES;
```

```sql
-- Menampilkan semua hak akses (grant) yang dimiliki user 'joni'
-- Berguna buat mastiin GRANT tadi sudah kesimpan dengan benar
SHOW GRANTS FOR 'joni'@'%';
```

###### Catatan tambahan soal xyz\_%:
- `xyz\_%` akan match: xyz_db1, xyz_db2, xyz_apapun ✅
- Kalau ditulis tanpa escape `(xyz_%)`, maka `_` dianggap wildcard "1 karakter bebas", jadi bisa saja ikut match ke nama seperti `xyzXdb1` (huruf X menggantikan underscore) ❌ — ini yang mau dihindari.

---
