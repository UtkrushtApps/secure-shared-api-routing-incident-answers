# Solution Steps

1. Verify the broken orchestration state by checking pod status and events: `kubectl -n api2 get pods`, then `kubectl -n api2 describe pod <pod>` and look for missing ConfigMap/Env errors or Pending scheduling due to resources.

2. Find the mismatch in the api2 environment configuration: confirm which namespace the ConfigMap `api2-runtime` exists in (`kubectl -n api2 get configmap api2-runtime` vs `kubectl -n api1 get configmap api2-runtime`).

3. Fix the broken state by updating the manifest so `api2-runtime` is created in the `api2` namespace (set `metadata.namespace: api2`) and re-apply the ConfigMap/Deployment manifests.

4. Confirm both api deployments can roll out cleanly: `kubectl -n api1 rollout status deployment/api1 --timeout=90s` and `kubectl -n api2 rollout status deployment/api2 --timeout=120s`. Ensure api2 reaches all replicas and pods are Ready.

5. Check single-node capacity constraints using the declared resource requests in the manifests: sum requests for api1 (1 replica) and api2 (2 replicas) and compare with node allocatable (`kubectl get nodes -o json` → `status.allocatable`). If CPU is over capacity, reduce only `resources.requests.cpu` in the manifests (keep behavior stable) so both services fit. On the 2-CPU node the system pods hold about 300m and api2's two replicas 1000m, so api1 must request at most 700m; 1000m leaves it Pending. Leave api2's requests as they are: changing its pod spec forces a rollout, and on a full node the replacement pod cannot be scheduled while the old ones still hold their CPU.

6. Re-apply the updated deployment manifests for api1/api2 (`kubectl apply -f ...`) and wait for readiness again using rollout status and `kubectl -n api2 get pods`.

7. Create the TLS material the shared entry point references: generate a self-signed certificate for `api.utkrusht.local` (`openssl req -x509 -nodes -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=api.utkrusht.local"`) and load it as `kubectl -n edge create secret tls shared-api-tls --cert=tls.crt --key=tls.key` (or apply `manifests/edge/tls-secret.yaml`). The Ingress already references `shared-api-tls`; without the Secret the controller serves its default certificate.
8. Validate the secure shared routing: port-forward ingress-nginx and curl both shared paths using the required Host header (e.g. `https://api.utkrusht.local/api1/health` and `/api2/health`). Confirm HTTP 200 for both.

