# Nginx Reverse Proxy for Raspberry Pi

Standalone nginx reverse proxy for hosting multiple web applications on one Raspberry Pi with **modular, per-app configuration**.

## Architecture

```
Internet → Port 8080 → nginx → /other-app/              → otherapp-web:8080
                             → /yet-another-app/        → yetanother-web:5000
```

## Key Features

- **Modular Configuration**: Each app gets its own config file in `conf.d/` - no more editing the main nginx.conf
- **Zero-Downtime Updates**: Reload nginx without restarting containers
- **Dynamic DNS Resolution**: Containers can restart freely without breaking nginx
- **Easy App Management**: Add or remove apps with a single configuration file
- **Centralized Logging**: All application traffic logged through one proxy
- **Lightweight**: Limited to 64MB RAM, perfect for Raspberry Pi

## Project Structure

```
raspi-reverse-proxy/
├── docker-compose.yml   # Main nginx container configuration
├── nginx.conf           # Main nginx config (rarely changes)
├── conf.d/              # App-specific configurations (one file per app)
│   └── other-app.conf   # Other app example
├── logs/                # Nginx access and error logs
├── TASKS.md             # Original setup tasks and documentation
└── DEPLOY.md            # Deployment instructions for Raspberry Pi
```

## Quick Start

See [DEPLOY.md](DEPLOY.md) for complete deployment instructions.

### Summary

1. **On Raspberry Pi**: Create the shared network
   ```bash
   docker network create webapps_network
   ```

2. **Deploy the reverse proxy**
   ```bash
   cd /apps/raspi-reverse-proxy
   docker compose up -d
   ```

3. **Configure your apps** to join the `webapps_network`

4. **Access** your apps at `http://<raspberry-pi-ip>:8080/<app-path>/`

## Adding a New Application

1. Create `conf.d/your-app.conf`:
   ```nginx
   location /your-app/ {
       set $upstream http://your-container-name:PORT;
       proxy_pass $upstream/;
       rewrite ^/your-app/(.*)$ /$1 break;

       proxy_set_header Host $host;
       proxy_set_header X-Real-IP $remote_addr;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;
   }
   ```

2. Reload nginx:
   ```bash
   docker compose exec nginx nginx -s reload
   ```

3. Done! No restart needed.

## Documentation

- [DEPLOY.md](DEPLOY.md) - Complete deployment and management guide

## Requirements

- Docker and Docker Compose
- Raspberry Pi (or any Linux system)
- Applications containerized with Docker

## License

MIT License - see [LICENSE](LICENSE) file for details