# Portainer Swarm Container

Docker Compose stack for deploying Portainer CE on a Docker Swarm cluster with Traefik reverse proxy.

## Overview

This stack deploys Portainer CE (v2.21.5) with its agent across a Docker Swarm cluster. Traefik handles TLS termination and routing via the `portainer.homelab.local` hostname.

### Architecture

```
                    Traefik (external)
                           |
                    traefik-network
                           |
                     +----------+
                     | Portainer |  <-- manager node only (1 replica)
                     +----------+
                           |
                   portainer-network (overlay)
                           |
              +------------------------+
              |                        |
        +----------+            +----------+
        | Agent    |            | Agent    |  <-- all Linux nodes (global)
        +----------+            +----------+
```

### Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| Docker Swarm | Required | Cluster must be initialized |
| Traefik | Required | Must be running with `traefik-network` |
| DNS | Required | `portainer.homelab.local` must resolve to the cluster |

## Prerequisites

- Docker Engine with Swarm mode enabled
- Traefik stack already deployed with an external network named `traefik-network`
- DNS record pointing `portainer.homelab.local` to your swarm manager

## Deployment

### 1. Verify prerequisites

```bash
docker node ls
docker network ls | grep traefik-network
```

### 2. Deploy the stack

```bash
docker stack deploy -c compose.yaml portainer
```

### 3. Verify services

```bash
docker stack ps portainer
```

### 4. Access Portainer

Open `https://portainer.homelab.local` in your browser.

## Configuration

### Networks

| Network | Scope | Purpose |
|---------|-------|---------|
| `traefik-network` | External (pre-existing) | Connects Portainer to Traefik for routing |
| `portainer-network` | Overlay (created) | Internal communication between Portainer and agents |

### Volumes

| Volume | Purpose |
|--------|---------|
| `portainer_data` | Persistent Portainer data (database, configs) |

### Services

| Service | Image | Mode | Placement |
|---------|-------|------|-----------|
| `portainer` | `portainer/portainer-ce:2.21.5` | Replicated (1) | Manager nodes only |
| `agent` | `portainer/agent:2.21.5` | Global | All Linux nodes |

## Runbooks

### Update Portainer version

Edit `compose.yaml` and update the image tags:

```yaml
portainer:
  image: portainer/portainer-ce:<new-version>
agent:
  image: portainer/agent:<new-version>
```

Then redeploy:

```bash
docker stack deploy -c compose.yaml portainer
```

### View logs

```bash
# Portainer server logs
docker service logs portainer_portainer

# Agent logs (all nodes)
docker service logs portainer_agent
```

### Remove the stack

```bash
docker stack rm portainer
```

> **Note:** The `portainer_data` volume persists after removal. Delete it manually if needed:
> ```bash
> docker volume rm portainer_portainer_data
> ```

## TLS / Certificates

Traefik handles TLS termination for Portainer via the `websecure` entrypoint.

### Certificate resolution

Traefik must have a certificate resolver configured for `portainer.homelab.local`. Common setups:

| Method | Traefik config | Notes |
|--------|---------------|-------|
| **Let's Encrypt (HTTP challenge)** | `certificatesResolvers.le.acme` | Requires port 80 open |
| **Let's Encrypt (DNS challenge)** | `certificatesResolvers.le.dnsChallenge` | Works behind NAT |
| **Self-signed / Local CA** | `tls.stores.default.defaultCertificate` | For internal use only |

### Verify TLS

```bash
# Check certificate expiry
echo | openssl s_client -connect portainer.homelab.local:443 -servername portainer.homelab.local 2>/dev/null | openssl x509 -noout -dates

# Check Traefik certificate store
docker exec <traefik-container> cat /etc/traefik/acme.json | jq '.[].Certificates[] | select(.domain.main == "portainer.homelab.local")'
```

### Using a custom certificate

If you need to provide your own certificate, mount it into the Traefik container and configure:

```yaml
# In your Traefik compose file
tls:
  certificates:
    - certFile: /certs/portainer.crt
      keyFile: /certs/portainer.key
```

## Troubleshooting

### Portainer service won't start

**Symptom**: Service stuck in `pending` state.
**Cause**: No manager node available or `traefik-network` doesn't exist.
**Fix**:
```bash
docker node ls                          # verify manager exists
docker network ls | grep traefik-network  # verify network exists
```

### Cannot access via hostname

**Symptom**: Browser cannot reach `portainer.homelab.local`.
**Cause**: DNS not configured or Traefik not routing correctly.
**Fix**:
- Verify DNS resolves: `nslookup portainer.homelab.local`
- Check Traefik dashboard for the route
- Verify Traefik labels in `compose.yaml` match your Traefik config

### Agent not connecting to Portainer

**Symptom**: Nodes not showing in Portainer UI.
**Cause**: Network connectivity issue between services.
**Fix**:
```bash
# Verify both services are on the same overlay network
docker service inspect portainer_portainer --format '{{json .Spec.TaskTemplate.Networks}}'
docker service inspect portainer_agent --format '{{json .Spec.TaskTemplate.Networks}}'
```
