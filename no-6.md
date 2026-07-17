# Rotasi Public SSH Key dengan Masa Berlaku 1 Tahun

Ada dua pendekatan utama: **(A) expiry di `authorized_keys`** (cara simpel, native OpenSSH) dan **(B) SSH Certificate** (cara lebih robust/scalable). Saya jelaskan keduanya.

---

## A. Menggunakan Expiry Option di `authorized_keys` (OpenSSH ≥ 8.2)

OpenSSH punya opsi `expiry-time` yang bisa ditempel langsung di entry `authorized_keys`, jadi key otomatis tidak berlaku lagi setelah tanggal tertentu — tanpa perlu manual hapus.

### 1. Cek versi OpenSSH dulu
```bash
ssh -V
```
Fitur ini butuh minimal OpenSSH 8.2 (Ubuntu 20.04+ sudah support).

### 2. Tambahkan key dengan opsi expiry
```bash
# Format: expiry-time="YYYYMMDD" ssh-rsa AAAA... comment
echo 'expiry-time="20270717" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... joni@laptop' >> ~/.ssh/authorized_keys
```
Tanggal `20270717` artinya key ini otomatis invalid setelah 17 Juli 2027 (persis 1 tahun dari sekarang, 17 Juli 2026).

### 3. Verifikasi
```bash
cat ~/.ssh/authorized_keys
```
Kalau tanggal sudah lewat, SSH otomatis menolak key tersebut walau baris-nya masih ada di file — tidak perlu manual hapus tepat waktu.

### Kekurangan Cara Ini
- Manual per-user, per-server — kalau banyak server & banyak user, jadi tidak scalable
- Harus ada proses **tracking** kapan expired, dan proses **rotasi ulang** (generate key baru + distribusi) tetap manual

---

## B. Menggunakan SSH Certificate (Direkomendasikan untuk Skala Lebih Besar)

Ini pendekatan yang lebih standar di environment production — pakai **CA (Certificate Authority) key** milik sendiri untuk sign public key user, dengan masa berlaku eksplisit.

### 1. Generate CA Key (sekali saja, disimpan aman)
```bash
ssh-keygen -t ed25519 -f /etc/ssh/ca_user_key -C "internal-ssh-ca"
```
> `ca_user_key` (private) harus disimpan sangat aman, idealnya di HSM/vault, bukan sembarang server.

### 2. Konfigurasi Server agar Trust CA
Di `/etc/ssh/sshd_config`:
```ini
TrustedUserCAKeys /etc/ssh/ca_user_key.pub
```
```bash
sudo systemctl restart sshd
```

### 3. Sign Public Key User dengan Validity 1 Tahun
User submit public key mereka (`id_ed25519.pub`), lalu admin sign:
```bash
ssh-keygen -s /etc/ssh/ca_user_key \
  -I "joni-2026" \
  -n joni \
  -V +52w \
  id_ed25519.pub
```
Penjelasan opsi:
- `-s` — CA private key untuk sign
- `-I` — identity/label sertifikat (buat tracking)
- `-n` — restrict certificate hanya untuk username `joni`
- `-V +52w` — **validity period, berlaku 52 minggu (~1 tahun) dari sekarang**, setelah itu otomatis expired
- Hasil: file `id_ed25519-cert.pub` (sertifikat), dikirim balik ke user

### 4. User Pakai Certificate untuk Login
```bash
ssh -i ~/.ssh/id_ed25519 -o CertificateFile=~/.ssh/id_ed25519-cert.pub user@server
```
Atau taruh di `~/.ssh/config`:
```
Host server1
  HostName server1.example.com
  User joni
  IdentityFile ~/.ssh/id_ed25519
  CertificateFile ~/.ssh/id_ed25519-cert.pub
```

### 5. Cek Detail Sertifikat & Masa Berlaku
```bash
ssh-keygen -L -f id_ed25519-cert.pub
```
Output menampilkan `Valid: from ... to ...` — jadi expiry jelas dan bisa diaudit.

### Keuntungan Cara Ini
- **Terpusat**: satu CA key bisa dipakai untuk semua server, tidak perlu edit `authorized_keys` di tiap server satu-satu
- **Otomatis expired** — tidak perlu manual hapus/tracking per file
- **Revocation** juga bisa dilakukan terpusat lewat `RevokedKeys` di `sshd_config`
- Cocok untuk environment dengan banyak server & banyak engineer

---

## Proses Rotasi Tahunan (Workflow)

```
1. H-30 hari sebelum expired → sistem/admin kirim reminder ke user
2. User generate keypair baru (atau reuse public key, sign ulang)
3. Admin sign public key baru dengan CA (cara B) atau update expiry-time (cara A)
4. Distribusikan certificate/key baru ke user
5. Key lama otomatis invalid setelah tanggal expired
6. Audit log — catat siapa yang key-nya di-rotate, kapan, oleh siapa
```

### Otomasi Reminder (contoh sederhana pakai cron + script)
```bash
#!/bin/bash
# cek certificate yang mendekati expired dalam 30 hari
for cert in /etc/ssh/certs/*.pub; do
  EXPIRE=$(ssh-keygen -L -f "$cert" | grep "Valid:" | awk '{print $NF}')
  # bandingkan tanggal, kirim notifikasi kalau < 30 hari
done
```

---

## Rekomendasi

| Skala | Rekomendasi |
|---|---|
| Server sedikit, tim kecil | Cara A (`expiry-time` di `authorized_keys`) — cukup dan simpel |
| Banyak server, banyak engineer, butuh audit trail | Cara B (SSH Certificate + CA) — lebih scalable dan terpusat |

Untuk kebutuhan **"rotasi rutin tiap 1 tahun"** secara formal/policy-driven, **SSH Certificate (cara B)** jauh lebih direkomendasikan karena expiry dan revocation-nya terpusat, tidak bergantung pada disiplin manual edit file di tiap server.
