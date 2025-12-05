# Deploying on Synology NAS

## Prerequisites

1. **Enable TUN/TAP Support**
   - SSH into your Synology NAS
   - Check if TUN device exists:
     ```bash
     ls -la /dev/net/tun
     ```
   - If it doesn't exist, create it:
     ```bash
     sudo mkdir -p /dev/net
     sudo mknod /dev/net/tun c 10 200
     sudo chmod 666 /dev/net/tun
     ```

2. **Load TUN Module** (if needed)
   ```bash
   sudo modprobe tun
   ```

3. **Verify Docker Compose Version**
   - Ensure you have Docker Compose v2.x or higher
   ```bash
   docker-compose --version
   ```

## Deployment Steps

1. **Copy files to NAS**
   ```bash
   scp -r /Users/tholtz/code/claude-poe user@your-nas-ip:/volume1/docker/
   ```

2. **SSH into your NAS**
   ```bash
   ssh user@your-nas-ip
   cd /volume1/docker/claude-poe
   ```

3. **Edit the .env file** with your keys
   ```bash
   nano .env
   ```

4. **Start the services**
   ```bash
   docker-compose up -d
   ```

5. **Check the status**
   ```bash
   docker-compose ps
   docker-compose logs -f
   ```

## Troubleshooting

### TUN/TAP Device Issues

If you get errors about `/dev/net/tun`, you need to enable TUN/TAP support:

```bash
# Create TUN device
sudo mkdir -p /dev/net
sudo mknod /dev/net/tun c 10 200
sudo chmod 666 /dev/net/tun

# Load TUN module
sudo modprobe tun

# Make it persistent (add to /etc/rc.local)
echo "modprobe tun" | sudo tee -a /etc/rc.local
```

### Permission Denied on /dev/net/tun

Some Synology models have restrictions on device access. You may need to:
1. Enable Docker in privileged mode in DSM
2. Or use Tailscale userspace mode (add to .env):
   ```
   TS_USERSPACE=true
   ```

### Container Fails to Start

1. **Check Tailscale health:**
   ```bash
   docker-compose logs tailscale
   docker exec claude-poe-tailscale-1 tailscale status
   ```

2. **Check if Tailscale is healthy:**
   ```bash
   docker inspect claude-poe-tailscale-1 | grep Health -A 10
   ```

3. **Restart services:**
   ```bash
   docker-compose down
   docker-compose up -d
   ```

### LiteLLM Worker Logs

You may see messages like "Child process died" in the LiteLLM logs. This is normal behavior:
- LiteLLM uses multiple worker processes
- Some workers may exit during initialization
- As long as `docker-compose ps` shows the container as "Up", the service is working
- Test the `/health` endpoint to verify the proxy is responding

### Network Namespace Error

If you see `no such file or directory` errors related to network namespaces:
- The healthcheck ensures Tailscale is fully started before LiteLLM tries to join its network
- Wait 30-60 seconds and check logs again
- If persistent, try restarting Docker daemon on NAS

### Tailscale iptables Warnings

You may see iptables error messages in Tailscale logs like:
```
iptables v1.8.11 (nf_tables): Could not fetch rule set generation id: Invalid argument
```

These warnings are harmless and can be ignored:
- Synology's kernel may have limited nftables support
- Tailscale will fall back to alternative methods
- As long as `tailscale status` shows "Running", the VPN is working correctly
- Check the health endpoint to confirm connectivity

## Testing from Remote Machine

Once deployed on your NAS, test from your local machine (not on the NAS):

```bash
# Make sure you're connected to Tailscale
tailscale status

# Get your Tailscale hostname or IP
# Look for the machine with hostname "litellm-proxy" in the output

# Test the health endpoint (replace with your Tailscale hostname/IP and master key)
curl http://litellm-proxy.tail1234.ts.net:4000/health \
  -H "Authorization: Bearer sk-your_litellm_master_key_here"

# Or use Tailscale IP directly:
curl http://100.x.y.z:4000/health \
  -H "Authorization: Bearer sk-your_litellm_master_key_here"

# Configure Claude Code (replace with your values)
export ANTHROPIC_BASE_URL=http://litellm-proxy.tail1234.ts.net:4000
export ANTHROPIC_AUTH_TOKEN=sk-your_litellm_master_key_here

# Test Claude Code
claude --verbose
```

## Synology-Specific Notes

- **Container Manager**: You can also manage this through Synology's Container Manager UI
- **Auto-start**: Enable auto-start in Container Manager to have services start on boot
- **Firewall**: No firewall rules needed - Tailscale handles all networking
- **Resource Limits**: Consider setting memory/CPU limits in Container Manager if your NAS has limited resources
