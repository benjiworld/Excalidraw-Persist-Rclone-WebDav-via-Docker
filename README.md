# Self-Hosted Excalidraw with persistence & Rclone WebDAV, with Caddy and via Docker

This repository contains the configuration to run [Excalidraw Persist](https://github.com/ozencb/excalidraw-persist) (with server-side SQLite storage) alongside an [Rclone WebDAV](https://rclone.org/) server, both routed securely through a [Caddy](https://caddyserver.com/) reverse proxy with automatic HTTPS.

## Prerequisites

- **Docker** and **Docker Compose** installed on the server.
- A registered domain name.
- DNS `A` records pointing to your server's IP address for your subdomains (e.g., `draw.your-domain.com` and `your-domain.com`).
- Ports `80` and `443` open on your server's firewall.

## Directory Structure

Set up your project directory like this:

```text
/home/user/Excalidraw-Persist-Rclone-WebDav-via-Docker/
├── docker-compose.yaml
├── Caddyfile
├── config/
│   └── rclone.conf       # Your existing Rclone configuration
├── data/                 # Where Rclone serves/stores its files
└── excalidraw-data/      # Where Excalidraw saves its SQLite DB
```

Make sure to create the empty directories before starting:
```bash
mkdir -p config data excalidraw-data
```

## Configuration Files

### 1. `docker-compose.yaml`

This file defines the three services: Caddy (Reverse Proxy), Rclone (WebDAV), and Excalidraw Persist.

```yaml
services:
  caddy-webdav:
    image: caddy:2
    container_name: caddy-webdav
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp" # Optional: enables HTTP/3
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config

  rclone-webdav:
    image: rclone/rclone:latest
    container_name: rclone-webdav
    user: "${UID:-1000}:${GID:-1000}"
    volumes:
      - ./config:/config/rclone
      - ./data:/data
    working_dir: /data
    command:
      - --config
      - /config/rclone/rclone.conf
      - --cache-dir
      - /data/cache
      - serve
      - webdav
      - backend:some/folder
      - --addr
      - 0.0.0.0:8080
      - --user
      - "${WEBDAV_USER:-admin}"
      - --pass
      - "${WEBDAV_PASS:-password}"
      - --vfs-cache-mode
      - writes
    restart: unless-stopped

  excalidraw-persist:
    image: ghcr.io/ozencb/excalidraw-persist:latest
    container_name: excalidraw-persist
    restart: unless-stopped
    volumes:
      - ./excalidraw-data:/app/data
    environment:
      - PORT=4000
      - NODE_ENV=production
      - DB_PATH=/app/data/database.sqlite

volumes:
  caddy_data:
  caddy_config:
```

### 2. `Caddyfile`

This routes incoming traffic to the correct Docker container based on the subdomain. Because they are on the same Docker network by default, Caddy routes to them using their `container_name` and internal port.

```caddyfile
# Rclone WebDAV
your-domain.com {
    reverse_proxy rclone-webdav:8080
}

# Excalidraw Persist
draw.your-domain.com {
    reverse_proxy excalidraw-persist:80
}
```

## Deployment

1. **Navigate to your directory:**
   ```bash
   cd /home/user/Excalidraw-Persist-Rclone-WebDav-via-Docker
   ```

2. **Start the stack:**
   ```bash
   docker compose up -d
   ```

3. **Check the logs** (to ensure Caddy got the SSL certificates and the apps started):
   ```bash
   docker compose logs -f
   ```
4. The services will be available at the following addresses:

   ```bash
   https://your-domain.com/webdav
   https://draw.your-domain.com/
   ```


## Maintenance

### Updating Caddy Configuration
If you modify the `Caddyfile`, you don't need to restart the whole stack. You can reload Caddy gracefully:
```bash
docker exec -w /etc/caddy caddy-webdav caddy reload
```

### Updating Images
To pull the latest versions of the images and recreate the containers:
```bash
docker compose pull
docker compose up -d --force-recreate
```
