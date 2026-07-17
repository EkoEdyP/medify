## Jawaban dengan `docker run`

```bash
docker run -d \
  --name apps_container \
  -p 5000:80 \
  -v /var/www/html/apps:/var/www/html \
  --restart unless-stopped \
  DockImg:1.1 \
  nginx -g "daemon off;"
```

**Label per poin:**

- **(a)** `-p 5000:80` → publish port 80 container ke port 5000 host
- **(b)** `-v /var/www/html/apps:/var/www/html` → memuat/mount direktori host ke dalam container
- **(c)** `nginx -g "daemon off;"` → menjalankan Nginx otomatis setiap container aktif
- **(d)** `--restart unless-stopped` → container otomatis nyala lagi setiap server di-restart

---

## Jawaban dengan `docker-compose.yml` (setara, opsional)

```yaml
version: '3.8'

services:
  apps:
    image: DockImg:1.1
    container_name: apps_container
    ports:
      - "5000:80"                          # (a)
    volumes:
      - /var/www/html/apps:/var/www/html   # (b)
    command: nginx -g "daemon off;"         # (c)
    restart: unless-stopped                 # (d)
```
```bash
docker-compose up -d
```

Keduanya sama-sama memenuhi 4 kondisi sekaligus — bedanya cuma cara eksekusinya (CLI langsung vs file config).
