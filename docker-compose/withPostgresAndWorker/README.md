# n8n Self-Hosting with HTTPS (Docker Compose + Nginx + Certbot)

This setup runs [n8n](https://n8n.io/) using Docker Compose with PostgreSQL and Redis, fronted by Nginx with HTTPS enabled via Let's Encrypt and Certbot.

---

## ✅ Requirements

- A domain pointing to your server (e.g. `n8n.example.com`)
- Docker & Docker Compose installed
- Open ports `80` and `443` on the server

---

## 📁 Folder Structure

```
project-root/
├── docker-compose.yml
├── nginx.conf
├── .env                         # credentials (not included here)
└── certbot/
    ├── www/                    # ACME challenge files
    └── letsencrypt/           # SSL certificates (volume)
```

---

## 🚀 Step-by-Step Setup

### 1. Create a temporary HTTP-only Nginx config

In `nginx.conf`:

```nginx
server {
    listen 80;
    server_name n8n.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 'Temporary Certbot Setup';
    }
}
```

Start the temporary Nginx container:

```bash
docker-compose up -d nginx
```

---

### 2. Issue HTTPS certificate with Certbot

Replace the email and domain as needed:

```bash
docker run --rm   -v "$(pwd)/certbot/www:/var/www/certbot"   -v "$(pwd)/certbot/letsencrypt:/etc/letsencrypt"   certbot/certbot   certonly --webroot   --webroot-path=/var/www/certbot   --email you@example.com   --agree-tos   --no-eff-email   -d n8n.example.com
```

Confirm that the certificates exist:

```bash
ls ./certbot/letsencrypt/live/n8n.example.com/
# expected: fullchain.pem, privkey.pem, etc.
```

---

### 3. Restore the full Nginx configuration

Replace `nginx.conf` with the full version:

```nginx
server {
    listen 80;
    server_name n8n.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://n8n:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Then restart Nginx:

```bash
docker-compose restart nginx
```

---

### 4. Launch all services

Make sure your `.env` is configured, then:

```bash
docker-compose up -d
```

Visit `https://n8n.example.com` in your browser to verify.

---

## 🔁 Automatic HTTPS Renewal

A background Certbot container is included:

```yaml
certbot:
  image: certbot/certbot
  restart: unless-stopped
  volumes:
    - ./certbot/letsencrypt:/etc/letsencrypt
    - ./certbot/www:/var/www/certbot
  entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

Make sure port `80` remains open for ACME HTTP challenges.

---
