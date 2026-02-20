# Deployment Instructions

This guide will help you deploy the nginx reverse proxy to your Raspberry Pi.

## Prerequisites

- Docker and Docker Compose installed on your Raspberry Pi
- SSH access to your Raspberry Pi
- The internet speed monitor application (or other apps) ready to connect

## Deployment Steps

### 1. Create Docker Network

First, create the shared network that all web applications will join:

```bash
docker network create webapps_network
```

### 2. Transfer Files to Raspberry Pi

Transfer this repository to your Raspberry Pi. You can use git clone, scp, or rsync:

**Option A: Git Clone (recommended)**
```bash
# On your Raspberry Pi
cd /apps
git clone https://github.com/sebastian-struecker/raspi-reverse-proxy raspi-reverse-proxy
cd raspi-reverse-proxy
```

**Option B: SCP**
```bash
# From your local machine
scp -r /path/to/raspi-reverse-proxy pi@<raspberry-pi-ip>:/apps/raspi-reverse-proxy
```

### 3. Deploy nginx Proxy

```bash
cd /apps/raspi-reverse-proxy
docker compose up -d
```

### 4. Verify Deployment

```bash
# Check container is running
docker ps | grep raspi-reverse-proxy

# Test health endpoint
curl http://localhost:8080/health
# Expected output: "healthy"

# Check logs
docker compose logs -f nginx
```

## Configuring Applications to Use the Reverse Proxy

### 1. Create App Configuration

Create a new file in `conf.d/` directory:

```bash
nano conf.d/your-app.conf
```

Use this template:

```nginx
# Your App Name
location /your-app/ {
    set $upstream http://your-container-name:PORT;
    proxy_pass $upstream/;
    rewrite ^/your-app/(.*)$ /$1 break;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 30s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

Replace:
- `your-app` - URL path (e.g., `/photo-gallery/`)
- `your-container-name` - Docker container name
- `PORT` - Internal port the app listens on

### 2. Reload nginx

```bash
docker compose exec nginx nginx -s reload
```

No restart needed!

### 3. Configure Your App

Ensure your app's `docker-compose.yml` joins the network:

```yaml
services:
  your-service:
    container_name: your-container-name  # Must match nginx config
    # ... other config ...
    # DON'T expose ports - nginx handles all external traffic
    networks:
      - webapps_network

networks:
  webapps_network:
    external: true
```

## Managing Applications

### Remove an Application

```bash
rm conf.d/your-app.conf
docker compose exec nginx nginx -s reload
```

### Temporarily Disable an Application

```bash
# Disable
mv conf.d/your-app.conf conf.d/your-app.conf.disabled
docker compose exec nginx nginx -s reload

# Re-enable
mv conf.d/your-app.conf.disabled conf.d/your-app.conf
docker compose exec nginx nginx -s reload
```

## Troubleshooting

### Check Configuration Before Reload

```bash
docker compose exec nginx nginx -t
```

### View Logs

```bash
# Follow logs
docker compose logs -f nginx

# View error log
tail -f logs/error.log

# View access log
tail -f logs/access.log
```

### Check Network Connectivity

```bash
# List all containers on the network
docker network inspect webapps_network
```

Should show `raspi-reverse-proxy` and your app containers.

### Common Issues

**502 Bad Gateway**
- App container not running: `docker ps | grep your-container`
- App not on network: `docker network inspect webapps_network`
- Container name mismatch: check `conf.d/*.conf` vs actual container name

**404 Not Found**
- URL path doesn't match location block
- Check trailing slash: `/app/` not `/app`

**Config Not Reloading**
- Test config first: `docker compose exec nginx nginx -t`
- Check logs: `docker compose logs nginx`

## Updating the Reverse Proxy

To update configuration:

```bash
cd /apps/raspi-reverse-proxy
git pull  # if using git
docker compose exec nginx nginx -t  # test config
docker compose exec nginx nginx -s reload  # reload
```

To restart the proxy (rarely needed):

```bash
docker compose restart nginx
```

## Stopping the Reverse Proxy

```bash
cd /apps/raspi-reverse-proxy
docker compose down
```

Note: This will make all proxied applications inaccessible from outside.

## Quick Reference

| Task | Command |
|------|---------|
| Start proxy | `docker compose up -d` |
| Stop proxy | `docker compose down` |
| Restart proxy | `docker compose restart nginx` |
| Reload config | `docker compose exec nginx nginx -s reload` |
| Test config | `docker compose exec nginx nginx -t` |
| View logs | `docker compose logs -f nginx` |
| List apps | `ls conf.d/*.conf` |
| Add app | Create `conf.d/app.conf` + reload |
| Remove app | Delete `conf.d/app.conf` + reload |
