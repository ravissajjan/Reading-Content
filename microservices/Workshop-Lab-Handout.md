# Workshop Lab Handout
### System Design for Modern Microservices — Principles, Patterns & Practices

---

## What you will be able to do by the end

1. Explain **when microservices help and when they hurt**, with supporting reasons.
2. Draw sensible **service boundaries** for a system you have never seen.
3. Name and recognise the **API Gateway**, **service discovery** and **graceful degradation** patterns in running code.
4. Read a **Dockerfile** and explain what an image and a container are.
5. Deploy a four-service application to **Kubernetes**, scale one service, and roll out a new version.

---

## LAB 0 — Set up (10 min)

1. On this repository: **Code** → **Codespaces** → **Create codespace on main**.
2. Wait for VS Code to open in your browser. In the terminal at the bottom:

```bash
cd lab-starter
node --version
docker --version
```

- [ ] Both print a version number

> 📁 **Work from `lab-starter`.** Every path in this handout is relative to it.
> To return there, run `cd /workspaces/*/lab-starter`.

> **If the terminal is not visible:** menu ☰ → *Terminal* → *New Terminal*.

**✅ CHECKPOINT 0** — everyone must have a working Codespace before continuing.

---

# SESSION 1 — Introduction to Microservices

## LAB 1A — Meet the monolith (10 min)

```bash
node monolith/server.js
```

Open port **8080**: click the **Ports** tab at the bottom, find 8080, click the 🌐 globe icon.

- [ ] I can see the BookStore page
- [ ] I read the "Think about it" panel at the bottom

Stop it with **Ctrl+C**.

> 📖 **DEFINITION — Monolith**: one codebase, one process, one deployable unit. Every feature
> ships together, scales together and fails together.

This file is simple and fast, and it makes no network calls. For a small team and a small
product, a monolith is often the correct choice.

---

## LAB 1B — Find the boundaries (11 min)

Work in pairs. Do not use a computer for this task.

A college wants one system for:

> student registration · course catalogue · fee payment · attendance tracking ·
> exam results · library book lending · hostel room allocation · alumni newsletter

**Your task:**

1. Group those eight things into **3–5 services**. Give each a name.
2. For each service write one line: *"This service owns ______."*
3. Pick **two** services and describe one piece of information they would need from each other.
4. Find one thing in the list that you would **leave out** of the microservices entirely.

- [ ] We agreed on our services
- [ ] We can justify why fee payment and attendance are (or are not) together

> 🎯 **Guideline:** if two things always change together and always deploy together,
> they probably belong in the **same** service. Splitting them adds network calls inside a
> single responsibility.

> 📖 **DEFINITION — Service boundary**: the line around one business capability that one team can
> own, change and deploy without asking anybody else.

**✅ CHECKPOINT 1** — two pairs present their grouping. There is no single correct answer, but
some groupings are clearly wrong.

➡️ **Now take QUIZ 1** in Part 3 of `Workshop-Guide.md` (5 questions).

---

# SESSION 2 — Architectural Patterns

## LAB 2A — Run it as four services (12 min)

One command starts all four services. Docker Compose is used here only to start containers;
Session 3 explains images and containers in detail.

```bash
docker compose up --build -d
docker compose ps
```

The first run takes about 90 seconds. `-d` starts the containers in the background.

- [ ] Four services are Up
- [ ] **Only `gateway` has a port** in the PORTS column

The containers take a moment to start listening after Compose returns. Check that the gateway
can reach catalog before you open the page:

```bash
curl -s localhost:8080/api/books | head -c 80
```

- [ ] I see book titles, not `Catalog is unavailable`

If you see the error, wait a few seconds and run the command again.

Open port **8080** in the **Ports** tab.

- [ ] The page looks almost identical to the monolith
- [ ] The Catalog panel says "from" a container ID
- [ ] I clicked **Buy** on a book and got an order number

> ⚠️ **The order is simulated.** No payment is processed, and `orders` stores it in memory only.
> Restarting that service discards it. Session 4 returns to this point.

> 📖 **DEFINITION — API Gateway**: one public entry point in front of many private services.
> Your browser talks only to the gateway. It cannot reach catalog, orders or recommendations
> at all — they have no published port.

### How the gateway finds the other services

One line in `docker-compose.yml` configures this:

```yaml
      CATALOG_URL: http://catalog:3001
```

`catalog` is a **name**, not an IP address. Prove it resolves, from inside the gateway container:

```bash
docker compose exec gateway wget -qO- http://catalog:3001/books
```

- [ ] I got JSON back, from one container talking to another **by name**

> 📖 **DEFINITION — Service discovery**: finding a service by **name** instead of by address.
> Containers are created and destroyed constantly, so their IP addresses change. Names do not.
> Kubernetes resolves the very same name in Session 4, with no code change.

---

## LAB 2B 🔥 — Break it on purpose (20 min)

**This lab shows how the system behaves when individual services fail.**

### Step 1 — make recommendations slow

```bash
docker compose exec recommendations wget -qO- http://localhost:3003/slow
```

Go back to the browser tab and wait up to 5 seconds (the page re-checks automatically).

- [ ] The **Recommended for you** panel turned yellow and says it is unavailable
- [ ] **The book list still works**
- [ ] **I can still buy a book**

> 📌 **This is graceful degradation.** One service has failed. The site still works and still
> accepts orders.

### Step 2 — see why

Open `gateway/server.js` and find the `/api/recommendations` block. Two settings produce this
behaviour:

```js
signal: AbortSignal.timeout(RECOMMENDATION_TIMEOUT_MS)   // 1. TIMEOUT: give up after 1 second
...
return json(200, { picks: [], degraded: true });          // 2. FALLBACK: answer anyway, with less
```

- [ ] I found the timeout and the fallback

> 📖 **DEFINITION — Timeout**: the longest you are willing to wait before giving up.
> 📖 **DEFINITION — Fallback**: the reduced answer you give when the real one is unavailable.
>
> Without a timeout, one slow service makes *every* page slow, then the gateway runs out of
> connections, and the whole site dies. That is a **cascading failure** — one service taking
> down a system that was supposed to survive it.

### Step 3 — stop the service completely

```bash
docker compose stop recommendations
```

- [ ] The panel still degrades gracefully. The shop is still open.

### Step 4 — stop an essential service

> 🎯 **Predict first:** what happens if I stop `catalog`? ______

```bash
docker compose stop catalog
```

**Refresh the page.** Only the recommendations panel re-checks itself every 5 seconds; the book
list is fetched once, when the page loads.

- [ ] After refreshing, the page went **red**. There is no shop without a catalog.

> 📌 **Design decision:** not every dependency needs a fallback. Decide, for each dependency,
> whether the system can operate without it. `recommendations` → yes. `catalog` → no.
> This is an architectural decision, not an implementation detail.

### Step 5 — bring it back

```bash
docker compose start catalog recommendations
docker compose exec recommendations wget -qO- http://localhost:3003/ok
```

The recommendations panel clears itself within 5 seconds. **Refresh the page** to bring the
book list back.

- [ ] Everything recovered without rebuilding or redeploying the gateway

**✅ CHECKPOINT 2**

➡️ **Now take QUIZ 2** (5 questions).

---

# SESSION 3 — Containerization & Orchestration

## LAB 3A — Build and run a container yourself (18 min)

Until now Compose has run the Docker commands for you. In this lab you run them directly.

```bash
docker images
```

- [ ] I can see the four `bookstore-*` images

Open `catalog/Dockerfile`. It is five lines:

```dockerfile
FROM node:22-alpine     # start from a small Linux that already has Node
WORKDIR /app            # work in this folder inside the image
COPY server.js ./       # copy my code in
EXPOSE 3001             # this service listens on 3001
CMD ["node", "server.js"]   # run this when the container starts
```

- [ ] I compared it with `orders/Dockerfile` — only the port differs

> 📖 **DEFINITION — Image**: a sealed, read-only package containing your app and everything it
> needs to run. Built once.
> 📖 **DEFINITION — Container**: a running copy of an image. One image → many containers.
> 📖 **DEFINITION — Dockerfile**: the recipe used to build the image.

```
Dockerfile  --build-->  Image  --run-->  Container
```

### Build the image

```bash
docker build -t bookstore-catalog:v1 ./catalog
```

- [ ] The build finished quickly and most steps reported `CACHED`

> The source has not changed since Compose built this image, so Docker reused every layer.
> Changing one line of `catalog/server.js` would rebuild the `COPY` step and every step after it.

### Run the container directly

```bash
docker run -d --name catalog-solo -p 3001:3001 bookstore-catalog:v1
docker ps
```

- [ ] **Two** catalog containers are running, from **one** image

Catalog answers on `/books` and `/health` only, so ask for `/books` directly:

```bash
curl -s localhost:3001/books
```

- [ ] I can see raw JSON straight from catalog, with no gateway in front of it

> Port **3001** also appears in the **Ports** tab now that you have published it. Its root URL
> returns `{"error":"not found"}` — add `/books` to the address.

> 📌 **`EXPOSE` vs `-p`.** `EXPOSE 3001` in the Dockerfile is documentation — it publishes
> nothing. `-p 3001:3001` is what actually connects the container to the outside world. The
> Compose services have no `ports:` entry, and that is the only reason your browser cannot
> reach them.

> 🎯 **Predict first:** is the gateway now sharing traffic with `catalog-solo`? ______
>
> **No.** It is on a different network under a different name. Containers do not find each other
> by accident — they need a shared network and a name. That is exactly what Compose set up for
> you, and what Kubernetes sets up in Session 4.

Remove it before starting the next lab:

```bash
docker rm -f catalog-solo
```

- [ ] `docker ps` shows the four Compose containers again

---

## LAB 3B — Why Kubernetes exists (10 min)

Docker Compose runs containers **on one machine**, and this project sets no restart policy.

> 🎯 **Predict first:** if I kill a container right now, what brings it back? ______

```bash
docker compose ps
docker kill $(docker compose ps -q catalog | head -1)
docker compose ps -a
```

- [ ] One catalog container is **Exited** and stays that way

`-a` also lists stopped containers, so you can confirm it exited rather than disappeared.
Nothing restarts it. Docker can restart containers if you configure a restart policy, but it
will not maintain a declared number of healthy copies. Kubernetes does.

> 📖 **DEFINITION — Orchestration**: running containers across many machines, and keeping them
> in the state you asked for — restarting, replacing, scaling and updating them for you.

| | Docker Compose | Kubernetes |
|---|---|---|
| Machines | One | Many |
| If a container dies | Stays dead unless you set a restart policy | A controller creates a replacement |
| Scaling | Manual | Manual or automatic |
| Rolling updates | No | Built in |
| Good for | Development | Production |

Stop Compose and start the cluster:

```bash
docker compose down
minikube start
```

This takes about two minutes. Leave it running.

- [ ] `kubectl get nodes` shows one node, STATUS **Ready**

---

## LAB 3C — The three objects (14 min)

| Object | Job | Everyday equivalent |
|---|---|---|
| **Pod** | One running container | One worker |
| **Deployment** | Keeps N pods alive, handles updates | A supervisor |
| **Service** | One stable address in front of pods | The shop's phone number |

Open `k8s/bookstore.yaml`. It is the same four services you have been running, described as
Kubernetes objects. Find:

- [ ] The `Deployment` named `catalog`
- [ ] The `Service` named `catalog`
- [ ] The gateway's `CATALOG_URL`, still `http://catalog:3001` — **the code did not change at all**

> 📌 That last point is the whole reason this pattern works. Compose resolved the name, now
> Kubernetes resolves the name. Your application never knew the difference.

**✅ CHECKPOINT 3**

➡️ **Now take QUIZ 3** (5 questions).

---

# SESSION 4 — Hands-on & Industry Insights

## LAB 4A — Deploy the whole shop to Kubernetes (10 min)

The cluster has its own image store, so load your images into it:

```bash
minikube image load bookstore-gateway:v1
minikube image load bookstore-catalog:v1
minikube image load bookstore-orders:v1
minikube image load bookstore-recommendations:v1
```

This takes about a minute. Then apply the manifest:

```bash
kubectl apply -f k8s/bookstore.yaml
kubectl get pods
```

- [ ] Four pods, all **Running** (give them 30 seconds)

Forward the gateway port:

```bash
kubectl port-forward service/gateway 8080:8080
```

Open port 8080 in the **Ports** tab.

- [ ] The BookStore works, now running on Kubernetes

> If a pod says `ErrImageNeverPull`, you missed one of the `minikube image load` commands.

---

## LAB 4B 🔥 — Self-healing (8 min)

> 🎯 **Predict first:** I am about to delete a running pod. What happens? ______

Open a **second terminal**, leave the port-forward running in the first:

```bash
cd lab-starter
kubectl get pods
kubectl delete pod <paste-a-catalog-pod-name>
kubectl get pods -w        # -w = watch. Press Ctrl+C after ~15 seconds
```

- [ ] I recorded what happened: ______________________

> 💡 Use `-w` instead of repeating `get pods`. The replacement appears within a few seconds and
> is easy to miss.

> 📖 **Desired state.** You declared `replicas: 1`. Kubernetes constantly compares what you
> asked for with what exists, and fixes the difference. You never wrote the *how*.

---

## LAB 4C — Independent scaling (10 min)

Scale one service without scaling the others.

```bash
kubectl scale deployment catalog --replicas=5
kubectl get pods
```

- [ ] Five catalog pods, **one** of everything else

> 📌 **A monolith cannot do this.** Browsing is frequent and ordering is rare, so you added
> capacity only where it is needed. A monolith would require five copies of the entire
> application, including the parts under no load.

Scale down to four replicas, which makes the rolling update in the next lab easier to observe:

```bash
kubectl scale deployment catalog --replicas=4
```

---

## LAB 4D — Ship a new version (14 min)

Edit `catalog/server.js` and change a book title — make it obvious, e.g. add `(2nd Edition)`.

```bash
docker build -t bookstore-catalog:v2 ./catalog
minikube image load bookstore-catalog:v2
kubectl set image deployment/catalog catalog=bookstore-catalog:v2
kubectl rollout status deployment/catalog
```

- [ ] Pods were replaced **a few at a time**, never all at once
- [ ] After refreshing the browser, the new title appears

> 📖 **DEFINITION — Rolling update**: replacing old versions with new ones gradually, so the old
> and new versions overlap instead of the service stopping completely.
>
> ⚠️ Gradual is not the same as *uninterrupted*. Staying fully available also needs readiness
> probes and spare capacity, and this manifest has neither — so expect brief gaps.
>
> 🧠 You did not need a maintenance window or a maintenance page, and you did not redeploy
> `orders`, `gateway` or `recommendations`. Only one service changed.

Roll back to the previous version:

```bash
kubectl rollout undo deployment/catalog
kubectl rollout status deployment/catalog
```

- [ ] The old title is back, in seconds

**✅ CHECKPOINT 4**

➡️ **Now take QUIZ 4** (5 questions), then read the assignment brief.

> 📚 **The revision bank in `Workshop-Guide.md`** contains twenty additional questions with
> answers, for revision before the viva.

---

## Clean up

```bash
kubectl delete -f k8s/bookstore.yaml
minikube stop
```

Then stop your Codespace: ☰ menu → *My Codespaces* → **Stop**.

---

## Command cheat sheet

```bash
# compose - starts the four containers with one command
docker compose up --build -d       docker compose ps / ps -a
docker compose logs                docker compose logs -f <service>
docker compose stop <service>      docker compose start <service>
docker compose exec <svc> wget -qO- <url>
docker compose down

# docker
docker images                      docker build -t name:tag ./folder
docker run -d --name x -p 3001:3001 name:tag
docker ps                          docker rm -f x

# kubernetes
minikube start / stop              minikube image load <image>
kubectl apply -f file.yaml         kubectl get pods / svc / deploy
kubectl delete pod <name>          kubectl scale deployment <name> --replicas=5
kubectl logs <pod>                 kubectl describe pod <name>
kubectl port-forward service/gateway 8080:8080
kubectl set image deployment/catalog catalog=bookstore-catalog:v2
kubectl rollout status deployment/catalog
kubectl rollout undo deployment/catalog

# simulate failures in the recommendations service
docker compose exec recommendations wget -qO- http://localhost:3003/slow
docker compose exec recommendations wget -qO- http://localhost:3003/fail
docker compose exec recommendations wget -qO- http://localhost:3003/ok
```

## Troubleshooting

| Problem | Fix |
|---|---|
| Port 8080 shows nothing | **Ports** tab → check 8080 is forwarded → click the globe icon |
| `docker compose up` fails on a port | Something else is on 8080: `docker compose down` first |
| Page is blank / spinner forever | Run `docker compose logs` and look for a crashed service |
| Catalog panel is red but you did not stop it | The book list loads once per page load. Wait a few seconds, then **refresh** |
| All four services Up, but catalog is still unreachable | Run `sudo iptables-legacy -P FORWARD ACCEPT`, then refresh |
| `port is already allocated` on `docker run` | `docker rm -f catalog-solo` and try again |
| `ErrImageNeverPull` in Kubernetes | You missed a `minikube image load`. Run all four |
| `ImagePullBackOff` | Same cause — the manifest uses `imagePullPolicy: Never` on purpose |
| Pods stuck `Pending` | minikube is still starting. `kubectl get nodes` and wait |
| `minikube: command not found` | **Ctrl+Shift+P** → *Codespaces: Rebuild Container* |
| minikube won't start | `minikube delete` then `minikube start`. Run `docker compose down` first to free memory |
| `curl localhost:30080` refused | Expected on minikube — use `kubectl port-forward` instead |
| Recommendations never recover | Run the `/ok` command; the page re-checks every 5 seconds |
| Lost in the terminal | `cd /workspaces/*/lab-starter` |
