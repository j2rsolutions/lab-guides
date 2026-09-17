# Start Secure — Chainguard × JFrog — Lab Guide

*Self-paced version of the five labs.*

Everything runs on your own lab server and **your own
JFrog Project** (`lab01`…`labNN`, on the shared JFrog platform) — nothing to install; your
repos, watches and policies live inside your project's walls, and only the `cgr` front door and
the golden `prod-docker` are shared. Work top to bottom; each lab stands alone if you fall
behind during the live session.

> [!NOTE]
> **Numbers in this guide are illustrative** (620 findings on the upstream build, 1 on the
> Chainguard build — the same OctoShop figures the CVE Zero deck uses). CVE feeds move daily;
> **your** numbers are the ones Xray shows in your project. The gap is the point, not the
> digits.

## The idea

Your scanner isn't the problem. Your *baseline* is. Point Xray at an app built on open source
and it returns hundreds of findings — a dashboard nobody triages is a dashboard nobody uses.
Start from a hardened base instead and the dashboard goes quiet — so when a finding *does*
appear, it's real, it's yours, and it's actionable. Then the pipeline gives that signal teeth:
Artifactory is the only front door, Xray watches everything that passes through it, and your
cluster ships only what promotion approved. Developers focus on shipping code; the platform
does the patching argument for them.

## Table of contents

- [Lab 0 — Get your seat](#lab-0--get-your-seat)
- [Lab 1 — One front door (~10 min)](#lab-1--one-front-door-10-min)
- [Lab 2 — Measure the noise (~15 min)](#lab-2--measure-the-noise-15-min)
- [Lab 3 — Development happens (~15 min)](#lab-3--development-happens-15-min)
- [Lab 4 — Give it teeth (~20 min)](#lab-4--give-it-teeth-20-min)
- [Lab 5 — Ship from the governed endpoint (~15 min)](#lab-5--ship-from-the-governed-endpoint-15-min)

---

## Lab 0 — Get your seat

> [!IMPORTANT]
> 🎫 **Have your table card handy — you have TWO sets of credentials today**, same username,
> different passwords: **🖥 LAB PORTAL** (the terminal in your browser) and **📦 JFROG**
> (the registry UI). You will need both, repeatedly. When a login fails, the first question
> is always: *which system am I talking to, and which password did I just use?*

1. Open the portal — **the URL is in your welcome email** (and on your table card at in-person events).
2. Log in with the credentials on your table card (`lab01`…`labNN`). Click the one tile you
   see: **Your Lab Server (labNN)** — a terminal opens in the browser.
3. Your card also carries your **JFrog** URL, login and **project key** (= your seat name,
   e.g. `lab01`) — open it in a second browser tab. You are the **Project Admin** of that
   project: pick it in the **project picker, top-left** of the JFrog UI, and its repos, Xray
   watches and policies are all yours today.

> [!NOTE]
> **New in this lab: one JFrog Project per seat.** Earlier runs of this lab put every seat in one
> flat namespace — everyone pushed to the same `dev-docker`, and two people creating a watch named
> `block-high-watch` collided. Now each seat is its own **JFrog Project**, the same boundary a real
> organisation draws around a team:
>
> | Yours (inside project `labNN`) | Shared by the whole room |
> |---|---|
> | `labNN-dev-docker` — your dev repo, where every push in this lab lands | `cgr` — the one warm remote to `cgr.dev` everyone pulls through |
> | `labNN-block-high` policy + `labNN-block-high-watch` — your Xray gate | `prod-docker` — the governed production repo: you can **promote into it** (to your own tags, `v2-labNN`) but never overwrite or delete |
> | your storage quota, your members, your project settings | the cluster's read-only pull credential (Lab 5) |
>
> You are **Project Admin**, not an instance admin: inside your walls you can do anything; outside
> them you have exactly the two grants above. Nothing another seat does can rename, watch, or
> block your images — and the names in your project are yours alone, so `lab07-block-high` and
> `lab12-block-high` never fight. Everywhere the guide says `$JF_DEV` or `$JF_PROJECT`, it means
> *your* repo and *your* project.
4. Clipboard: **Ctrl+Alt+Shift** opens the side menu → paste into its clipboard box.
5. Your seat already knows where it lives — three variables are exported in every shell:

   ```bash
   echo $JF_HOST $JF_PROJECT $JF_DEV
   ```

   Something like `jfrog.<zone>  lab01  lab01-dev-docker`: the platform host, your project
   key, and your project's dev repo (`$JF_PROJECT-dev-docker`). Every command below uses
   them. `JF_HOST` and `JF_DEV` must not be empty — if they are, open a fresh terminal
   before going on. `JF_PROJECT` may be empty on a **legacy (flat) seat** — your facilitator
   will tell you; in that case there is no project picker for you, and you set the two
   variables by hand once per terminal, with your seat name as the project:

   ```bash
   export JF_DEV=${JF_DEV:-dev-docker} JF_PROJECT=${JF_PROJECT:-lab01}   # legacy seats only — use YOUR seat name
   ```

Your seat runs Docker and a k3s cluster with OctoShop's proxy and database already up —
hardened images, naturally. Sanity check:

```bash
kubectl get pods -n octoshop && curl -s -o /dev/null -w '%{http_code}\n' localhost:30080
```

Two pods (`proxy`, `db`) and a **502**. Not broken — honest: the web tier doesn't exist yet.
**The shelf is empty until you ship something through the pipeline.** That's Lab 5.

---

## Lab 1 — One front door (~10 min)

Your project is already wired into the seat (`$JF_HOST`, `$JF_PROJECT` and `$JF_DEV` are set
in your shell; Docker is logged in). Confirm the front door works by pulling a Chainguard image
**through** the platform's `cgr` remote repo instead of straight from the internet:

```bash
docker login $JF_HOST
docker pull $JF_HOST/cgr/chainguard/python:latest
docker images | grep $JF_HOST
```

The `cgr` repo is a *remote* repo proxying `cgr.dev`: one URL for every Dockerfile in the
org, cached at the edge, and — the part that matters — **everything that passes through it is
visible to Xray**. In the JFrog UI: Artifactory → Artifacts → `cgr` — there's your pull,
already indexed. (`cgr` is a shared, platform-wide repo — if it's missing from the tree, set the
top-left project picker to *All*.)

---

## Lab 2 — Measure the noise (~15 min)

Same app, two supply chains. OctoShop's web source is in `~/octoshop/web` with both
Dockerfiles from the CVE Zero lab. **Read them first** — the entire lab is in the diff:

```bash
cd ~/octoshop/web
cat Dockerfile
cat Dockerfile.cg
```

Three differences to notice before you build: the base image (`python:3.12` hauls all of
Debian; `cgr.dev/chainguard/python` is distroless), one stage vs two (the `.cg` build's
toolbox never ships), and the `.cg` runtime deliberately uninstalls pip/setuptools — build
tools do the building, then get out of the container. Now build and push both through your
front door (builds are cached on your seat — seconds each):

```bash
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v1 .
docker push $JF_HOST/$JF_DEV/octoshop-web:v1
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v2 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v2
```

Now the money shot. In the JFrog UI, **with your project selected in the top-left picker**:
**Xray → Scans List** (or Artifactory → your `$JF_DEV` repo → select the artifact → Xray tab).
Scans List is filtered by that picker — with no project selected you'll see nothing. Give
indexing a minute, then put v1 and v2 side by side:

- `octoshop-web:v1` (FROM `python:3.12`): **~620 findings** (illustrative)
- `octoshop-web:v2` (Chainguard, same `app.py`): **~1 finding**

Ask the honest question: *which of these dashboards would your team actually work?* 620
findings isn't security visibility — it's a queue nobody will ever drain. Write both numbers
on the scorecard at the bottom.

---

## Lab 3 — Development happens (~15 min)

Starting at zero doesn't mean staying at zero — it means every finding from now on is **real
and yours**. Prove it. A developer on your team is shipping a feature and pins Flask back to
an ancient version, because — direct quote — *"that's the only one I've tested my code on."*
Sound familiar? Play the developer:

```bash
sed -i 's/^flask.*/flask==2.0.1/' requirements.txt
cat requirements.txt
```

There it is: one line, a three-year-old Flask, and a perfectly sincere reason. Nobody
attacked anything — this is how vulnerabilities actually enter production. Build and ship
it like any other change (the pin is cached on your seat, so the rebuild is instant and
offline):

```bash
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v3 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v3
```

Back to Xray: v3 shows **one new finding** — the known Flask CVE you just introduced
(illustratively CVE-2023-30861, High). On the v1 dashboard this would have been invisible:
finding number 621, buried on page 13. On a quiet baseline it's the *only* thing on screen —
attributable to a specific change, by a specific person, today. That's the difference between
noise and signal, and it's the entire case for pairing a clean start (Chainguard) with
continuous watching (Xray).

### Lab 3B — The poison you didn't choose (~10 min)

v3 was a vulnerability you *chose* — a bad pin in code you own. Images also rot a second
way: **build tooling that ships without anyone choosing it**. Your Dockerfile guards
against this with one line; see what happens without it. Restore the good Flask first,
then temporarily remove the guard:

```bash
sed -i 's/^flask.*/flask>=3.0/' requirements.txt
sed -i 's| && \\$||; /pip uninstall -y pip setuptools/d' Dockerfile.cg
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v5 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v5
```

In Xray: v5 carries **~4 High findings your app never asked for** — open one and read the
impact path: every one threads through `site-packages/pip/`. `python -m venv` seeds pip and
setuptools into the venv; pip vendors its own private copies of setuptools and msgpack and
ships an SBOM confessing to it; the scanner reads the confession. Clean requirements, dirty
image — **the vulnerability came from the packaging default, not the code.**

Now remediate — not by patching, by *subtraction*. Put the guard back (this is the exact
line your Dockerfile normally carries):

```bash
sed -i 's|RUN /home/nonroot/venv/bin/pip install -r requirements.txt|RUN /home/nonroot/venv/bin/pip install -r requirements.txt \&\& \\\n    /home/nonroot/venv/bin/pip uninstall -y pip setuptools|' Dockerfile.cg
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v5-fixed -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v5-fixed
```

v5 and v5-fixed side by side in Scans List: findings to zero, and the diff is a line that
*removes* software. Nobody waited for upstream patches — the components that carried the
CVEs simply aren't aboard anymore. **The fastest patch is the component you don't ship** —
the distroless argument, executed by your own hands, one layer up the stack. Between v3 and
v5 you've now seen both ways images rot: the dependency you chose badly, and the freight
you never chose at all.

---

## Lab 4 — Give it teeth (~20 min)

> [!NOTE]
> **Verified live (2026-08-20, self-hosted E+):** the block fires on the **pull** (HTTP 403
> from Xray), never on `docker-promote` — promotion is a server-side copy and flows
> untouched. Gating promotion itself is a Release Bundles / RLM capability (v1.1 candidate).

A dashboard informs; a **policy** enforces. Your project came with both pieces already built
**inside its walls** — today you inspect them, arm them, and watch them bite. In the JFrog UI,
**select your project in the top-left project picker** (`$JF_PROJECT`), then:

1. **Xray → Watches & Policies → Policies**: open `$JF_PROJECT-block-high` (`lab01-block-high`
   for seat 01 — every project's objects carry its key). Type Security, one rule:
   minimal severity **High** → action **Block Download**. Read it — this is the whole gate,
   and it is *yours*: a Project Admin can loosen it, tighten it, or add a second rule.
2. **Watches**: open `$JF_PROJECT-block-high-watch`. Resource: your `$JF_DEV` repo
   (`lab01-dev-docker` for seat 01); assigned policy: `$JF_PROJECT-block-high`. That is the
   governed door for everything you push. The shared `prod-docker` sits *outside* your
   project and is governed a different way — the cluster's read-only pull credential, which
   is Lab 5's story.

> [!NOTE]
> **Legacy (flat) seat?** Your picker has no projects and nothing is pre-seeded. Build the two
> pieces yourself as *global* objects — policy `block-high` (Security, min severity High →
> Block Download) and watch `block-high-watch` on repository `dev-docker` with that policy —
> then carry on from step 3 with those names.

> [!WARNING]
> **The UI creates watches and policies in whatever project context is selected** — silently.
> With the picker on *All* (or on the wrong project) a "new watch" lands somewhere your repo
> can't see, never fires, and the 403 below never comes. The facilitator lost an hour to this
> exact trap. Before you touch anything under Xray, look at the top-left picker: it must say
> `$JF_PROJECT`. If you don't see `$JF_PROJECT-block-high-watch` at all, that's the tell — fix
> the picker, don't create a second one.

> [!NOTE]
> **Why the clean image survives a High-severity gate:** its Dockerfile uninstalls
> pip/setuptools from the venv after the install. Build tools in a runtime image don't just
> add bloat — pip *vendors* its own dependency tree (setuptools, msgpack…), and at T2 every
> High finding on this image was pip's freight, not the app's. Ship the app, not the
> package manager. (Facilitator note: if the CVE feed ever tags the golden again, the play
> is a scoped, **expiring Ignore Rule** with written justification — the POA&M workflow,
> not a weakened gate.)

3. **Apply it to what's already there**: watches judge scan events, and the CVE feed keeps
   moving after a scan lands — so make the policy pass judgement on everything already in
   your repo. In the watch list (project `$JF_PROJECT`): row menu (⋮) on
   `$JF_PROJECT-block-high-watch` → **Apply on Existing Content** → pick a range covering
   today → run. (Menu greyed out? Your project was created without the *Manage Security
   Assets* privilege — wave the facilitator over; they run this one step for you and the
   gate below still fires.) This back-fills violations for everything already in `$JF_DEV` —
   and enforcement trails it by a few minutes, so **verify the gate is armed before the demo**:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -u <you>:<your-jfrog-password> https://$JF_HOST/v2/$JF_DEV/octoshop-web/manifests/v3
   ```

   Repeat until it prints **403** (a 200 means the back-fill is still running — a pull now
   would slip through the closing door and quietly succeed).

Now promote the clean build to production (the CLI is pre-configured as `lab`). `prod-docker`
is shared by the whole room and you can deploy into it but not overwrite — so every seat
promotes to **its own tags**, suffixed with your project key:

```bash
jf rt docker-promote octoshop-web $JF_DEV prod-docker --copy --source-tag v2 --target-tag v2-$JF_PROJECT
docker pull $JF_HOST/prod-docker/octoshop-web:v2-$JF_PROJECT
```

The clean build flows. Now try the poisoned one:

```bash
jf rt docker-promote octoshop-web $JF_DEV prod-docker --copy --source-tag v3 --target-tag v3-$JF_PROJECT
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null
docker system prune -af
docker pull $JF_HOST/$JF_DEV/octoshop-web:v3
```

Read the refusal out loud — docker relays Xray naming the policy, the repo, and the artifact
(the repo is *your* project's, e.g. `lab01-dev-docker`):

```
Error response from daemon: ... "DENIED" ... octoshop-web/v3/manifest.json was not
downloaded due to the download blocking policy configured in Xray for lab01-dev-docker.
```

> [!NOTE]
> **Why the wipe first?** You BUILT v3 on this machine. A builder's own daemon holds the
> layers, the config, and digest bookkeeping (which survives plain `rmi`), so it can
> re-materialize the image from metadata checks (always allowed) plus content-addressed
> blobs (shared with clean images, unblockable by design) — without ever issuing the one
> request the policy refuses: the manifest download. The gate governs **distribution**,
> not possession. Wiping makes your daemon an honest consumer again — the same position
> every other machine is in, including the k8s cluster in Lab 5. (Your docker work is done
> by this point; anything you rebuild later re-pulls through your `cgr` remote.) Defense
> in depth exists precisely because no single gate covers the machine that already has
> the goods — that's the admission-controller's job, one layer down.

> [!TIP]
> 🧹 Cache acting suspicious at any point (pulls "succeeding" that shouldn't, stale tags)?
> `/opt/lab/docker-clean.sh` resets your docker to an honest consumer in one command.

**Expect the Xray 403** — the platform refuses to serve an artifact that violates the policy
you just armed — a policy that lives in *your* project, that you own. No memo, no training: the
unsafe path simply stops working. (Sound familiar?
It's the admission-controller moment from the advanced lab, one layer earlier in the pipeline.)

---

## Lab 5 — Ship from the governed endpoint (~15 min)

*This lab works even if you skipped 2–4: a known-good build is already in the shared
`prod-docker`, pushed at seed time.*

Your cluster's pull secret can read **`prod-docker` and nothing else**. Ship the web tier:

```bash
kubectl apply -f ~/octoshop/k8s/
kubectl rollout status deploy/web -n octoshop
curl -s localhost:30080
```

```
OctoShop up · 3 items in catalog · served via proxy→web→db
```

The 502 is gone — the app is serving, and the image came from exactly one place: the repo
your policy governs. Now try the ungoverned path:

```bash
kubectl set image deploy/web web=$JF_HOST/$JF_DEV/octoshop-web:v2 -n octoshop
kubectl get pods -n octoshop
```

`ErrImagePull` / `ImagePullBackOff` — same image bits, wrong door. The cluster's credentials
open the prod repo only, so *nothing* reaches production runtime without passing your gate.
Restore the governed ref:

```bash
kubectl set image deploy/web web=$JF_HOST/prod-docker/octoshop-web:v2 -n octoshop
```

> [!NOTE]
> **☸ The permission trick, spelled out:** the deny you just saw isn't Kubernetes policy —
> it's a JFrog permission. The `regcred` secret belongs to a `k8s-puller` user granted read on
> `prod-docker` only — and it isn't a member of your project, so your `$JF_DEV` repo is
> invisible to it. One permission target turns "please only deploy approved images" from a
> memo into a mechanical fact. Pair it with blueprint #3's admission controller and you have
> belt *and* suspenders.

## Your signal-to-noise scorecard

| Image | Base | Xray findings | How many are YOURS | Would you action this list? |
|---|---|---|---|---|
| `octoshop-web:v1` | `python:3.12` | | 0 | |
| `octoshop-web:v2` | Chainguard | | 0 | |
| `octoshop-web:v3` | Chainguard + your pin | | 1 | |

Keep this. The v3 row is the argument: a quiet baseline is what makes your scanner's output an
action list instead of an archive.

## Next steps

1. **Wire one real repository through an Artifactory remote to `cgr.dev`** — one URL change,
   and everything your team pulls becomes visible and governable.
2. **Turn on one blocking policy in a staging repo** — start with Critical-only if High feels
   bold; the point is that *some* gate exists and machines enforce it.
3. **Bring your promotion flow to a working session** — J2R maps the dev→prod repo model and
   the prod-only pull credential onto your clusters with your team.

Jonathan Spigler · J2R Solutions · jonathan@j2rsolutions.io · (717) 491-2827

*Seat misbehaving? `sudo /opt/lab/bootstrap.sh` rebuilds the cluster side in ~90 s. Project
misbehaving? You're its admin — and the facilitator can re-seed it in seconds, like a spare seat.*
