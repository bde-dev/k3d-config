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

## Devpod

A `devcontainer` spec is available, using `docker-in-docker` to be bale to run the `k3d` clusters inside the `devcontainer`.

> The `devcontainer` provider used in this repo is `devpod`.

`.devcontainer/devcontainer.json` defines an array of `forwardPorts` to allow host browser access to web apps and UI's running in the cluster.
