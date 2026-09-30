# Hosting My Web App On My Home Server
Hi! This is a description of what i did to host my own web app, [Aurora]([zj6pxpr5hd-creator/aurora_prot](https://github.com/zj6pxpr5hd-creator/aurora_prot.git))  (check the aurora_prot repo for the code), on my own home server.
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
This is a list of every step that is needed to do that (in my case).
1. **Update Express Backend for Static Serving:**
Add static file middleware to the Express app so it serves the built React bundle (`/dist`) and handles client-side routing fallbacks.


2. **Configure SQLite Path for Environment Variables:**
Ensure the database handler reads the database file path from an environment variable (e.g., `process.env.DB_FILE`) so it points to the container's mounted data directory.


3. **Create Multi-Stage Dockerfile:**
Add a `Dockerfile` at the monorepo root that uses `pnpm` to compile the React/Vite frontend, install backend production dependencies, and bundle both into a single lightweight Node.js runtime image.


4. **Create docker-compose.yml:**
Define the container service, map the selected port, set environment variables, and create a host bind mount (`./data:/app/server/data`) to persist your SQLite database file across container restarts.


5. **Commit and Push Changes to GitHub:**
Save and push all new configuration files, Docker specifications, and backend routing updates to your `development` branch.


6. **Clone or Pull Code onto Server:**
SSH into your HP server and pull the updated codebase into your designated project directory.


7. **Build and Launch the Container:**
Run `docker compose up -d --build` on the server to build the multi-stage image and launch your application service in the background.


8. **Verify Local Access and Database Persistence:**
Test access via `[http://192.168.178.200:5000](http://192.168.178.200:5000)`, perform a test write action, and verify that the SQLite database file is created inside the host's `./data` folder.
