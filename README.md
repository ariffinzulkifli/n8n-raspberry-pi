# N8n on Raspberry Pi using Docker Compose

A complete guide to installing and running n8n workflow automation on your Raspberry Pi using Docker Compose.

## 📋 Prerequisites

Before you begin, ensure you have:

- **Raspberry Pi 3B+ or newer** (4GB RAM recommended for Pi 4/5)
- **Raspberry Pi OS** (64-bit recommended) or Ubuntu Server
- **Stable internet connection**
- **At least 8GB free storage space**
- **Basic command line knowledge**

## 🚀 Installation Steps

### Step 1: Update Your System

Update and upgrade your system packages:

```bash
sudo apt update && sudo apt upgrade -y
```

After the upgrade completes, reboot your Pi:

```bash
sudo reboot
```

### Step 2: Install Docker and Docker Compose Plugin

Install Docker using the official installation script:

```bash
curl -sSL https://get.docker.com | sh
```

Add your user to the docker group (replace `$USER` with your username if needed):

```bash
sudo usermod -aG docker $USER
```

Install Docker Compose plugin:

```bash
sudo apt install -y docker-compose-plugin
```

**Important:** Log out and log back in for group changes to take effect, or run:

```bash
newgrp docker
```

### Step 3: Verify Installation

Check that Docker and Docker Compose are installed correctly:

```bash
docker --version
docker compose version
```

**Expected output:**
```
Docker version 28.x.x, build xxxxxxx
Docker Compose version v2.x.x
```

### Step 4: Find Your Raspberry Pi Hostname and IP Address

To find your Pi's hostname, run:

```bash
hostname
```

To find your Pi's IP address, run:

```bash
hostname -I
```

The first IP address shown (usually starting with `192.168.` or `10.`) is your local network IP.

### Step 5: Create the n8n Project Directory

```bash
mkdir -p ~/n8n && cd ~/n8n
```

### Step 6: Local HTTPS with Self-Signed Certificate

This method allows you to use HTTPS on your local network without external services. Perfect for local development and testing.

```bash
mkdir -p certs
cd certs
```

Generate private key and certificate (valid for 365 days)

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout privkey.pem \
  -out fullchain.pem \
  -subj "/C=US/ST=State/L=City/O=Organization/OU=Department/CN=raspberrypi.local"
```

**Note:** Replace `raspberrypi.local` with your Pi's hostname or local IP address.

### Step 7: Create the docker-compose.yaml File

```bash
cd ~/n8n
```

Create and edit the Docker Compose configuration file:

```bash
nano docker-compose.yaml
```

Copy and paste the following configuration:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=password
      - N8N_HOST= # YOUR_PI_IP e.g. raspberrypi.local or 192.168.0.168
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - N8N_SECURE_COOKIE=false
      - NODE_ENV=production
      - WEBHOOK_URL=https://YOUR_PI_IP:5678
      - GENERIC_TIMEZONE=Asia/Kuala_Lumpur
      - TZ=Asia/Kuala_Lumpur
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - n8n-network

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./certs:/certs
      - caddy_data:/data
    networks:
      - n8n-network

volumes:
  n8n_data:
    driver: local
  caddy_data:
    driver: local

networks:
  n8n-network:
    driver: bridge
```

**Note:** Replace `raspberrypi.local` and `YOUR_PI_IP` with your Pi's hostname or local IP address.

Save and exit (in nano: `Ctrl+X`, then `Y`, then `Enter`).

### Step 8: Create Caddyfile

```bash
cd ~/n8n
nano Caddyfile
```

Add this configuration:

```
https://raspberrypi.local {
    tls /certs/fullchain.pem /certs/privkey.pem
    reverse_proxy n8n:5678
}

http://raspberrypi.local {
    redir https://raspberrypi.local{uri}
}
```

**Note:** Replace `raspberrypi.local` with your Pi's hostname or local IP address.

### Step 9: Start the n8n Container

Start n8n in detached mode:

```bash
docker compose up -d
```

**Expected output:**
```
[+] Running 3/3
 ✔ Network n8n_n8n-network  Created
 ✔ Volume n8n_n8n_data      Created
 ✔ Container n8n            Started
```

### Step 10: Verify n8n is Running

Check the container status:

```bash
docker compose ps
```

You should see the n8n container with status "Up".

View logs to ensure it started correctly:

```bash
docker compose logs -f
```

Look for a line like: `Editor is now accessible via: http://0.0.0.0:5678/`

Press `Ctrl+C` to exit the logs.

## 🌐 Accessing n8n

Open your web browser and navigate to:

```
http://YOUR_PI_IP:5678
```

For example: `http://192.168.0.168:5678`

On first access, you'll be prompted to:
1. Create an owner account (email and password)
2. Log in using the Basic Auth (admin / password).
3. Set up your workspace

## 🔧 Useful Commands

### Check n8n Status
```bash
docker compose ps
```

### View Live Logs
```bash
docker compose logs -f n8n
```

### Stop n8n
```bash
docker compose down
```

### Restart n8n
```bash
docker compose restart
```

### Update n8n to Latest Version
```bash
docker compose pull
docker compose up -d
```

### Remove n8n (Keep Data)
```bash
docker compose down
```

### Remove n8n and All Data
```bash
docker compose down -v
```

## 🛠️ Troubleshooting

### Can't Access n8n Web Interface

1. **Check if container is running:**
   ```bash
   docker compose ps
   ```

2. **Check firewall settings:**
   ```bash
   sudo ufw status
   ```
   If active, allow port 5678:
   ```bash
   sudo ufw allow 5678/tcp
   ```

3. **Verify correct IP address:**
   ```bash
   hostname -I
   ```

### Container Keeps Restarting

Check the logs for errors:
```bash
docker compose logs n8n
```

### Port Already in Use

If port 5678 is already in use, change it in `docker-compose.yaml`:
```yaml
ports:
  - "8080:5678"  # Change 8080 to any available port
```

Then restart:
```bash
docker compose down
docker compose up -d
```

## 📊 Data Persistence

Your n8n workflows and data are stored in a Docker volume named `n8n_data`. This means:
- ✅ Data persists across container restarts
- ✅ Data survives container recreation
- ✅ Can be backed up using Docker volume commands

### Backup Your Data

```bash
docker run --rm -v n8n_n8n_data:/data -v $(pwd):/backup alpine tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz -C /data .
```

### Restore From Backup

```bash
docker run --rm -v n8n_n8n_data:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xzf /backup/YOUR_BACKUP_FILE.tar.gz"
```

## 📚 Additional Resources

- [n8n Official Documentation](https://docs.n8n.io/)
- [n8n Community Forum](https://community.n8n.io/)
- [n8n GitHub Repository](https://github.com/n8n-io/n8n)
- [Docker Documentation](https://docs.docker.com/)

## 📝 Notes

- n8n runs on port 5678 by default
- The container will automatically restart unless stopped manually
- Data is persisted in Docker volumes, so your workflows won't be lost on restart
- For remote access outside your local network, consider setting up a VPN or reverse proxy with HTTPS

---

**Happy Automating! 🎉**
