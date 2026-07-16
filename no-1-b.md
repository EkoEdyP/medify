b. Backup Otomatis Jam 2 Pagi, Retention 10 Hari
Script backup `/opt/scripts/backup_xyz.sh`

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/backup/mysql/xyz"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=10

# gunakan mysql_config_editor / ~/.my.cnf, jangan hardcode password di script
MYSQL_CMD="mysql --defaults-extra-file=/etc/mysql/backup_cred.cnf"
DUMP_CMD="mysqldump --defaults-extra-file=/etc/mysql/backup_cred.cnf"

mkdir -p "$BACKUP_DIR"

DATABASES=$($MYSQL_CMD -e "SHOW DATABASES LIKE 'xyz\_%';" -s -N)

for db in $DATABASES; do
  $DUMP_CMD --single-transaction --routines --triggers --events "$db" \
    | gzip > "${BACKUP_DIR}/${db}_${DATE}.sql.gz"
done

# hapus backup lebih tua dari 10 hari
find "$BACKUP_DIR" -type f -name "*.sql.gz" -mtime +"$RETENTION_DAYS" -delete
```

File credential terpisah `/etc/mysql/backup_cred.cnf`

```bash
[client]
user=joni
password=password_yang_kuat
host=localhost
```

untuk add permission file:

```bash
chmod 600 /etc/mysql/backup_cred.cnf
chmod +x /opt/scripts/backup_xyz.sh
```

untuk menjalankan dan menyimpan output cronjob:

```bash
0 2 * * * /opt/scripts/backup_xyz.sh >> /var/log/xyz_backup.log 2>&1
```
