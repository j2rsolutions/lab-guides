# Open Source vs. Chainguard — Container Lab Guide

*Self-paced version of the five labs.*

Everything runs on your own lab server in the browser —
nothing to install locally. Work top to bottom; each lab stands alone if you fall behind.

> [!NOTE]
> The images and CVE feeds move daily, so **your** numbers are the ones that count — capture them
> on the scorecard at the bottom as you go. Nothing here asks you to trust a vendor claim; you
> prove each point yourself.

## Table of contents

- [Lab 0 — Get your seat](#lab-0--get-your-seat)
- [Lab 1 — Pull both, weigh both (~15 min)](#lab-1--pull-both-weigh-both-15-min)
- [Lab 2 — The scan showdown (~20 min)](#lab-2--the-scan-showdown-20-min)
- [Lab 3 — Look inside (~15 min)](#lab-3--look-inside-15-min)
- [Lab 4 — Prove where it came from (~15 min)](#lab-4--prove-where-it-came-from-15-min)
- [Lab 5 — Migrate a real Dockerfile (~25 min)](#lab-5--migrate-a-real-dockerfile-25-min)

---

## Lab 0 — Get your seat

1. Open the portal (the URL on your table card, e.g. the portal URL from your welcome email).
2. Log in with the username/password on your card (`lab01`…`labNN`).
3. Click the one tile you see: **Your Lab Server (labNN)**. A terminal opens in the browser tab.
4. Clipboard: press **Ctrl+Alt+Shift** to open the Guacamole side menu, paste into its clipboard
   box, close the menu, then paste into the terminal. (Paste one command at a time — a multi-line
   paste can merge lines.)

Sanity check:

```bash
docker version && grype version
```

---

## Lab 1 — Pull both, weigh both (~15 min)

Same software, two supply chains — both already staged on your seat at image-build time.
Weigh them:

```bash
docker images | grep -E 'nginx|python|REPOSITORY'

# bonus: confirm multi-arch (amd64 + arm64)
crane manifest cgr.dev/chainguard/nginx:latest | jq -r '.manifests[].platform.architecture'
```

> [!NOTE]
> Why no `docker pull`? Everything is pre-staged so a room full of seats isn't hammering
> registries over venue wifi — and because `:latest` moves daily, a live pull could re-tag
> your validated, pre-scanned image to a build nobody has tested today. The images you have
> ARE the lab.


**Record** the two nginx sizes on the scorecard — then the python pair right below them
(`python` vs `cgr.dev/chainguard/python`; the gap is even wider). Smaller means faster
pulls/cold starts, less registry and network cost, a smaller scan surface, and fewer files
for an attacker to use.

---

## Lab 2 — The scan showdown (~20 min)

Scan both images with grype. Record totals and Critical/High counts.

First, confirm your vulnerability data is current — the seat refreshes it at boot, and a
scan is only as honest as its feed:

```bash
grype db status
```

```bash
# upstream first — let it scroll
grype nginx:latest
# now the Chainguard build of the same software
grype cgr.dev/chainguard/nginx:latest

# just the numbers, for the scorecard
grype -q nginx:latest | tail -n +2 | wc -l
grype -q cgr.dev/chainguard/nginx:latest | tail -n +2 | wc -l

# the scorecard's python row works the same way
grype -q python:latest | tail -n +2 | wc -l
grype -q cgr.dev/chainguard/python:latest | tail -n +2 | wc -l

# skeptics welcome: cross-check with a second scanner
trivy image --severity HIGH,CRITICAL nginx:latest
```

Read it like an assessor: Criticals and Highs drive POA&M clocks; note how many findings sit in
packages your app never calls. Same scanner, same day, same software — only the base changed.

> [!NOTE]
> **Grype and Trivy won't report the same totals** — different vuln databases, different severity
> sources (grype leans on NVD/CVSS, Trivy on the distro's rating), and different counting. That's
> expected; the point isn't the absolute number, it's that the delta between the two *bases* is
> huge under *any* scanner.

---

## Lab 3 — Look inside (~15 min)

Count what's inside each image, then try to get a shell in both.

```bash
# rough package inventory, upstream vs Chainguard
syft -q nginx:latest | wc -l
syft -q cgr.dev/chainguard/nginx:latest | wc -l

# now try to land a shell in each
docker run --rm -it --entrypoint sh nginx:latest
#   ...you're in. ls, apt, curl — all there. (type 'exit')

docker run --rm -it --entrypoint sh cgr.dev/chainguard/nginx:latest
#   error: no such file or directory: sh — nothing to land on. That's the point.
```

No shell and no package manager means "living off the land" has no land, and the image runs as a
non-root user by default. When you must debug, use the `:latest-dev` variant — in the build stage,
never in production.

---

## Lab 4 — Prove where it came from (~15 min)

Verify the image signature with cosign, then export the SBOM your assessors will ask for.

```bash
# verify the Sigstore signature on the public image
cosign verify \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity=https://github.com/chainguard-images/images/.github/workflows/release.yaml@refs/heads/main \
  cgr.dev/chainguard/nginx | jq

# the upstream image the same way -> no matching signatures
cosign verify nginx:latest || echo "no signature — as expected"

# export an SBOM to hand to your assessor (single line so a paste can't split it)
syft cgr.dev/chainguard/nginx:latest -o spdx-json > nginx-sbom.json && jq '.packages | length' nginx-sbom.json
```

Signature verification proves the *build path*, not just a checksum — the certificate records the
repo, commit, workflow, and branch that produced the image. SPDX SBOMs ship with every image. This
is the artifact trail continuous-ATO packages are built from.

---

## Lab 5 — Migrate a real Dockerfile (~25 min)

Ship a working app on a distroless base and *feel* the migration — one `FROM` line. Your files are
in `~/labs/lab5` (`app.py` and a `Dockerfile` that **starts on the upstream base**).

**Step 1 — build and scan the upstream version (the "before"):**

```bash
cd ~/labs/lab5
docker build -t hello-oss .
docker run --rm hello-oss        # prints: hello from my app
grype -q hello-oss | tail -n +2 | wc -l
docker images | grep hello-oss   # note the size
```

**Step 2 — migrate: change the one `FROM` line and rebuild (the "after"):**

Edit `Dockerfile` and change the first line to:

```dockerfile
FROM cgr.dev/chainguard/python:latest
```

```bash
docker build -t hello-cg .
docker run --rm hello-cg         # same output — your code didn't change
grype -q hello-cg | tail -n +2 | wc -l
docker images | grep hello-cg    # smaller, near-zero findings
```

The app code never changed — only the base. **Record hello-oss vs hello-cg** (size + findings) on
the scorecard.

**The pattern for real apps** — build tooling in a `-dev` stage, distroless at runtime:

```dockerfile
FROM cgr.dev/chainguard/python:latest-dev AS builder
WORKDIR /app
# ...create a venv and pip install your deps into it...

FROM cgr.dev/chainguard/python:latest
WORKDIR /app
COPY --from=builder /app/venv /app/venv
COPY app.py .
ENTRYPOINT ["python", "app.py"]
```

Build tooling lives only in the builder stage; the runtime image stays distroless. **Bring your own
image? Migrate it now — swap its base, rebuild, and scan.**

## Your scorecard

| Image | Size · upstream | Size · Chainguard | CVEs · upstream | CVEs · Chainguard | Crit+High Δ |
|---|---|---|---|---|---|
| `nginx:latest` | | | | | |
| `python:latest` | | | | | |
| your app (`hello-oss` → `hello-cg`) | | | | | |

Keep this. In the readout, the CVE delta becomes hours: findings × triage time per finding ×
release cadence. That number — not a feature list — is the business case.

## Where to go next

1. Re-run Labs 1–2 against the images your programs actually deploy — keep the scorecard.
2. Pilot one workload on the free tier.
3. Need pinned versions, FIPS, or patch SLAs? That's the Production tier — scope it with your
   Chainguard SE.

Learn more: `edu.chainguard.dev` · `images.chainguard.dev` · `j2rsolutions.io`
