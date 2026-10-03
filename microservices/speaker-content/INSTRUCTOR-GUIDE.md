# Instructor Guide
### System Design for Modern Microservices · 4 hours · 3 October 2026

> ⚠️ **Instructor material — do not hand this folder to students.**
> Everything in `speaker-content/` is ignored by `.gitignore`, so it will not reach the student
> repository: this guide and `Slides.md`, the presenter script with speaker notes.
>
> Share the slides afterwards if you want to. The "Predict first" answers and the quiz key now
> live in Part 4 of the student `Workshop-Guide.md`.

---

## The spine

One application, four lenses. No new app is introduced at any point.

```
monolith → same thing as 4 services → open the container → run it on Kubernetes
```

Everything is plain Node with zero dependencies, so `docker compose up --build` is ~90 seconds
and nothing can fail on an npm install.

**Nobody needs a Docker Hub account.** Images stay local and are handed to the cluster with
`minikube image load`. This removes the single biggest source of first-workshop failures.

---

## Timing

| Time | Segment | Hands-on |
|---|---|---|
| 9:30 – 9:40 | Welcome, outcomes, **start Codespaces immediately** | Lab 0 |
| 9:40 – 10:05 | Monolith vs microservices; the trade; when *not* to | — |
| 10:05 – 10:15 | Run the monolith | Lab 1A |
| 10:15 – 10:26 | Boundaries exercise, in pairs, **on paper** | Lab 1B |
| 10:26 – 10:30 | Two pairs present · **Quiz 1** | — |
| 10:30 – 10:50 | API Gateway · service discovery · sync vs async | — |
| 10:50 – 11:02 | Start four services; prove name resolution | Lab 2A |
| 11:02 – 11:22 | **Break things on purpose** | Lab 2B |
| 11:22 – 11:26 | Two pairs report what degraded and what did not | — |
| 11:26 – 11:30 | **Quiz 2** | — |
| 11:30 – 11:45 | Docker: image, container, Dockerfile | — |
| 11:45 – 12:03 | Read the Dockerfile, **build and run a container by hand** | Lab 3A |
| 12:03 – 12:13 | Kill a container, nothing returns → why orchestration. **`minikube start` runs here** | Lab 3B |
| 12:13 – 12:27 | Pod / Deployment / Service; read the manifest | Lab 3C |
| 12:27 – 12:30 | **Quiz 3** | — |
| 12:30 – 12:40 | Load the four images, deploy, reach it | **Lab 4A** |
| 12:40 – 12:48 | Delete a pod, watch it return | **Lab 4B** |
| 12:48 – 12:58 | Scale catalog only — the payoff for the whole day | **Lab 4C** |
| 12:58 – 13:12 | Rolling update and rollback (needs a rebuild, so allow the time) | **Lab 4D** |
| 13:12 – 13:24 | Security, the operational bill, case studies, when it goes wrong | — |
| 13:24 – 13:30 | **Quiz 4**, assignment brief, close | — |

**Session 4 is the tightest hour of the day** — 42 minutes of lab in a 60-minute slot. If you are
behind when you reach 12:58, **skip Lab 4D** and demonstrate the rolling update on the projector
instead; it is the one lab whose lesson survives being watched rather than done.

**There is no scheduled break**, because the college's slot is exactly four hours. If the room
needs one, take ten minutes at 11:20 and pay for it by cutting Lab 4D and the case studies.

**Running late?** Cut in this order: Lab 4D (rolling update) → the `docker run` half of Lab 3A →
the case studies. **Never cut Lab 2B.** It is the reason the workshop exists.

**Compose is deliberately not a topic.** It is one command that starts four containers and
builds the four images Session 4 hands to Kubernetes. If a student asks why we are not teaching
it properly: local dev convenience, one machine, no reconciliation — the moment you want
replacement, scaling or rolling updates you are asking for an orchestrator. Do not spend Session
3 on Compose syntax; spend it on `docker build`, `docker run` and the Kubernetes objects.

---

## 🔴 Landmines

1. **Start Codespaces in the first five minutes.** Creation takes ~2 minutes and Lab 1A needs it.
2. **`minikube start` must begin at ~12:05**, during the Lab 3B discussion. It takes ~2 minutes
   and Session 4 is tight. Do not leave it until 12:30.
3. **Run `docker compose down` before `minikube start`.** A 2-core Codespace running four
   containers *and* minikube will be short of memory. If minikube fails: `minikube delete`,
   `docker compose down`, retry.
4. **All four `minikube image load` commands are required.** Miss one and that pod sits in
   `ErrImageNeverPull`. It is the most likely Session 4 failure — put the four commands on the
   projector.
5. **`curl localhost:30080` will not work** even though the Service is a NodePort. minikube runs
   on its own Docker network. Use `kubectl port-forward`, as the handout does.
6. **The recommendations page refreshes every 5 seconds.** Students who expect an instant change
   after `/slow` will think it failed. Tell them to wait.
7. **Ports tab, not localhost.** In Codespaces, opening `localhost:8080` in their own browser does
   nothing. They must use the **Ports** tab and the globe icon. Expect to say this three times.
8. **"All four Up but catalog unreachable."** Docker-in-Docker can leave the legacy iptables
   `FORWARD` policy at `DROP`, which silently blocks traffic between containers on the Compose
   network. `.devcontainer` now clears this on start. If a student still hits it,
   `sudo iptables-legacy -P FORWARD ACCEPT` fixes it immediately.
9. **`/slow` persists until that container restarts.** If you rehearse Lab 2B and go back, reset
   it with the `/ok` command or the room will see timeouts you did not intend.
9. **`/slow` survives a page refresh.** The mode is held in the recommendations process, so it
   persists until that container restarts. If you rehearse Lab 2B and jump back, reset it with
   the `/ok` command or the room will see timeouts you did not intend.

---

## Teaching notes

- **Lab 2B is the workshop.** Everything before it is setup and everything after is consequence.
  Do it on the projector at the same time as the room. When the recommendations panel goes yellow
  and someone still successfully buys a book, pause and let it land: *"One service is completely
  broken. The shop is still taking orders. Nobody was paged."* Say plainly that the order is a
  simulation held in memory — the lesson is the failure boundary, not the checkout.
- **Then immediately stop `catalog`.** The contrast is the actual lesson — not "add fallbacks
  everywhere" but "**decide, per dependency, whether you can survive without it**." Say plainly
  that this decision is architecture and cannot be automated.
- **Be genuinely fair to the monolith in Session 1.** If the room leaves thinking monoliths are
  obsolete, the workshop has failed. The `monolith/server.js` comments are written to support
  this — read them aloud.
- **Lab 1B is deliberately on paper.** It buys Codespace-creation time, and boundary-drawing is
  better done by arguing in pairs. There is no single right answer; there are wrong ones
  (splitting by "controllers/models", or one service per database table).
- **Session 3 has an ordering advantage — use it.** They have been running containers for an hour
  without knowing what one is, and without typing a Docker command. Open with: *"You have been
  using Docker all morning. Let's open the box."* Lab 3A is the first time they build and run a
  container themselves — protect that time.
- **Lab 4C is the payoff for the entire day.** Five catalog pods, one orders pod. Say explicitly:
  *"This is the thing the monolith could not do. Everything you paid for this morning was to buy
  this."*
- **Close on Uber.** They grouped ~2,200 services into ~70 domains, each behind a gateway, so
  callers depend on one interface instead of twenty — not a return to a monolith, and not a merge
  into a handful of giant services. More services is not better. It leaves the room with
  judgement rather than enthusiasm.

---

## What was deliberately left out

| Topic | Why |
|---|---|
| Databases per service | No time to run four databases; covered conceptually in Notes §17 |
| Message brokers (Kafka/Rabbit) | `orders` logs an event instead; the concept lands without the install |
| Circuit breakers in code | Explained; implementing one needs a library and 20 minutes |
| Service mesh (Istio/Linkerd) | Far beyond a first workshop. Name it, move on |
| CI/CD pipelines | A different workshop entirely |

If asked "why no database?" the honest answer is: four databases would consume the entire
hands-on budget and teach less than breaking a service does.

---

## Before you teach

- [ ] Create a fresh Codespace and run **the whole handout end to end**, timing each lab
- [ ] Confirm `docker compose up --build` from cold — note the real duration for your timings
- [ ] Confirm all four `minikube image load` commands, then `kubectl get pods` all Running
- [ ] Check memory: run compose **and** minikube together to see whether your Codespace copes
- [ ] Have the Ports tab demo ready — it is the most common student confusion
- [ ] Decide whether to read quiz answers aloud or project them
- [ ] Confirm `speaker-content/` is still ignored — `git status` should never list this folder

---

