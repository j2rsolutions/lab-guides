# Start Compliant, Stay Compliant — STIG Lab Guide

*Self-paced version of the labs. Everything runs on your own lab server — nothing on your laptop.*

> [!NOTE]
> **Numbers here are illustrative.** The DISA benchmark and image contents move; **your** numbers are
> what `./scan.sh` prints on your seat. The point is that you *start* compliant and *stay* compliant.

**Table of contents**

- [Lab 0 — Get your seat](#lab-0--get-your-seat)
- [Lab 1 — Which STIG applies to a container? (~10 min)](#lab-1--which-stig-applies-to-a-container-10-min)
- [Lab 2 — Start compliant (~10 min)](#lab-2--start-compliant-10-min)
- [Lab 3 — Develop on it, stay compliant (~15 min)](#lab-3--develop-on-it-stay-compliant-15-min)
- [Lab 4 — The guardrail (~10 min)](#lab-4--the-guardrail-10-min)
- [Lab 5 — The evidence package (~10 min)](#lab-5--the-evidence-package-10-min)

## The situation

Your team is up for an ATO. The assessor runs the DISA benchmark against your container images, and if
they come back with a wall of findings, the accreditation stalls. The usual approach is to *end* the
project by hand-remediating hundreds of OS settings. This lab does the opposite: you **start** on a
Chainguard STIG-hardened image that's already compliant, and the work becomes *keeping* it that way as
you build — which, it turns out, is nearly free.

The fair question this lab exists to answer: **"If I start on a hardened image, doesn't it break the
moment I install anything?"** No — and understanding *why not* is the whole point. STIG/GPOS compliance
is a property of the **OS layer** (no remote-access services, no login users, FIPS crypto, file
permissions). Your application and its language libraries live *above* that layer and don't touch it.
You develop normally. Compliance isn't fragile; it's measurable — and the scanner tells you the instant
you actually reintroduce a controlled risk.

## Lab 0 — Get your seat

1. Open the portal — **the URL is in your welcome email** (and on your table card at in-person events).
2. Log in with your table-card username/password (`lab01`…`labNN`).
3. You get two tiles: **a terminal** and **a desktop** (Firefox + a file manager, for reading the HTML scan report).

> **Tip — keep both open at once.** A tile normally opens in the same browser tab. **Ctrl-click** (or
> middle-click) a tile to open it in a **new browser tab** instead — so you can keep the **terminal**
> running in one tab and the **desktop** (to view reports in Firefox) in another, side by side.

Everything is staged in `~/labs/stig`: a `scan.sh` wrapper around OpenSCAP, the app source
(`app.py` + `Dockerfile.app`), the guardrail demo (`Dockerfile.broken`), and two images already built —
`stig-demo:app` (your app on the hardened base) and `stig-demo:broken`.

```bash
cd ~/labs/stig && ls && docker images | grep -E 'stig-demo|python-fips|openscap'
```

## Lab 1 — Which STIG applies to a container? (~10 min)

DISA has **never published a container STIG**, which is why "STIG a container" confuses everyone. Four
documents get conflated; knowing which is which is half the battle at an ATO:

| Layer | What assesses it |
|---|---|
| The container **image** (base-OS) ← *this lab* | **General Purpose OS SRG (GPOS SRG)** |
| The orchestrator (Kubernetes) | Container Platform SRG + Kubernetes STIG |
| The app inside | Application Security & Development STIG + code analysis |
| The end-to-end workflow | DoD Container Hardening Process Guide (a *process*, not a scannable benchmark) |

You're measuring the **image**, so it's the **GPOS SRG** — with host-inherited and container-inapplicable
checks documented Not Applicable, exactly as DISA's Container Hardening Process Guide prescribes.
Chainguard formally aligns its images to that SRG and publishes the scan content you're about to use.

## Lab 2 — Start compliant (~10 min)

Your base is `python-fips` — a Chainguard image that is **STIG-hardened and FIPS** out of the box.

First, don't take the wrapper on faith. `scan.sh` is three lines around a stock tool — read it:

```bash
cat scan.sh
```

No magic: it runs upstream **`oscap-docker`** against the DISA GPOS datastream and counts the failures.
The proof that it isn't rigged to print "0" is to run it on a stock image and watch it find things. So
scan the everyday baseline first — Docker Hub's own `python`:

```bash
./scan.sh python:latest
```

```
  STIG rules FAILED: 6
  report: /home/labuser/Desktop/stig-reports/report-python_latest.html   (open it in Firefox on your desktop)
```

Six real findings, against 396 evaluated rules — the scanner works, and it isn't grading on a curve.
Now the Chainguard base, the exact same command:

```bash
./scan.sh $(docker images --format '{{.Repository}}:{{.Tag}}' | grep python-fips | grep -v dev | head -1)
```

```
  STIG rules FAILED: 0   (the rest are documented Not Applicable — no login service, no local users)
```

**Zero — same scanner, same datastream, same day.** That's the starting line most projects would kill a
quarter to reach, before you've done anything. A minimal image satisfies most GPOS controls *by
construction*: no local user accounts, no configured passwords, no remote-access service; the rest carry
no automated check and are documented N/A with rationale.

> **Open both reports and see for yourself.** On your desktop, open the **STIG Reports** folder (or the
> File Manager) and double-click each `report-*.html` — Firefox renders OpenSCAP's full CAT I/II/III
> breakdown, rule by rule, pass and fail. The stock report shows you exactly which six rules tripped;
> the python-fips report shows a clean sheet. That side-by-side IS the lab.

Record both on the scorecard:

| Image | Size | STIG rules failed |
|---|---|---|
| `python:latest` (Docker Hub stock) | 1.12 GB | 6 |
| `python-fips` (Chainguard, hardened) | ~69 MB | 0 |

Note where the gap really lives: the STIG delta is real but modest (6 → 0), because STIG/GPOS measures
the **OS layer**. The dramatic difference is **16× the size** and — as the next lab shows — a vastly
larger CVE surface. Compliance and vulnerabilities are two different axes; you're about to meet the
second one.

> How does OpenSCAP even scan an image with no shell? `oscap-docker` does an **offline scan** — it
> mounts the image's filesystem read-only from *outside* and assesses it from the host. Nothing runs
> inside the target, which is exactly why it works on minimal/distroless images — and why the same
> wrapper scans stock python and hardened python-fips identically.

## Lab 3 — Develop on it, stay compliant (~15 min)

Now the real question: build an actual app on this base and see what it does to your compliance. Look at
how the app image is built — the normal Chainguard dev→runtime pattern:

```bash
cat Dockerfile.app        # pip install flask + your code, on the hardened base
```

It's already built as `stig-demo:app`. Confirm it's a real, working app:

```bash
docker run --rm -d --name myapp -p 8080:8080 stig-demo:app && sleep 2 && curl -s localhost:8080
docker rm -f myapp
```

```
compliant app, still serving
```

Now re-scan the image **with your app and Flask installed**:

```bash
./scan.sh stig-demo:app
```

```
  STIG rules FAILED: 0
```

**Still zero.** You added Python, Flask, and your application code — and the STIG posture didn't budge,
because none of that is an OS-layer control. This is the answer to "won't it break if I touch it?":
you develop your application normally and stay compliant, because compliance lives below your app, not
in it.

### Compliance and vulnerabilities are two different questions

Here's the distinction that makes this click — and the honest answer to "so adding code changes
*nothing*?" Adding your app and its dependencies **shouldn't change your compliance posture** (STIG/GPOS
is about the OS layer — services, users, crypto config — which your code doesn't touch). But it **can
change your vulnerability surface**: that Flask install pulled a dependency tree, and any of it could
carry a CVE. That's a *different question, measured by a different tool*:

| Question | Tool | Layer | Who owns it |
|---|---|---|---|
| Is the image **compliant**? (STIG) | OpenSCAP / `oscap` | OS baseline | **Inherited** from the hardened base — stays clean |
| Does the image have **vulnerabilities**? (CVEs) | grype / Trivy / **JFrog Xray** | your dependencies | **Yours**, and it grows as you add libraries |

So the clean division of labor: the Chainguard base hands you **STIG compliance, inherited and
maintained** (you're off the upstream-variance treadmill), and what remains is your app layer — your
**CVEs** (that's the CVE-Zero and JFrog Xray labs) and your application-security controls (the ASD
STIG). No base image can do those for you; this one just makes sure they're *all that's left*.

## Lab 4 — The guardrail (~10 min)

So what *does* break it? A real controlled risk. `stig-demo:broken` is the same base with one addition —
a remote-access (SSH) service someone bolted on "for debugging." Scan it:

```bash
cat Dockerfile.broken       # apk add dropbear
./scan.sh stig-demo:broken
```

```
  STIG rules FAILED: 46
```

One login service fails the GPOS **RemoteAccessServices** rule and the dozens of rules that hang off it — MFA,
session lock, the DoD banner, 3-strikes lockout, the idle timeout — because the SRG assumes that if you
run a login service you must run every control that protects one. The lesson isn't "images are
fragile." It's the opposite: **your compliance posture is measurable, and the scanner is a guardrail
that fires the instant you introduce a genuine risk** — not a tripwire that punishes you for installing
your app's dependencies. You'd only ever add SSH deliberately, and now you'd see it immediately, in CI,
before the assessor ever does.

## Lab 5 — The evidence package (~10 min)

A compliant image nobody can *prove* is compliant doesn't move an ATO. You already have the artifacts
from scanning `stig-demo:app`:

```bash
ls ~/Desktop/stig-reports/   # per-image: results-<image>.xml (machine-readable) + report-<image>.html (human-readable)
```

Hand the ISSO: the scan report on **your actual app image** (near-zero, with N/A rationale), plus the
image's provenance — because the STIG story and the supply-chain story are one story:

```bash
# org-entitled images are signed by Chainguard's build infrastructure
# (issuer.enforce.dev), not GitHub Actions
cosign verify --certificate-oidc-issuer=https://issuer.enforce.dev \
  --certificate-identity-regexp='^https://issuer\.enforce\.dev/' \
  $(docker images --format '{{.Repository}}' | grep python-fips | head -1) >/dev/null && echo "signature verified"
```

That's an ATO evidence package for a real, working application — not a promise, and not a stripped-down
demo image you can't actually ship.

## Your compliance log

| Stage | Image | STIG rules failed | Works? |
|---|---|---|---|
| start (hardened base) | `python-fips` | | n/a |
| your app on it | `stig-demo:app` | | |
| + remote-access service | `stig-demo:broken` | | |

## Next steps

1. **Bring your real app to a working session** — we'll rebuild it on the matching Chainguard
   STIG-hardened base (python-fips, and the other `-fips` variants: nginx, node, …) and scan it with you.
2. **Wire the scan into CI** — the guardrail belongs in your pipeline, so a compliance regression is
   caught at build time, not at assessment.
3. **Leave with the evidence-package pattern** — the finding report on your shipping image + the
   signature an assessor signs.

Jonathan Spigler · J2R Solutions · jonathan@j2rsolutions.io
