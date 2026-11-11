# Smart Parking

A containerized dashboard that shows which vehicles are currently inside a parking lot, based on license-plate images stored on a file server.

DevOps assignment, 13th grade.

---

## System Description

The system has two services, each running in its own Docker container:

- **web** (Nginx): serves the dashboard (HTML/CSS/JS) and acts as a **reverse proxy** to the storage server. It is the only service exposed to the outside.
- **storage** (Apache): serves the license-plate images and a directory listing of them. It is reachable **only from inside** the Docker network.

There is no backend application. The browser fetches the Apache directory listing through Nginx, extracts the image filenames, and renders one card per vehicle.

**Dashboard features:**

- One card per vehicle (plate image + name).
- Counters: vehicles inside / free spaces (capacity: 20).
- Auto-refresh every 10 seconds.
- A clear error message when the storage server is unavailable.
- XSS-safe rendering of filenames (`textContent` / `createElement`, never `innerHTML`).
- **Custom features:** a duration-based parking fee calculator and a dark-mode toggle (remembered between visits).
- High-contrast, large-font design that stays readable on a projector.

---

## Architecture

```
                      Host machine (Windows + Docker Desktop)
 ┌──────────────┐
 │   Browser    │
 └──────┬───────┘
        │ http://localhost:8080
        │ (only published port)
 ┌──────┼───────────────── Docker network: parking-net (bridge) ─────────────┐
 │      ▼                                                                    │
 │ ┌──────────────────────────┐   /plates/*  ->   ┌──────────────────────────┐│
 │ │ web                      │   http://storage/ │ storage                  ││
 │ │ nginx:1.27-alpine        │ ─────────────────>│ httpd:2.4-alpine         ││
 │ │                          │      plates/*     │                          ││
 │ │  /         -> dashboard  │                   │  /plates/ -> listing     ││
 │ │  /health   -> "OK"       │                   │             + images     ││
 │ │  /plates/  -> proxy      │                   │  port 80 (internal only) ││
 │ └──────────────────────────┘                   └────────────▲─────────────┘│
 │                                                             │              │
 └─────────────────────────────────────────────────────────────┼──────────────┘
                                                               │ bind mount (read-only)
                                                  ./license_plates  (host folder)
```

**Request flow for an image:**

1. The browser requests `http://localhost:8080/plates/license1.jpg`.
2. Nginx matches `location /plates/` and forwards to `http://storage/plates/license1.jpg`.
3. Docker's internal DNS resolves `storage` to the Apache container.
4. Apache reads the file from the mounted folder and returns it through Nginx.

Because everything comes from the same origin (`localhost:8080`), the browser has no CORS issues.

---

## Services

| Service | Image | Role | Port | Exposed to host? | Volume |
|---|---|---|---|---|---|
| `web` | `nginx:1.27-alpine` | Dashboard + reverse proxy + health check | `8080:80` | **Yes** (`ports`) | — |
| `storage` | `httpd:2.4-alpine` | Serves plate images + directory listing | `80` | **No** (`expose` only) | `./license_plates` → `/usr/local/apache2/htdocs/plates` (`ro`) |

### Endpoints (via `web`)

| Path | Response |
|---|---|
| `/` | The dashboard |
| `/health` | `OK` (`text/plain`, 200), used by CI |
| `/plates/` | Apache directory listing (proxied) |
| `/plates/<file>` | A plate image (proxied) |

---

## Running the Project

**Requirements:** Windows with Docker Desktop running. All commands are for PowerShell and run from the project root.

Start (build images and run in the background):

```powershell
docker compose up -d --build
```

Open the dashboard:

```powershell
Start-Process http://localhost:8080
```

Check the health endpoint:

```powershell
curl.exe http://localhost:8080/health
```

Check the full pipeline (browser → Nginx → Apache → file). Expected output: `200 image/jpeg`.

```powershell
curl.exe -s -o NUL -w "%{http_code} %{content_type}\n" http://localhost:8080/plates/license1.jpg
```

Confirm that `storage` is **not** reachable from the host. In the `PORTS` column, `web` shows `0.0.0.0:8080->80/tcp`, while `storage` shows only `80/tcp`, with no host mapping:

```powershell
docker compose ps
```

Simulate a storage outage. The dashboard shows a red error within 10 seconds, then recovers by itself:

```powershell
docker compose stop storage
docker compose start storage
```

Stop and remove everything:

```powershell
docker compose down
```

> Use `curl.exe`, not `curl`. In Windows PowerShell, `curl` is an alias for `Invoke-WebRequest`.

---

## Volume Proof (Proof of Scale)

This proves that the images are read **live from the host folder** through the volume. Adding or removing vehicles requires no rebuild and no restart.

1. Start the project and open the dashboard. It shows **5** vehicles and **15** free spaces.

2. **Add one vehicle** while the containers are running:

   ```powershell
   Copy-Item .\license_plates\license1.jpg .\license_plates\license6.jpg
   ```

   Within 10 seconds the dashboard shows **6** vehicles and **14** free spaces.

3. **Scale up past capacity.** Add 20 more vehicles (26 in total, capacity is 20):

   ```powershell
   1..20 | ForEach-Object { Copy-Item .\license_plates\license1.jpg ".\license_plates\scale_$_.jpg" }
   ```

   Within 10 seconds the dashboard shows **26** vehicles and **0** free spaces. The free count never goes negative.

4. **Prove the mount is read-only.** The container cannot change the images:

   ```powershell
   docker compose exec storage touch /usr/local/apache2/htdocs/plates/test.jpg
   ```

   Expected: `Read-only file system`.

5. **Clean up.** Within 10 seconds the dashboard returns to 5 vehicles:

   ```powershell
   Remove-Item .\license_plates\license6.jpg, .\license_plates\scale_*.jpg
   ```

During the whole proof, `docker compose ps` shows the same containers with an unchanged uptime. Nothing was rebuilt or restarted.

---

## CI/CD Pipeline

> **Status: planned.** The workflow file (`.github/workflows/ci.yml`) has not been written yet. This section describes the intended design and will be updated when it is added.

The pipeline runs on GitHub Actions on every `push` and `pull_request` to `main`:

| # | Step | What it checks |
|---|---|---|
| 1 | Checkout the repository | — |
| 2 | `docker compose build` | Both Dockerfiles build successfully |
| 3 | `docker compose up -d` | The stack starts |
| 4 | `curl` `/health` until it returns `OK` | Nginx is up and its config is valid |
| 5 | `curl` `/plates/` and an image, expecting `200` | The proxy and the storage server work end to end |
| 6 | `docker compose logs` (on failure) | Shows the reason for a failure in the CI log |
| 7 | `docker compose down` | Cleans up the runner |

If any step fails, the run is marked red and the pull request should not be merged.

---

## Repository Structure

```
smart_parking_pj/
├── .github/
│   └── workflows/
│       └── ci.yml          # (planned) GitHub Actions pipeline
├── license_plates/         # Plate images, mounted read-only into storage
├── storage/
│   ├── Dockerfile          # Apache image + includes plates.conf
│   └── plates.conf         # Enables directory listing only for /plates
├── web/
│   ├── Dockerfile          # Nginx image + config + static files
│   ├── nginx.conf          # Static site, /health, /plates/ reverse proxy
│   └── index.html          # Dashboard: HTML + CSS + JS in one file (RTL, Hebrew)
├── docker-compose.yml      # Services, network, volume
├── AI_USAGE.md             # Documentation of AI use in the project
├── CLAUDE.md               # Working rules for the AI assistant
└── README.md               # This file
```

---

## Known Limitations

- **Fixed capacity.** Parking capacity is a constant (`TOTAL_SPOTS = 20` in `web/index.html`), since there is no backend or database.
- **Fee calculator uses a typed-in duration.** The system does not know when a car entered, so the calculator works from a duration the user enters, not from real entry times.
- **Nginx resolves `storage` once at startup.** If the storage container is recreated with a new IP address, restart `web` with `docker compose restart web`.

---

## Team

| Name | ID | Role | GitHub |
|---|---|---|---|
| | | | |
| | | | |
| | | | |
