# Claude Code Instructions - Raspi Reverse Proxy

## Project Overview

This is a standalone nginx reverse proxy designed for Raspberry Pi deployments. It provides a modular, per-app configuration system for hosting multiple web applications on a single Pi.

**Key Purpose**: Centralized reverse proxy that allows multiple Docker-based web apps to be accessed through a single entry point (port 8080) with different URL paths.

## Project Structure

```
raspi-reverse-proxy/
├── docker-compose.yml   # Main nginx container configuration
├── nginx.conf           # Core nginx config (rarely changes)
├── conf.d/              # Per-app configurations (one file per app)
│   └── *.conf          # Individual app proxy configs
├── logs/                # Nginx access and error logs
├── DEPLOY.md            # Deployment instructions
├── README.md            # Project documentation
└── CLAUDE.md            # This file
```

## Key Conventions

### Path Standards
- **Base deployment path**: `/apps/raspi-reverse-proxy`
- **Container name**: `raspi-reverse-proxy`
- **Network name**: `webapps_network` (external, shared across all apps)
- **Port**: 8080 (external access)

### Configuration Pattern
Each app gets its own configuration file in `conf.d/` following this template:

```nginx
# App Name
location /app-path/ {
    set $upstream http://container-name:PORT;
    proxy_pass $upstream/;
    rewrite ^/app-path/(.*)$ /$1 break;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_connect_timeout 30s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

### File Naming
- App configs: `conf.d/<app-name>.conf` (lowercase, hyphen-separated)
- Example: `conf.d/speedmon.conf`, `conf.d/photo-gallery.conf`

## Architecture Principles

1. **Modular Design**: Each app is a separate config file - no monolithic configuration
2. **Zero-Downtime Updates**: Use `nginx -s reload` instead of container restarts
3. **Dynamic DNS**: Uses Docker's internal DNS (127.0.0.11) for container resolution
4. **Resource Limits**: nginx container limited to 64MB RAM for Pi efficiency
5. **Shared Network**: All apps communicate via `webapps_network`

## Common Workflows

### Adding a New App
1. Create `conf.d/new-app.conf`
2. Reload nginx: `docker compose exec nginx nginx -s reload`
3. No restart needed

### Removing an App
1. Delete `conf.d/app.conf`
2. Reload nginx: `docker compose exec nginx nginx -s reload`

### Testing Configuration
Always test before reload:
```bash
docker compose exec nginx nginx -t
```

### Deployment Path
All documentation references should use `/apps/raspi-reverse-proxy` as the deployment directory.

## Important Notes

### DO NOT Change
- `nginx.conf` - Core configuration, rarely needs modification
- Container name in docker-compose.yml without discussing first
- Network name (`webapps_network`) - shared across multiple projects

### Prefer
- Reload over restart
- Individual app configs over modifying nginx.conf
- Docker Compose commands over direct docker commands

### Testing
- Always test config with `nginx -t` before reloading
- Check logs at `logs/error.log` and `logs/access.log`
- Health check endpoint: `http://localhost:8080/health`

## Documentation Style

When updating documentation:
- Use `/apps` as the base directory for all examples
- Use `raspi-reverse-proxy` as the directory name (not `nginx-proxy`)
- Keep examples concise and copy-paste ready
- Include both git clone and SCP options for deployment

## Container Requirements

Apps that want to use this reverse proxy must:
1. Join the `webapps_network` (external network)
2. NOT expose ports externally (nginx handles all external traffic)
3. Use a stable container name that matches the proxy config
4. Run on the same Docker host

## Security Considerations

- No TLS/SSL configured (designed for internal network use)
- Port 8080 should be behind a firewall or accessible only on local network
- Apps should handle their own authentication/authorization

## Performance Notes

- Resource-constrained environment (Raspberry Pi)
- nginx limited to 64MB RAM
- Worker connections set to 512
- Single worker process
- gzip compression enabled for text/json/js

## User Preferences

- Deployment target: Raspberry Pi (ARM architecture)
- Container orchestration: Docker Compose
- Configuration management: File-based, no orchestration tools
- Updates: Manual git pull + reload pattern
