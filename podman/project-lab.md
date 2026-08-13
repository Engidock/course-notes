# 🦭 Podman Project Mastery

> **👋 Hey Fresher — Read This First!**

> Podman is a container engine — like Docker — that lets you build and run containers, but with one fundamental architectural difference: Podman has no background daemon running as root. Docker's classic architecture relies on `dockerd`, a privileged process running as root that every `docker` command talks to; if that daemon or the Docker socket is ever compromised, an attacker effectively has root on the host. Podman runs containers as regular, unprivileged child processes directly from the CLI you invoke — no root daemon required, no daemon at all, and full support for running entire multi-container applications as your own regular Linux user. This project builds a real multi-container backend the fully rootless way.

> **Company in this project:** OctaneForge Games — a mobile gaming studio in Chennai that runs the live backend for a battle-royale style game: a real-time leaderboard service backed by Redis, serving score updates during matches with brutal latency requirements. A recent security audit flagged that their build servers run Docker containers as root, and any engineer with Docker socket access effectively has root on every build machine. You just joined as a Junior Infrastructure Engineer. Your mentor is Sneha, and the platform security lead who ordered the audit is Rahul. Let's rebuild OctaneForge's leaderboard backend on rootless Podman.

#### What You Will Learn and Build in This Project

You will build OctaneForge's leaderboard backend — an app container plus a Redis container — as a rootless Podman pod, understand exactly what "rootless" buys you versus Docker's daemon model, wire the pod into systemd so it survives a server reboot, and export it to real Kubernetes YAML so the exact same pod definition can graduate from a developer's laptop to a production cluster with no rewrite.

Podman Architecture, Rootless Containers, Podman Pods, fork/exec vs Client-Server, Dockerfile Compatibility, systemd Integration, Podman Kube Play, Container Security

> **📦 Phase 1 — Why Rootless Matters**
>
> Understand the security gap between Docker's root daemon model and Podman's daemonless, rootless architecture — with a real incident to motivate it.

> **📦 Phase 2 — Install and Verify Rootless Podman**
>
> Get Podman running, confirm it's actually operating rootless, and understand what user namespaces are doing under the hood.

> **📦 Phase 3 — Build the Leaderboard Image**
>
> Write a Dockerfile (Podman uses the same format) and build it with `podman build` — no daemon required.

> **📦 Phase 4 — Group Containers into a Pod**
>
> Create a Podman pod holding the leaderboard app and Redis together, sharing one network namespace exactly like a Kubernetes pod does.

> **📦 Phase 5 — Survive a Reboot with systemd**
>
> Generate a systemd unit so the pod comes back automatically after the build server restarts — no manual `podman start` required.

> **📦 Phase 6 — Export to Kubernetes YAML**
>
> Generate real Kubernetes manifests directly from your running pod, proving Podman pods and Kubernetes pods are genuinely the same concept.

**Scene 1 — OctaneForge Security Audit, Chennai | "Everyone With Docker Access Has Root"**

> **Rahul** _Platform Security Lead — OctaneForge_
>
> Aarav, here's what the audit found. Our build servers run Docker. Every engineer who needs to build or test a container gets added to the `docker` group so they can run `docker` commands without `sudo`. What most people don't realize is that being in the `docker` group is functionally equivalent to having root on that machine — the Docker daemon runs as root, and anyone who can talk to its socket can mount the host's root filesystem into a container and read or write anything. We gave forty engineers root access without meaning to.

> **Sneha** _Senior Infrastructure Engineer — OctaneForge_
>
> And it's not theoretical — this is a well-documented Docker privilege escalation pattern. A container run with `docker run -v /:/host busybox chroot /host` gives you a root shell on the *host*, not just the container, if you're in the docker group. Podman closes this specific gap architecturally, not just with a policy saying "please don't do that." When you run `podman run` as your normal user, the container process runs as your normal user too — no daemon, no root, no group membership that secretly grants host root.

> **Aarav (You)** _Junior Infrastructure Engineer — Day 1_
>
> So Podman isn't just "Docker with a different CLI" — it's solving an actual architectural problem?

> **Sneha** _Senior Infrastructure Engineer_
>
> Right. And the good news for you: the command syntax is almost identical, and it uses the exact same Dockerfile format. You're not learning a new tool from scratch — you're learning a safer way to run the same containers you already know how to build.

### 1. Phase 1 — Why Rootless Matters: Docker's Daemon vs Podman's fork/exec Model

**Business Problem:** OctaneForge's build servers run untrusted, third-party CI job definitions from open-source dependencies and community game mod tooling. If any of that tooling can escape a container and reach the Docker daemon's root socket, it can compromise the entire build fleet — including signing keys for game client releases.

```
Docker Architecture                    Podman Architecture
======================                 =====================
   docker CLI                             podman CLI
       │                                       │
       │ talks to socket                       │ fork/exec directly
       ▼                                       ▼
  dockerd (root daemon)                   container process
       │  runs as root, always                 runs as the SAME
       │  even if you're not root)              user who ran podman
       ▼
  containerd → runc
       │
       ▼
  container process (root inside,
  and daemon-managed on the host)

Compromise the daemon or socket = root on host.    No daemon to compromise.
                                                    Rootless containers run
                                                    with your uid, mapped
                                                    into a user namespace.
```

> **📖 The Structural Difference, Not Just a Feature Difference**
>
> In Docker's model, `docker run` is really "ask the root daemon, on my behalf, to start a container" — the actual container process is a child of the daemon, and the daemon is what has root. In Podman's model, `podman run` directly forks and execs the container process as a child of your own shell, using **user namespaces** to remap your regular unprivileged UID to appear as root *only inside the container's own view of the filesystem* — while remaining your unprivileged UID on the actual host. There is no daemon process to compromise, and no root-owned socket that grants effective root to anyone who can write to it.

**Docker vs Podman — When Each Makes Sense**

- **Docker** — mature, huge ecosystem, `docker-compose` is extremely well established, and its client-server model is genuinely convenient for remote container management (talk to a daemon on another machine over TCP). Still the right default for teams already deeply invested in Docker Desktop workflows and Docker-specific tooling.
- **Podman** — the better default when rootless operation is a hard security requirement (shared build servers, multi-tenant CI, security-sensitive environments like OctaneForge's build fleet), when you want pods (multiple containers sharing a network namespace, exactly like Kubernetes) without installing a full orchestrator, or when you want your container-to-Kubernetes-YAML path to be a single native command rather than a third-party conversion tool.

### 2. Phase 2 — Install and Verify Rootless Podman

**Business Problem:** "Rootless" isn't automatic just because you installed Podman — you need to verify it's actually running rootless, and understand the subsystem (`/etc/subuid`, `/etc/subgid`) that makes user namespace remapping work at all.

#### 2.1 Installing Podman

```bash
# Fedora / RHEL / CentOS
sudo dnf install -y podman

# Debian / Ubuntu
sudo apt-get update && sudo apt-get install -y podman

# Verify installation and rootless status
podman info --format '{{.Host.Security.Rootless}}'
# Expected output: true
```

> **📖 Confirming Rootless, Not Assuming It**
>
> `podman info --format '{{.Host.Security.Rootless}}'` queries Podman's own view of how it's currently running and prints `true` or `false` directly — this is the authoritative check, more reliable than just assuming "I didn't use sudo, so it must be rootless." If this prints `false`, Podman was likely installed or invoked with `sudo podman`, which runs it rootful (as root) — defeating the entire point of this project.

#### 2.2 Verifying the User Namespace Mapping

```bash
# Check your subuid/subgid ranges — these let your user "own"
# a range of UIDs inside containers without actually having them on the host
cat /etc/subuid
# aarav:100000:65536

cat /etc/subgid
# aarav:100000:65536

# Run a container and check what UID it thinks it's running as
podman run --rm alpine id
# uid=0(root) gid=0(root) groups=0(root)   <- root INSIDE the container

# Now check what that same process actually looked like on the host
podman run --rm alpine sh -c 'sleep 30' &
ps -eo user,pid,cmd | grep sleep
# aarav    48213  sleep 30      <- your real, unprivileged user on the HOST
```

> **📖 The Trick Behind Rootless — UID Remapping**
>
> `/etc/subuid` grants user `aarav` a range of 65,536 UIDs (starting at 100000) that only exist inside containers he runs — the kernel's user namespace feature maps container UID 0 (root) to one of these host UIDs, which has zero actual privileges on the host system. Inside the container, `id` reports `uid=0(root)` — the software running inside genuinely believes it's root, so things that expect to run as root (like `apt-get install` during image builds) work normally. But on the host, `ps` shows the exact same process running as the unprivileged `aarav` user, PID owned by a regular account with no special permissions. This is the entire security model in one command pair.

**Quiz: An engineer at OctaneForge runs `podman run --rm -v /:/host alpine chroot /host` on a rootless Podman setup, hoping to get a root shell on the host — the same attack that works against a Docker daemon accessible via the `docker` group. What happens?**
- It works exactly the same as the Docker attack, giving full host root
- The command fails or the resulting "root" only has the unprivileged host UID's actual permissions — because the container's root is mapped to an unprivileged UID via user namespaces, not real host root
- Podman blocks all volume mounts by default for security
- The container silently does nothing

> **Answer/explanation:** The second option is correct, and it's the entire point of rootless Podman. Even though the container's `chroot /host` command believes it's running as UID 0 (root) and has mounted the host's root filesystem, that "root" is mapped through the user namespace to the invoking user's actual unprivileged host UID. Any file operations the container attempts against the mounted host filesystem are subject to the real host permissions of that unprivileged user — it cannot read root-owned files it shouldn't be able to, cannot write to files it doesn't own, and gains no elevated capability on the host at all. This is fundamentally different from the equivalent Docker attack against a root daemon, where the container process really is root as far as the host kernel is concerned. Podman does allow the volume mount (it's not blocked outright) — the security boundary is enforced by the UID mapping underneath, not by refusing the mount.

### 3. Phase 3 — Build the Leaderboard Image

**Business Problem:** OctaneForge's leaderboard service is a small Node.js app that reads and writes match scores to Redis. It needs to be containerized the same way it would be for Docker — Podman's whole design goal is Dockerfile compatibility, so nothing here should feel unfamiliar.

#### 3.1 The Leaderboard Dockerfile

```dockerfile
# leaderboard-service/Dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

COPY src/ ./src/

# Podman honors USER just like Docker — but remember, even
# "root" here is remapped to an unprivileged host UID rootless
USER node

EXPOSE 3000
CMD ["node", "src/server.js"]
```

#### 3.2 Building with Podman — No Daemon Required

```bash
cd leaderboard-service
podman build -t octaneforge/leaderboard:1.0.0 .

# List the image — stored in your own user's local storage,
# not a shared daemon-owned image store
podman images

# Inspect exactly which registries/user this build used
podman info --format '{{.Store.GraphRoot}}'
# /home/aarav/.local/share/containers/storage
```

> **📖 podman build — Same Command Shape, No Daemon Behind It**
>
> `podman build -t octaneforge/leaderboard:1.0.0 .` reads the exact same Dockerfile syntax `docker build` would — `FROM`, `WORKDIR`, `COPY`, `RUN`, `USER`, `EXPOSE`, `CMD` all mean exactly what they mean in Docker, because Podman implements the OCI (Open Container Initiative) image spec, the same standard Docker images conform to. The important difference is *where the build happens*: there's no long-running daemon process building the image on your behalf — `podman build` runs the build steps directly, and the resulting image is stored under `~/.local/share/containers/storage`, owned entirely by your own user account, not a shared root-owned Docker image store that every user on the machine implicitly has some access to.

### 4. Phase 4 — Group Containers into a Pod

**Business Problem:** The leaderboard service needs Redis running alongside it, reachable over a fast local connection with minimal latency — this is a real-time gaming backend where every millisecond of network hop matters during a live match. Podman's **pod** concept groups multiple containers to share one network namespace, exactly like a Kubernetes pod.

**Scene 2 — Design Discussion | "Why Not Just Two Separate Containers?"**

> **Aarav (You)** _Junior Infrastructure Engineer_
>
> Couldn't I just run the leaderboard container and a separate Redis container, and link them with `--network`?

> **Sneha** _Senior Infrastructure Engineer_
>
> You could, but a Podman pod gives you something better for this use case: containers in the same pod share one network namespace, meaning they talk to each other over `localhost`, at the lowest possible latency, exactly the way containers inside one Kubernetes pod do. And when you eventually deploy this for real on Kubernetes, the mental model transfers directly — you already understand what "sharing a pod" means, because you built it this way from day one.

#### 4.1 Creating the Pod

```bash
# Create a pod, exposing only the port the leaderboard app needs
podman pod create --name octaneforge-leaderboard-pod \
  -p 3000:3000

# Add Redis to the pod — no port mapping needed, it's only
# reachable from inside the pod, over localhost
podman run -d --pod octaneforge-leaderboard-pod \
  --name leaderboard-redis \
  -v leaderboard-redis-data:/data \
  redis:7-alpine redis-server --appendonly yes

# Add the leaderboard app to the same pod
podman run -d --pod octaneforge-leaderboard-pod \
  --name leaderboard-app \
  -e REDIS_HOST=localhost \
  -e REDIS_PORT=6379 \
  octaneforge/leaderboard:1.0.0
```

> **📖 Pods Share a Network Namespace — That's the Whole Point**
>
> `podman pod create -p 3000:3000` declares the pod-level port mapping — only the pod as a whole gets a mapped port, individual containers inside it don't need their own `-p` flags. `-e REDIS_HOST=localhost` on the leaderboard app is the key detail: because both containers share the pod's network namespace, the leaderboard app reaches Redis over plain `localhost:6379` — the same as if they were two processes on one machine — instead of needing a separate container-to-container DNS name or a custom network. Redis itself has no `-p` mapping at all, because it's intentionally never reachable from outside the pod — only the leaderboard app inside the same pod can reach it, which is exactly the right exposure boundary for a datastore that should never be internet-facing.

#### 4.2 Verifying Pod Connectivity

```bash
podman pod ps
# POD ID        NAME                          STATUS     CONTAINERS
# a1b2c3d4e5f6  octaneforge-leaderboard-pod    Running    2

podman exec leaderboard-app sh -c "nc -zv localhost 6379"
# localhost (127.0.0.1:6379) open

podman logs leaderboard-app
# Leaderboard service listening on :3000
# Connected to Redis at localhost:6379
```

> **Key takeaways**
> - Podman pods are not a Podman-only abstraction invented for convenience — they use the same shared-network-namespace concept as Kubernetes pods, so the mental model transfers directly.
> - Containers sharing a pod talk to each other over `localhost` — no custom bridge network or service discovery mechanism needed for containers that belong together.
> - Only map ports at the pod level for services that genuinely need external access (the app); keep internal dependencies like Redis unmapped so they're unreachable from outside the pod entirely.

### 5. Phase 5 — Survive a Reboot with systemd

**Business Problem:** OctaneForge's build servers get rebooted for kernel patches roughly monthly. Without something managing the pod's lifecycle, an engineer has to remember to manually restart the leaderboard pod every time — which inevitably gets forgotten, causing a preventable outage during a patch window.

#### 5.1 Generating the systemd Unit

```bash
# Generate systemd unit files for the running pod
podman generate systemd --new --files --name octaneforge-leaderboard-pod

# This produces three unit files in the current directory:
#   pod-octaneforge-leaderboard-pod.service
#   container-leaderboard-redis.service
#   container-leaderboard-app.service

# Move them into the user's systemd directory (rootless — no sudo needed)
mkdir -p ~/.config/systemd/user
mv pod-octaneforge-leaderboard-pod.service container-*.service \
   ~/.config/systemd/user/

# Reload systemd and enable the pod to start on boot
systemctl --user daemon-reload
systemctl --user enable --now pod-octaneforge-leaderboard-pod.service

# Enable "lingering" so the user's systemd services keep running
# even when aarav isn't logged in via SSH
sudo loginctl enable-linger aarav
```

> **📖 Rootless systemd — `--user` Units, Not System-Wide**
>
> `--new --files` tells `podman generate systemd` to produce unit files that recreate the pod from scratch on each start (rather than assuming the pod container objects already exist), which behaves correctly across a full reboot when the previous containers are gone. Because this is running rootless, the units go into `~/.config/systemd/user/` and are managed with `systemctl --user`, entirely as your own unprivileged user — no `sudo systemctl` required for day-to-day start/stop/restart. The one command that genuinely needs `sudo` is `loginctl enable-linger`, because without it, systemd would normally stop all of a user's `--user` services the moment their SSH session ends — lingering tells systemd to keep them running as a genuine background service independent of any login session, which is what you want for a production leaderboard backend.

#### 5.2 Podman Quadlet — The Newer, Simpler Alternative

```ini
# ~/.config/containers/systemd/leaderboard.pod
[Pod]
PodName=octaneforge-leaderboard-pod
PublishPort=3000:3000

[Install]
WantedBy=default.target
```

> **📖 Quadlet — Declarative Instead of Generated**
>
> Newer Podman versions (4.4+) support **Quadlet**: instead of running a live pod and then generating systemd units from it, you write a small declarative `.pod`/`.container` file directly, and `podman-system-generator` turns it into the equivalent systemd unit automatically at boot. This is closer to how Kubernetes YAML feels — you declare desired state in a file, checked into Git, rather than generating units from an already-running imperative setup. For new OctaneForge projects, Sneha's team now writes Quadlet files from the start rather than running `podman generate systemd` against a manually created pod.

### 6. Phase 6 — Export to Kubernetes YAML

**Business Problem:** OctaneForge's leaderboard service has outgrown a single build server — it needs to run on the company's Kubernetes cluster for real production traffic during live matches. Rewriting the pod definition into Kubernetes YAML by hand would be slow and error-prone; Podman can generate it directly from the running pod.

#### 6.1 Generating Kubernetes YAML from a Running Pod

```bash
podman kube generate octaneforge-leaderboard-pod -f leaderboard-k8s.yaml
```

```yaml
# leaderboard-k8s.yaml (generated, trimmed for readability)
apiVersion: v1
kind: Pod
metadata:
  name: octaneforge-leaderboard-pod
  labels:
    app: octaneforge-leaderboard-pod
spec:
  containers:
    - name: leaderboard-redis
      image: docker.io/library/redis:7-alpine
      args: ["redis-server", "--appendonly", "yes"]
      volumeMounts:
        - name: leaderboard-redis-data
          mountPath: /data
    - name: leaderboard-app
      image: localhost/octaneforge/leaderboard:1.0.0
      env:
        - name: REDIS_HOST
          value: localhost
        - name: REDIS_PORT
          value: "6379"
      ports:
        - containerPort: 3000
          hostPort: 3000
  volumes:
    - name: leaderboard-redis-data
      persistentVolumeClaim:
        claimName: leaderboard-redis-data
```

> **📖 Real Kubernetes YAML, Not an Approximation**
>
> This is a standard, valid Kubernetes Pod manifest — `apiVersion: v1`, `kind: Pod`, the same schema Kubernetes itself uses. Notice both containers are listed inside **one** `spec.containers` list, sharing one Pod — exactly mirroring the pod you built manually in Phase 4, because a Podman pod and a Kubernetes pod are the same underlying concept: one shared network namespace, multiple containers. The `localhost/octaneforge/leaderboard:1.0.0` image reference will need to be pushed to a real registry (not left as a local-only image) before this YAML can actually be applied to a remote cluster — Podman generates a technically correct manifest, but it can't push your local-only image for you.

#### 6.2 Testing the Round Trip — podman kube play

```bash
# Tear down the manually created pod
podman pod rm -f octaneforge-leaderboard-pod

# Recreate it entirely from the generated YAML
podman kube play leaderboard-k8s.yaml

podman pod ps
# POD ID        NAME                          STATUS     CONTAINERS
# f7g8h9i0j1k2  octaneforge-leaderboard-pod    Running    2
```

> **📖 `podman kube play` — Kubernetes YAML as a Local Dev Tool**
>
> `podman kube play` reads standard Kubernetes YAML and recreates the pod locally — proving the round trip actually works: pod → generated YAML → pod again, byte-for-byte the same shape. This makes Kubernetes YAML a legitimate way to define local development environments even before a real cluster is involved — engineers can write or receive a Kubernetes manifest and run it locally with Podman for fast iteration, with high confidence that what runs locally will behave the same way once genuinely deployed to the OctaneForge production cluster.

**Quiz: The generated Kubernetes YAML references `localhost/octaneforge/leaderboard:1.0.0` as the image. Before this manifest can be applied to OctaneForge's real production Kubernetes cluster, what has to happen?**
- Nothing — Kubernetes can pull from any local Podman image store automatically
- The image needs to be pushed to a registry the cluster can actually reach, and the `image:` field updated to that registry's address
- The pod needs to be renamed
- Kubernetes will automatically convert `localhost/` references to the correct registry

> **Answer/explanation:** The second option is correct. `localhost/octaneforge/leaderboard:1.0.0` is a local-only image reference meaningful only on the machine where `podman build` ran — it exists in that machine's local container storage, not in any registry a remote Kubernetes cluster's nodes could reach. Before this manifest is usable on a real cluster, the image must be pushed with `podman push octaneforge/leaderboard:1.0.0 registry.octaneforge.in/leaderboard:1.0.0` (or similar) to a registry every cluster node can pull from, and the YAML's `image:` field updated to that registry address. Kubernetes has no special handling for `localhost/` image references from a completely different machine — it will simply fail to pull the image, since as far as any cluster node is concerned, no such image exists anywhere reachable.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a third container to the pod:** Add a small Prometheus `redis_exporter` sidecar container to the leaderboard pod so match-time Redis metrics (memory usage, connected clients, ops/sec) are scraped alongside the app.
2. **Enforce rootless in CI:** Write a CI check that runs `podman info --format '{{.Host.Security.Rootless}}'` and fails the pipeline if it doesn't print `true`, preventing anyone from accidentally reintroducing a rootful Podman setup on a build server.
3. **Convert to Quadlet:** Rewrite the systemd units from Phase 5 as Quadlet `.pod` and `.container` files, check them into Git, and confirm `systemctl --user daemon-reload` picks them up correctly without ever running `podman generate systemd` again.
4. **Push the image and apply for real:** Push `octaneforge/leaderboard:1.0.0` to a real container registry, update the generated YAML's image reference, and apply it to a real (or local kind/minikube) Kubernetes cluster with `kubectl apply -f leaderboard-k8s.yaml` — confirm it runs identically to the Podman pod.
5. **Compare resource isolation:** Run the same leaderboard pod under Podman with `podman pod create --cpus 1 --memory 512m`, then intentionally load-test past that limit, and observe how Podman enforces the same cgroup-based resource limits Docker and Kubernetes rely on — proving rootless operation doesn't mean giving up resource isolation.

### Podman Project Complete 🎉

You rebuilt OctaneForge's leaderboard backend as a fully rootless Podman pod — the leaderboard app and Redis sharing one network namespace exactly like a Kubernetes pod, with zero root daemon anywhere in the picture. You wired the pod into systemd so it survives a server reboot without manual intervention, and you generated real, valid Kubernetes YAML directly from the running pod — proving the exact same pod definition can move from a build server to a production cluster with no rewrite required.

> **Rahul**
>
> "The audit finding that started this project was that forty engineers effectively had root on our build fleet through Docker group membership. After this migration, that sentence is no longer true — rootless Podman means a compromised container process on a build server has exactly the privileges of the unprivileged CI user that started it. Nothing more, no matter what the container thinks it's running as internally."

> **Sneha**
>
> "What I wanted you to walk away with isn't just 'Podman is more secure' — it's that you now understand *why*, at the user-namespace level, instead of taking it on faith. That's the difference between someone who can configure a tool and someone who can explain a security decision to an auditor."

> **Aarav (You)**
>
> "The moment it really clicked was running `id` inside the container and seeing root, then checking `ps` on the host and seeing my own unprivileged username on the exact same process. Same process, two completely different privilege views — that's the whole rootless model in one command."

> **Next: Kubernetes Fundamentals — From One Pod to a Real Cluster**

> - Pods, Deployments, and Services — the same pod concept you built manually, now managed automatically by a control plane across many nodes
> - kubectl and the Kubernetes API — the tools you'll use daily once the leaderboard pod is running on a real cluster
> - ConfigMaps and Secrets — replacing the plain `-e` environment variables used here with properly managed configuration and credentials
> - Horizontal Pod Autoscaling — scaling the leaderboard app automatically during peak match hours, the way OctaneForge's traffic actually spikes
> - CRI-O and container runtimes in production Kubernetes — understanding what actually runs your containers once Podman's job (build and local dev) is done
