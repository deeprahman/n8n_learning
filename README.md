# 1. Create directory structure
mkdir -p nginx/conf.d

# 2. Place the files
#    - docker-compose.yml in root
#    - .env in root
#    - default.conf in nginx/conf.d/

# 3. Generate encryption key and update .env
openssl rand -hex 32

# 4. Generate strong postgres password and update .env
openssl rand -base64 24

# 5. Start postgres and nginx first (nginx will fail, that's ok)
docker compose up -d postgres

# 6. Get initial SSL certificate
docker compose run --rm certbot certonly \
  --webroot \
  -w /var/www/html \
  -d n8n.yourdomain.com \
  --email your@email.com \
  --agree-tos \
  --no-eff-email

# 7. Start everything
docker compose up -d

# 8. Check logs
docker compose logs -f

# 9. Certificate renewal
# Add to crontab (crontab -e)
0 3 * * * cd /path/to/your/compose && docker compose exec certbot certbot renew --quiet && docker compose exec nginx nginx -s reload