# AI Usage

This document records how AI tools were used in this project: what was asked, what was produced, what went wrong, and which decisions were made.

**Tools used:** Claude Code (CLI, on Windows), model Claude Opus 5.5. Other tools: Unknown – to be completed manually.

---

## 1. Prompts

One row per significant prompt. "Verification" describes how we checked the output ourselves, for example a command we ran or a line we read and understood.

| # | Date | Tool | Prompt (summary) | Output (files / result) | Verification | Changes we made |
|---|---|---|---|---|---|---|
| 1 | 2026-11-10 | Claude Code | Create `CLAUDE.md` with working rules: educational DevOps project (13th grade), every line must be defended; write code one step at a time; explain each file line by line in Hebrew; mention alternatives; at the end of each step give one verification command, ask one question and wait; use only Docker, Docker Compose, Nginx, Apache, HTML/CSS/JS, GitHub Actions; Windows + Docker Desktop, PowerShell commands. "Do not write any other code at this stage." | `CLAUDE.md` | Unknown – to be completed manually. | None found (file matches the AI output). |
| 2 | 2026-11-10 | Claude Code | "Set up only the web server for me, without connecting it to the storage server yet." Dockerfile based on the official Nginx Alpine image, a custom config replacing the default, a temporary `index.html` with only a heading, and a static-text health check endpoint for CI. | `web/Dockerfile`, `web/nginx.conf`, `web/index.html` (temporary heading page). The AI tried to build and run the image, but `docker` was not found in its shell, so it did not test the files itself. | Unknown – to be completed manually. | None found. `web/index.html` was later replaced by the AI in prompt #4. |
| 3 | 2026-11-10 | Claude Code | "Connect the two services in `docker-compose.yml` and resolve the communication issue between the browser and the storage server, following the plan we established in Step 1." Explicit shared network; only the web server reachable from outside; define the browser path for images; explain each key, how Nginx builds the proxied address and why the trailing slash matters, what `depends_on` does and does not guarantee; give one end-to-end `curl` command. | `docker-compose.yml`; new `location /plates/` block (`proxy_pass http://storage/plates/;`) in `web/nginx.conf`. The AI said it did not have the "Step 1" plan and based the design on the existing `storage/` files. | Unknown – to be completed manually. | None found. |
| 4 | 2026-11-10 | Claude Code | "Now, build the actual dashboard." Fetch the Apache directory listing through the defined path, extract image filenames, one card per vehicle, counters for vehicles inside / free spaces, auto-refresh every 10 seconds. Custom feature: duration-based parking fee calculator + dark-mode toggle. Clear message when the server is unavailable, XSS protection for filenames, clean design that works on a projector. | `web/index.html` (rewritten), `web/style.css`, `web/app.js`; `web/Dockerfile` updated to copy all three files. The AI ran a JavaScript syntax check (`node --check`) that passed; the page itself was not run by the AI. | Unknown – to be completed manually. | None found. |
| 5 | 2026-11-10 | Claude Code | "Write a complete `README.md` for the project": system description, text architecture diagram, services table, run instructions, volume proof (proof of scale), CI/CD explanation, repository structure, empty team table. Also an `AI_USAGE.md` template with a prompts table, an errors and corrections section and an architectural decisions section. | `README.md` (CI/CD section marked "planned", because no workflow file exists yet); `AI_USAGE.md` template. The AI corrected two lines in the README right after writing it (see errors #1 and #2). | Unknown – to be completed manually. | None found. |
| 6 | 2026-11-10 | Claude Code | Populate the `AI_USAGE.md` tables using only the actual conversation history and project files; write "Unknown – to be completed manually" where there is no clear source; keep the original structure; list all "Unknown" fields at the end. | This version of `AI_USAGE.md`. | Unknown – to be completed manually. | Unknown – to be completed manually. |

---

## 2. Errors and Corrections

Copy this block once for each case where the AI's output was wrong, incomplete, or did not fit the requirements.

### Error #1

- **What the AI produced:** In `README.md`, volume proof step 3: "Add 20 more vehicles (25 in total, capacity is 20)".
- **What was wrong:** The count was wrong. Step 2 had already added `license6.jpg`, so the total is 5 + 1 + 20 = 26. The same step's expected result already said 26.
- **How we found it:** The AI noticed the mismatch while reviewing the README right after writing it (prompt #5).
- **Correction:** Changed to "26 in total".
- **What we learned:** Unknown – to be completed manually.

### Error #2

- **What the AI produced:** In `README.md`, a check that `storage` is not reachable from the host: `docker compose port storage 80`, described as "This should fail to connect".
- **What was wrong:** The description did not match the command. `docker compose port` does not try to connect to anything; it only prints a published port mapping. The AI was not sure of its exact output for an unpublished port, so the check was unreliable.
- **How we found it:** The AI reviewed the README right after writing it (prompt #5).
- **Correction:** Replaced with `docker compose ps`. In the `PORTS` column, `web` shows `0.0.0.0:8080->80/tcp` while `storage` shows only `80/tcp`.
- **What we learned:** Unknown – to be completed manually.

### Error #3

- **What the AI produced:** In prompts #3, #4 and #5, the AI went ahead with the next step even though the student had not answered the comprehension question from the previous step. It mentioned the open questions at the end of each answer.
- **What was wrong:** `CLAUDE.md` says: "Wait for the student's answer before continuing."
- **How we found it:** Visible in the conversation: the questions about `EXPOSE 80`, the `8081` port and the 502 status are still unanswered.
- **Correction:** None yet. Answering the open questions is still pending.
- **What we learned:** Unknown – to be completed manually.

### Error #4

- **What the AI produced:** In prompt #4, the AI checked `web/app.js` with `node --check` (Node.js).
- **What was wrong:** `CLAUDE.md` allows only Docker, Docker Compose, Nginx, Apache, HTML/CSS/JS and GitHub Actions. Node.js was used only as a one-off check on the AI's side and is not part of the project, but it is still outside the allowed list.
- **How we found it:** Visible in the conversation (the AI described it as "only on my side, not part of the project").
- **Correction:** None. No project file depends on Node.js.
- **What we learned:** Unknown – to be completed manually.

---

## 3. Architectural Decisions

One row for each decision that shaped the system. "Source" records who proposed it: the team, the AI, or the AI and then changed by the team.

| # | Decision | Alternatives considered | Why we chose it | Source |
|---|---|---|---|---|
| 1 | Allowed technologies only: Docker, Docker Compose, Nginx, Apache, HTML/CSS/JS, GitHub Actions; PowerShell commands for Windows + Docker Desktop. | — | Only technologies covered in the course (`CLAUDE.md`). | Team |
| 2 | Web server image `nginx:1.27-alpine` (Alpine, pinned version). | `nginx:latest`, `nginx:alpine` (unpinned), Debian-based `nginx`. | Alpine is small (faster pulls and CI, fewer vulnerabilities); a pinned version keeps builds reproducible. | Both (team required Nginx Alpine; AI pinned version 1.27) |
| 3 | Replace the default config by copying `nginx.conf` over `/etc/nginx/conf.d/default.conf`. | `RUN rm` the default then copy under another name; replace the whole `/etc/nginx/nginx.conf`. | One instruction, no extra layer; keeps the main Nginx settings (`worker_processes`, `events`, `mime.types`) at their defaults. | Both (team required replacing the default config; AI chose the method) |
| 4 | Health endpoint `location = /health` → `return 200 "OK\n"` as `text/plain`, with `access_log off`. | Prefix match `location /health`; a static `health.txt` file; `add_header Content-Type`. | Exact match catches only `/health`; the response comes from Nginx itself, not a file on disk; no log noise from repeated CI checks. | Both (team required a static-text health endpoint for CI; AI chose the implementation) |
| 5 | Storage server: Apache `httpd:2.4-alpine`, directory listing enabled only for `/plates` (`storage/Dockerfile`, `storage/plates.conf`). | Unknown – to be completed manually. | Unknown – to be completed manually. | Unknown – to be completed manually (these files were not created in the recorded conversation). |
| 6 | Nginx is a reverse proxy: `location /plates/` → `proxy_pass http://storage/plates/;`. The browser only talks to Nginx. | Publishing a port for Apache (e.g. `8081:80`) and letting the browser access it directly; `proxy_pass http://storage;` without a path. | Keeps storage internal; same origin for the browser, so no CORS; the explicit path shows the full proxied address. | Both (team asked to follow the "Step 1" plan, whose content is not in this history; AI wrote the implementation) |
| 7 | Explicitly defined shared network `parking-net` (`driver: bridge`) for both services. | The Compose default network (`<project>_default`). | The requirement asked for an explicit network; it makes clear which services are connected. | Both (team required an explicit network; AI chose the name and driver) |
| 8 | Storage is internal only: no `ports` on `storage`; `expose: "80"` only as documentation. Only `web` publishes `8080:80`. | Publishing a port for storage. | Without `ports`, Docker opens no port on the host; storage is reachable only from the Docker network. | Both (team required internal-only storage; AI chose the setting) |
| 9 | Images are mounted with a bind mount: `./license_plates` → `/usr/local/apache2/htdocs/plates:ro`. | `COPY` the images into the image at build time. | New images appear without a rebuild; `:ro` stops the container from changing files (least privilege). | AI |
| 10 | Simple `depends_on: [storage]` for `web`, without a healthcheck condition. | `healthcheck` + `condition: service_healthy`. | Guarantees start order so Nginx can resolve `storage` at startup; the healthcheck was deferred to keep the step small. | AI |
| 11 | No backend: the browser fetches the Apache listing and parses it with `DOMParser`; only plain image filenames are accepted (`IMAGE_PATTERN` regex); duplicates removed with a `Set`. | Extracting names with a regex over the raw HTML text. | Parsing HTML with a regex is fragile; `DOMParser` creates an inert document (no scripts run, no images load). | Both (team required no backend and parsing the listing; AI chose `DOMParser` and the filter) |
| 12 | XSS protection: elements built with `createElement`, text set with `textContent`, image URL built with `encodeURIComponent`; never `innerHTML`. | Building an HTML string and assigning it to `innerHTML`. | `textContent` always inserts text, never HTML, so a malicious filename cannot run code. | Both (team required XSS protection; AI chose the method) |
| 13 | Refresh every 10 s with `setInterval`; `fetch` uses `cache: 'no-store'` and a 5 s timeout (`AbortSignal.timeout`); `response.ok` checked explicitly. | Self-scheduling `setTimeout`. | No stale cached listing; a hung server cannot block the page; the 5 s timeout prevents overlapping requests. | Both (team required a 10 s refresh; AI chose the details) |
| 14 | When storage is unavailable: show a red error message, reset the counters to `—` and clear the cards. | Keeping the last cards on screen. | Stale data that looks current is riskier on a parking display than an explicit "no data" message. | Both (team required a clear error message; AI chose to clear the old data) |
| 15 | Parking capacity is a constant: `TOTAL_SPOTS = 20`; free spaces never go below 0. | Unknown – to be completed manually. | There is no backend or database to store capacity. | AI |
| 16 | Fee calculator: user enters a duration; 12 ₪ first hour, 8 ₪ per additional started hour, at most 80 ₪ per day, each full day billed separately. | A single cap on the whole stay (`Math.min(fee, 80)`); hard-coding rates in the HTML. | The system has no entry times; a per-day cap stays correct for multi-day stays; rates are defined once in JS and shown from there. | Both (team chose a duration-based fee calculator; AI chose the rates and rules) |
| 17 | Dark mode: CSS variables switched by `data-theme` on `<html>`; choice saved in `localStorage` (inside `try/catch`); defaults to the OS preference. | Per-element `.dark` classes; an inline `<head>` script to avoid a light flash on load. | One switch changes the whole page; a storage error cannot break the rest of the script. | Both (team chose the dark-mode toggle; AI chose the implementation) |
| 18 | Projector-friendly design: 20px base font, very large counters, high-contrast colors, 2px borders, responsive grid without media queries. | Unknown – to be completed manually. | Readability from a distance on a projector. | Both (team required a projector-friendly design; AI chose the values) |
| 19 | Separate `index.html`, `style.css` and `app.js` files, copied explicitly in the Dockerfile. | Inline `<style>`/`<script>`; `COPY . /usr/share/nginx/html/`. | Easier to read and defend; copying everything would expose the Dockerfile and `nginx.conf` to the browser. | AI |
| 20 | `README.md` in English; the CI/CD section is marked "planned" until `.github/workflows/ci.yml` exists. | A Hebrew README; describing the pipeline as if it already existed. | Mixed Hebrew and code renders poorly on GitHub; the README should not describe a workflow that does not exist. | AI |

---

## 4. Reflection

- **Where the AI helped most:** Helped in understanding the required architecture,He helped develop the actual code for the website and other elements.
- **Where it was wrong or risky:** Actually, we haven't encountered any instances where it made a mistake, but it could easily lead to various problems—such as the incorrect definition of things.
- **What we would do differently:** We learn so much from this project, Next time, we’ll broaden our horizons much further.