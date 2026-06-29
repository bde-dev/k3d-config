# k3d Config

Repo containing various configuration files for k3d cluster environments.

Each folder contains a `k3d-config.yaml` file and a `README.md`.

The structure of an environment is as follows:

```bash
environment-name          # Environment folder
├── environment-name.yaml # k3d cluster file identified as <env-name>.yaml
├── ingress.yaml          # Ingress resource allowing access to endpoints in the cluster
├── namespace.yaml        # Namespace resource (Not always required)
├── README.md             # Description of the environment and any bespoke setup instructions
└── other-files.yaml      # Other required files such as values.yaml for helm charts
```


## k3d loadBalancer considerations

`k3d` creates a `loadBalancer` container for each cluster to effectively allow access to that cluster.

Each environment explicitly sets a port mapping for its `loadBalancer` container to use.

This avoids traffic conflicts if there are multiple environments running at once.

It also allows host browser access to web apps and UI's running in the containers via `ingress` resources.

## Private Registries

Clusters can optionally be configured to pull images from private registries or registry mirrors. Each environment references a `registries.yaml` file at the repo root.

**To use private registries:**

1. Copy `registries-example.yaml` to `registries.yaml`
2. Update the endpoints and credentials with your registry details
3. The `registries.yaml` file is gitignored to prevent committing credentials

**To use public registries only:**

Remove the `registries` block from the environment's k3d config file:

```yaml
registries:
  config: ../registries.yaml
```

See the [k3d registries documentation](https://k3d.io/v5.7.5/usage/registries/) for more details.

## Devpod

A `devcontainer` spec is available, using `docker-in-docker` to be bale to run the `k3d` clusters inside the `devcontainer`.

> The `devcontainer` provider used in this repo is `devpod`.

`.devcontainer/devcontainer.json` defines an array of `forwardPorts` to allow host browser access to web apps and UI's running in the cluster.

## Port traversal: host → service

A request from the host laptop to a service inside a k3d cluster passes through four hops:

```
host browser
   │  http://localhost:<port>
   ▼
devpod / devcontainer        ← VSCode forwards listed ports (devcontainer.json → forwardPorts)
   │  same localhost:<port> inside the container (DinD daemon)
   ▼
k3d loadBalancer container   ← published by k3d (cluster yaml → ports: <hostPort>:<nodePort>)
   │  forwards to port 80 on the k3s server node
   ▼
Traefik (k3s default ingress controller, :80)
   │  matches Ingress rule
   ▼
Kubernetes Service → Pod
```

### Worked example: prometheus-lab on port 4006

**1. Devcontainer forwards the port out to the host** — `.devcontainer/devcontainer.json`:

```jsonc
"forwardPorts": [
  4001,
  4002,
  4003,
  4004,
  4005,
  4006
],
"portsAttributes": {
  "4006": { "label": "prometheus-lab" }
}
```

**2. k3d publishes the port and maps it to Traefik (:80) on the server node** — `prometheus-lab/prometheus-lab.yaml`:

```yaml
ports:
  - port: 4006:80
    nodeFilters:
      - loadbalancer
```

**3. Ingress routes the request to the Service** — `prometheus-lab/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-lab
  namespace: monitoring
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-lab-kube-promet-prometheus
                port:
                  number: 9090
```

### What works from the host

- Open `http://localhost:4006` in a browser → hits the Prometheus UI. The full chain (host → devpod → k3d LB → Traefik → Ingress → Service → Pod) runs end-to-end.
- Nothing else is reachable from the host directly. `kubectl` from the host is only available if you have a kubeconfig pointing at the cluster on the host machine; by default k3d writes the kubeconfig inside the devcontainer.

### What works from inside the devpod

- `curl http://localhost:4006` — same chain as the host case (the k3d loadbalancer container publishes to the DinD daemon, which is `localhost` inside the devcontainer). Useful for scripted smoke tests.
- `kubectl port-forward -n monitoring svc/prometheus-lab-kube-promet-prometheus 9090:9090` — bypasses the loadBalancer container and Traefik entirely, talking straight to the Service. Useful when you want to skip Ingress to test the service in isolation.
- `kubectl exec` into any pod, then `curl http://prometheus-lab-kube-promet-prometheus.monitoring.svc:9090` — cluster DNS works **only inside pods**, not from the devcontainer shell.

### Adapting the pattern to other environments

Each environment in this repo follows the same shape — the only things that change are the host port (in both `devcontainer.json` and the cluster yaml) and the Ingress backend service name/port. The `cilium-cni-ingress` environment is an exception: it disables Traefik and uses Cilium's ingress controller instead, so step 4 above changes, but the rest of the chain is identical.
