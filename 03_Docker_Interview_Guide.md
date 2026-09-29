# Docker — Complete Interview Guide

> Document 3 of your DevOps interview series: **Linux → Networking → Docker → Kubernetes → CI/CD → AWS → Terraform.**
> Covers what a 3–5 YOE DevOps engineer needs to actually operate and troubleshoot Docker — not a command encyclopedia.

**Callout legend:** 🔵 `NOTE` = Anchor Fact · 🟡 `WARNING` = Failure Scenario · 🟢 `TIP` = Interview Answer · 🔧 `IMPORTANT` (HANDS-ON) = Practical Lab · 🟣 `IMPORTANT` (LIVE-CODING/LIVE-DIAGNOSIS) = Interview Task.

**Core vs Stretch:** `CORE` = must be instantly fluent for a 3–5 YOE interview. `STRETCH` = good depth that differentiates you.

**The one framing to hold onto through this whole document:** current research on real Docker interviews is consistent on this point — almost every hard Docker question reduces to one of two ideas: **layers** (the build, the cache, image size) or **isolation** (namespaces, cgroups, the shared kernel). When a question feels unfamiliar, ask which of the two it's really about, and the answer usually falls out.

---

## Quick Jump Index

**Part 1 — Foundations**
1. [What a Container Actually Is, Physically](#1-what-a-container-actually-is-physically) `CORE`
2. [Image vs Container](#2-image-vs-container) `CORE`
3. [The Dockerfile & Image Layers](#3-the-dockerfile--image-layers) `CORE`
4. [The Build Cache](#4-the-build-cache) `CORE`
5. [RUN vs CMD vs ENTRYPOINT](#5-run-vs-cmd-vs-entrypoint) `CORE`
6. [What Actually Happens on docker run](#6-what-actually-happens-on-docker-run) `CORE`
7. [Docker Networking](#7-docker-networking) `CORE`
8. [Data Persistence: Volumes, Bind Mounts, tmpfs](#8-data-persistence-volumes-bind-mounts-tmpfs) `CORE`
9. [Multi-Stage Builds](#9-multi-stage-builds) `CORE`
10. [Container Security Basics: Non-Root & Minimal Images](#10-container-security-basics-non-root--minimal-images) `CORE`

**Part 2 — Deeper Mechanics**
- [PID 1 and Signal Handling](#pid-1-and-signal-handling) `CORE`
- [The Four Levers for Image Size](#the-four-levers-for-image-size) `CORE`
- [Secrets: What Never Goes in an Image](#secrets-what-never-goes-in-an-image) `CORE`
- [Docker Compose](#docker-compose) `CORE`
- [EXPOSE vs Publishing a Port](#expose-vs-publishing-a-port) `CORE`
- [Distroless & SBOM/Image Signing](#distroless--sbomimage-signing) `STRETCH`

**Part 3 — Interview Corner**
- [Images & Layers — full 9-angle drill](#images--layers-drill)
- [Networking & Compose — full 9-angle drill](#networking--compose-drill)
- [Security & Isolation — full 9-angle drill](#security--isolation-drill)
- [Troubleshooting — full 9-angle drill](#troubleshooting-drill)
- [Quick-Coverage Round](#quick-coverage-round)
- [Hot Topics Right Now](#hot-topics-right-now)
- [Research-Backed Additions (Current Market, 2025–2026)](#research-backed-additions-current-market-20252026)

**Part 4 — Real Incident / Story Bank**
- [Story 1: The ECR Tag Immutability Break](#story-1-the-ecr-tag-immutability-break)
- [Story 2: The Three-Container Network with Trivy in the Pipeline](#story-2-the-three-container-network-with-trivy-in-the-pipeline)
- [Practice Scenario (rehearsal only)](#practice-scenario-rehearsal-only)

**Part 5 — Master Glossary (Active Recall)**
- [A–Z Quick Recall](#master-glossary-active-recall)

**Part 6 — Additional Practical Exercises**
- [1: Shrink a Real Image With a Multi-Stage Build](#exercise-1-shrink-a-real-image-with-a-multi-stage-build)
- [2: Debug Two Containers That Can't Talk](#exercise-2-debug-two-containers-that-cant-talk)

---

# Part 1 — Foundations

## 1. What a Container Actually Is, Physically
`CORE`

**Why it exists:** Before containers, "works on my machine" was the default failure mode — an app's behavior depended on whatever libraries, versions, and config happened to already be installed on that one specific host. A container packages an application with everything it needs so it behaves identically anywhere it runs, without the overhead of virtualizing an entire machine to get that isolation.

**Physical flow — what the kernel actually does on `docker run`:**
1. The Docker daemon checks if the image exists locally; pulls from a registry if not.
2. It asks the Linux kernel to create a new set of **namespaces** for this one process — restricted views of the system: its own process tree (PID namespace — the container's first process becomes PID 1 *inside* that namespace, even though it has a different, ordinary PID on the host), its own network stack (NET namespace), its own filesystem view (MNT namespace), its own hostname (UTS namespace).
3. The kernel applies **cgroups** to that process — hard limits on how much CPU and memory it's allowed to consume.
4. The image's layers are mounted read-only, with one thin writable layer added on top for anything the container changes at runtime.
5. The image's `ENTRYPOINT`/`CMD` runs as that first process inside the new namespaces.

**Components involved:** Namespaces (isolation — what the process can see), cgroups (limits — what it can consume), the container runtime (containerd, under Docker), the shared host kernel.

**Decision point — the honest trade-off:** A container is NOT a lightweight VM. There's no hypervisor and no separate kernel — it's a regular Linux process with the kernel's view of the world scoped down. This is exactly why containers start in milliseconds and VMs take seconds: no kernel to boot. But it's also why container isolation is weaker than a VM's — every container on a host shares that host's actual kernel, so a kernel vulnerability or a misconfigured privileged container can cross the isolation boundary in ways that are far harder against a VM's hypervisor.

**Plain example:** A VM is like giving each tenant their own separate house, complete with their own plumbing and electrical system. A container is like giving each tenant their own locked apartment in the same building — much faster to set up, much cheaper to run many of, but they're all still connected to the same building's foundation and utilities underneath.

> [!NOTE]
> **Anchor Fact:** Namespaces provide isolation (what a process can see — PIDs, network, filesystem). Cgroups provide limits (what it can consume — CPU, memory). Together, on a shared kernel, that's the entire container model — no hypervisor, no separate kernel anywhere.

> [!TIP]
> **Interview Answer:** "A container isn't a lightweight VM — it's a regular Linux process that the kernel has scoped down using namespaces for isolation and cgroups for resource limits, still sharing the host's actual kernel. That's exactly why containers start in milliseconds instead of the seconds a VM takes to boot its own kernel, and it's also the honest trade-off — weaker isolation than a VM, since every container shares that one kernel."

---

## 2. Image vs Container
`CORE`

**Why it exists:** You need a repeatable, shareable template (the image) that's separate from any specific running instance of it (the container) — otherwise every deployment would be a fresh, unrepeatable, hand-built environment.

**Physical flow:**
1. An **image** is a read-only, layered filesystem snapshot plus metadata (what command to run, what ports it expects, environment defaults) — it never runs by itself, it's a template.
2. A **container** is a running (or stopped) instance created from that image, with a thin writable layer added on top for anything it changes.
3. Many containers can be created from the same image simultaneously, each with its own independent writable layer — none of them affect the underlying image or each other's writable layers.
4. When a container is removed, its writable layer is destroyed with it. The image is untouched and can spawn a fresh container identical to how the first one started.

**Decision point:** This is the single idea that explains why data written inside a container disappears when you remove it (Section 8) — nothing was misconfigured, that's just how the model works. Anything meant to survive has to live outside the container's own writable layer.

**Plain example:** An image is like a class in object-oriented programming, or a stamped cookie cutter — a container is an object instantiated from that class, or one cookie cut from that cutter. Cutting many cookies from the same cutter doesn't change the cutter, and eating (removing) one cookie doesn't affect the others or the cutter itself.

> [!NOTE]
> **Anchor Fact:** Image = read-only template, never runs by itself. Container = a running/stopped instance with its own writable layer. One image, many independent containers.

> [!TIP]
> **Interview Answer:** "The relationship is like a class and an object — the image is the immutable template, and a container is one running instance of it, with its own thin writable layer for whatever it changes at runtime. That writable layer is ephemeral by design — it's destroyed when the container is removed, which is exactly why anything meant to persist has to live outside it, in a volume or bind mount."

---

## 3. The Dockerfile & Image Layers
`CORE`

**Why it exists:** You need a text-based, version-controllable recipe that anyone can use to rebuild the exact same image — not a manual, undocumented sequence of commands someone ran once and forgot.

**Physical flow:**
1. A Dockerfile is a sequence of instructions, read top to bottom: `FROM` (the base image), `RUN` (execute a command at build time), `COPY` (bring files into the image), `CMD`/`ENTRYPOINT` (what runs when a container starts).
2. Each instruction that changes the filesystem creates a new, separate **layer** — Docker builds the image as a stack of these layers.
3. Layers are content-addressed and cached — `docker history <image>` shows every layer and its size, letting you see exactly which instruction is responsible for how much of the image's weight.

**Components involved:** Base image, instructions, layers, the image manifest tying them together.

**Exact identifiers:**
```bash
docker history myapp:latest
# CREATED BY                    SIZE
# CMD ["python" "app.py"]       0B
# COPY app.py .                 12.3kB
# RUN pip install -r reqs.txt   13.1MB
# COPY requirements.txt .       12.3kB
```

**Decision point:** `COPY` vs `ADD` is a real, recurring point of confusion — `COPY` does exactly one thing (copies local files in). `ADD` does that plus two extra, "magic" behaviors: it can fetch a remote URL, and it auto-extracts a local tar archive. The safe default is `COPY`; reach for `ADD` only when you specifically want the tar auto-extract behavior, since the magic has genuinely surprised people in real builds.

**Plain example:** A Dockerfile is a recipe card, and each instruction is one step that leaves a physical trace (a layer) — like a photo taken after each cooking step, stacked into a flipbook that recreates the exact same dish every time someone follows the same steps.

> [!TIP]
> **Interview Answer:** "A Dockerfile is a top-to-bottom recipe, and most instructions each create their own layer, which is why docker history shows the image as a stack of sized layers instead of one blob. I default to COPY over ADD — ADD's extra behaviors, fetching URLs and auto-extracting tarballs, are easy to trigger by accident, so I only reach for it when I specifically want that auto-extract."

---

## 4. The Build Cache
`CORE`

**Why it exists:** Rebuilding every layer from scratch on every single build would make iteration painfully slow — most changes between builds are small (one line of application code), so Docker caches each layer and reuses it whenever nothing relevant has changed.

**Physical flow:**
1. Docker evaluates each instruction top to bottom against its cache.
2. If an instruction and its inputs are identical to a previous build, Docker reuses that cached layer instantly instead of re-executing it.
3. The moment ONE instruction's inputs change, that layer AND every layer after it must be rebuilt — the cache is valid only up to the first point of change, never selectively past it.
4. This means instruction ORDER directly controls how much a typical code change actually costs to rebuild.

**Decision point — the practical rule this produces:** put instructions that change rarely (installing dependencies) BEFORE instructions that change on every commit (copying application source code). Copying dependency manifests and installing them first, then copying source code last, means a routine one-line code change only invalidates the final layer or two — not a full dependency reinstall.

**Plain example:** It's like packing a moving truck — load the heavy furniture that's rarely rearranged first, at the back, and the small daily-use boxes last, near the door. If you did it the other way around, every time you needed one small box, you'd have to unload the entire truck to get to it.

> [!WARNING]
> **Failure Scenario:** A Dockerfile copies the entire source tree first, then installs dependencies. **Symptom:** builds are consistently slow, even for a one-line code change. **Investigation:** watch the build output — the dependency-install `RUN` step shows as re-executing on every build instead of `CACHED`. **Root cause:** because the source `COPY` happens before the dependency install, ANY source change invalidates the cache from that point forward, forcing a full dependency reinstall every time, even though the dependencies themselves never changed. **Fix:** reorder — copy only the dependency manifest first, install dependencies, then copy the rest of the source. **Verification:** rebuild after a source-only change and confirm the dependency-install step now shows `CACHED`.

> [!TIP]
> **Interview Answer:** "The cache invalidates top-down — once one layer's inputs change, every layer after it rebuilds too, even if those later instructions are identical to last time. So I order a Dockerfile from least to most frequently changing: copy the dependency manifest and install dependencies before copying application source code, since code changes on every commit but dependencies rarely do. Getting that order backwards means a one-line code change silently triggers a full dependency reinstall on every build."

---

## 5. RUN vs CMD vs ENTRYPOINT
`CORE`

**Why it exists:** A Dockerfile needs to distinguish between things that happen once, permanently, when the IMAGE is built (installing a package) versus what happens every time a CONTAINER starts (launching the actual application) — and further, what's a fixed executable versus overridable default arguments to it.

**Physical flow:**
1. `RUN` executes at **build time** — its result (files created, packages installed) is baked into a layer, permanently part of the image.
2. `CMD` and `ENTRYPOINT` both define what happens at **container start**, not build time — neither runs until `docker run` actually launches a container from the image.
3. `ENTRYPOINT` sets the fixed executable that always runs. `CMD` provides default arguments — easily overridden by anything passed after the image name on `docker run`.
4. A common, deliberate pattern: `ENTRYPOINT ["python"]` with `CMD ["app.py"]` — `docker run myimage other.py` swaps the script but keeps the interpreter fixed.

**Decision point — exec form vs shell form, and why it matters later:** `CMD ["python", "app.py"]` (exec form) runs the binary directly as the container's PID 1. `CMD python app.py` (shell form) silently wraps it as `/bin/sh -c "python app.py"` — meaning a shell process, not your application, becomes PID 1. This distinction seems cosmetic until Section on PID 1/signals (Part 2), where it directly explains why `docker stop` sometimes hangs for the full grace period.

**Plain example:** `ENTRYPOINT` is the fixed name of the restaurant on the sign — it's not changing. `CMD` is today's default menu recommendation — easy to override by just ordering something else off the same menu when you walk in.

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run myimage           -> runs: python app.py
# docker run myimage other.py  -> runs: python other.py  (CMD overridden, ENTRYPOINT fixed)
```

> [!NOTE]
> **Anchor Fact:** `RUN` = build time, baked into a layer. `CMD`/`ENTRYPOINT` = container start time, not build time. Exec form (`["cmd", "arg"]`) makes your app PID 1 directly; shell form (`cmd arg`, no brackets) silently wraps it in `/bin/sh -c`, putting a shell in front of it instead.

> [!TIP]
> **Interview Answer:** "RUN executes once at build time and is baked into a layer — CMD and ENTRYPOINT define what happens when a container actually starts, not during the build. ENTRYPOINT is the fixed executable, CMD is the overridable default argument to it — a common pattern is ENTRYPOINT python with CMD pointing at the default script, so you can swap the script at run time while the interpreter stays fixed. I always use the exec form, the bracketed array syntax, so my application becomes PID 1 directly instead of a shell silently sitting in front of it."

---

## 6. What Actually Happens on `docker run`
`CORE`

**Why it exists:** `docker run` looks like one command, but understanding it as three distinct steps is what lets you diagnose exactly WHERE a failure happened, instead of treating "it didn't work" as one undifferentiated mystery.

**Physical flow — pull, create, start:**
1. **Pull:** Docker checks if the requested image exists locally; if not, it pulls it from the configured registry — this step can fail on authentication, a missing tag, or network issues.
2. **Create:** Docker creates the container — allocating its writable layer, its namespaces, and its configuration (env vars, mounts, network) — but doesn't start the process yet. This step can fail on a bad configuration (invalid volume mount, port conflict).
3. **Start:** Docker actually launches the `ENTRYPOINT`/`CMD` as the container's first process inside its namespaces. This step can fail on a bad command, a missing binary, or the application itself crashing immediately.

**Decision point:** Knowing these three steps means a failure message tells you a lot before you even read the details — a pull failure is a registry/network/auth problem, a create failure is a configuration problem, and a start failure is an application/command problem. Conflating all three into "docker run failed" wastes diagnostic time.

> [!TIP]
> **Interview Answer:** "docker run is really three steps — pull the image if it's not already local, create the container with its config and namespaces, then start the actual process. Knowing which of the three failed narrows the problem immediately: pull failures are registry or auth issues, create failures are configuration issues, start failures are about the command or the application itself."

---

## 7. Docker Networking
`CORE`

**Why it exists:** Containers on the same host need to reach each other predictably, without relying on IP addresses that change every time a container restarts — and Docker needs sensible default network isolation so containers aren't all flatly exposed to each other and the host by default.

**Physical flow:**
1. Docker installs three networks out of the box: **bridge** (the default — a private internal network where each container gets its own IP), **host** (the container shares the host's network namespace directly, no isolation), and **none** (no networking at all).
2. Unless told otherwise, a new container joins the **default bridge** network.
3. The default bridge has NO built-in DNS — containers on it can only reach each other by IP, not by container name.
4. A **user-defined bridge network** (`docker network create mynet`) is different — Docker runs embedded DNS for it, so containers on that network resolve each other automatically by container name or Compose service name.

**Exact identifiers:**
```bash
# Default bridge: name resolution fails
$ docker run --rm busybox nslookup web2
** server can't find web2: NXDOMAIN

# User-defined network: name resolution works
$ docker network create mynet
$ docker run --rm --network mynet busybox nslookup web
Name:   web
Address: 172.19.0.2
```

**Decision point:** This exact NXDOMAIN failure is one of the most common real "it works sometimes, not others" Docker networking issues — two containers started without an explicit `--network` land on the default bridge, and their names simply don't resolve, even though both are running fine.

> [!NOTE]
> **Anchor Fact:** Default bridge = no built-in DNS, containers can't resolve each other by name. User-defined bridge network = Docker's embedded DNS works, name resolution works automatically. This is exactly why `docker compose` "just works" for service-name resolution — Compose creates a user-defined network for you by default.

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Reproduce and fix the NXDOMAIN failure**
> **Concept:** Directly reproduce a commonly reported Docker networking confusion, so the fix becomes muscle memory instead of a fact you memorized.
> **Implementation:**
> ```bash
> docker run -d --name web --rm nginx:alpine
> docker run --rm busybox nslookup web        # fails: NXDOMAIN (default bridge)
>
> docker network create mynet
> docker run -d --name web2 --network mynet --rm nginx:alpine
> docker run --rm --network mynet busybox nslookup web2   # succeeds
> ```
> **Failure:** The first lookup fails even though the `web` container is clearly running (`docker ps` confirms it).
> **Investigation:** Check which network each container landed on — `docker inspect web --format '{{.NetworkSettings.Networks}}'`.
> **Fix:** Put both containers on the same user-defined network.
> **Verification:** The second lookup succeeds and returns a real IP.
> **Interview Explanation:** "Two containers not resolving each other by name is almost always a default-bridge problem, not a DNS misconfiguration — the default bridge simply has no embedded DNS at all. Moving both containers onto a user-defined network, or letting Compose do it automatically, fixes it immediately, because Docker's embedded DNS only runs on user-defined networks."

---

## 8. Data Persistence: Volumes, Bind Mounts, tmpfs
`CORE`

**Why it exists:** A container's writable layer is destroyed with the container (Section 2) — anything meant to survive a container restart, redeploy, or crash needs to live somewhere outside that ephemeral layer.

**Physical flow:**
1. A **volume** is storage Docker itself manages, outside any specific container's filesystem — you don't need to know or care exactly where it physically lives on the host.
2. A **bind mount** links a specific, explicit host path directly into the container — useful for local development (edit code on the host, see it live in the container with no rebuild), but ties the container to that host's exact directory layout.
3. **tmpfs** is in-memory only — never touches disk at all, used for sensitive data you specifically don't want persisted anywhere, even temporarily.

**Exact identifiers:**
```bash
docker run --rm -v mydata:/data busybox sh -c 'echo hello > /data/file.txt'
# container removed — data survives because it's in a volume, not the writable layer:
docker run --rm -v mydata:/data busybox cat /data/file.txt
# hello
```

**Decision point:** Bind mounts give the container direct access to a host filesystem path — genuinely useful for development, but a real security concern in production if that path or its permissions aren't tightly controlled, since the container can now read/write directly into the host's filesystem at that location.

> [!TIP]
> **Interview Answer:** "By default, container storage is ephemeral — a container's writes go to its own writable layer, which is deleted with the container. Volumes are Docker-managed storage that outlives any specific container, the right default for anything that needs to persist. Bind mounts link a specific host path in directly, which is great for live-editing code in development, but I'd be cautious using them in production since it gives the container direct access to that host path."

---

## 9. Multi-Stage Builds
`CORE`

**Why it exists:** Building an application often needs a full toolchain — compilers, dev dependencies, source code — that's only needed DURING the build. Shipping all of that inside the production image makes it needlessly large and expands the attack surface with tools an attacker could use if they ever got in.

**Physical flow:**
1. A Dockerfile can contain more than one `FROM` — each one starts a new, independent **stage**.
2. An early stage (`FROM golang:1.23 AS builder`) has the full toolchain and compiles the application.
3. The FINAL stage starts fresh from a minimal base image, and uses `COPY --from=builder` to bring in ONLY the specific compiled artifact — nothing else from the builder stage carries over automatically.
4. Everything else from the builder stage — the compiler, source code, build-time dependencies — is simply left behind, never part of the final image at all.

**The real, verified numbers (same Go application, two Dockerfiles):**
```
REPOSITORY   TAG      SIZE
demo         single   1.27GB
demo         multi    15.5MB
```
A single-stage build ships the entire Go toolchain — over a gigabyte of compilers never run in production. The multi-stage build ships Alpine plus one static binary.

> [!NOTE]
> **Anchor Fact:** A multi-stage build's final image contains ONLY what's explicitly copied via `COPY --from=<stage>` — nothing else from earlier stages carries over by default, even files that happen to exist at the same path.

> [!IMPORTANT]
> 🔧 **HANDS-ON LAB — Build the same app single-stage and multi-stage, compare**
> **Concept:** The size difference isn't theoretical — see it directly, because a number you've personally reproduced sticks far better than one you've read about.
> **Implementation:**
> ```dockerfile
> # Dockerfile.single
> FROM golang:1.23
> WORKDIR /app
> COPY . .
> RUN go build -o server .
> CMD ["./server"]
> ```
> ```dockerfile
> # Dockerfile.multi
> FROM golang:1.23 AS builder
> WORKDIR /app
> COPY . .
> RUN CGO_ENABLED=0 go build -o server .
>
> FROM alpine:3.20
> COPY --from=builder /app/server /server
> CMD ["/server"]
> ```
> **Hands-on:** `docker build -f Dockerfile.single -t demo:single .` and `docker build -f Dockerfile.multi -t demo:multi .`, then `docker images` and compare.
> **Failure:** The multi-stage build fails with "no such file or directory" on the `COPY --from=builder` line.
> **Investigation:** Confirm the exact path the binary was built to in the builder stage matches the path referenced in `COPY --from=builder` exactly — a very common typo-class mistake.
> **Fix:** Correct the path, or add `RUN ls /app` temporarily in the builder stage to confirm the actual output location.
> **Verification:** `docker images` shows both, and the size gap is dramatic and real, not a rounding difference.
> **Interview Explanation:** "The gap between single-stage and multi-stage builds isn't subtle — for a compiled language especially, moving the toolchain into an earlier stage that never ships routinely takes an image from well over a gigabyte down to tens of megabytes, sometimes less. The final stage only gets what I explicitly COPY --from the builder stage, so the toolchain and source code never make it into what actually ships."

---

## 10. Container Security Basics: Non-Root & Minimal Images
`CORE`

**Why it exists:** A container process runs as root by default — and that root maps to the REAL root (UID 0) on the host kernel, since containers share the host kernel (Section 1). If an attacker escapes a root container through some vulnerability, they're already much closer to real root on the host than they would be escaping a properly-configured non-root container.

**Physical flow:**
```bash
$ docker run --rm busybox id
uid=0(root) gid=0(root) groups=0(root)

$ docker run --rm --user 1000:1000 busybox id
uid=1000 gid=1000
```

**Decision point:** The fix is a `USER` instruction in the Dockerfile (or `--user` at run time) so the process runs unprivileged inside the container — combined with granting only the specific Linux capabilities it actually needs, rather than the full default set.

**Plain example:** Running a container as root is like giving a new employee the building master key on day one, "just in case" — running as a scoped non-root user is giving them a key to exactly the rooms their job requires, so a compromised badge only exposes those rooms, not the whole building.

> [!NOTE]
> **Anchor Fact:** A container's default `root` (UID 0) maps to the actual host's UID 0, because there's no separate kernel translating identities the way a VM's hypervisor would. Adding `USER <non-root>` in the Dockerfile is one of the highest-leverage, lowest-effort container security fixes for most applications.

> [!TIP]
> **Interview Answer:** "By default a container runs as root, and because containers share the host kernel, that's the real UID 0 on the host — not a sandboxed root the way a VM would give you. Adding a USER instruction so the process runs unprivileged is one of the highest-value, lowest-effort security fixes for most applications, since it directly limits what an attacker gains if they ever escape the container."
---

# Part 2 — Deeper Mechanics

## PID 1 and Signal Handling
`CORE`

**Why it exists:** This is where Section 5's exec-vs-shell-form distinction stops being cosmetic. On a normal Linux machine, PID 1 is a real init system with special, built-in signal-forwarding and child-reaping behavior. Inside a container, whatever your image's startup command is BECOMES PID 1 — but it usually wasn't written to behave like an init system, it's just your application.

**Physical flow:**
1. `docker stop` sends `SIGTERM` to the container's PID 1, expecting it to clean up and exit gracefully.
2. Docker waits a grace period (10 seconds by default).
3. If the process hasn't exited by then, Docker sends `SIGKILL` — immediate, forced, zero chance for cleanup.

**The two real causes of "docker stop always takes exactly 10 seconds":**
1. **Shell-form CMD** (Section 5) can leave `/bin/sh` as PID 1 instead of your application — and a bare shell does not forward `SIGTERM` to the child process it launched. The signal is sent, but never actually reaches your app.
2. Your application genuinely doesn't handle `SIGTERM` in its own code, regardless of form.

**Decision point — the actual fixes, in order of what to try:**
- Use the exec form (`CMD ["app"]`, not `CMD app`) so your application is PID 1 directly, receiving signals with nothing in front of it.
- Handle `SIGTERM` explicitly in application code for a genuine graceful shutdown.
- If you need a real init process (e.g., you're running something that forks children of its own), use `docker run --init`, which inserts a minimal init (`tini`) as PID 1 specifically to forward signals correctly and reap zombies — solving the problem structurally instead of relying on the app alone.

> [!NOTE]
> **Anchor Fact:** `docker stop` = SIGTERM, wait for the grace period (10s default), then SIGKILL if still alive. `docker kill` = SIGKILL immediately, no grace period at all. A container that always takes exactly the full grace period to stop is a strong signal that SIGTERM either isn't reaching PID 1 or isn't being handled.

> [!WARNING]
> **Failure Scenario:** Every deploy takes an extra 10 seconds longer than expected during the stop phase, across many services. **Symptom:** consistent, exact grace-period delay. **Investigation:** `docker inspect <container> --format '{{.Path}} {{.Args}}'` and check whether the image's `CMD` uses shell form; separately, check whether the application has any `SIGTERM`/`SIGINT` handler in its code at all. **Root cause:** commonly shell-form CMD leaving a shell as PID 1, absorbing the signal before it ever reaches the actual application. **Fix:** switch to exec form, or add `--init` if the process tree is more complex than one simple command. **Verification:** re-deploy and confirm stop time drops close to instant instead of hitting the full grace period.

> [!TIP]
> **Interview Answer:** "A container that always takes the full stop grace period is telling me SIGTERM isn't reaching or isn't being handled by PID 1. The most common cause is shell-form CMD leaving a shell as PID 1 instead of the actual application, which doesn't forward the signal — switching to exec form so the app is PID 1 directly usually fixes it immediately, and for anything with a more complex process tree, docker run --init adds a minimal init specifically to forward signals and reap zombies correctly."

---

## The Four Levers for Image Size
`CORE`

**Why this is its own section:** "Why is your image so big" is one of the most consistently reported senior-signal questions — the answer is about trade-offs across a few concrete levers, not one silver bullet.

1. **Multi-stage builds** (Section 9) — by far the biggest single lever; keeps the entire build toolchain out of the shipped image.
2. **A slim or alpine base image** instead of a full OS image — fewer pre-installed packages means less weight and fewer things to patch/scan.
3. **Combine and order `RUN` steps deliberately** — each `RUN` is its own layer, and cleanup (like clearing a package manager's cache) has to happen in the SAME layer as the install to actually save space; doing it in a later `RUN` leaves the bloat baked into the earlier layer permanently, since layers are immutable once created.
4. **A proper `.dockerignore`** — without one, the entire build context (including `.git`, `node_modules`, local artifacts) gets sent to the Docker daemon and can end up copied in via a broad `COPY . .`, even if unintentionally.

> [!WARNING]
> **Failure Scenario:** A Dockerfile runs `apt-get update && apt-get install -y build-essential` in one `RUN`, then a SEPARATE later `RUN rm -rf /var/lib/apt/lists/*` to "clean up." **Symptom:** the image is still large despite the cleanup step being present. **Investigation:** `docker history` shows the install layer is still large — the cleanup ran in its OWN layer, which only removes those files from that later layer's view, not from the earlier layer where they were actually written; layers are immutable, so the disk space from the install layer is permanently part of the image regardless of what a later layer deletes. **Root cause:** cleanup separated from the install it's supposed to clean up after. **Fix:** combine install and cleanup into a single `RUN` command chained with `&&`, so the cache is never actually written into any layer that ships. **Verification:** `docker history` shows the install layer's size drop correspondingly.

> [!TIP]
> **Interview Answer:** "Four real levers: multi-stage builds to keep the toolchain out entirely, a slim or alpine base, combining install-and-cleanup into a single RUN layer since layers are immutable and a later cleanup step can't shrink an earlier layer, and a proper .dockerignore so the build context itself isn't bloated. Multi-stage is the biggest single lever by far — I've personally seen it take an image from over a gigabyte to about fifteen megabytes for the same application."

---

## Secrets: What Never Goes in an Image
`CORE`

**Why it exists:** Anything baked into an image layer — including something later deleted in a subsequent layer — is still recoverable from the image's layer history by anyone who can pull it, exactly like the cleanup problem above. Secrets need a fundamentally different handling path than regular config.

**Physical flow:**
- Regular, non-sensitive configuration: environment variables (`-e`, or an `env_file`), or mounted config files — fine to pass at runtime, and the same image runs correctly across dev/staging/prod with different values.
- Secrets specifically: NEVER baked into the image via `ENV`, `ARG`, or a `COPY`'d file that stays in the final layer — `docker history` (or simply `docker save` and inspecting the layers) can reveal anything ever written into any layer, even if a later layer deletes the file.
- The correct mechanism: runtime-injected environment variables for lower-sensitivity values, or a real secrets store (cloud provider secrets manager, Docker/Swarm secrets, or the orchestrator's secret mechanism) mounted as a file at runtime, never during build.

> [!NOTE]
> **Anchor Fact:** A secret baked into any layer — even one later "deleted" in a subsequent layer — is still recoverable from the image, because layers are immutable and additive; deleting a file in a later layer only hides it from the final filesystem view, it doesn't remove it from the earlier layer's actual stored content.

> [!TIP]
> **Interview Answer:** "Secrets never go in the Dockerfile, full stop — not as an ENV, not as a file that gets deleted in a later layer, because layers are immutable and additive, so anything ever written into any layer is still pullable out of the image's history. Config that isn't sensitive can be an environment variable passed at runtime; actual secrets need a real secrets mechanism injected at runtime, never baked in at build time."

---

## Docker Compose
`CORE`

**Why it exists:** A real application is rarely one container — it's an API, a database, a cache, maybe a reverse proxy. Compose defines that whole multi-container application declaratively in one YAML file, so it comes up together, reproducibly, with one command.

**Physical flow:**
1. `docker-compose.yml` declares services, networks, and volumes.
2. `docker compose up` builds/pulls what's needed and starts every declared service.
3. Compose automatically creates a user-defined network for the whole stack — which is exactly why services can resolve each other by service name out of the box (Section 7's DNS distinction, solved automatically).

**Decision point:** Compose is for a single host — local development, a small single-machine deployment, or a CI test environment. It is NOT a production multi-host orchestrator — that's what the Kubernetes document in this series covers next.

> [!TIP]
> **Interview Answer:** "Compose defines a multi-container application in one YAML file — services, networks, volumes — brought up together with one command. It's a genuine convenience for local development and single-host setups, and it solves the container-name-resolution problem automatically since it creates a user-defined network by default. It's not a production orchestrator, though — for multi-host scaling and self-healing, that's Kubernetes' job, not Compose's."

---

## EXPOSE vs Publishing a Port
`CORE`

**Why it exists:** These are commonly conflated, and the distinction is a real, common early mistake — one is documentation, the other actually opens network access.

**Physical flow:**
- `EXPOSE 8080` in a Dockerfile is purely informational/documentation — it records which port the application listens on inside the container. It does NOT open anything or make the port reachable from outside.
- `docker run -p 8080:80` (or `--publish`) is what actually maps a host port to a container port, making it reachable from outside the container.

> [!NOTE]
> **Anchor Fact:** `EXPOSE` documents; `-p`/`--publish` actually opens the mapping. A container with `EXPOSE 8080` and no `-p` flag at runtime is not reachable from outside the container's network, no matter what the Dockerfile says.

> [!TIP]
> **Interview Answer:** "EXPOSE is documentation only — it doesn't open anything by itself. Only -p or --publish at run time actually maps a host port to the container and makes it reachable from outside. I've seen this conflated enough that stating it plainly, unprompted, is a small but real signal that you've actually published a port in a real deployment, not just read about it."

---

## Distroless & SBOM/Image Signing
`STRETCH`

**Why it exists:** Distroless and minimal images push the "smaller is safer" principle (Section 10's Four Levers) further — no shell, no package manager, nothing beyond the application runtime itself. An attacker who lands inside has almost no tools available. SBOM (Software Bill of Materials) and image signing are the current, growing supply-chain-security layer on top of that: knowing exactly what's in an image, and proving it hasn't been tampered with since it was built.

**The trade-off worth naming:** No shell means you can't `docker exec` in to debug interactively — some teams keep a debug variant of a distroless image specifically for troubleshooting, accepting the extra image to maintain in exchange for a genuinely harder-to-exploit production image.

> [!NOTE]
> **Anchor Fact:** SBOM = a complete inventory of every package/library/dependency in an image, generated automatically by modern build tooling, used for vulnerability tracking and compliance. Image signing verifies an image came from a trusted publisher and hasn't been modified since — a growing baseline expectation in security-conscious pipelines, distinct from vulnerability scanning itself.

> [!TIP]
> **Interview Answer:** "Distroless takes the minimal-base idea to its logical end — no shell, no package manager, just the app and its runtime, which massively shrinks what an attacker has to work with if they land inside. The trade-off is you lose the ability to exec in and poke around interactively, so some teams keep a debug variant around just for that. SBOM and image signing are the layer above that — knowing exactly what's inside an image and proving it hasn't been tampered with, which is increasingly a baseline expectation, not just a nice-to-have, in security-conscious pipelines."
---

# Part 3 — Interview Corner

## Images & Layers Drill

1. **Basic:** What's the actual difference between an image and a container? — Image is the read-only template; container is a running instance with its own writable layer on top. One image, many independent containers.
2. **Why:** Why does Docker build images as stacked layers instead of one flat filesystem snapshot? — Caching and sharing — unchanged layers are reused across builds (faster builds) and across images that share a base (less duplicated storage on disk and less to pull over the network).
3. **How:** Walk through what happens between running `docker build` and getting a finished image. — Each instruction is evaluated in order; for each one, Docker checks the cache — reuse if inputs match a previous build, otherwise execute it and create a new layer — until every instruction has produced or reused a layer, then the final layer stack becomes the image.
4. **Scenario:** Your image is 1.2GB for an app whose actual binary is 10MB. Where do you even start looking? — `docker history <image>` first, to see which specific layer is actually heavy — almost always either the base image choice or a dependency-install step that wasn't cleaned up in the same layer it ran in.
5. **Troubleshooting:** After a rebuild, your code changes aren't showing up in the running container. — Check whether the build actually re-ran the source `COPY` step or printed `CACHED` for it — if cached unexpectedly, something upstream didn't change as expected; also confirm you're actually running the freshly-built image (retag/pull confusion is common here too). Force with `docker build --no-cache` to rule out a cache issue definitively.
6. **Architecture:** Design a Dockerfile strategy for a monorepo with three services sharing common dependencies. — A shared base image (built once, containing common dependencies) that each service's Dockerfile extends via `FROM`, combined with multi-stage builds per service so only that service's specific artifact ships — balancing build-time reuse against keeping each final image minimal and specific to what it actually runs.
7. **Comparison:** `COPY` vs `ADD` — when, if ever, would you actually choose `ADD`? — Only when you specifically want its auto-extract-a-local-tarball behavior; its ability to fetch remote URLs is generally considered a bad practice today (no retry logic, no cache invalidation control) versus a dedicated `RUN curl`/`wget` step.
8. **Follow-up:** If a base image itself gets a security patch, does rebuilding your image automatically pick it up? — Only if you're not pinning to an exact digest and the tag you're using (like `python:3.12-slim`) has been updated to point at a new digest upstream — pinning by digest instead of a mutable tag gives full reproducibility but means you must deliberately bump it to receive upstream security fixes; it's a real trade-off, not a free win either way.
9. **Pressure:** An interviewer says "reduce this image by 80% without seeing the Dockerfile — what do you check first, in order?" — Base image choice (full OS vs slim/alpine/distroless) first, since it's usually the single biggest fixed cost; then whether a multi-stage build is even being used at all; then whether install-and-cleanup are combined into single layers; then `.dockerignore` presence. State the order and why, not just the list.

---

## Networking & Compose Drill

1. **Basic:** What are Docker's three default networks? — bridge (default, private internal network with per-container IPs), host (shares the host's network stack directly), none (no networking at all).
2. **Why:** Why doesn't the default bridge network support container-name DNS resolution? — It's Docker's original, simpler network type, predating the embedded-DNS feature that was later added specifically to user-defined networks — the default bridge was never upgraded to add it, partly for backward compatibility.
3. **How:** How does Compose solve the container-name-resolution problem without you doing anything extra? — It creates a user-defined bridge network for the whole stack automatically, and Docker's embedded DNS on that network resolves each service by its Compose service name.
4. **Scenario:** Two containers, started with plain `docker run` (no `--network` flag), can't resolve each other by name, but `docker ps` shows both healthy and running. — Almost certainly both landed on the default bridge — confirm with `docker inspect <container> --format '{{.NetworkSettings.Networks}}'` for each, then move both onto a shared user-defined network.
5. **Troubleshooting:** `EXPOSE 8080` is in the Dockerfile, but the service isn't reachable from the host. — `EXPOSE` alone never opens anything — confirm whether `-p`/`--publish` was actually used at `docker run` time; without it, nothing is mapped to the host regardless of what the Dockerfile declares.
6. **Architecture:** Design the networking for a Compose stack: a public-facing API, an internal-only database, and a cache neither should be reachable from outside. — Multiple Compose networks — a "frontend" network the API and any reverse proxy join, and a "backend" network the API, database, and cache all join, with the database/cache NOT joining the frontend network at all — so nothing external can reach them even indirectly, only the API can bridge between the two networks.
7. **Comparison:** Bridge network vs host network — when would you actually choose host? — Host networking removes all network isolation for maximum raw performance and simplicity (no port mapping needed) — appropriate for niche cases like certain monitoring agents that need to see the host's real network interfaces directly; it sacrifices the isolation that's usually the entire point of containerizing something, so it's a deliberate, narrow exception, not a default choice.
8. **Follow-up:** Can a container be on more than one Docker network at once? — Yes — `docker network connect <network> <container>` attaches an already-running container to an additional network, which is exactly the mechanism behind the frontend/backend network-segmentation pattern above (a reverse proxy or API container bridging two otherwise-isolated networks).
9. **Pressure:** Production shows intermittent DNS resolution failures between two containers that normally resolve each other fine. What's your investigation path? — Check whether anything recently changed network membership (a redeploy that didn't explicitly rejoin the right network, a Compose file change), check Docker daemon logs for embedded DNS errors, and rule out a resource issue on the host itself (Docker's embedded DNS is a real process that can be resource-starved under host-level pressure) before assuming it's purely a configuration problem.

---

## Security & Isolation Drill

1. **Basic:** Why does a container run as root by default, and why is that worth changing? — No `USER` instruction means the default user is root — and since containers share the host kernel, that's the actual host UID 0, not a sandboxed equivalent, making a container escape far more damaging than it needs to be.
2. **Why:** Why is container isolation considered weaker than a VM's, specifically? — All containers on a host share that host's actual kernel — a kernel-level vulnerability or a misconfigured privileged container can cross the boundary in ways a VM's separate-kernel-plus-hypervisor model makes structurally much harder.
3. **How:** How do namespaces and cgroups concretely divide the isolation and limiting responsibilities? — Namespaces control visibility (what the process can see: its own PIDs, its own network stack, its own filesystem view). Cgroups control consumption (how much CPU/memory/IO it's allowed to use). A process can be fully namespace-isolated and still starve the host without cgroup limits, or vice versa — they solve genuinely different problems.
4. **Scenario:** A container needs to bind to port 80, which normally requires root privileges on Linux. How do you allow that without running the whole container as root? — Grant only the specific Linux capability needed (`CAP_NET_BIND_SERVICE`) via `--cap-add`, rather than running as root wholesale — the principle of least privilege applied at the capability level, not just the user level.
5. **Troubleshooting:** A security scan flags a running production container for having a shell and package manager present, calling it an unnecessary attack surface. — This is exactly what distroless/minimal base images address — the fix is migrating that image's final stage to a distroless or minimal base via a multi-stage build, accepting the debugging trade-off (no shell to exec into) in exchange for a smaller attack surface.
6. **Architecture:** Design the security posture for a container that must run as root internally (a legacy app you can't easily modify) but shouldn't compromise the host if breached. — Layer other controls since you can't fix the root-user issue directly: drop all unnecessary Linux capabilities explicitly, run with a read-only root filesystem where possible, apply a strict seccomp/AppArmor profile, and treat this container as higher-risk in monitoring — least privilege applied at every OTHER layer available, since the user-level fix isn't available.
7. **Comparison:** Vulnerability scanning vs SBOM vs image signing — three different things people often conflate. — Scanning checks known packages against a CVE database at a point in time. SBOM is the actual inventory of what's in the image, which scanning depends on and which persists for future re-checking as new CVEs are discovered. Signing proves an image's origin and integrity, unrelated to whether it has vulnerabilities at all — you can sign a vulnerable image; signing answers "did this come from who it claims to," not "is this safe."
8. **Follow-up:** Why can't a later Dockerfile layer truly delete something a build secret left behind in an earlier layer? — Layers are immutable and additive — a later layer can only mark a file as deleted in the union filesystem view presented to a running container, it cannot rewrite or shrink an earlier layer's actual stored content, which is still fully present and extractable from the image.
9. **Pressure:** You inherit a production image that's been running as root for two years with no incidents. The team says "it's fine, nothing's happened." How do you argue for fixing it without being alarmist? — Reframe from "has something happened" to "what's the actual blast radius if something does" — the absence of an incident isn't evidence of low risk, it's evidence of no successful attack yet; propose it as a low-effort, low-risk change (add a USER instruction, test in staging) rather than an emergency, since the fix itself is genuinely low-cost relative to the exposure it removes.

---

## Troubleshooting Drill

1. **Basic:** A container exits immediately after `docker run`. First command? — `docker logs <container>` to see what it printed before exiting, and `docker ps -a` to see its exit code and status.
2. **Why:** Why does a container with no long-running foreground process just... stop, instead of erroring? — A container lives only as long as its PID 1 process — if that process finishes (even successfully, exit code 0), the container has nothing left to run and stops, exactly as designed, not as a bug.
3. **How:** Walk through diagnosing a container that exits with code 137. — 137 = 128 + 9 = killed by SIGKILL. Confirm the specific cause with `docker inspect <container> --format 'OOMKilled={{.State.OOMKilled}}'` — `true` means the kernel's OOM killer terminated it for exceeding its memory limit; if `false`, something else sent SIGKILL (worth checking orchestrator-level health check failures or a manual `docker kill`).
4. **Scenario:** A container that ran fine for weeks suddenly starts exiting with 137 repeatedly after a routine deploy. — Check whether the deploy changed the memory limit configuration, or whether application behavior changed (a new code path with a memory leak, a larger workload) pushing it against an unchanged limit — compare the container's actual memory usage trend (via `docker stats` or equivalent monitoring) against its configured limit before assuming either side is "wrong."
5. **Troubleshooting:** "It works in my local container but fails from the exact same tag pulled from the registry." — Three usual suspects, checked in order: tag drift (confirm the digest, not just the tag name, since `latest` and even specific tags can silently point at different content over time if repushed); a missing build artifact present locally but not in a clean build context; and outright `latest`-tag ambiguity if that's what's being used anywhere in the chain. The fix in every case is the same principle — pin by digest and rebuild from a clean checkout to get determinism back.
6. **Architecture:** Design a deploy process that makes exit-code-137-style failures visible immediately instead of being discovered by users reporting errors. — Container-level memory metrics and OOM-kill events wired into alerting (not just uptime/health-check monitoring, which can miss a kill-and-immediate-restart cycle), combined with sane restart-policy limits so a crash-looping container surfaces as a clear alert rather than quietly cycling forever.
7. **Comparison:** `docker logs` vs `docker inspect` vs `docker events` — when do you reach for each while debugging? — `logs` for what the application itself printed. `inspect` for the container's actual configured state and exit/OOM details at this moment. `events` for a live or historical stream of what Docker itself did to a container (started, killed, died, OOM) — useful specifically when you need to know WHEN something happened, not just what its current state is.
8. **Follow-up:** A container keeps restarting in a loop under a `Restart=always`-equivalent policy, and each restart's logs look identical. — Get the logs from the PREVIOUS failed attempt specifically, not just the newest restart (which may not have logged the real error yet, or may be too early in its lifecycle) — most container runtimes/orchestrators support a "previous" or historical log flag for exactly this reason, mirroring the same `--previous` pattern used for a crash-looping Kubernetes pod.
9. **Pressure:** Production is down, a container is stuck, and you're being watched while you debug live. What's your sequencing? — State the plan out loud before typing: check status/exit code first (fast, non-destructive), then logs (fast, non-destructive), then inspect for OOM/config details (fast, non-destructive) — only reaching for anything that touches the container's state (restart, recreate) after you've gathered enough non-destructive evidence to have an actual hypothesis, not a guess. Narrating this order is itself part of what's being evaluated.

---

## Quick-Coverage Round

**Images & Registries**
- Why is the `latest` tag risky? It's not "the newest version" — it's just the default tag applied when none is specified, and it's a moving target: two pulls weeks apart can silently return different images.
- What's the fix for `latest`-tag ambiguity in anything that matters? Pin explicit version tags, or better, pin by digest for full reproducibility.

**Compose & Multi-Container**
- What three things does a Compose file typically declare? Services, networks, volumes.
- Does Compose scale across multiple hosts? No — that's an orchestrator's job (Kubernetes, covered next in this series), not Compose's.

**Cleanup**
- What does `docker system prune` remove? Stopped containers, unused networks, and dangling (untagged) images.
- What does adding `-a` to that command additionally remove? Any image not currently used by at least one container, not just dangling ones — a much more aggressive cleanup.

**Signals & Lifecycle**
- `docker stop` vs `docker kill`? Stop = SIGTERM, grace period, then SIGKILL. Kill = SIGKILL immediately, no grace period.
- What silently breaks SIGTERM delivery? Shell-form CMD leaving a shell as PID 1 instead of the application.

---

## Hot Topics Right Now

- **Rootless Docker / Podman as a daemonless alternative** — running the container engine itself without a root daemon, or without a central daemon at all — worth name recognition given the ongoing industry conversation around reducing Docker's traditional root-daemon attack surface.
- **BuildKit as the modern build engine** — the default builder in current Docker versions, offering better caching, parallel stage execution, and build secrets that don't leak into image layers (`RUN --mount=type=secret`) — a genuine, current improvement over the older, simpler build path.
- **SBOM and image signing as baseline expectations** — covered in Part 2, and increasingly asked about directly rather than as a stretch topic, reflecting real current supply-chain-security emphasis across the industry.

---

## Research-Backed Additions (Current Market, 2025–2026)

> [!NOTE]
> **What's actually confirmed from current interview-prep material:** A detailed, dated (mid-2026) guide built specifically around real, run-on-real-Docker command output frames nearly every hard Docker question as reducing to one of two ideas — layers or isolation — and structures its troubleshooting section entirely around scenarios rather than trivia: a container exiting immediately, code changes not appearing after a rebuild, exit code 137, and data loss after `docker rm`. All four of those exact scenarios are covered directly in this guide's Troubleshooting Drill and Deeper Mechanics sections above.

> [!NOTE]
> **Docker rarely gets a dedicated interview slot** — it typically shows up inside a broader DevOps, backend, or platform round, in three recurring shapes: quick fundamentals (image vs container, what a layer is), "why" questions (why is your image big, why avoid latest, why run as non-root — this is explicitly where senior-level signal lives, since these are trade-off questions, not syntax recall), and live troubleshooting you debug out loud, where the method is scored as much as the fix itself.

> [!NOTE]
> **India market context:** Docker is listed as a core, expected skill across both product companies (reports specifically name Flipkart, Razorpay, Paytm among companies asking Docker questions in backend/system-design rounds) and major services firms (TCS, Infosys, Accenture, Wipro are named as top recruiters for Docker-related roles) — reinforcing that this isn't a niche or optional topic for the Indian DevOps market specifically.
---

# Part 4 — Real Incident / Story Bank

## Story 1: The ECR Tag Immutability Break

**Symptom:** A CI pipeline pushing Docker images to ECR started failing on the push step, after previously working fine — the pipeline was re-pushing an image using the `latest` tag, the same tag reused across builds.

**Initial hypothesis:** Something in the pipeline's authentication or permissions had broken, since the push itself was failing outright rather than the application behaving wrong post-deploy.

**Evidence/investigation:** The actual ECR error pointed at tag immutability — the ECR repository had immutable tags enabled (a deliberate security/traceability setting), meaning once a specific tag is pushed, that exact tag can never be overwritten by a different image again. Reusing `latest` across builds is precisely what immutable tags are designed to reject.

**Narrowing:** The real fix wasn't disabling immutability (that would remove a genuinely valuable safety property — knowing a tag always points at one specific, unchanging image) but changing the tagging strategy itself to never need to overwrite a tag in the first place.

**Root cause:** The pipeline's tagging strategy (reuse `latest`) was fundamentally incompatible with the repository's immutable-tag setting — not a bug in either one individually, a mismatch between the two.

**Fix:** Moved to a "build once, promote — never rebuild" pattern: each build gets a unique, immutable tag (the Git commit SHA), pushed once. Promoting that exact image through environments (staging → production) is done by re-tagging the SAME already-built image (via `aws ecr put-image`, pointing a new tag at the existing image's digest) — never rebuilding or re-pushing under a reused tag.

**Verification:** Confirmed staging and production were provably running the identical image (same digest) that passed testing, rather than two separately-built images that happened to share a tag name — closing a subtle but real risk (a "works in staging" image not being bit-for-bit identical to what reaches production).

**Prevention/what this taught:** Immutable tags force a better practice rather than just blocking a bad one — SHA-based tagging plus explicit promotion means you can always answer "what exact code is running in production" with certainty, and rollback becomes "point back at the previous known-good digest," not "hope a rebuild produces the same thing."

> [!TIP]
> **How to tell this in an interview:** "I hit an ECR push failure caused by tag immutability rejecting a reused latest tag — which was actually the immutability setting doing its job, catching a tagging strategy that would have made it impossible to know exactly what was running where. I moved to tagging every build with its Git commit SHA and promoting the same already-built image through environments by re-tagging it, rather than rebuilding — which means staging and production are provably running the identical image, not just two builds that happen to share a name."

---

## Story 2: The Three-Container Network With a Reverse Proxy

**Symptom:** A frontend, a backend API, and an Nginx reverse proxy needed to run as three separate containers, with the reverse proxy routing requests to the backend — but hardcoding the backend's IP address anywhere was clearly going to break the moment any container restarted and got reassigned a different IP.

**Initial hypothesis:** Some kind of service discovery or static IP assignment would be needed to make the reverse proxy reliably find the backend container.

**Evidence/investigation:** Confirmed that Docker's default bridge network doesn't support container-name resolution at all — the three containers needed to be deliberately placed on a shared, user-defined network specifically to get Docker's embedded DNS.

**Narrowing:** Once on a user-defined network, the reverse proxy's config could reference the backend simply by its container name, letting Docker's own DNS resolve it to whatever IP that container currently has — no hardcoded IP, no static assignment needed at all.

**Root cause:** Not a bug — this was designing the network topology correctly from the start, once the default-bridge DNS limitation was understood.

**Fix:** All three containers joined one explicit, user-defined Docker network; the Nginx config's `proxy_pass` directive pointed at the backend by container name (e.g., `proxy_pass http://backend:8000;`) rather than any IP.

**Verification:** Confirmed the reverse proxy correctly routed to the backend even after deliberately restarting the backend container (which gets a fresh IP on restart) — the name-based resolution meant nothing needed to be reconfigured.

**Prevention/what this taught:** Designing container networking around names instead of IPs from the outset avoids an entire category of fragile, restart-sensitive configuration — the user-defined network isn't just a "nice to have," it's what makes a multi-container architecture actually resilient to normal container lifecycle events like restarts and redeploys.

> [!TIP]
> **How to tell this in an interview:** "My project runs as three containers — frontend, backend, and an Nginx reverse proxy — all on one user-defined Docker network specifically so the reverse proxy can reach the backend by container name through Docker's embedded DNS, rather than a hardcoded IP that would break the moment the backend container restarted and got reassigned a different address. It's a small design decision, but it's exactly the kind of thing that separates a demo that only works once from an architecture that actually tolerates normal container lifecycle events."

---

## Practice Scenario (rehearsal only — not a personal account)

The following is a worked example for practicing the troubleshooting-narration format under pressure — not something to present as your own history.

*A container running a batch-processing job starts exiting with code 137 every few hours, always after processing roughly the same volume of data. First check: `docker inspect <container> --format 'OOMKilled={{.State.OOMKilled}}'` confirms `true` — this is genuinely an out-of-memory kill, not an unrelated SIGKILL. Second check: the container's configured memory limit versus its actual observed usage trend via monitoring — usage climbs steadily through each run rather than staying flat, a shape suggesting an in-process memory leak rather than the workload simply being too big for a correctly-sized limit. Third check: application-level — does the batch job process records in a way that accumulates memory across iterations instead of releasing it (an unbounded cache, a growing in-memory list never cleared)? Immediate fix: raise the memory limit enough to get the current backlog processed without further OOM kills, buying time. Real fix: the application code itself needs the leak addressed — raising the limit alone only delays the same failure at a larger scale. Verification: after the code fix, the same monitoring shows memory usage plateauing instead of climbing across a run, confirming the leak (not just the symptom) is actually resolved.*

The lesson worth keeping regardless of whose story it is: **raising a memory limit after an OOM kill treats the symptom, not necessarily the cause** — a climbing-then-crashing memory pattern across repeated runs is a leak signature worth checking before just giving the container more room to leak into.

---

# Part 5 — Master Glossary (Active Recall)

| Term | Definition |
|---|---|
| Namespace | Kernel isolation mechanism — what a container process can see (PIDs, network, filesystem) |
| Cgroup | Kernel limiting mechanism — how much CPU/memory/IO a container process can consume |
| Image | Read-only, layered template; never runs by itself |
| Container | A running/stopped instance of an image, with its own writable layer |
| Layer | The filesystem change produced by one Dockerfile instruction; images are a stack of layers |
| Build cache | Reuses a layer if its instruction and inputs are unchanged; invalidates top-down from the first real change |
| COPY vs ADD | COPY copies files only; ADD also fetches URLs and auto-extracts tarballs — prefer COPY unless you need that extraction |
| RUN | Executes at build time, baked permanently into a layer |
| CMD / ENTRYPOINT | Define container-start behavior; ENTRYPOINT is the fixed executable, CMD is overridable default arguments |
| Exec form vs shell form | `["cmd","arg"]` makes your app PID 1 directly; `cmd arg` wraps it in /bin/sh -c, a shell becomes PID 1 instead |
| Pull, create, start | The three real steps behind `docker run`, each with its own distinct failure mode |
| Default bridge | Docker's default network; no built-in DNS, no name resolution between containers |
| User-defined network | A created Docker network with embedded DNS — containers resolve each other by name |
| Volume | Docker-managed persistent storage, outlives any specific container |
| Bind mount | A specific host path mounted directly into a container; great for dev, a security concern in prod |
| tmpfs | In-memory-only storage, never touches disk |
| Multi-stage build | Multiple FROM stages; only what's explicitly COPY --from'd reaches the final image |
| EXPOSE | Documentation only — does not open any port |
| -p / --publish | Actually maps a host port to a container port |
| docker stop vs docker kill | SIGTERM+grace-period-then-SIGKILL vs immediate SIGKILL, no grace |
| PID 1 signal handling | Whatever the container's startup process is, it must itself handle/forward signals — no free init behavior unless added (`--init`) |
| Exit code 137 | 128 + 9 — SIGKILL; check `OOMKilled` to confirm whether it was an out-of-memory kill specifically |
| USER (Dockerfile) | Sets a non-root user for the container process; one of the highest-leverage security fixes for most apps |
| Distroless | A minimal base image with no shell or package manager — smaller attack surface, harder to debug interactively |
| SBOM | Software Bill of Materials — full inventory of everything in an image, used for vulnerability tracking |
| Image signing | Verifies an image's publisher and integrity — separate concern from vulnerability scanning |
| Compose | Declares a multi-container application in one YAML file; single-host only, not a production orchestrator |
| BuildKit | The modern default build engine — better caching, parallel stages, leak-proof build secrets |
| Immutable tag (ECR) | A registry setting where a tag, once pushed, can never point at a different image again |
| Build once, promote | Tag by commit SHA once; promote the same image between environments by re-tagging its digest, never rebuilding |

---

# Part 6 — Additional Practical Exercises

## Exercise 1: Shrink a Real Image With a Multi-Stage Build

**Concept:** The single highest-value, most-cited Docker exercise in current interview-prep material — and the one most likely to come up as a live-coding task.

**Implementation:** Take any app you have handy (or a simple Python/Node/Go "hello world"), write a single-stage Dockerfile, build it and note the size, then write a multi-stage version and compare.

**Hands-on:**
```bash
docker build -f Dockerfile.single -t demo:single .
docker build -f Dockerfile.multi -t demo:multi .
docker images | grep demo
docker history demo:single    # see exactly what's heavy
docker history demo:multi
```

**Failure:** The multi-stage final image is still surprisingly large.

**Investigation:** `docker history demo:multi` — is the final stage actually using a slim/alpine base, or was that step accidentally left as a full OS image? Is anything besides the final artifact being `COPY --from`'d unintentionally (a broad `COPY --from=builder /app /app` instead of copying just the specific binary)?

**Fix:** Narrow the final `COPY --from` to exactly the artifact needed, and confirm the final stage's `FROM` is actually minimal.

**Verification:** Compare `docker images` sizes again — expect at minimum a several-times reduction, often far more.

**Interview Explanation:** "Once you've actually run this comparison yourself, the number stops being abstract — moving the build toolchain into an earlier stage that never ships routinely takes an image from over a gigabyte down to tens of megabytes for a compiled app. I'd rather say that having tested it than just recite it from a blog post."

---

## Exercise 2: Debug Two Containers That Can't Talk

**Concept:** Directly reproduce and fix a commonly reported real Docker networking confusion, cold, without looking at the answer first.

**Implementation:**
```bash
# INSPECT — reproduce the failure first
docker run -d --name svc-a --rm nginx:alpine
docker run --rm busybox nslookup svc-a     # expect NXDOMAIN

# VALIDATE — confirm which network svc-a actually landed on
docker inspect svc-a --format '{{json .NetworkSettings.Networks}}'

# EXECUTE — fix by moving both onto a shared user-defined network
docker network create fixnet
docker run -d --name svc-b --network fixnet --rm nginx:alpine
docker run --rm --network fixnet busybox nslookup svc-b

# VERIFY
docker network inspect fixnet   # confirm both containers are listed as members
```

**Failure:** Even after creating the network, resolution still fails for one of the two containers.

**Investigation:** Confirm BOTH containers were actually started with `--network fixnet` — a very easy step to forget for one of the two while fixing the other.

**Fix:** Restart the missed container with the correct `--network` flag, or `docker network connect fixnet <container>` to attach it without a restart.

**Verification:** `docker network inspect fixnet` should list both containers as current members; the `nslookup` succeeds both directions.

**Interview Explanation:** "The first thing I check when two containers can't resolve each other is which network each one is actually on — this is almost always a default-bridge-versus-user-defined-network issue, not a DNS misconfiguration, since the default bridge simply doesn't run embedded DNS at all."

---

*End of Docker — Document 3 of 7. Next up: Kubernetes.*
