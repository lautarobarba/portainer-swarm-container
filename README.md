# Portainer Swarm Container

Docker Compose stack for deploying Portainer CE on a Docker Swarm cluster with Traefik reverse proxy.

## Overview

This stack deploys Portainer CE with its agent across a Docker Swarm cluster. Traefik handles TLS termination and routing via the `portainer.homelab.local` hostname.

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

| Dependency   | Type     | Notes                                                 |
| ------------ | -------- | ----------------------------------------------------- |
| Docker Swarm | Required | Cluster must be initialized                           |
| Traefik      | Required | Must be running with `traefik-network`                |
| DNS          | Required | `portainer.homelab.local` must resolve to the cluster |

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

### 2. Crear la estructura de directorios en el NFS

```bash
mkdir -p /srv/nfs/portainer
chmod 777 /srv/nfs/portainer
chown -R nobody:nogroup /srv/nfs/portainer
```

### 3. Deploy the stack

```bash
docker stack deploy -c compose.yaml portainer
```

### 4. Verify services

```bash
docker stack ps portainer
```

### 5. Token para registro

Para crear la primer cuenta de administrador necesitamos el setup_token

```bash
$ docker service logs -f portainer_portainer
# Buscar dentro de los primeros logs el output con el setup_token
...
portainer_portainer > github.com/portainer/portainer/api/cmd/portainer/main.go:256 > no administrator account configured; admin initialization and backup restore require this setup token in the X-Setup-Token header. Start with --no-setup-token to disable. | setup_token=9999999999999999999999999999
...
```

Open `https://portainer.homelab.local` in your browser.

### 6. Access Portainer

Open `https://portainer.homelab.local` in your browser.

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

> **Note:** The `portainer-data` volume persists after removal. Delete it manually if needed:
>
> ```bash
> docker volume rm portainer-portainer_data
> ```

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
