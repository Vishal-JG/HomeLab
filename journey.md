## Day 1
Installed K3s via the official install script. Verified node status with kubectl get nodes.

## Day 2
Set up Tailscale Kubernetes Operator via Helm to expose cluster access securely over my tailnet without opening ports.

## Day 3
Deployed nginx and exposed it via a Tailscale Ingress (ingressClassName: tailscale)
instead of a LoadBalancer, to keep resource usage low on older hardware and to set up
a pattern that scales to multiple apps behind one entry point.

Hit a CrashLoopBackOff on the Tailscale operator pod — logs showed a 403 error:
"calling actor does not have enough permissions to perform this function" when the
operator tried to create an auth key. Root cause was the OAuth client scope missing
Auth Keys write access. Fixed by regenerating the OAuth client with correct scopes
and reapplying via Helm.

Confirmed working: app reachable at https://nginx.taild425ba.ts.net from my laptop
in Sydney, proxying to the cluster running in Singapore.
