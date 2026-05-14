# n8n + Nginx Proxy Manager Setup on AWS EC2

## 1. Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose-plugin -y
sudo service docker start
sudo usermod -aG docker $USER
```

Reconnect SSH after adding yourself to docker group.

---

# 2. Create n8n Directory

```bash
mkdir ~/n8n
cd ~/n8n
```

---

# 3. Create `.env`

```env
POSTGRES_USER=n8n
POSTGRES_PASSWORD=CHANGE_THIS_PASSWORD
POSTGRES_DB=n8n

N8N_ENCRYPTION_KEY=GENERATE_RANDOM_KEY

N8N_HOST=n8n.yourdomain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://n8n.yourdomain.com/

N8N_RUNNERS_ENABLED=true
N8N_PUSH_BACKEND=websocket
N8N_SECURE_COOKIE=true
N8N_PROXY_HOPS=1

NODE_ENV=production
```

Generate encryption key:

```bash
openssl rand -base64 32
```

---

# 4. Create Docker Volume

```bash
docker volume create n8n_data
```

---

# 5. Run n8n

```bash
docker run -d \
  --name n8n \
  --env-file .env \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n
```

Check logs:

```bash
docker logs -f n8n
```

---

# 6. Setup Nginx Proxy Manager

Create directory:

```bash
mkdir ~/nginx
cd ~/nginx
```

Create `docker-compose.yml`

```yaml
volumes:
  nginx_data:
  nginx_letsencrypt:
  mysql_db_storage:

services:
  nginx:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: ${MYSQL_USER}
      DB_MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      DB_MYSQL_NAME: ${MYSQL_DATABASE}
      DISABLE_IPV6: "true"
    volumes:
      - nginx_data:/data
      - nginx_letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: jc21/mariadb-aria:latest
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MARIADB_AUTO_UPGRADE: "1"
    volumes:
      - mysql_db_storage:/var/lib/mysql
```

Create `.env`

```env
MYSQL_ROOT_PASSWORD=CHANGE_THIS
MYSQL_DATABASE=nginx
MYSQL_USER=nginx_user
MYSQL_PASSWORD=CHANGE_THIS
```

Start NPM:

```bash
docker compose up -d
```

---

# 7. AWS Security Group

Open inbound ports:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 81 | Nginx Proxy Manager Admin |

Optional:
- Remove public access to 5678 after proxy works.

---

# 8. DNS

Point:

```text
n8n.yourdomain.com
```

to your EC2 public IP.

Verify:

```bash
dig +short n8n.yourdomain.com
curl ifconfig.me
```

IPs must match.

---

# 9. Login to Nginx Proxy Manager

Open:

```text
http://YOUR_SERVER_IP:81
```

Default login:

```text
admin@example.com
changeme
```

Change immediately.

---

# 10. Create Proxy Host

Domain:

```text
n8n.yourdomain.com
```

Forward:

```text
Scheme: http
Host: YOUR_EC2_PRIVATE_IP_OR_PUBLIC_IP
Port: 5678
```

SSL:
- Request new certificate
- Force SSL
- Enable HTTP/2

---

# 11. Troubleshooting

Check n8n logs:

```bash
docker logs -f n8n
```

Check NPM logs:

```bash
docker logs -f nginx-nginx-1
```

Check Let's Encrypt logs:

```bash
docker exec -it nginx-nginx-1 sh
cat /data/logs/letsencrypt.log
```

Common issue:
- Port 80 not open in AWS Security Group.
