# FastBite – Deployment Runbook

**For:** Bilal Ahmed (CTO) and whoever runs FastBite next
**Prepared by:** Taha Yasir (22I-2429)
**Scope:** the six Node.js services in `Q2-cafe-system/` and the GitHub pipeline in `.github/workflows/fastbite-ci.yml`

This document is written so that someone who has never seen the project can run it, ship it and repair it from this repository alone, without needing me or my laptop.

---

## 1. What was wrong

| Problem | Why it matters |
|---|---|
| The platform ran in **six terminal windows**, with a command typed by hand in each one | Only the person who remembers the commands can start it, and one closed window takes a service down |
| **Configuration files containing passwords** were being committed | Anyone with read access to the repository has the database password, permanently, because Git keeps history |
| Every service fell back to **`localhost`** to find the others | That only works when all six run on one laptop. Inside a container, `localhost` is the container itself, so the services cannot find each other |
| **No build or packaging step** | Nothing tested or packaged a merged change, so what ran was whatever was on someone's machine |
| The API gateway was **nested inside `inventory-service/`** | It would have been copied into the inventory image, and the layout no longer matched the six-service design |

## 2. What was done, in order, and why

The order matters: each step depends on the one before it.

1. **Restore and tidy the code.** The Q2 files had been deleted from `main`, so they were restored first. The gateway was then moved to its own `api-gateway/` folder, so that each of the six services is one folder, one image and one container.
2. **Move configuration out of the code.** The `localhost` defaults were removed from all six `index.js` files. Service addresses and `MONGO_URI` must now come from the environment, and a service that starts without them prints `Missing required environment variable …` and exits straight away. `PORT` keeps each service's correct port as its default. This comes before Docker because a container cannot work while the code still assumes `localhost`.
3. **Package each service on its own (`<service>/Dockerfile`).** Each image is `node:22-alpine` and does the following:
   - installs exact dependency versions with `npm ci`
   - sets and exposes its own port
   - runs as the unprivileged `node` user
   - checks its own `/health` endpoint
4. **Exclude what Git and Docker must never contain.**
   - `.gitignore` excludes `node_modules/`, every `.env*` file except `.env.example`, keys, logs, PM2 runtime files and editor clutter.
   - Each `.dockerignore` keeps `node_modules`, logs, `.env*` and `.git` out of the image build. Dependencies are installed inside the image and secrets are supplied at run time, so neither is ever baked into an image.
5. **Keep secrets out of the repository.** The only secret, the MongoDB root user and password, lives in a local `.env` file that both ignore files exclude. `.env.example` is the committed template. `docker compose` refuses to start if either value is missing. The pipeline creates a random throwaway password on every run and logs in to the registry with GitHub's built-in `GITHUB_TOKEN`. No secret is stored anywhere in the repository or in GitHub settings.
6. **Run everything with one command (`docker-compose.yml`).** This replaces the six terminal windows:
   - MongoDB 7 runs with authentication turned on, and its data is kept in the `mongo-data` volume.
   - The six services each get their own database, `fastbite_menu`, `fastbite_orders` and so on.
   - Containers find each other by **service name** on the private `fastbite` network, for example `http://menu-service:3001`.
   - Only the gateway is published, on port 3000. The other services and the database cannot be reached from outside that network.
   - Start-up waits on health checks: MongoDB first, then the services, then the gateway.
   - `restart: unless-stopped` restarts any service that crashes.
7. **Automate testing and publishing on GitHub (`.github/workflows/fastbite-ci.yml`).** On every push to `main`, which includes every merged pull request:
   1. **Smoke test.** The pipeline starts the whole platform with `docker compose`. Through the gateway, customer Hamza (`C-1042`) orders a Zinger Burger and Fries, and pays for them. The test checks each of these:
      - the total is Rs. 650
      - he earns 65 points (1 point per Rs. 10)
      - his balance goes from 150 to 215 points and the free drink is unlocked
      - the order is marked `PAID`
      - Zinger Burger stock drops from 80 to 79
   2. **Publish.** Only if the smoke test passes, the pipeline builds and pushes a fresh image for **each of the six services** to GitHub Container Registry, tagged `latest` and `sha-<commit>`.

   Pull requests run the smoke test but publish nothing. The workflow can also be started by hand from the Actions tab using **Run workflow**.
8. **Write this runbook**, so the work can be repeated and checked.

Each step was committed separately with a message that says what changed and why (`git log --oneline`).

## 3. Run FastBite on a new machine

**You need:** Git and Docker (Docker Desktop on Windows or macOS, or Docker Engine with the Compose plugin on Linux). You do not need Node.js or MongoDB installed.

```bash
git clone https://github.com/Taha9876/SCD-LAB-MID.git
cd SCD-LAB-MID/Q2-cafe-system

cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env
# edit .env: choose a real username and a long password (letters, digits, hyphens)

docker compose up -d --build  # build from this checkout
#   or, to run exactly what the pipeline published:
# docker compose pull && docker compose up -d

docker compose ps             # all seven containers should be "healthy"
```

### Check that it works (Hamza's order)

```bash
curl http://localhost:3000/menu
curl http://localhost:3000/customers/C-1042          # loyaltyPoints: 150

curl -X POST http://localhost:3000/orders -H "Content-Type: application/json" \
  -d '{"customerId":"C-1042","items":[{"itemId":"M-101","quantity":1},{"itemId":"M-102","quantity":1}]}'
# expect: total 650, pointsEarned 65, loyalty.loyaltyPoints 215, freeDrinkUnlocked true

curl -X POST http://localhost:3000/payments -H "Content-Type: application/json" \
  -d '{"orderId":"<orderId from the previous reply>","amount":650}'
# expect: "Payment successful", status PAID
```

`localhost:3000` here is your own computer talking to the published gateway port, which is how the mobile app reaches the platform. The services never use `localhost` to reach each other.

## 4. Shipping a change

1. Push a branch and open a pull request. The smoke test runs on it.
2. Merge into `main`. The pipeline tests again and publishes six new images:

   | Service | Image | Port |
   |---|---|---|
   | API gateway | `ghcr.io/taha9876/fastbite-api-gateway` | 3000 |
   | Menu | `ghcr.io/taha9876/fastbite-menu-service` | 3001 |
   | Order | `ghcr.io/taha9876/fastbite-order-service` | 3002 |
   | Inventory | `ghcr.io/taha9876/fastbite-inventory-service` | 3003 |
   | Customer | `ghcr.io/taha9876/fastbite-customer-service` | 3004 |
   | Payment | `ghcr.io/taha9876/fastbite-payment-service` | 3005 |

3. On the server, run `docker compose pull && docker compose up -d`. Only containers whose image changed are recreated.
4. **To roll back**, put `TAG=sha-<older commit>` in `.env` and run `docker compose pull && docker compose up -d`. Every published commit stays available under its own tag. Remove the line again to go back to `latest`.

The images are listed under **Packages** on the repository page. If a package is private, either run `docker login ghcr.io` with a token that has `read:packages`, or set the package to Public in its settings.

## 5. Day-to-day operations

| Task | Command |
|---|---|
| Follow logs (all services, or one) | `docker compose logs -f` / `docker compose logs -f order-service` |
| Restart one service | `docker compose restart payment-service` |
| Stop everything (data is kept) | `docker compose down` |
| Wipe the data and start again with demo data | `docker compose down -v` |
| Use another host port for the gateway | set `GATEWAY_PORT=8080` in `.env` |

**Changing the database password.** The `.env` values only create the user the first time the `mongo-data` volume is set up. To change the password on a system that already has data:

```bash
docker compose exec mongo mongosh -u <user> -p <old> --authenticationDatabase admin \
  --eval 'db.getSiblingDB("admin").changeUserPassword("<user>", "<new>")'
# put <new> in .env, then:
docker compose up -d
```

A password that was ever committed to Git must be treated as leaked and changed this way, even after the file is deleted, because it stays in the repository history.

## 6. Bilal's brief, point by point

| Requirement | How it is met | Where |
|---|---|---|
| Stop depending on one person's memory or laptop | One documented command on any machine with Docker; packaging happens on GitHub | `docker-compose.yml`, this file |
| Every service runs on its own | One image and one container per service | `*/Dockerfile` |
| No secrets in the repository | `.env` ignored by Git and Docker; template only; Compose refuses to start without values; the pipeline uses a throwaway password and `GITHUB_TOKEN` | `.env.example`, `.gitignore`, `*/.dockerignore` |
| Each service on its correct port | `PORT` set in each image and in Compose; only 3000 published | `*/Dockerfile`, `docker-compose.yml` |
| Services must not find each other through `localhost` | `localhost` defaults removed from the code; service names on a private network; a missing address stops the service | `*/index.js`, `docker-compose.yml` |
| Every merge to `main` produces a fresh package for every service | Publish job builds all six images after the smoke test passes | `.github/workflows/fastbite-ci.yml` |
| Ignore everything Git and Docker should not include | `.gitignore` and one `.dockerignore` per service | as listed |
| Automated publishing runs on GitHub | GitHub Actions, pushing to GitHub Container Registry | `.github/workflows/fastbite-ci.yml` |
| Meaningful commits as the work goes | One commit per step | `git log` |
| Someone else can repeat the work | This runbook | `DEPLOYMENT.md` |

**Why not PM2?** PM2 restarts crashed processes, but only on one machine that has Node installed, and services started that way still reach each other through `localhost`. Docker's restart policy restarts crashed services in the same way, while each service stays isolated and the same images run on every machine.

## 7. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `required variable MONGO_ROOT_PASSWORD is missing … set MONGO_ROOT_PASSWORD in .env` | There is no `.env` file, or it is missing that value. Copy `.env.example` to `.env` and fill it in |
| A container exits with `Missing required environment variable …` | The service was started without Compose, or a variable was removed from `docker-compose.yml`. Run it through Compose |
| `could not connect to MongoDB: Authentication failed` | `.env` was changed after the volume was created. Change the password as in section 5, or run `docker compose down -v` (this deletes the data) |
| `port is already allocated` on 3000 | Something else is using the port. Set `GATEWAY_PORT` in `.env` |
| A pipeline job is "not started" with a billing message | The GitHub account is locked over billing. Clear it under **Settings → Billing**, then re-run the workflow from the Actions tab |

## 8. Status at hand-in

Checked before pushing:

- the Compose file parses and resolves every variable (`docker compose config`)
- the workflow YAML parses
- all six services pass `node --check`
- every `package-lock.json` matches its `package.json`, so `npm ci` will install cleanly
- no password appears anywhere in the Git history

The first pipeline run was not started by GitHub because of the account billing lock described in section 7. Once that is cleared, run the workflow from the Actions tab, or merge any change into `main`. That run is the end-to-end proof: the smoke test with Hamza's order, and then six images published.
