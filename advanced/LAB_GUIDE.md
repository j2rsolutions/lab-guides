# Chainguard Advanced — Beyond the Basics — Lab Guide

*Self-paced version of the five labs.*

Everything runs on your own lab server — nothing to
install, nothing on your laptop. Work top to bottom; each lab stands alone if you fall behind
during the live session.

Day one proved the images. Today proves the operating model: your libraries, your own builds,
your cluster's admission policy, and your air gap. Each lab produces one **artifact** — collect
them in the evidence sheet at the bottom; together they're the outline of a supply chain SOP.

## Table of contents

- [Lab 0 — Get your seat](#lab-0--get-your-seat)
- [Lab 1 — Point your libraries at a trusted source (~15 min)](#lab-1--point-your-libraries-at-a-trusted-source-15-min)
- [Lab 2 — Build your own image, the Chainguard way (~20 min)](#lab-2--build-your-own-image-the-chainguard-way-20-min)
- [Lab 3 — Migrate the hard one: Java (~20 min)](#lab-3--migrate-the-hard-one-java-20-min)
- [Lab 4 — Make the cluster refuse unsigned images (~25 min)](#lab-4--make-the-cluster-refuse-unsigned-images-25-min)
- [Lab 5 — Cross the air gap, with the evidence attached (~15 min)](#lab-5--cross-the-air-gap-with-the-evidence-attached-15-min)

---

## Lab 0 — Get your seat

1. Open the portal — **the URL is in your welcome email** (and on your table card at in-person events).
2. Log in with the username and password on your table card (`lab01`…`labNN`).
3. Click the one tile you see: **Your Lab Server (labNN)**. A terminal opens in the browser tab.
4. Clipboard: press **Ctrl+Alt+Shift** to open the side menu, paste into its clipboard box,
   close the menu, then paste normally.

Your seat runs Docker, a single-node Kubernetes cluster (kind) with the Sigstore
policy-controller already installed, and a local "high-side" registry at `localhost:5000`. Lab
files live in `~/labs`. Sanity check:

```bash
kubectl get nodes && crane version
```

A `Ready` node and a crane version string. (If the cluster is ever missing —
`sudo /opt/lab/bootstrap.sh` rebuilds everything in about 90 seconds.)

---

## Lab 1 — Point your libraries at a trusted source (~15 min)

pip, npm, and Maven pull from public repos where credential theft and malicious packages are
now routine. Chainguard Libraries serves the same packages **rebuilt from source** in a hardened
pipeline — and your tooling barely notices the switch. Your seat already has the auth wired
(`~/.netrc`, `~/.npmrc`); *selecting the index* is the entire migration, and that's your job:

```bash
pip config set global.index-url https://libraries.cgr.dev/python/simple/
pip install flask
```

Watch the first line of output — that's the receipt:

```
Looking in indexes: https://libraries.cgr.dev/python/simple/
```

Same move for npm:

```bash
npm config set registry https://libraries.cgr.dev/npm/
npm install axios
```

Prove the source of truth (this output is your evidence-sheet artifact):

```bash
pip config list && npm config get registry
```

Every future install on this machine now resolves from source-rebuilt packages, not public
mirrors — the poisoned-package class of attack never reaches your build. In CI, these same
config lines protect every pipeline at once. (Maven works the same way via a `settings.xml`
mirror — same endpoint family, not labbed today.)

---

## Lab 2 — Build your own image, the Chainguard way (~20 min)

Every image you pulled on day one was assembled by **apko** from signed Wolfi packages — no
Dockerfile, no `RUN`, no build-time shell. Today you drive the same machinery:

```bash
cd ~/labs/apko
cat hello.yaml
```

```yaml
contents:
  packages:
    - wolfi-base
    - python-3.12
entrypoint:
  command: python3 /hello.py
```

The YAML *is* the image — reviewable in a pull request like any other code. Assemble it (apko
runs from its own container; the package cache was pre-warmed at image-build, so this is fast — a much older seat may re-fetch a little):

```bash
./build.sh
```

The build wrote two artifacts: `hello.tar` (the image) and `sbom-x86_64.spdx.json` — the SBOM
was **generated during the build**, not reverse-engineered afterwards, because a declared image
already knows its own inventory. Scan what you built:

```bash
grype hello:adv
```

Expect a clean (or near-clean) result on an image you assembled yourself.

> [!NOTE]
> **Why isn't `hello.py` inside the image?** apko has no `COPY` — contents come from declared,
> signed packages, full stop. That constraint is the point: nothing enters the image that isn't
> declared and inventoried. App files become packages (built with melange, apko's sibling) in
> production; for a quick run today, mount it:
> `docker run --rm -v "$PWD/hello.py":/hello.py hello:adv` →
> `hello from an image with no Dockerfile`

---

## Lab 3 — Migrate the hard one: Java (~20 min)

"That works for Python toys, but we run JVMs." This lab is the answer. Two stages: Chainguard's
Maven `-dev` image compiles, the distroless JRE runs:

```bash
cd ~/labs/java
cat Dockerfile
```

```dockerfile
FROM cgr.dev/chainguard/maven:latest-dev AS build
WORKDIR /app
COPY pom.xml ./
COPY src ./src
RUN mvn -q package

FROM cgr.dev/chainguard/jre:latest
COPY --from=build /app/target/app.jar /app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

Build and run it (Maven dependencies are pre-cached on your seat — the build finishes in
seconds, offline):

```bash
docker build -t javapp:adv .
docker run -d --rm -p 8080:8080 javapp:adv && curl -s localhost:8080
```

```
Hello from distroless Java
```

```bash
grype javapp:adv
```

> [!TIP]
> 💡 If grype ever says `Pulled image` and then errors, you've typo'd a local tag — when a
> name isn't found locally, grype goes looking on Docker Hub. Check `docker images`.

A dramatically quieter scan than any stock JVM base — zero to a handful of findings depending on feed age — is your artifact. The toolbox (`maven:latest-dev`) never ships to
production; the runtime is non-root with no shell — nothing for an implant to land on. The same
two-stage shape covers Spring Boot, Quarkus, and your legacy WARs.

---

## Lab 4 — Make the cluster refuse unsigned images (~25 min)

The signature you verified by hand on day one becomes something the cluster checks on **every
deploy**, with no human in the loop. Apply the policy, then opt the namespace in:

```bash
kubectl apply -f ~/labs/policy/signed-only.yaml
kubectl label ns default policy.sigstore.dev/include=true
```

Now deploy the image every tutorial on the internet uses:

```bash
kubectl run web --image=nginx:latest
```

```
error: admission webhook "policy.sigstore.dev" denied the request: … no matching signatures
```

Read that out loud. Nobody was trained, nobody read a memo — the unsafe path simply stopped
working. (The exact wording varies slightly across policy-controller versions; the rejection
doesn't.) Now the signed build of the same software:

```bash
kubectl run web --image=cgr.dev/chainguard/nginx:latest
kubectl get pods
```

`pod/web created`, `Running`. The deny event is itself evidence — enforcement you can show an
assessor. Policies scope by namespace label and image glob, so break-glass namespaces are a
design choice, not a workaround.

---

## Lab 5 — Cross the air gap, with the evidence attached (~15 min)

`localhost:5000` is your "high side" registry today. The tool choice is the lesson: signatures
are separate artifacts in the registry, so a plain image copy **leaves the proof behind** —
`cosign copy` moves the image *and* its signatures together:

```bash
cosign copy cgr.dev/chainguard/nginx:latest localhost:5000/nginx:latest
docker pull localhost:5000/nginx:latest
```

Verify **on the far side** — same identity, new registry:

```bash
cosign verify \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity=https://github.com/chainguard-images/images/.github/workflows/release.yaml@refs/heads/main \
  localhost:5000/nginx:latest | jq
```

Trust survived the transfer. The same pattern feeds Harbor or Artifactory syncs from unclass to
high side. (Fully-offline verification with bundled transparency-log proofs exists as the
production pattern for truly disconnected enclaves; today's path verifies online.)

## Your evidence sheet

One artifact per lab — a screenshot or pasted command output each. Five artifacts, one story:
consumption → construction → enforcement → transfer.

| Lab | Artifact you produced | What it proves | ✓ |
|---|---|---|---|
| 1 · Libraries | pip/npm config output showing `libraries.cgr.dev` | Dependencies resolve from a source-rebuilt index | |
| 2 · Build | `hello.tar` + generated SPDX SBOM | You can build declarative, SBOM-native images | |
| 3 · Java | `javapp:adv` quiet scan | JVM workloads fit the -dev builds / distroless runs pattern | |
| 4 · Enforce | the admission deny event | The cluster rejects unsigned images without human help | |
| 5 · Air gap | verified image at `localhost:5000` | Signatures survive transfer to a disconnected registry | |

## The maturity ladder (where this leaves you)

1. **Consume** — hardened images from a trusted source *(day one)*
2. **Extend** — libraries and your own builds inherit the same trust *(Labs 1–2)*
3. **Migrate** — legacy workloads, Java included, on distroless patterns *(Lab 3)*
4. **Enforce** — the cluster rejects what policy forbids, automatically *(Lab 4)*
5. **Transport** — trust survives air gaps and enclave syncs *(Lab 5)*

Most programs live on rung 1. You just climbed all five before dinner — rungs 4 and 5 are the
ones assessors are starting to ask about by name.

## Next steps

1. **Wire one CI pipeline to Chainguard Libraries this week** — two config lines, immediate
   protection against the poisoned-package class.
2. **Pick one namespace and turn on signature enforcement in staging** — we'll pair on the
   policy design.
3. **Map your enclave sync to the signature-preserving pattern** — J2R will architect it with
   your registry team.

Jonathan Spigler · J2R Solutions · jonathan@j2rsolutions.io · (717) 491-2827
Phil Brooks · Partner Solutions Engineer, Chainguard
