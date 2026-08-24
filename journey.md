## Homelab Journey
**Overview**
This document narrates how I built out my homelab, moving from a bare Ubuntu box to a working Kubernetes cluster exposed over Tailscale, with a CI/CD pipeline and Ansible automation reproducing the whole setup. 


**1. Setting Up K3s and Exposing It Over Tailscale**
I started by installing K3s on my Lenovo AIO running Ubuntu, then worked on exposing the cluster to the outside world without opening it up to the public internet. Tailscale was the natural choice since it let me reach the cluster securely from anywhere, including from Sydney while on exchange, without managing my own VPN infrastructure. The plan was to install the Tailscale Kubernetes Operator via Helm, using OAuth credentials from the Tailscale admin console, then deploy a simple app and expose it through a Tailscale typed LoadBalancer Service or Ingress.

The biggest challenge here was kubeconfig permissions breaking silently in non interactive shells
The first real snag was a permission denied error on etc rancher k3s k3s.yaml when running kubectl manually. I fixed it at the time by exporting KUBECONFIG in my bashrc, and moved on.
This fix turned out to be fragile in a way I did not realize until much later. When I later tried to run kubectl commands through a CI deploy step using tailscale ssh into the box, the exact same permission error came back. The root cause was that tailscale ssh opens a non interactive session, which skips bashrc entirely, so the environment variable I relied on was never set. This was a good lesson in the difference between fixing a problem for how I personally use a machine versus fixing it at the root. The real fix was to either chmod the kubeconfig file itself so permissions were not an issue regardless of shell type, or use an inline export that does not depend on any shell profile loading.

Getting the Tailscale Operator running was not a one shot process. It first crash looped with a 403 calling actor does not have enough permissions error. The OAuth client had Devices Core write access, but the tag it needed was not actually wired to tagOwners correctly, so the operator could not do what it needed to do even though the scope looked broad enough on paper.

A similar issue resurfaced later when setting up a CI runner to join the tailnet. This time the OAuth client had Devices Core scope, but was missing the separate Auth Keys scope needed specifically to generate ephemeral keys for GitHub Actions runners. The lesson here is that Tailscale splits manage existing devices and create new auth keys into two distinct scopes that look related but are not interchangeable, and the generic 403 error gives no hint which one is actually missing. I had to learn to treat any Tailscale permission error as a scope audit problem first, not a config problem.


**2. Building and Deploying a Real App**
Once the test deployment worked, I replaced it with my own app, Briefly, containerizing it with a Dockerfile, pushing the image to a registry, and adding a Service, Ingress, ConfigMap, and Secret so it was reachable at a proper ts.net URL with config and credentials mounted in cleanly.

Twice, the Briefly pod failed to start because the Deployment's env references did not match what was actually created. The first time it was a key name mismatch, TELEGRAM_CHAT_ID singular in the Secret versus the app expecting TELEGRAM_CHAT_IDS plural. The second time it was a full secret name mismatch, briefly-telegram created versus briefly-secrets actually referenced in the deployment spec.
What made both of these unusually annoying to debug was that the old pod, which was nine days old at the time, kept serving traffic while the new pod sat failing. Running kubectl exec against the app kept hitting the still healthy old pod and showing stale placeholder values instead of surfacing the real error, which masked the problem for a while. The eventual fix was simple in both cases, matching names exactly, but it reinforced the habit of checking which pod is actually receiving traffic before trusting any output from exec or logs.


**3. Building the CI/CD Pipeline**
With the app running manually, I moved on to automating builds and deploys. I set up a GitHub Actions workflow to build the Docker image and push it to a registry on every push to main, then added a deploy job that reaches into the homelab over Tailscale to update the running app.

The Tailscale SSH ACL authorization was the most persistent and layered problem in the whole pipeline setup. It started with a tailnet policy does not permit you to SSH to this node error, which took a while to understand because Tailscale SSH authorization is governed by a completely separate ssh policy block from normal network ACLs. Being reachable on the tailnet and being allowed to SSH in are two entirely different permission systems, and I had assumed they were the same thing.
Fixing that surfaced a second issue, that the dst field in SSH rules only accepts users, groups, or tags, never a raw hostname or IP. I tried setting dst to bytecave directly and then to its IP, and both were rejected, because SSH rules operate on identity rather than network addressing the way normal ACL grants do.

The fix for that, tagging the machine with tag:homelab, triggered a third issue: tagging a device instantly strips it out of autogroup:self, which cut off my own manual SSH access mid session the moment I ran tailscale up --advertise-tags. The full fix ended up needing two separate SSH rules, one for my personal user and one for the CI runner's tag:ci, plus a grants entry explicitly opening port 22. This whole chain was a strong lesson in how Tailscale layers network reachability, SSH authorization, and device identity as three separate systems that all have to line up.

Next, CI/CD pipeline only handled image updates, not full manifest reconciliation. Once deploys were running, I noticed the CronJob manifest that had existed in the repo from day one was never actually live on the cluster, kubectl get cronjobs returned nothing days into using the app. The original deploy job only ran kubectl set image, which patches the container image tag on an existing Deployment but never applies changes to the CronJob, ConfigMap, Service, or Ingress files sitting in the repo.
The fix was to rewrite the deploy step to git pull on the box and run kubectl apply -f . before the image patch, turning the pipeline from an image only deploy into genuine GitOps style reconciliation where the whole manifest directory is the source of truth, not just the image tag.


**4. Automating the Setup With Ansible**
To make the whole build reproducible from a bare Ubuntu box, I wrote an Ansible playbook covering K3s install, Tailscale setup, and app deploy, structured as roles so it could be torn down and re-run to prove idempotency.

Authentication chain failures across SSH key, passphrase, host key, and sudo was the longest and most tangled issue of the whole project, spanning many back and forth debugging exchanges. It was not one bug, it was a stack of four independent auth layers failing in sequence. First, an unrecognized host key under a new IP alias. Then an SSH key protected by a forgotten passphrase. Then no SSH agent running to cache credentials. Then finally a missing sudo password for Ansible's become true setting. Each layer looked like the problem until I resolved it, only to reveal the next one underneath.

The lesson that generalizes well beyond this one project is that Ansible refuses to answer any interactive prompt, whether that is a passphrase, a password, or a host key confirmation. Every one of those needs a non interactive path before automation can work, such as ssh-keyscan for host keys, a passphrase-less key or an agent for the SSH key, and either ask-become-pass or a sudoers rule for sudo. 

Several distinct failures all traced back to the same root cause, assuming a file path would resolve relative to something intuitive when Ansible actually resolves it differently at runtime. This showed up three separate times. The original relative manifest paths broke because the kubernetes.core.k8s module executes on the control node rather than the target machine. An inventory file could not be found because the command was not run from the expected working directory. And a leftover flat tasks block in the playbook ran in addition to the new roles list, silently duplicating work and making it unclear which version of a task was actually executing.
The fix pattern that solved this, anchoring paths explicitly with playbook_dir followed by the relative path, is the transferable lesson: never trust bare relative paths in Ansible, always anchor them explicitly to something Ansible actually guarantees, like the playbook's own directory.

The final stretch of failures were not logic bugs at all, they were files containing the wrong thing while looking superficially fine. A stray literal b character got appended to configmap.yaml. A UTF-8 BOM got invisibly prepended to files created via PowerShell's Out-File with utf8 encoding. And most notably, ingress.yaml contained an empty kind List object, clearly pasted from a kubectl get ingress -o yaml command that had been run before any Ingress actually existed, instead of a real resource definition.

These were by far the hardest bugs to spot, because just catting the file looked right at a glance, and the error messages Ansible produced, things like FileNotFoundError or an empty results list, did not obviously point back to the content itself being wrong. They ended up disproportionately time consuming relative to how simple the actual fixes were once I found them, which taught me to always diff or hexdump a file when something inexplicable is happening rather than trusting a visual read.


**What I Would Add Next**
Looking back, the recurring theme across almost every challenge was that different layers of the stack, network reachability, SSH authorization, shell environment, and file resolution, all look similar on the surface but are governed by completely separate rules underneath. The next additions I would want to make are proper secrets management instead of manually created Kubernetes Secrets, monitoring and alerting so pod failures do not get masked by an old healthy pod still serving traffic, and extending the Ansible playbook to handle multi node scenarios instead of a single bare box.

Last but not least, I would like to thank Claude and Perplexity for helping me with this project as I could not have done it without them.