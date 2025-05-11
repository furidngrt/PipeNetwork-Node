# POP Cache Node Setup Guide (Linux)

This guide outlines how to set up and configure a Pipe Network POP Cache Node on a Linux server, using `screen` for persistent session handling. Make sure to have your **invite code** ready, sent via email after registration.

## 1. System Requirements

**Recommended specs:**

* 4+ CPU cores
* 16 GB+ RAM
* 100 GB+ SSD
* 1 Gbps+ internet connection

## 2. Preparing the System

### A. Install dependencies

```bash
sudo apt update
sudo apt install -y libssl-dev ca-certificates
```

### B. Optimize networking

```bash
sudo bash -c 'cat > /etc/sysctl.d/99-popcache.conf << EOL
net.ipv4.ip_local_port_range = 1024 65535
net.core.somaxconn = 65535
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.core.wmem_max = 16777216
net.core.rmem_max = 16777216
EOL'

sudo sysctl -p /etc/sysctl.d/99-popcache.conf
```

### C. Increase file limits

```bash
sudo bash -c 'cat > /etc/security/limits.d/popcache.conf << EOL
*    hard nofile 65535
*    soft nofile 65535
EOL'
```

Logout and back in to apply.

## 3. Install POP Cache Binary

### A. Create directory

```bash
sudo mkdir -p /opt/popcache
cd /opt/popcache
```

### B. Download binary

Go to: [https://download.pipe.network/](https://download.pipe.network/) and download the latest `pop` binary.

Example (replace with latest version):

```bash
wget https://download.pipe.network/pop-v0.3.0-linux-x64.tar.gz
sudo tar -xzf pop-v0.3.0-linux-x64.tar.gz -C /usr/local/bin
sudo chmod +x /usr/local/bin/pop
```

## 4. Configuration

### A. Create cache directory

```bash
mkdir -p /opt/popcache/cache
```

### B. Create config.json

```bash
cd /opt/popcache
nano config.json
```

Example content:

```json
{
  "pop_name": "your-pop-name",
  "pop_location": "Your Location, Country",
  "server": {
    "host": "0.0.0.0",
    "port": 443,
    "http_port": 80,
    "workers": 40
  },
  "cache_config": {
    "memory_cache_size_mb": 4096,
    "disk_cache_path": "./cache",
    "disk_cache_size_gb": 100,
    "default_ttl_seconds": 86400,
    "respect_origin_headers": true,
    "max_cacheable_size_mb": 1024
  },
  "api_endpoints": {
    "base_url": "https://dataplane.pipenetwork.com"
  },
  "identity_config": {
    "node_name": "your-node-name",
    "name": "Your Name",
    "email": "your.email@example.com",
    "website": "https://your-website.com",
    "discord": "your_discord",
    "telegram": "your_telegram",
    "solana_pubkey": "YOUR_SOLANA_WALLET"
  },
  "invite_code": "YOUR_INVITE_CODE"
}
```

Replace `YOUR_INVITE_CODE` with the actual code you received.

## 5. Run the Node Using screen

### A. Start a screen session

```bash
screen -S popcache
```

### B. Run the node

```bash
pop --config /opt/popcache/config.json
```

To detach from screen: `Ctrl+A` then `D`

To reattach: `screen -r popcache`

## 6. View Logs

```bash
tail -f /opt/popcache/logs/stdout.log
journalctl -u popcache -n 100 -f
```

## 7. Enable Systemd Service (Optional)

### A. Create service file

```bash
sudo nano /etc/systemd/system/popcache.service
```

Paste:

```ini
[Unit]
Description=POP Cache Node
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/popcache
ExecStart=/usr/local/bin/pop --config /opt/popcache/config.json
Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

### B. Enable the service

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now popcache
```

## 8. Monitor Metrics

```bash
curl http://localhost/state
curl http://localhost/metrics
curl http://localhost/health
```

---

## 🔐 Firewall Rules

```bash
sudo ufw allow 443/tcp
sudo ufw allow 80/tcp
```

## 🔁 Log Rotation (Optional)

```bash
sudo nano /etc/logrotate.d/popcache
```

Paste:

```conf
/opt/popcache/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 root root
    sharedscripts
    postrotate
        systemctl reload popcache >/dev/null 2>&1 || true
    endscript
}
```

---

## ✅ Final Checks

```bash
sudo netstat -tuln | grep -E ':(80|443)'
```

Your POP Cache Node should now be fully operational!
