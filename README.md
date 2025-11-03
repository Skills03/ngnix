# Substrate Blockchain Load Balancer

High-availability HTTPS/WSS load balancer for Substrate blockchain validators using Nginx.

## Architecture

```
Frontend → wss://substrate-compute-27586.duckdns.org
           ↓
       Load Balancer (34.238.51.245)
           ↓
   ┌───────┼───────┐
   ↓       ↓       ↓
Val-1   Val-2   Val-3
:9944   :9944   :9947
```

## Features

- **Auto-failover**: Automatic failover with health checks (max_fails=2, fail_timeout=10s)
- **SSL/TLS**: Let's Encrypt certificate with auto-renewal
- **WebSocket Support**: Optimized for Polkadot.js API connections
- **Load Balancing**: Round-robin across 3 validators
- **High Availability**: Any validator can crash without downtime

## Configuration

### Active Validators

- Validator-1: `172.31.35.84:9944`
- Validator-2: `172.31.89.170:9944`
- Validator-3: `172.31.89.170:9947`

### Endpoints

- **HTTPS**: https://substrate-compute-27586.duckdns.org
- **WSS**: wss://substrate-compute-27586.duckdns.org
- **Health Check**: https://substrate-compute-27586.duckdns.org/health
- **Docker API**: https://substrate-compute-27586.duckdns.org/docker/ (proxies to validator-1:8080)

### Nginx Optimizations

- `proxy_buffering off` - Disable buffering for WebSocket streaming
- `proxy_read_timeout 300s` - Extended timeout for metadata downloads
- `proxy_buffer_size 128k` - Large buffers for metadata responses
- `client_max_body_size 50M` - Support large requests

## Installation

### Prerequisites

- Ubuntu Server
- Nginx
- Certbot (Let's Encrypt)
- DNS configured to point to load balancer IP

### Setup

1. Install dependencies:
```bash
sudo apt-get update
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

2. Copy configuration:
```bash
sudo cp nginx-loadbalancer.conf /etc/nginx/sites-available/blockchain-lb
sudo ln -sf /etc/nginx/sites-available/blockchain-lb /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
```

3. Obtain SSL certificate:
```bash
sudo certbot certonly --nginx -d substrate-compute-27586.duckdns.org \
  --non-interactive --agree-tos \
  --email admin@substrate-compute-27586.duckdns.org \
  --no-eff-email
```

4. Test and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Security Groups

### Load Balancer Security Group

**Inbound Rules:**
- Port 443 (HTTPS): `0.0.0.0/0`
- Port 80 (HTTP): `0.0.0.0/0`
- Port 22 (SSH): `0.0.0.0/0`

**Outbound Rules:**
- Port 9944 (RPC): `0.0.0.0/0`
- Port 9947 (RPC): `0.0.0.0/0`
- All traffic: `0.0.0.0/0`

### Validator Security Group

**Inbound Rules:**
- Port 9944 (RPC): `0.0.0.0/0` or Load Balancer IP
- Port 9947 (RPC): `0.0.0.0/0` or Load Balancer IP
- Port 30333-30336 (P2P): `0.0.0.0/0`

## Testing

### Health Check
```bash
curl https://substrate-compute-27586.duckdns.org/health
```

### RPC Query
```bash
curl -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_chain"}' \
  https://substrate-compute-27586.duckdns.org
```

### WebSocket Connection
```javascript
const { ApiPromise, WsProvider } = require('@polkadot/api');

const wsProvider = new WsProvider('wss://substrate-compute-27586.duckdns.org');
const api = await ApiPromise.create({ provider: wsProvider });
```

## Monitoring

Check nginx logs:
```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

Check SSL certificate expiry:
```bash
sudo certbot certificates
```

## Troubleshooting

### WebSocket Connection Timeouts

If Polkadot.js API initialization times out:
1. Check `proxy_read_timeout` is set to 300s or higher
2. Ensure `proxy_buffering off` is set
3. Verify `client_max_body_size` is at least 50M
4. Check validator RPC endpoints are accessible from load balancer

### SSL Certificate Issues

Renew certificate manually:
```bash
sudo certbot renew --force-renewal
sudo systemctl reload nginx
```

## Maintenance

Certificate auto-renewal is handled by certbot systemd timer. Check status:
```bash
sudo systemctl status certbot.timer
```

## License

MIT
