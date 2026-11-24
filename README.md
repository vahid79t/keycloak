# Keycloak behind HAProxy with Let's Encrypt (Docker Compose)
 
Deploys Keycloak 26.7.0 with PostgreSQL 18 using Docker Compose, in a low-resource configuration. HAProxy terminates public HTTPS on `:443` using a Let's Encrypt certificate and forwards to Keycloak on `:9443`.
 
## Architecture
 
```
Browser / kubectl
        ↓
https://keycloak.tokeli.ir
        ↓
HAProxy :443
        ↓
Let's Encrypt certificate
        ↓
Keycloak :9443
```
 
## Prerequisites
 
- Docker and Docker Compose
- HAProxy installed on the host
- `certbot` installed on the host
- DNS A record for `keycloak.tokeli.ir` pointing to the server
- Port 80 free temporarily (for the standalone certificate challenge)
## 1. Issue the Let's Encrypt certificate
 
Stop HAProxy so certbot can bind port 80, verify port 80 is free, and check DNS:
 
```bash
systemctl stop haproxy
 
ss -lntp | grep ':80 '
 
dig +short keycloak.tokeli.ir
```
 
Request the certificate:
 
```bash
certbot certonly \
  --standalone \
  -d keycloak.tokeli.ir \
  --email YOUR_REAL_EMAIL \
  --agree-tos \
  --no-eff-email
```
 
Resulting files:
 
| Item        | Path                                                      |
|-------------|-----------------------------------------------------------|
| Certificate | `/etc/letsencrypt/live/keycloak.tokeli.ir/fullchain.pem`  |
| Private key | `/etc/letsencrypt/live/keycloak.tokeli.ir/privkey.pem`    |
 
Start HAProxy again:
 
```bash
systemctl start haproxy
```
 
## 2. Docker Compose file
 
Save as `docker-compose.yml`:
 
```yaml
services:
  kc-postgres:
    image: public.ecr.aws/docker/library/postgres:18
    container_name: kc-postgres
    restart: unless-stopped
 
    # Resource limits
    mem_limit: 160m
    cpus: "0.20"
    pids_limit: 60
 
    environment:
      POSTGRES_USER: kcuser
      POSTGRES_PASSWORD: "123"          # CHANGE ME
      POSTGRES_DB: kc
 
    command:
      - postgres
      - -c
      - shared_buffers=16MB
      - -c
      - effective_cache_size=64MB
      - -c
      - work_mem=1MB
      - -c
      - maintenance_work_mem=8MB
      - -c
      - max_connections=15
      - -c
      - max_worker_processes=2
      - -c
      - max_parallel_workers=0
      - -c
      - max_parallel_workers_per_gather=0
      - -c
      - autovacuum_max_workers=1
      - -c
      - max_wal_size=128MB
 
    ports:
      - "127.0.0.1:30001:5432"
 
    volumes:
      - pgdata18:/var/lib/postgresql
 
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U kcuser -d kc
      interval: 60s
      timeout: 5s
      retries: 3
      start_period: 30s
 
    networks:
      - kc_network
 
 
  keycloak:
    image: quay.io/keycloak/keycloak:26.7.0
    container_name: rhbk
    restart: unless-stopped
 
    # Resource limits
    mem_limit: 640m
    cpus: "0.75"
    pids_limit: 150
 
    depends_on:
      kc-postgres:
        condition: service_healthy
 
    # Keycloak is reachable publicly through HAProxy.
    # Do not expose 9443 directly to the Internet.
    ports:
      - "127.0.0.1:9443:9443"
 
    environment:
      # Bootstrap admin
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: "12345678"   # CHANGE ME
 
      # PostgreSQL
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://kc-postgres:5432/kc
      KC_DB_USERNAME: kcuser
      KC_DB_PASSWORD: "123"                     # CHANGE ME
 
      # Small connection pool
      KC_DB_POOL_INITIAL_SIZE: "1"
      KC_DB_POOL_MIN_SIZE: "1"
      KC_DB_POOL_MAX_SIZE: "5"
 
      # Public/external Keycloak URL
      KC_HOSTNAME: "https://keycloak.tokeli.ir"
      KC_HOSTNAME_STRICT: "true"
 
      # HTTPS only
      KC_HTTP_ENABLED: "false"
      KC_HTTPS_PORT: "9443"
 
      # TLS certificate inside container
      KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/certs/server.crt
      KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/certs/server.key
 
      # Lightweight configuration
      KC_HEALTH_ENABLED: "false"
      KC_METRICS_ENABLED: "false"
      KC_CACHE: local
 
      # Logging
      KC_LOG_LEVEL: WARN
 
      # JVM memory
      JAVA_OPTS_KC_HEAP: "-Xms64m -Xmx320m"
 
    volumes:
      # Let's Encrypt certificate copied to these host paths
      - type: bind
        source: /opt/keycloak/server.crt
        target: /opt/keycloak/certs/server.crt
        read_only: true
        bind:
          create_host_path: false
 
      - type: bind
        source: /opt/keycloak/server.key
        target: /opt/keycloak/certs/server.key
        read_only: true
        bind:
          create_host_path: false
 
      # Custom themes
      - /opt/keycloak/themes:/opt/keycloak/themes:ro
 
      # Custom CAPTCHA provider
      - /opt/keycloak/providers/keycloak-captcha.jar:/opt/keycloak/providers/keycloak-captcha.jar:ro
 
    command:
      - start
 
    networks:
      - kc_network
 
 
volumes:
  pgdata18:
 
 
networks:
  kc_network:
    driver: bridge
```
 
## 3. Copy the certificate for Keycloak and start it
 
Copy the Let's Encrypt files (dereferencing symlinks) to the paths mounted by the container and set permissions:
 
```bash
cp -L /etc/letsencrypt/live/keycloak.tokeli.ir/fullchain.pem \
  /opt/keycloak/server.crt
 
cp -L /etc/letsencrypt/live/keycloak.tokeli.ir/privkey.pem \
  /opt/keycloak/server.key
 
chmod 644 /opt/keycloak/server.crt
chmod 640 /opt/keycloak/server.key
```
 
Start (or recreate) Keycloak:
 
```bash
docker compose up -d --force-recreate keycloak
```
 
## Notes
 
- **Change all default passwords** (`POSTGRES_PASSWORD`, `KC_DB_PASSWORD`, `KC_BOOTSTRAP_ADMIN_PASSWORD`) before using this anywhere real, and avoid committing real secrets to GitHub (use an `.env` file listed in `.gitignore`).
- Keycloak (`9443`) and PostgreSQL (`30001`) are bound to `127.0.0.1` only; public access goes through HAProxy.
- Let's Encrypt certificates expire every 90 days. After each renewal, re-run the `cp -L` and `chmod` commands above and recreate the Keycloak container (this can be automated with a certbot `--deploy-hook`).
- Because the certbot standalone challenge needs port 80, renewals will conflict with HAProxy if it listens on 80. Consider the `--http-01-port` option or a webroot/HAProxy-based challenge for renewals.
- The custom themes directory (`/opt/keycloak/themes`) and CAPTCHA provider (`/opt/keycloak/providers/keycloak-captcha.jar`) must exist on the host.
 
