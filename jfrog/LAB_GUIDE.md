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

> [!TIP]
> **How to read this guide.** Anything you have to *do* is marked — the rest is context.
>
> | Marker | Means |
> |---|---|
> | ▶ **DO:** | click, type or run something — a command block always follows a DO line |
> | ✅ **CHECK:** | what you should see before you move on |
> | ✍️ **RECORD:** | write it on your scorecard |

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

1. ▶ **DO:** Open the portal — **the URL is in your welcome email** (and on your table card at in-person events).
2. ▶ **DO:** Log in with the credentials on your table card (`lab01`…`labNN`).
3. **Your folder today: JFrog lab.** Sign in once. The portal shows one **folder per lab track**
   running today; your welcome email says which folder is which.

   ▶ **DO:** For this guide open the folder named **JFrog lab**. On a single-lab day there are no
   folders — the same two tiles sit at the portal root as **Your Lab Server (labNN)** and
   **Your Lab Server desktop (labNN)**.

   Inside your folder you get **two tiles** — both are your own seat:
   - **JFrog lab (labNN)** — a **terminal** in the browser tab. The lab happens here.
   - **JFrog lab desktop (labNN)** — a **desktop** on the *same* server (Firefox, a File
     Manager and a Terminal launcher; it's already logged in, no password to type). Today it's
     for the JFrog Platform UI — and OctoShop at `http://localhost:30080` in Lab 5.

   Other folders are **other servers**: files and images do not carry across. Everywhere below,
   **your desktop tile** means **JFrog lab desktop (labNN)** — or **Your Lab Server desktop (labNN)**
   on a single-lab day.

   ▶ **DO:** **Keep both open.** A tile normally opens in the same browser tab — **Ctrl-click** (or
   middle-click) a tile to open it in a **new browser tab**, so the terminal and the desktop sit
   side by side.

   Firefox on the desktop reaches anything listening on your seat
   (`http://localhost:<port>`), and the File Manager reaches every file in your home folder.
   Every desktop step below names a launcher *and* the URL or path to type into Firefox — if the
   launcher isn't on your desktop, type it; same result.
4. **Open the JFrog UI.** Your card — and the JFrog block of your welcome email — also carries
   your **JFrog** URL, login and **project key** (= your seat name, e.g. `lab01`).

   ▶ **DO:** Two places to open it — pick one and stay there:
   - **(a)** a second tab on your laptop, or
   - **(b)** **Firefox on your desktop tile** (**JFrog lab desktop (labNN)** — or **Your Lab Server
     desktop (labNN)** on a single-lab day) — double-click **JFrog Platform**, or type
     `https://<your JFrog host>` (it's `echo $JF_HOST` in the terminal).

   **Use (b) if your laptop's browser is locked down or can't reach the host** — the seat always
   can. Same JFROG password either way. You are the **Project Admin** of that project.

   ▶ **DO:** Pick it in the **project picker, top-left** of the JFrog UI, and its repos, Xray
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
5. Clipboard: **Ctrl+Alt+Shift** opens the side menu → paste into its clipboard box.
6. Your seat already knows where it lives — three variables are exported in every shell.

   ▶ **DO:** Print them:

   ```bash
   echo $JF_HOST $JF_PROJECT $JF_DEV
   ```

   Something like `<event>-jfrog.lab.aws.j2rsolutions.com  lab01  lab01-dev-docker`: the
   platform host, your project key, and your project's dev repo (`$JF_PROJECT-dev-docker`).
   Every command below uses them.

   ✅ **CHECK:** None of the three may be empty — if one is, open a fresh terminal and check
   again.

Your seat runs Docker and a k3s cluster with OctoShop's proxy and database already up —
hardened images, naturally.

▶ **DO:** Run this sanity check:

```bash
kubectl get pods -n octoshop && curl -s -o /dev/null -w '%{http_code}\n' localhost:30080
```

✅ **CHECK:** Two pods (`proxy`, `db`) and a **502**. Not broken — honest: the web tier doesn't exist yet.
**The shelf is empty until you ship something through the pipeline.** That's Lab 5.

---

## Lab 1 — One front door (~10 min)

Your project is already wired into the seat (`$JF_HOST`, `$JF_PROJECT` and `$JF_DEV` are set
in your shell; Docker is logged in).

▶ **DO:** Confirm the front door works by pulling a Chainguard image **through** the platform's
`cgr` remote repo instead of straight from the internet:

```bash
docker login $JF_HOST
docker pull $JF_HOST/cgr/chainguard/python:latest
docker images | grep $JF_HOST
```

The `cgr` repo is a *remote* repo proxying `cgr.dev`: one URL for every Dockerfile in the
org, cached at the edge, and — the part that matters — **everything that passes through it is
visible to Xray**. (Docker may answer *Image is up to date*: your seat already pulled every base
image through this same door when it was seeded, so the layers are here — the record in `cgr` is
the point.)

▶ **DO:** In the JFrog UI — your laptop tab, or Firefox on your desktop tile (**JFrog lab
desktop (labNN)** — or **Your Lab Server desktop (labNN)** on a single-lab day):
Artifactory → Artifacts → `cgr`.

✅ **CHECK:** There's your pull, already indexed. (`cgr` is a shared, platform-wide repo — if it's missing from the tree, set the
top-left project picker to *All*.)

---

## Lab 2 — Measure the noise (~15 min)

Same app, two supply chains. OctoShop's web source is in `~/octoshop/web` with both
Dockerfiles from the CVE Zero lab.

▶ **DO:** **Read them first** — the entire lab is in the diff:

```bash
cd ~/octoshop/web
cat Dockerfile
cat Dockerfile.cg
```

First the `FROM` lines: both bases come **through your platform** — `$JF_HOST/dockerhub/library/python:3.12`
and `$JF_HOST/cgr/chainguard/python:…` — the seat wrote your host into every Dockerfile at seed.
Nothing in this lab pulls from the internet directly: one front door, for the build as well as the pull.

Three differences to notice before you build: the base image (`python:3.12` hauls all of
Debian; `cgr.dev/chainguard/python` is distroless), one stage vs two (the `.cg` build's
toolbox never ships), and the `.cg` runtime deliberately uninstalls pip/setuptools — build
tools do the building, then get out of the container.

▶ **DO:** Now build and push both through your front door (builds are cached on your seat —
seconds each):

```bash
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v1 .
docker push $JF_HOST/$JF_DEV/octoshop-web:v1
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v2 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v2
```

Now the money shot.

▶ **DO:** In the JFrog UI (laptop tab or desktop-tile Firefox), **with your project
selected in the top-left picker**:
**Xray → Scans List** (or Artifactory → your `$JF_DEV` repo → select the artifact → Xray tab).

Scans List is filtered by that picker — with no project selected you'll see nothing.

✅ **CHECK:** Give indexing a minute, then put v1 and v2 side by side:

- `octoshop-web:v1` (FROM `dockerhub/library/python:3.12`): **~620 findings** (illustrative)
- `octoshop-web:v2` (Chainguard, same `app.py`): **~1 finding**

Ask the honest question: *which of these dashboards would your team actually work?* 620
findings isn't security visibility — it's a queue nobody will ever drain.

✍️ **RECORD:** Write both numbers on the scorecard at the bottom.

---

## Lab 3 — Development happens (~15 min)

Starting at zero doesn't mean staying at zero — it means every finding from now on is **real
and yours**. Prove it. A developer on your team is shipping a feature and pins Flask back to
an ancient version, because — direct quote — *"that's the only one I've tested my code on."*
Sound familiar?

▶ **DO:** Play the developer:

```bash
sed -i 's/^flask.*/flask==2.0.1/' requirements.txt
cat requirements.txt
```

There it is: one line, a three-year-old Flask, and a perfectly sincere reason. Nobody
attacked anything — this is how vulnerabilities actually enter production.

▶ **DO:** Build and ship it like any other change (the pin is cached on your seat, so the
rebuild is instant and offline):

```bash
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v3 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v3
```

✅ **CHECK:** Back to Xray: v3 shows **one new finding** — the known Flask CVE you just
introduced (illustratively CVE-2023-30861, High).

On the v1 dashboard this would have been invisible:
finding number 621, buried on page 13. On a quiet baseline it's the *only* thing on screen —
attributable to a specific change, by a specific person, today. That's the difference between
noise and signal, and it's the entire case for pairing a clean start (Chainguard) with
continuous watching (Xray).

### Lab 3B — The poison you didn't choose (~10 min)

v3 was a vulnerability you *chose* — a bad pin in code you own. Images also rot a second
way: **build tooling that ships without anyone choosing it**. Your Dockerfile guards
against this with one line; see what happens without it.

▶ **DO:** Restore the good Flask first, then temporarily remove the guard:

```bash
sed -i 's/^flask.*/flask>=3.0/' requirements.txt
sed -i 's| && \\$||; /pip uninstall -y pip setuptools/d' Dockerfile.cg
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v5 -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v5
```

✅ **CHECK:** In Xray: v5 carries **~4 High findings your app never asked for**.

▶ **DO:** Open one and read the impact path: every one threads through `site-packages/pip/`.

`python -m venv` seeds pip and
setuptools into the venv; pip vendors its own private copies of setuptools and msgpack and
ships an SBOM confessing to it; the scanner reads the confession. Clean requirements, dirty
image — **the vulnerability came from the packaging default, not the code.**

Now remediate — not by patching, by *subtraction*.

▶ **DO:** Put the guard back (this is the exact line your Dockerfile normally carries):

```bash
sed -i 's|RUN /home/nonroot/venv/bin/pip install -r requirements.txt|RUN /home/nonroot/venv/bin/pip install -r requirements.txt \&\& \\\n    /home/nonroot/venv/bin/pip uninstall -y pip setuptools|' Dockerfile.cg
docker build -t $JF_HOST/$JF_DEV/octoshop-web:v5-fixed -f Dockerfile.cg .
docker push $JF_HOST/$JF_DEV/octoshop-web:v5-fixed
```

✅ **CHECK:** v5 and v5-fixed side by side in Scans List: findings to zero, and the diff is a
line that *removes* software.

Nobody waited for upstream patches — the components that carried the
CVEs simply aren't aboard anymore. **The fastest patch is the component you don't ship** —
the distroless argument, executed by your own hands, one layer up the stack. Between v3 and
v5 you've now seen both ways images rot: the dependency you chose badly, and the freight
you never chose at all.

---

## Lab 4 — Give it teeth (~20 min)

A dashboard informs; a **policy** enforces. Your project came with both pieces already built
**inside its walls** — today you inspect them, arm them, and watch them bite.

▶ **DO:** In the JFrog UI, **select your project in the top-left project picker**
(`$JF_PROJECT`), then:

1. ▶ **DO:** **Xray → Watches & Policies → Policies**: open `$JF_PROJECT-block-high`
   (`lab01-block-high` for seat 01 — every project's objects carry its key).

   ✅ **CHECK:** Type Security, one rule:
   minimal severity **High** → action **Block Download**. Read it — this is the whole gate,
   and it is *yours*: a Project Admin can loosen it, tighten it, or add a second rule.
2. ▶ **DO:** **Watches**: open `$JF_PROJECT-block-high-watch`.

   ✅ **CHECK:** Resource: your `$JF_DEV` repo (`lab01-dev-docker` for seat 01); assigned
   policy: `$JF_PROJECT-block-high`.

   That is the
   governed door for everything you push. The shared `prod-docker` sits *outside* your
   project and is governed a different way — the cluster's read-only pull credential, which
   is Lab 5's story.

> [!WARNING]
> **The UI creates watches and policies in whatever project context is selected** — silently.
> With the picker on *All* (or on the wrong project) a "new watch" lands somewhere your repo
> can't see, never fires, and the 403 below never comes.

> [!NOTE]
> **Why the clean image survives a High-severity gate:** its Dockerfile uninstalls
> pip/setuptools from the venv after the install. Build tools in a runtime image don't just
> add bloat — pip *vendors* its own dependency tree (setuptools, msgpack…), and at T2 every
> High finding on this image was pip's freight, not the app's. Ship the app, not the
> package manager. (Facilitator note: if the CVE feed ever tags the golden again, the play
> is a scoped, **expiring Ignore Rule** with written justification — the POA&M workflow,
> not a weakened gate.)

3. **Apply it to what's already there.** Watches judge scan events, and the CVE feed keeps
   moving after a scan lands — so the policy has to pass judgement on everything already in
   your repo.

   ▶ **DO:** In the watch list (project `$JF_PROJECT`), open the row menu (⋮) on
   `$JF_PROJECT-block-high-watch` → **Apply on Existing Content** → pick a range covering
   today → **Run**.

   > Menu greyed out? Your project was created without the *Manage Security Assets* privilege —
   > wave the facilitator over; they run this one step for you and the gate below still fires.

   This back-fills violations for everything already in `$JF_DEV`, and enforcement trails it
   by a few minutes.

   ▶ **DO:** Check the gate is armed — run this, and repeat it every 30 s:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -u <you>:<your-jfrog-password> https://$JF_HOST/v2/$JF_DEV/octoshop-web/manifests/v3
   ```

   ✅ **CHECK:** it prints **403**. A 200 means the back-fill is still running — a pull now
   would slip through the closing door and quietly succeed. Artifactory also asks Xray "is this
   blocked?" on the **first** request for an artifact and **caches the answer for several
   minutes**, so an early 200 keeps coming back for a while even after the violations exist.
   Give it five minutes before you conclude the gate is broken.

Now promote the clean build to production (the CLI is pre-configured as `lab`). `prod-docker`
is shared by the whole room and you can deploy into it but not overwrite — so every seat
promotes to **its own tags**, suffixed with your project key.

▶ **DO:** Promote v2 and pull it back from production:

```bash
jf rt docker-promote octoshop-web $JF_DEV prod-docker --copy --source-tag v2 --target-tag v2-$JF_PROJECT
docker pull $JF_HOST/prod-docker/octoshop-web:v2-$JF_PROJECT
```

✅ **CHECK:** The clean build flows.

> [!NOTE]
> **FYI — why this step wipes Docker before the pull.** You *built* v3 on this machine, so your
> Docker daemon already holds everything it needs to rebuild the image locally:
>
> - **The layers and the config are already cached here.** Plain `docker rmi` removes the tag,
>   but the daemon keeps its digest bookkeeping and the content-addressed blobs.
> - **Those blobs aren't blocked.** They are shared with clean images, so the policy can't refuse
>   them. The metadata checks a pull makes are always allowed too.
> - **The one request the policy refuses is the manifest download.** A daemon that already has the
>   goods can skip it, so your pull would "succeed" and the gate would look broken.
>
> Wiping containers, images and the build cache turns your laptop-equivalent into an honest
> consumer — the same position every other machine is in, including the k8s cluster in Lab 5. The
> gate governs **distribution**, not possession; stopping a machine that already has the image is
> the admission controller's job, one layer down. Your Docker work is done by this point, and
> anything you rebuild later re-pulls through your `cgr` remote.

▶ **DO:** Now try the poisoned one:

```bash
jf rt docker-promote octoshop-web $JF_DEV prod-docker --copy --source-tag v3 --target-tag v3-$JF_PROJECT
docker rm -f $(docker ps -aq) 2>/dev/null; docker rmi -f $(docker images -aq) 2>/dev/null
docker system prune -af
docker pull $JF_HOST/$JF_DEV/octoshop-web:v3
```

✅ **CHECK:** Read the refusal out loud — docker relays Xray naming the policy, the repo, and the artifact
(the repo is *your* project's, e.g. `lab01-dev-docker`):

```
Error response from daemon: ... "DENIED" ... octoshop-web/v3/manifest.json was not
downloaded due to the download blocking policy configured in Xray for lab01-dev-docker.
```

> [!TIP]
> 🧹 Cache acting suspicious at any point (pulls "succeeding" that shouldn't, stale tags)?
>
> ▶ **DO (optional):** `/opt/lab/docker-clean.sh` resets your docker to an honest consumer in one command.

✅ **CHECK:** **Expect the Xray 403** — the platform refuses to serve an artifact that violates the policy
you just armed — a policy that lives in *your* project, that you own. No memo, no training: the
unsafe path simply stops working. (Sound familiar?
It's the admission-controller moment from the advanced lab, one layer earlier in the pipeline.)

---

## Lab 5 — Ship from the governed endpoint (~15 min)

*This lab works even if you skipped 2–4: a known-good build is already in the shared
`prod-docker`, pushed at seed time.*

> [!NOTE]
> `prod-docker` also holds the database, proxy and cluster system images. They came through the
> same governed door, so expect a busier repo view.

Your cluster's pull secret can read **`prod-docker` and nothing else**.

▶ **DO:** Ship the web tier:

```bash
kubectl apply -f ~/octoshop/k8s/
kubectl rollout status deploy/web -n octoshop
curl -s localhost:30080
```

✅ **CHECK:** The last command prints:

```
OctoShop up · 3 items in catalog · served via proxy→web→db
```

> [!TIP]
> ▶ **DO:** **On your desktop** (**JFrog lab desktop (labNN)** — or **Your Lab Server desktop (labNN)** on a
> single-lab day): double-click **OctoShop** — or open **Firefox** at `http://localhost:30080`.
>
> ✅ **CHECK:** The 502 is gone in a real browser too.
>
> Keep the tab: after the `ErrImagePull` swap below,
> refresh it — **it still serves**, because Kubernetes never took the working pod down for an
> image it couldn't pull.

✅ **CHECK:** The 502 is gone — the app is serving, and the image came from exactly one place:
the repo your policy governs.

▶ **DO:** Now try the ungoverned path:

```bash
kubectl set image deploy/web web=$JF_HOST/$JF_DEV/octoshop-web:v2 -n octoshop
kubectl get pods -n octoshop
```

✅ **CHECK:** `ErrImagePull` / `ImagePullBackOff` — same image bits, wrong door. The cluster's
credentials open the prod repo only, so *nothing* reaches production runtime without passing
your gate.

▶ **DO:** Restore the governed ref:

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
