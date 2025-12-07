# Caddy with Podman Rootless + Quadlet

Rootless Podman deployment of Caddy using systemd Quadlet with Cloudflare DNS challenge for SSL certificates.

## Prerequisites

```bash
# Enable user lingering (keeps services running after logout)
sudo loginctl enable-linger <user>

# Allow unprivileged ports 80/443
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/podman-ports.conf
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80
```

**Recommended: Increase UDP buffer sizes for QUIC (HTTP/3) performance**

QUIC transfers on high-bandwidth connections can be limited by UDP buffer sizes. Increase the maximum buffer size:

```bash
sudo sysctl -w net.core.rmem_max=7500000
sudo sysctl -w net.core.wmem_max=7500000
```

To persist across reboots:
```bash
echo "net.core.rmem_max=7500000" | sudo tee /etc/sysctl.d/99-udp-buffer.conf
echo "net.core.wmem_max=7500000" | sudo tee -a /etc/sysctl.d/99-udp-buffer.conf
```

## File Structure

- `caddy-secrets.example.yaml` - Template for Cloudflare API token secret
- `caddy-pod.yaml` - Kubernetes pod definition for Caddy
- `caddy-pod.kube` - Quadlet file that creates systemd service
- `my-caddy/Caddyfile` - Caddy configuration (mounted into container)

## Configuration

**Caddyfile** - Edit `my-caddy/Caddyfile` to set your domain(s):
```caddyfile
{
	acme_dns cloudflare {env.CF_API_TOKEN}
}

test.example.com {
	respond "Hello!"
}
```

**Secrets** - Copy `caddy-secrets.example.yaml` to `caddy-secrets.yaml` and add token:
```yaml
stringData:
  CF_API_TOKEN: your_cloudflare_api_token_here
```

> [!TIP]
> **How to get Cloudflare API Token?**
>
> Go to the "API Tokens" section in your Cloudflare dashboard (https://dash.cloudflare.com/profile/api-tokens).
> Create a new token using the "Edit zone DNS" template, selecting the specific zone(s) you want Caddy to manage.

**Quadlet** - Edit `caddy-pod.kube` line 8 to absolute path:
```ini
Yaml=/home/youruser/caddy-podman/caddy-pod.yaml
```

## Deployment

```bash
# 1. Deploy secrets (after configuring caddy-secrets.yaml)
podman kube play caddy-secrets.yaml

# 2. Install Quadlet service (after setting correct Yaml path)
mkdir -p ~/.config/containers/systemd
cp caddy-pod.kube ~/.config/containers/systemd/
systemctl --user daemon-reload
systemctl --user start caddy-pod.service
```

## Verification

```bash
systemctl --user status caddy-pod.service
journalctl --user -u caddy-pod.service -f
podman pod ps
```

## Updating

```bash
# After config changes
systemctl --user restart caddy-pod.service
```

## Troubleshooting

Quadlet unit fails to generate:
```bash
/usr/libexec/podman/quadlet --user --dryrun
journalctl --user -xe
```

Port permission denied:
```bash
sysctl net.ipv4.ip_unprivileged_port_start  # Should be ≤80
```

Missing container image:
```bash
podman build -t my-caddy:latest ./my-caddy/
```