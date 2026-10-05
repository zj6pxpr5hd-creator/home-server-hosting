# Hosting My Web App On My Home Server
Hi! This is a description of what i did to host my own web app, [Aurora](https://github.com/zj6pxpr5hd-creator/aurora_prot)  (check the aurora_prot repo for the code), on my own home server.
The home server started as a side project to implement AI automations but by using i found it it can do other things as well. This is one of those things.
I am writing this as i am deploying the architecture so I don't know how it's going to go, so wish me well!

## The Server
The hardware itself isn't anything special, I used my old laptop, a HP pavillion with an i7 8th gen with 16GB of RAM, to implement this.
I have quickly found out that 16GB of RAM aren't enough to run local LLM's models, but they should be enough to run my app, which I will talk about more later.
To maximize the performance of my laptop i switched from Windows to Ubuntu operating self-hosted DevOps and AI automation host using Docker Compose
The environment is optimized to run local containerized workloads, execute privileged host tasks securely and manage Git-driven workflows.

### Current Server Infrastructure & Core Services
**Host System:** \
Ubuntu Linux running Docker Engine and Docker Compose.

**Automation Hub (n8n v2.39.9):**\
Self-hosted instance running under the unprivileged node user.

**AI & LLM Stack:**\
Gemini API (goated for it's free usage limits): Cloud-based LLM integration used inside n8n workflows for complex analysis (e.g., codebase mapping).

**Networking & Ingress:**\
Local subnet access (192.168.178.***).
Cloudflare Tunnel (cloudflared): Secure external HTTPS access without exposing open router ports.

**Active Automations:**\
AGENTS.md Generator Workflow: Automated GitHub integration using fine-grained Personal Access Tokens (PAT) to analyze repository structures, process schemas via Gemini, and commit updates directly to the development branch.

## The App
The app I will be deploying is [Aurora]([zj6pxpr5hd-creator/aurora_prot](https://github.com/zj6pxpr5hd-creator/aurora_prot.git)), an AI secretary that remembers the user future and past events, tasks and goals and uses the given information to select what items are relevant in the moment.
I chose this app because it's the one I was working on at the moment, but the stack i used to build it should make deploying on my own not too difficult.
The stack i used is the following:
- Frontend: React, Typescript and Vite
- Backend: Node.js, Express
- Database: SQLite
- Package Manager: pnpm

What is the role of each technology in the deployment setup?
- Frontend: Compiled into static assets via multi-stage Docker build.
- Backend: Acts as the unified server—handling API routes and serving built frontend static assets (/dist) on a single port.
- Database: Lightweight file-based DB. Persisted via Docker host bind mount
- Package Manager: Efficient dependency management with frozen lockfile support during multi-stage image builds.

If this sounds like gibberish to you, don't worry it did to me too before i asked (insert favourite chatbot) what it all meant. I will write my learnings here so you don't have to waste tokens.

### Frontend
What it is: The visual user interface—everything you see, click, and interact with in your browser (buttons, forms, theme toggles, and chat windows).
How it gets deployed: Your browser cannot directly run raw React or TypeScript files. During deployment, a build tool (Vite) compiles all your source code into a bundle of standard HTML, CSS, and JavaScript files (stored in a /dist folder). When you visit the app URL, the server sends this static bundle to your browser to render the page.

### Backend
What it is: The engine. It handles application logic, processes user requests, executes time utilities, and manages communication with the database.
How it gets deployed: In a single-container architecture (the one I am using for Aurora), Express acts as a unified server performing two key jobs:
1. Static File Host: It serves the compiled React frontend files (/dist) to your browser when you first open the app.
2. API Provider: It listens on the selected port for data requests sent from your browser (like saving a record or making an API call), processes the logic, and returns the response.

### Database
What it does: The permanent storage layer where application data (like chat history or user settings) lives across sessions.
How it gets deployed: SQLite is a lightweight file-based database, the entire database resides inside a single file on disk (e.g., app.db). Because Docker containers are temporary (anything created inside them is erased when the container stops or rebuilds), the deployment mounts this file to the HP server's physical hard drive using a Docker Volume.

### Package Manager
What it does: The automation tool that downloads, tracks, and manages all third-party code libraries (dependencies) required by both your frontend and backend.
How it gets deployed: It works behind the scenes during the Docker build phase. It uses the pnpm-lock.yaml file to ensure the exact same library versions used during development are installed inside the server container.

## Deploying
Ok now that we have everything we need, we just have to deploy. 
I'm going to deploy the app as a single container on my Ubuntu server. A single-container deployment bundles the entire application, including the compiled frontend UI, backend API routes, and runtime dependencies, into one unified Docker container. This streamlines the infrastructure by running the full-stack app on a single server port, significantly reducing memory overhead and deployment complexity.

This is a list of every step that is needed to do that (in my case).

1. **Tell the backend to serve the website screens:** Right now, the backend code only handles raw data requests (specific requests at the endpoints defined in /routes). I have to add a few lines of code so that when someone visits the website URL, the backend also sends them the visual React screens (buttons, menus, and text).\
The code is the folowing:
- `const distPath = path.join(__dirname, '../../dist');`\
This allows the Express server to identify the /dist diractory that contains the frontend files compiled by Vite.
- `app.use(express.static(distPath));`\
This tells Express that if it gets a request for a specific file, it has to look in the /dist directory to find it and then send it to the user.
- `app.get('*', (req, res) => {
  res.sendFile(path.join(distPath, 'index.html'));
});`\
The * acts as a wildcard catch-all. If a request is not an API endpoint and does not match a physical file in /dist, Express defaults to sending the index.html file. Once loaded in the browser, React reads the URL path and renders the correct view.

**Crucial Rule:** API endpoints must always be declared above the static middleware and fallback route. If the wildcard route app.get('*') comes first, it will intercept all API calls and return HTML instead of JSON.

2. **Set a permanent location for the database file:** SQLite stores all the  app's memory inside a single file (just like a Word document). In Docker, if the application creates app.db inside an unmounted folder (like /app/server/src/db/app.db), all the data will disappear whenever the container restarts or rebuilds. To prevent that I have to mount the db file to the host disk in a Docker volume and make sure that the db handler points to that container mounted directory.\
In practice, i just have to write more code.
- `import fs from "fs";` \
To make sure the app doesn't crash if the correct directory doesn't exists yet, but instead it creates that right away.
- `const dbPath = process.env.DB_FILE || path.join(__dirname, "aurora.db");`\
Selects the correct db directory path depending on the tipe of enviroment (development or production).
In the docker-compose.yml file I will specify the correct DB_FILE directory making sure the correct directory is selected, while in development the app will default to the standard aurora.db file
- `const dbDir = path.dirname(dbPath);
if (!fs.existsSync(dbDir)) {
  fs.mkdirSync(dbDir, { recursive: true });
}`\
Making sure the directory exists before opening the database. (uses the library imported first)

3. **Write the Dockerfile:** Docker needs instructions to package the application. A Dockerfile is simply a text recipe that tells the server: "Download Node.js, compile the React design, install the code dependencies using pnpm, and package everything into one tidy box."\
In practice, you guessed it, more code.\
Since i had never used Docker before (but already heard about it so this was a deliberate choice) I got some help from my trusty AI chatbot.
I won't explain all the details here, you can check out the Dockerfile here => [Dockerfile](https://github.com/zj6pxpr5hd-creator/aurora_prot/blob/ae4e6da019ac67adc3b655e9414eb230af00fbea/Dockerfile)

4. **Writing docker-compose.yml:** This file acts like a remote control for the container box. It tells the server: "Run this box on port 5000, and connect the box's database folder to a real folder on the server's hard drive."\
So more code.
Again I had never wrote any Docker code before, but I did use it to run the other services that my server was running, so I already knew how a docker-compose file looked like. With some AI help it wasn't too difficult setting this up as I wanted. You can find the file here => [docker-compose.yml](https://github.com/zj6pxpr5hd-creator/aurora_prot/blob/caff80428e45f2376a0a6b8066d40fe09d3dfabc/docker-compose.yml)

5. **Upload changes to GitHub:** Push all changes made to the code (New Dockerfile and docker-compose-yml included) to Github so that I can clone the repo from the server and run the code on there.

6. **Download the new files onto the server:** Log into the server's terminal and run a git pull command to fetch the latest changes from GitHub onto the server's drive. Now all the app's code is on the server ready to be started.

7. **Build and start the application:** Run one command (docker compose up -d --build) on your server. Docker will read your recipe, assemble the app box, and start running it quietly in the background.

8. **Open the app in the browser and test it!** 

- Optional Make it accessible outside home: right now I can connect to the app through Tailscale from my other devices even when I am outside of the house. Maybe in the future I will make it accessable for everyown with a Cloudflare tunnel, but that is not neccessary now.


## What went wrong
This is probably the most important part and the whole reason I decided to deploy the project on my own server, so that I would get errors, many errors.
Most of them where fixed with the help of AI because I had no idea what I was doing, but I still tried my hardest to understand every single one of them so that I could move on from this project smarter and able to deploy the next one in a smoother way.
This is the list of the problems i encountered while deploying Aurora:


### 1. `COPY --from=backend-builder /app/node_modules` failed: "not found"

**Symptom:** build error `"/app/node_modules": not found`.

**Cause:** `pnpm-workspace.yaml` only had `allowBuilds` and no `packages` key. So `server` was not a workspace member, `pnpm install --filter ./server...` matched nothing and installed nothing (and exited without errors).

**Fix:**
```yaml
packages:
  - 'server'
allowBuilds:
  better-sqlite3: true
  '@google/genai': false
  protobufjs: false
```
Then regenerate and commit `pnpm-lock.yaml` (otherwise `--frozen-lockfile` fails).

**Runner stage needs both folders** (the symlinks in `server/node_modules` point to the root `.pnpm` store):
```dockerfile
COPY --from=backend-builder /app/node_modules ./node_modules
COPY --from=backend-builder /app/server/node_modules ./server/node_modules
COPY server ./server   # after the node_modules copies
```

**How I diagnosed it:** `docker build --target backend-builder -t dbg .` then `docker run --rm dbg sh -c "ls -la /app"` to see what the builder stage really contains.

---

### 2. Container restarting with exit code 139

**Symptom:** `aurora_service exited with code 139 (restarting)`.

**Cause:** 139 = 128 + 11 = SIGSEGV. Node crashed while loading the native module `better-sqlite3@13` on Node 20. Image, architecture and libc were all consistent (alpine, x86_64), so it was a version incompatibility.

**Fix:** switch all stages to `node:22-alpine` and rebuild with `--no-cache`.

**Test command (isolates the crash without starting the app):**
```bash
docker compose run --rm --entrypoint sh aurora -c "cd server && node -e \"const D=require('better-sqlite3'); console.log('ok', new D(':memory:').prepare('select sqlite_version() v').get())\""
```
A segfault prints nothing after the shell starts; success prints `ok { v: '...' }`.

**Rule of thumb:** a segfault in a Node container almost always means a native module, so check the module's supported Node versions first.

---

### 3. Frontend not served (saw "Aurora server is running" instead)

**Cause 1:** in `server/app.js`, `distPath = path.join(__dirname, '../../dist')` resolved to `/dist`. `app.js` lives in `/app/server`, so the frontend at `/app/dist` is **one** level up.

**Fix:**
```javascript
const distPath = path.join(__dirname, '../dist');
```

**Cause 2:** the health check `app.get('/')` was answering requests that `express.static` could not satisfy, which hid the real problem. Moved it to its own path:
```javascript
app.get('/health', (req, res) => res.send('Aurora server is running'));
```

**Order that works in `app.js`:** `express.json()`, `cors()`, `express.static(distPath)`, API routes, then the `app.get('*splat', ...)` catch-all last. Express 5 needs `'*splat'`, not `'*'`.

---

### 4. Gemini API key missing in the container

**Symptom:** `WARN The "GEMINI_API_KEY" variable is not set` and `missing` inside the container.

**Cause:** the repo has no secrets (correct), so on the server the `.env` simply didn't exist. `dotenv/config` only reads a file from the working dir, and the image doesn't (and must not) contain one.

**Fix:**
1. Create `.env` on the server, next to `docker-compose.yml`: `GEMINI_API_KEY=...` (no spaces around `=`, no quotes). `chmod 600 .env`.
2. In compose: `env_file: - .env` (don't also keep an empty `environment: GEMINI_API_KEY=${...}` line, it would override).
3. `docker compose up -d --force-recreate` (env vars only apply at container creation).
4. Check without printing the key: `docker compose exec aurora sh -c 'test -n "$GEMINI_API_KEY" && echo set || echo missing'`

**Also:** `.env` stays in `.gitignore` and `.dockerignore`; `.env.example` (names, no values) is committed with `!.env.example` in `.gitignore` if needed.

---

### 5. Frontend loaded but no data, backend received no requests

**Cause:** hardcoded `const API_BASE_URL = 'http://localhost:3000';` in the frontend. In the browser, `localhost` is the user's own machine, not the server. It only worked in local dev.

**Fix:**
```typescript
const API_BASE_URL = '';   // relative URLs, same origin as the backend
```
(or `import.meta.env.VITE_API_URL ?? ''` with `VITE_API_URL` set only in the local `.env`).

**Remember:** the frontend is baked into the image, so changes need `docker compose build --no-cache && docker compose up -d --force-recreate`, then a hard reload (Ctrl+Shift+R). Vite env vars are read at **build time**, not runtime.

**How I diagnosed it:** browser DevTools, Network tab, filter Fetch/XHR, look at the Request URL.

---

### 6. Pinning pnpm broke the build (`ERR_PNPM_FROZEN_LOCKFILE_WITH_OUTDATED_LOCKFILE`)

**Cause:** adding `"packageManager": "pnpm@12.9.1"` to `package.json` makes pnpm want to record that version in `pnpm-lock.yaml`. With `--frozen-lockfile` it refuses to update the lockfile.

**Fix used:** remove `packageManager` from `package.json` and pin in the Dockerfile instead (both stages using pnpm):
```dockerfile
RUN corepack enable && corepack prepare pnpm@12.9.1 --activate
```

**Alternative:** keep `packageManager` and regenerate the lockfile in a throwaway container, then commit it:
```bash
docker run --rm -v "$PWD":/app -w /app node:22-alpine sh -c "corepack enable && pnpm install --lockfile-only"
```

**Find the version the build uses:** `docker build --target backend-builder -t dbg . && docker run --rm dbg pnpm -v`. pnpm is not installed on the host, and doesn't need to be.

---

### 7. Database persistence

- The SQLite file is `/app/server/data/aurora.db`. The volume must mount the **folder** (`/app/server/data`), not the single file, so `-wal`/`-shm` files are covered too.
- Check it's a real mount: `docker compose exec aurora sh -c 'grep " /app/server/data " /proc/self/mountinfo'`
- Real test: create data, `docker compose up -d --force-recreate`, reload. Never use `docker compose down -v` (deletes named volumes).
- The app runs as **root** (the file is owned by root). If `USER node` is ever added, first run `docker compose exec -u root aurora chown -R node:node /app/server/data`, or SQLite fails with "attempt to write a readonly database".

---

### Handy commands

| Goal | Command |
|---|---|
| Rebuild everything fresh | `docker compose build --no-cache && docker compose up -d --force-recreate` |
| Inspect a build stage | `docker build --target <stage> -t dbg .` then `docker run --rm dbg sh -c "ls -la /app"` |
| Live logs | `docker compose logs -f aurora` |
| Deploy update | `git pull` then `docker compose build` then `docker compose up -d` |
| Free disk space | `docker system prune -f` (never with `--volumes`) |

### Still open / nice to have

- **No backups** (decided to skip for now since it's a test app). The data folder is the only thing that matters: copy it while the container is stopped.
- No login on the app and `cors()` is open: fine while it's only reachable via SSH/Tailscale. To be safe, bind the port to the Tailscale IP or `127.0.0.1` (plus `tailscale serve`) and don't forward port 5000 on the router.
- Set a spending cap or alert on the Gemini key (if using a paid plan in the future).
- Re-check the Node base image every few months, and test native module upgrades locally before deploying.