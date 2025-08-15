# RustDesk Docker Server Setup

Complete Docker deployment solution for RustDesk server with Cloudflare DNS and Tunnel configuration.

## Features

- **Complete RustDesk Server Stack**: hbbs (signal server) + hbbr (relay server)
- **Docker Compose**: Easy deployment and management
- **Persistent Storage**: Data persistence across container restarts
- **Environment Configuration**: Customizable server settings
- **Cloudflare Integration**: DNS and Tunnel setup for secure access
- **Production Ready**: Health checks and restart policies

## Prerequisites

- Ubuntu/Debian server with public IP
- Docker and Docker Compose installed
- Domain name managed by Cloudflare
- Cloudflare account with API access

## Quick Start

### 1. Clone Repository and Deploy

```bash
# Clone the repository
git clone https://github.com/a-ariff/rustdesk-docker.git
cd rustdesk-docker

# Set your server IP
export RUSTDESK_SERVER_IP="your-server-public-ip"

# Create environment file
echo "RUSTDESK_SERVER_IP=$RUSTDESK_SERVER_IP" > .env

# Deploy the services
docker compose up -d

# Verify deployment
docker compose ps
docker compose logs
```

### 2. Configure Firewall

```bash
# Allow RustDesk ports
sudo ufw allow 21115/tcp
sudo ufw allow 21116/udp
sudo ufw allow 21117/tcp
sudo ufw allow 21118/tcp
sudo ufw allow 21119/tcp

# Enable firewall if not already enabled
sudo ufw --force enable
sudo ufw status
```

## Cloudflare DNS Configuration

### Create A Record

```bash
# Replace with your values
DOMAIN="your-domain.com"
SUBDOMAIN="rust"
SERVER_IP="your-server-public-ip"
CF_API_TOKEN="your-cloudflare-api-token"
ZONE_ID="your-zone-id"

# Create DNS A record
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "A",
    "name": "'$SUBDOMAIN'",
    "content": "'$SERVER_IP'",
    "ttl": 300,
    "proxied": false
  }'
```

### Alternative: Manual DNS Setup

1. Go to Cloudflare Dashboard
2. Select your domain
3. Go to DNS > Records
4. Add A record:
   - **Name**: `rust`
   - **IPv4 address**: `your-server-ip`
   - **Proxy status**: DNS only (gray cloud)
   - **TTL**: Auto

## Cloudflare Tunnel Setup

### 1. Install cloudflared

```bash
# Download and install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Verify installation
cloudflared --version
```

### 2. Authenticate and Create Tunnel

```bash
# Authenticate with Cloudflare (opens browser)
cloudflared tunnel login

# Create a new tunnel
cloudflared tunnel create rustdesk-server

# Note the tunnel ID from output
TUNNEL_ID="your-tunnel-id"
```

### 3. Configure Tunnel Services

```bash
# Create tunnel configuration
sudo mkdir -p /etc/cloudflared

# Create config file
sudo tee /etc/cloudflared/config.yml > /dev/null <<EOF
tunnel: $TUNNEL_ID
credentials-file: /root/.cloudflared/$TUNNEL_ID.json

ingress:
  # RustDesk Signal Server (hbbs)
  - hostname: rust.$DOMAIN
    service: tcp://localhost:21115
  
  # RustDesk Relay Server (hbbr) - TCP services
  - hostname: rust-relay.$DOMAIN
    service: tcp://localhost:21117
  
  # Catch-all rule (required)
  - service: http_status:404
EOF
```

### 4. Create DNS Records for Tunnel

```bash
# Create CNAME records for tunnel endpoints
cloudflared tunnel route dns $TUNNEL_ID rust.$DOMAIN
cloudflared tunnel route dns $TUNNEL_ID rust-relay.$DOMAIN
```

### 5. Install and Start Tunnel Service

```bash
# Install as system service
sudo cloudflared service install

# Start the service
sudo systemctl start cloudflared
sudo systemctl enable cloudflared

# Check status
sudo systemctl status cloudflared
```

## Port Configuration Summary

| Service | Port | Protocol | Purpose |
|---------|------|----------|----------|
| hbbs | 21115 | TCP | RustDesk signal server |
| hbbs | 21116 | UDP | RustDesk discovery |
| hbbr | 21117 | TCP | RustDesk relay server |
| hbbr | 21118 | TCP | RustDesk relay server |
| hbbr | 21119 | TCP | RustDesk relay server |

## Client Configuration

### Method 1: Direct IP Connection

1. Open RustDesk client
2. Click the **⚙️ Settings** button
3. Go to **Network** tab
4. Set **ID Server**: `your-server-ip:21115`
5. Set **Relay Server**: `your-server-ip:21117`
6. Click **OK**

### Method 2: Domain Connection (with Cloudflare DNS)

1. Open RustDesk client
2. Click the **⚙️ Settings** button  
3. Go to **Network** tab
4. Set **ID Server**: `rust.your-domain.com:21115`
5. Set **Relay Server**: `rust.your-domain.com:21117`
6. Click **OK**

### Method 3: Cloudflare Tunnel Connection

1. Open RustDesk client
2. Click the **⚙️ Settings** button
3. Go to **Network** tab  
4. Set **ID Server**: `rust.your-domain.com`
5. Set **Relay Server**: `rust-relay.your-domain.com`
6. Click **OK**

### Client Configuration Commands

```bash
# For automated client deployment
RUSTDESK_SERVER="rust.your-domain.com"
RUSTDESK_RELAY="rust-relay.your-domain.com"

# Windows Registry (Run as Administrator)
reg add "HKEY_CURRENT_USER\\Software\\RustDesk\\config" /v "option" /t REG_SZ /d "{\"relay-server\":\"$RUSTDESK_RELAY\",\"id-server\":\"$RUSTDESK_SERVER\"}"

# Linux config file
mkdir -p ~/.config/rustdesk
echo '{"relay-server":"'$RUSTDESK_RELAY'","id-server":"'$RUSTDESK_SERVER'"}' > ~/.config/rustdesk/RustDesk2.toml
```

## Management Commands

### Server Management

```bash
# View logs
docker compose logs -f

# Restart services
docker compose restart

# Stop services
docker compose down

# Update to latest images
docker compose pull
docker compose up -d

# View running containers
docker compose ps
```

### Tunnel Management

```bash
# Check tunnel status
sudo systemctl status cloudflared

# View tunnel logs
sudo journalctl -u cloudflared -f

# Restart tunnel
sudo systemctl restart cloudflared

# List active tunnels
cloudflared tunnel list

# Test tunnel connectivity
cloudflared tunnel info $TUNNEL_ID
```

## Troubleshooting

### Common Issues

1. **Connection Failed**
   ```bash
   # Check if services are running
   docker compose ps
   
   # Check port accessibility
   telnet your-server-ip 21115
   ```

2. **Firewall Issues**
   ```bash
   # Check UFW status
   sudo ufw status
   
   # Check iptables
   sudo iptables -L
   ```

3. **DNS Resolution**
   ```bash
   # Test DNS resolution
   nslookup rust.your-domain.com
   dig rust.your-domain.com
   ```

4. **Tunnel Issues**
   ```bash
   # Check tunnel configuration
   sudo cat /etc/cloudflared/config.yml
   
   # Test tunnel connectivity
   curl -I https://rust.your-domain.com
   ```

### Log Locations

- **Docker logs**: `docker compose logs`
- **Cloudflare tunnel logs**: `sudo journalctl -u cloudflared`
- **System logs**: `/var/log/syslog`

## Security Considerations

1. **Firewall**: Only allow necessary ports
2. **Updates**: Keep Docker images and system updated
3. **Monitoring**: Monitor access logs regularly
4. **Encryption**: RustDesk uses end-to-end encryption by default
5. **Access Control**: Use strong passwords for RustDesk connections

## Production Deployment Checklist

- [ ] Server has static IP address
- [ ] Domain DNS configured correctly
- [ ] Firewall rules applied
- [ ] Docker services running
- [ ] Cloudflare tunnel configured (if used)
- [ ] Client configuration tested
- [ ] Backup strategy implemented
- [ ] Monitoring setup

## Support

For issues and questions:

- **RustDesk Documentation**: https://rustdesk.com/docs/
- **Docker Documentation**: https://docs.docker.com/
- **Cloudflare Tunnel Docs**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
