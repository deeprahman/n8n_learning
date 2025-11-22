In the project root directory I have docker-compose.yml in the project root directory i have ./nginx/conf.d/default.conf . In the root directory I have .env file having sensitive data.

I am running docker on wsl2 Ubunut -24 . I want to set the domain name to palmwavestays.lo in suc a way when called from the host machine terminal, browser etc, it resolve to wsl2. and when is called from the host or from wsl2 using domain auto.palmwavestays.lo it will be handaled by this docker instance describe by given docker-compose.yml


```conf

map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

# HTTP - redirect to HTTPS & handle ACME challenges
server {
    listen 80;
    listen [::]:80;
    server_name n8n.yourdomain.com;

    # Let's Encrypt challenge
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Redirect all other traffic to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name n8n.yourdomain.com;

    # SSL certificates
    ssl_certificate /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem;

    # SSL configuration
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # Proxy to n8n
    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_buffering off;
        proxy_cache off;
        chunked_transfer_encoding off;
    }
}
```

```env
# Domain Configuration
DOMAIN=n8n.yourdomain.com

# Timezone
TZ=UTC

# PostgreSQL Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your-secure-postgres-password-here
POSTGRES_DB=n8n

# n8n Configuration
# Generate with: openssl rand -hex 32
N8N_ENCRYPTION_KEY=rin7Gch8CLPh7AT1Ry2mbkGwjRZXiHsN
```

```yml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: unless-stopped
    container_name: n8n
    environment:
      - N8N_HOST=${DOMAIN}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${DOMAIN}/
      - N8N_EDITOR_BASE_URL=https://${DOMAIN}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_SECURE_COOKIE=true
      - GENERIC_TIMEZONE=${TZ}
      - EXECUTIONS_MODE=regular
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - n8n_network

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    container_name: n8n-postgres
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10
    networks:
      - n8n_network

  nginx:
    image: nginx:stable-alpine
    restart: unless-stopped
    container_name: n8n-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot-webroot:/var/www/html
      - certbot-certs:/etc/letsencrypt:ro
    depends_on:
      - n8n
    networks:
      - n8n_network

  certbot:
    image: certbot/certbot:latest
    container_name: certbot
    volumes:
      - certbot-webroot:/var/www/html
      - certbot-certs:/etc/letsencrypt
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  n8n_data:
  postgres_data:
  certbot-certs:
  certbot-webroot:

networks:
  n8n_network:
    driver: bridge
```
