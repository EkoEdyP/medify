# Langkah-Langkah Pengamanan Standar Server Ubuntu

## 1. Update & Patch Management

```bash
# Update system secara berkala
sudo apt update && sudo apt upgrade -y

# Aktifkan automatic security updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Konfigurasi `/etc/apt/apt.conf.d/50unattended-upgrades` untuk memastikan hanya security patch yang auto-install (hindari auto-upgrade major version yang bisa break aplikasi).

---

## 2. Hardening Akses SSH

**Ganti port default** (opsional tapi mengurangi noise dari bot scanning):
```bash
sudo nano /etc/ssh/sshd_config
```
```ini
Port 2222
```

**Disable root login langsung:**
```ini
PermitRootLogin no
```

**Disable password authentication, wajib pakai SSH key:**
```ini
PasswordAuthentication no
PubkeyAuthentication yes
```

**Batasi user yang boleh SSH:**
```ini
AllowUsers joni deploy
```

**Set limit percobaan login:**
```ini
MaxAuthTries 3
LoginGraceTime 30
```

Restart service:
```bash
sudo systemctl restart sshd
```

> **Penting**: sebelum disable password auth, pastikan SSH key sudah ter-setup dan bisa login duluan, supaya tidak lockout dari server sendiri.

---

## 3. Firewall (UFW)

```bash
sudo apt install ufw -y

# Default policy: block semua masuk, allow semua keluar
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Buka hanya port yang dibutuhkan
sudo ufw allow 2222/tcp    # SSH (custom port)
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS

sudo ufw enable
sudo ufw status verbose
```

---

## 4. Fail2ban (Proteksi Brute Force)

```bash
sudo apt install fail2ban -y
```

Konfigurasi `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 3600
findtime = 600

[nginx-http-auth]
enabled = true
```

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

---

## 5. Manajemen User & Privilege

```bash
# Hindari kerja pakai root, buat user biasa dengan sudo
sudo adduser joni
sudo usermod -aG sudo joni

# Audit user yang punya akses sudo
grep -Po '^sudo.+:\K.*$' /etc/group

# Set password policy (kompleksitas & expiry)
sudo apt install libpam-pwquality -y
sudo nano /etc/security/pwquality.conf
```

Hapus/lock user yang tidak terpakai:
```bash
sudo passwd -l nama_user_tidak_terpakai
```

---

## 6. Konfigurasi Firewall Level Aplikasi & Kernel

**Disable/limit unused services:**
```bash
sudo systemctl list-unit-files --type=service --state=enabled
sudo systemctl disable <service_tidak_perlu>
```

**Kernel hardening** via `/etc/sysctl.conf`:
```ini
# Cegah IP spoofing
net.ipv4.conf.all.rp_filter = 1

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0

# Cegah SYN flood
net.ipv4.tcp_syncookies = 1

# Disable ICMP redirect
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Log paket mencurigakan
net.ipv4.conf.all.log_martians = 1
```
```bash
sudo sysctl -p
```

---

## 7. Sertifikat & Enkripsi (SSL/TLS)

```bash
# Pasang Let's Encrypt untuk HTTPS
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d domain.com

# Auto-renewal
sudo certbot renew --dry-run
```

Pastikan hanya TLS versi aman yang diaktifkan di Nginx (`nginx.conf`):
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers HIGH:!aNULL:!MD5;
```

---

## 8. Antivirus/Malware Scanning (opsional, tergantung kebutuhan)

```bash
sudo apt install rkhunter clamav -y
sudo rkhunter --update
sudo rkhunter --check
```

---

## 9. Audit & Logging

```bash
# Install auditd untuk tracking aktivitas sistem
sudo apt install auditd -y
sudo systemctl enable auditd

# Cek log auth untuk percobaan login mencurigakan
sudo tail -f /var/log/auth.log

# Setup centralized logging (opsional, kirim log ke server terpisah)
```

Pertimbangkan integrasi dengan **Prometheus + Grafana + Loki** atau **ELK Stack** untuk monitoring dan alerting real-time.

---

## 10. Backup & Disaster Recovery

- Backup rutin (sesuai kebutuhan no. 1 sebelumnya) dengan retention yang jelas
- Simpan backup di lokasi terpisah (offsite/cloud storage), jangan cuma di server yang sama
- Test restore backup secara berkala untuk memastikan backup valid dan bisa dipakai

---

## 11. Hardening Level Aplikasi

- **Nginx/Apache**: sembunyikan versi server (`server_tokens off;` di Nginx)
- **PHP**: disable fungsi berbahaya di `php.ini`
  ```ini
  disable_functions = exec,passthru,shell_exec,system,proc_open,popen
  expose_php = Off
  ```
- **Database**: bind MySQL hanya ke localhost/internal network kalau tidak butuh akses eksternal langsung
  ```ini
  bind-address = 127.0.0.1
  ```
- Environment variable/credential sensitif jangan hardcode di kode, pakai `.env` dengan permission ketat

---

## 12. File Permission & Ownership

```bash
# Pastikan file sensitif tidak world-readable
sudo chmod 600 /etc/mysql/backup_cred.cnf
sudo chown -R www-data:www-data /var/www/apps

# Cari file dengan permission terlalu terbuka
find / -perm -o+w -type f 2>/dev/null
```

---

## 13. Two-Factor Authentication (opsional, untuk akses lebih kritikal)

```bash
sudo apt install libpam-google-authenticator -y
google-authenticator
```
Integrasikan dengan SSH untuk 2FA login.

---

## Ringkasan Prioritas

| Kategori | Tindakan Utama |
|---|---|
| **Akses** | SSH key only, disable root login, firewall (UFW) |
| **Brute force protection** | Fail2ban |
| **Update** | Unattended-upgrades untuk security patch |
| **Enkripsi** | HTTPS/SSL via Let's Encrypt |
| **Monitoring** | auditd, log terpusat, alerting |
| **Backup** | Rutin, offsite, tervalidasi lewat test restore |
| **Least privilege** | User terpisah dari root, permission file dibatasi ketat |
