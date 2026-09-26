# Redis Setup Guide

Panduan install, configure, dan monitor Redis untuk Fofa Shop.

## Install

```bash
# Ubuntu/Debian
sudo apt install -y redis-server

# Enable dan start
sudo systemctl enable redis-server
sudo systemctl start redis-server

# Verify
redis-cli ping  # → PONG
```

## Konfigurasi

### Default Config (sudah cukup untuk 2GB RAM)

Redis default config sudah optimal untuk small VPS. Yang perlu di-check:

```bash
sudo nano /etc/redis/redis.conf
```

Pastikan:
```
# Bind ke localhost saja (security)
bind 127.0.0.1

# Max memory 256MB (cukup untuk cache + queue + session)
maxmemory 256mb

# Evict policy: hapus key paling lama dipakai kalau penuh
maxmemory-policy allkeys-lru

# Simpan data ke disk (optional, untuk session persistence)
save 900 1
save 300 10
save 60 10000
```

### Set Password (Recommended)

```bash
sudo sed -i 's/^# requirepass foobared/requirepass your_secure_password/' /etc/redis/redis.conf
sudo systemctl restart redis-server

# Test
redis-cli -a your_secure_password ping  # → PONG
```

Kalau set password, update `.env`:
```env
REDIS_PASSWORD=your_secure_password
```

## Laravel Integration

### .env Configuration

```env
# Cache (DB 1 — terpisah dari default)
CACHE_STORE=redis
REDIS_CACHE_DB=1

# Session (DB 0 — shared dengan default)
SESSION_DRIVER=redis

# Queue (DB 0 — default)
QUEUE_CONNECTION=redis

# Connection
REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

### Redis DB Allocation

| DB | Purpose | Key Pattern |
|---|---|---|
| DB 0 | Queue + Session + Rate Limiter | `fofa-shop-*` (prefix) |
| DB 1 | Application Cache | `fofa-shop-cache-*` (prefix) |

Pisahkan DB supaya `flushdb` di satu service tidak menghapus data service lain.

## Monitoring

### Basic Health Check

```bash
# Apakah Redis hidup?
redis-cli ping

# Memory usage
redis-cli info memory | grep used_memory_human
# → used_memory_human:1.23M

# Jumlah keys
redis-cli dbsize
# → (integer) 42

# Connected clients
redis-cli info clients | grep connected_clients
```

### Real-time Monitor

```bash
# Monitor semua commands (hati-hati di production, verbose!)
redis-cli monitor

# Filter specific pattern
redis-cli monitor | grep "fofa-shop"
```

### Memory Analysis

```bash
# Top 10 keys by memory
redis-cli --bigkeys

# Key per database
redis-cli info keyspace
# db0:keys=42,expires=38,avg_ttl=3600000
# db1:keys=15,expires=15,avg_ttl=86400000
```

## Troubleshooting

### Redis Connection Refused

```bash
# Check status
sudo systemctl status redis-server

# Restart
sudo systemctl restart redis-server

# Check if port is in use
sudo ss -tlnp | grep 6379
```

### High Memory Usage

```bash
# Check what's using memory
redis-cli info memory | grep used_memory_human

# Check number of keys per DB
redis-cli info keyspace

# Flush specific DB (hati-hati!)
redis-cli -n 1 flushdb  # Flush DB 1 (cache only)

# Flush all (DANGER — removes queue + session too!)
redis-cli flushall
```

### Queue Jobs Not Processing

```bash
# Check queue size
php artisan queue:size

# Check worker status
supervisorctl status

# Restart worker
sudo supervisorctl restart fofo-queue-worker_00

# Check failed jobs
php artisan queue:failed
```

### Session Expired Unexpectedly

```bash
# Check session keys
redis-cli keys "fofa-shop-session:*" | wc -l

# Check TTL on a session
redis-cli ttl "fofa-shop-session:abc123"

# Default TTL = 120 minutes (SESSION_LIFETIME)
```

## Backup

Redis data bersifat ephemeral (cache + queue). Tidak perlu backup routine. Yang perlu di-backup:

- **MySQL** — source of truth untuk semua data
- **Redis** — bisa di-rebuild dari MySQL (cache auto-rebuild, queue bisa re-dispatch)
- **`.env`** — konfigurasi aplikasi

Kalau butuh Redis snapshot:
```bash
# Manual backup
redis-cli bgsave
sudo cp /var/lib/redis/dump.rdb /backup/redis-dump-$(date +%Y%m%d).rdb

# Restore
sudo systemctl stop redis-server
sudo cp /backup/redis-dump-20260828.rdb /var/lib/redis/dump.rdb
sudo chown redis:redis /var/lib/redis/dump.rdb
sudo systemctl start redis-server
```
