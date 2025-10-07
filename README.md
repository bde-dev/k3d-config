# k3d Config

Repo containing various configuration files for k3d cluster environments.

Each folder contains a `k3d-config.yaml` file.

Due to how the `loadBalancer` works, each environment explicitly sets a port for the `loadBalancer` to use.

This avoids traffic conflicts if there are multiple environments running at once.

The devcontainer exposes port `8090` on the local machine.

This allows access to services running in clusters by using a `kubectl port-forward`.
