# Deployment Guide

Panduan lengkap deploy Fofa Shop ke VPS production.

## Prerequisites

- VPS 2 VCPU + 2 GB RAM + 2 GB SWAP (minimum)
- Ubuntu 22.04+ / Debian 12+
- Domain name + SSL certificate (Let's Encrypt)

## 1. Server Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y nginx mysql-server php8.3-fpm php8.3-mysql \
  php8.3-redis php8.3-mbstring php8.3-xml php8.3-curl php8.3-zip \
  php8.3-bcmath php8.3-gd php8.3-intl redis-server supervisor git

# Enable services
sudo systemctl enable nginx mysql redis-server php8.3-fpm supervisor
```

## 2. MySQL Setup

```bash
# Create database and user
sudo mysql -e "CREATE DATABASE fofa_shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
sudo mysql -e "CREATE USER 'fofa_user'@'127.0.0.1' IDENTIFIED BY 'your_secure_password';"
sudo mysql -e "GRANT ALL PRIVILEGES ON fofa_shop.* TO 'fofa_user'@'127.0.0.1';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Apply optimized config
sudo cp deploy/mysql-optimized.cnf /etc/mysql/conf.d/fofa-optimized.cnf
sudo systemctl restart mysql
```

## 3. Redis Setup

```bash
# Redis should already be installed from step 1
# Verify it's running
redis-cli ping  # Should return PONG

# Optional: set password for Redis
sudo sed -i 's/^# requirepass foobared/requirepass your_redis_password/' /etc/redis/redis.conf
sudo systemctl restart redis-server
```

## 4. Deploy Application

```bash
# Clone repo
cd /var/www
sudo git clone https://github.com/your-repo/fofa-topup-shop.git
sudo chown -R www-data:www-data fofa-topup-shop
cd fofa-topup-shop

# Install PHP dependencies
composer install --no-dev --optimize-autoloader

# Install Node dependencies and build assets
npm ci
npm run build

# Setup environment
cp .env.example .env
php artisan key:generate

# Edit .env — set these values:
# DB_CONNECTION=mysql
# DB_HOST=127.0.0.1
# DB_DATABASE=fofa_shop
# DB_USERNAME=fofa_user
# DB_PASSWORD=your_secure_password
#
# CACHE_STORE=redis
# SESSION_DRIVER=redis
# QUEUE_CONNECTION=redis
#
# APP_ENV=production
# APP_DEBUG=false
# APP_URL=https://yourdomain.com

# Run migrations
php artisan migrate --force

# Seed admin account (if needed)
php artisan db:seed --class=AdminSeeder

# Cache config for production
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Set permissions
sudo chown -R www-data:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
```

## 5. PHP-FPM Config

```bash
sudo cp deploy/php-fpm-pool.conf /etc/php/8.3/fpm/pool.d/www.conf
sudo systemctl restart php8.3-fpm
```

Key settings:
- `pm.max_children = 25` (25 workers × ~50MB = 1.25GB RAM)
- `pm.max_requests = 500` (restart workers to prevent memory leak)
- OPcache JIT enabled

## 6. Nginx Config

```bash
# Edit deploy/nginx-site.conf — replace 'yourdomain.com' with actual domain
sudo cp deploy/nginx-site.conf /etc/nginx/sites-available/fofa-topup
sudo ln -sf /etc/nginx/sites-available/fofa-topup /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

## 7. SSL Certificate

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
sudo certbot renew --dry-run  # Verify auto-renewal
```

## 8. Supervisor (Queue + Scheduler)

```bash
sudo cp deploy/supervisor-worker.conf /etc/supervisor/conf.d/fofa-topup.conf
sudo supervisorctl reread && sudo supervisorctl update
sudo supervisorctl status  # Should show fofo-queue-worker_00 and fofo-scheduler running
```

## 9. Swap (if not configured)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 10. Firewall

```bash
sudo apt install -y ufw
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

---

## Rate Limiting

Rate limiting sudah di-configure di `config/fofa.php`:

| Endpoint | Limit | Decay | Config Key |
|---|---|---|---|
| Checkout per IP | 5 req | 1 menit | `FOFA_RATE_CHECKOUT_IP` |
| Checkout per User | 20 req | 1 jam | `FOFA_RATE_CHECKOUT_USER` |
| Checkout per Target | 6 req | 10 menit | `FOFA_RATE_CHECKOUT_TARGET` |
| Admin | 100 req | 1 menit | `FOFA_RATE_ADMIN` |
| Login | 5 attempt | 15 menit | `FOFA_RATE_LOGIN` |

Untuk override, set env vars di `.env`:
```env
FOFA_RATE_CHECKOUT_USER=30
FOFA_RATE_ADMIN=120
```

---

## Monitoring

```bash
# Redis
redis-cli ping                              # PONG = healthy
redis-cli info memory | grep used_memory_human  # Memory usage
redis-cli dbsize                            # Keys count

# PHP-FPM
curl -s http://localhost/fpm-status          # Worker status

# MySQL
mysql -e "SHOW STATUS LIKE 'Threads_connected';"  # Active connections

# Queue
php artisan queue:size                      # Pending jobs
supervisorctl status                        # Worker status

# System
free -h                                     # RAM + SWAP usage
vmstat 1 5                                  # CPU + IO stats
htop                                        # Interactive monitor

# Logs
tail -f storage/logs/laravel.log            # App logs
sudo tail -f /var/log/nginx/fofa-topup-error.log  # Nginx errors
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| 502 Bad Gateway | PHP-FPM down | `sudo systemctl restart php8.3-fpm` |
| Queue jobs stuck | Worker crashed | `sudo supervisorctl restart fofo-queue-worker_00` |
| Slow page loads | MySQL slow queries | Check `slow-query.log`, add indexes |
| High SWAP usage | Too many PHP workers | Lower `pm.max_children` in FPM config |
| Redis connection refused | Redis not running | `sudo systemctl start redis-server` |
| Rate limit too strict | Default too low | Set `FOFA_RATE_*` env vars |

---

## Memory Budget

```
OS + SWAP overhead:     ~200 MB
MySQL:                  ~400 MB
Redis:                  ~50 MB
PHP-FPM (25 workers):   ~1250 MB
Nginx:                  ~20 MB
Supervisor:             ~50 MB
─────────────────────────────────
Total:                  ~1970 MB  ← fits in 2 GB
Swap:                   2 GB backup (light swap at 60 req/s)
```
