# Docker: Dev vs Prod with One Dockerfile and Multiple Compose Files

These are my personal journey-style notes for setting up **two different environments**:

- Local **development** (with bind mounts, nodemon, live reload)
- **Production** (no bind mounts, no dev dependencies, clean image)

All of this is done using:

- **One `Dockerfile`**
- **Three compose files**:
  - `docker-compose.yml` → shared config
  - `docker-compose.dev.yml` → dev-only config
  - `docker-compose.prod.yml` → prod-only config

---

## 1. Why do we even need “dev vs prod” for Docker?

At first, everything lived in a single `docker run` command:

- Mounted code with a bind mount
- Anonymous volume for `node_modules`
- Port mappings
- Env vars
- `npm run dev` for nodemon

That works fine **just for development**.

But for a *real* app, we actually have different needs:

### Development needs

- **Bind mount**: sync local source ⇒ container `/app`
- **nodemon**: restart Node whenever code changes
- Command: `npm run dev`
- Lots of restarting, debugging, experimenting

### Production needs

- **No bind mount** (we don’t edit code in prod)
- No `nodemon` (we don’t hot-reload prod)
- Only **production dependencies** installed
- Command: `node index.js` (or `npm start`)
- Clean, repeatable image

So we need a way to:

- Use **one base definition** (Dockerfile, shared settings)
- But still have **different behavior** per environment

---

## 2. File structure I ended up with

Final layout looks roughly like this:

```text
project/
├─ index.js
├─ package.json
├─ package-lock.json
├─ Dockerfile
├─ docker-compose.yml           # shared config (dev + prod)
├─ docker-compose.dev.yml       # dev-only overrides
├─ docker-compose.prod.yml      # prod-only overrides
├─ .dockerignore
└─ ... (other app files)
```

The idea:

- `Dockerfile` → how to build the image (same for dev & prod)
- `docker-compose.yml` → common configs for all environments
- `docker-compose.dev.yml` → stuff **only dev** cares about
- `docker-compose.prod.yml` → stuff **only prod** cares about

And I pick which environment to run using `docker-compose -f ... -f ...`.

---

## 3. Shared Dockerfile for both dev and prod

We keep **one Dockerfile** and make it smart enough to behave differently
based on an argument `NODE_ENV`.

### 3.1 Dockerfile (final version)

```dockerfile
# 1. Base image
FROM node:15

# 2. Set working directory inside container
WORKDIR /app

# 3. Copy package.json first (to leverage Docker layer cache)
COPY package.json package-lock.json* ./

# 4. Build-time argument to control dev vs prod behavior
#    This is NOT the same as the runtime env var yet.
ARG NODE_ENV=development

# 5. Conditionally install dependencies
#    If NODE_ENV=development  -> install ALL deps (including devDependencies)
#    If NODE_ENV!=development -> install ONLY production deps
RUN if [ "$NODE_ENV" = "development" ]; then       echo "Installing ALL dependencies (dev + prod)";       npm install;     else       echo "Installing ONLY production dependencies";       npm install --only=production;     fi

# 6. Copy the rest of the app source code
COPY . .

# 7. Default runtime env var for the app (can be overridden by compose)
ENV PORT=3000

# 8. Expose is just documentation; actual access is via -p in docker / compose
EXPOSE $PORT

# 9. Default command (used mainly for prod; dev will override via compose)
CMD ["node", "index.js"]
```

Key ideas here:

- `ARG NODE_ENV` controls **build-time** behavior.
- `ENV PORT` is a **runtime** env var (app uses it via `process.env.PORT`).
- The `RUN if [ "$NODE_ENV" = "development" ] ...` block is a tiny Bash script
  that decides how to install dependencies.

So the Dockerfile can install:

- all deps when building dev images
- only prod deps when building prod images

---

## 4. `.dockerignore` – stop copying junk into the image

We don’t want to bloat our image with:

- Local `node_modules`
- Git stuff
- Compose files
- Editor configs, etc.

So we create a `.dockerignore`:

```dockerignore
node_modules
npm-debug.log
.git
.gitignore

# Ignore compose files
docker-compose*.yml

# Ignore local editor/project junk
.vscode
.idea
*.swp
```

This keeps the context small and avoids copying `docker-compose` files
and local `node_modules` into the image.

---

## 5. Shared `docker-compose.yml` – base config for all environments

This file contains **only what is common** to dev and prod.

```yaml
version: "3"

services:
  node-app:
    build:
      context: .
      # The actual NODE_ENV arg is overridden by dev/prod files
      args:
        NODE_ENV: development

    # Common port mapping (same in dev and prod here)
    ports:
      - "3000:3000"

    # Common runtime env vars
    environment:
      - PORT=3000
```

Important:

- We define the **service name** as `node-app`.
- We factor in the `build` section with `context` and `args`.
- Even though we set `NODE_ENV: development` here, **dev/prod override it**.

Think of this as the “base class”; dev/prod files will “extend/override” it.

---

## 6. Dev-only config: `docker-compose.dev.yml`

In dev, I want:

- Bind mount: local code → `/app`
- Anonymous volume: `/app/node_modules` (so bind mount doesn’t erase it)
- `NODE_ENV=development`
- `npm run dev` (nodemon) instead of plain `node index.js`

```yaml
version: "3"

services:
  node-app:
    # Override / extend build section from base
    build:
      context: .
      args:
        NODE_ENV: development

    # Dev-only volumes
    volumes:
      # Bind mount: local source -> container /app
      - ./:/app:ro
      # Anonymous volume to protect node_modules in container
      - /app/node_modules

    # Dev-only env vars
    environment:
      - NODE_ENV=development

    # Dev-only command (override Dockerfile CMD)
    command: ["npm", "run", "dev"]
```

Why two volumes?

- `- ./:/app:ro`
  - Mirror local folder into container at `/app`
  - The `:ro` makes it **read-only** from the container’s POV
- `- /app/node_modules`
  - Anonymous Docker volume attached only to `node_modules`
  - Prevents the bind mount from replacing `node_modules` with whatever
    you have locally (possibly nothing)

So in dev:

- Code changes locally ⇒ nodemon sees changes inside container ⇒ reloads
- `node_modules` are inside the container and *not* destroyed by the bind mount

---

## 7. Prod-only config: `docker-compose.prod.yml`

In prod, we want:

- No bind mounts
- No devDependencies
- `NODE_ENV=production`
- `node index.js` or `npm start`

```yaml
version: "3"

services:
  node-app:
    build:
      context: .
      args:
        NODE_ENV: production

    environment:
      - NODE_ENV=production

    # Use Dockerfile’s default CMD or override explicitly:
    command: ["node", "index.js"]
```

Notice:

- No `volumes:` section → no bind mounts.
- No `nodemon` here.
- Build arg `NODE_ENV=production` triggers `npm install --only=production`
  in the Dockerfile.

---

## 8. Running dev vs prod with multiple compose files

Here’s how Docker Compose decides configuration:

- You can pass **multiple `-f`** files.
- Compose merges them **in order**.
- Later files override earlier ones.

So:

- Base file → `docker-compose.yml`
- Then environment file → `docker-compose.dev.yml` or `.prod.yml`

### 8.1 Start dev environment

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   up -d --build
```

Explanation:

- `-f docker-compose.yml` → load shared config
- `-f docker-compose.dev.yml` → apply dev overrides
- `up -d` → start in detached mode
- `--build` → force rebuild if Dockerfile / build args changed

Now you can:

```bash
docker ps           # see running container
curl localhost:3000 # or open in browser
```

Edit `index.js`, hit refresh, and nodemon should reload.

### 8.2 Stop dev environment

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   down -v
```

Notes:

- `down` → stop and remove containers + network
- `-v` → also remove anonymous volumes created by this compose stack
  (e.g., `/app/node_modules`)

---

### 8.3 Start prod environment

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   up -d --build
```

What changes vs dev?

- `NODE_ENV=production` passed as build arg
- Dockerfile executes `npm install --only=production`
- Command = `node index.js`
- No bind mounts; code is baked into the image

### 8.4 Stop prod environment

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   down -v
```

Same idea: tears down containers, networks, and volumes.

---

## 9. Sanity checks inside the container

Just to prove everything works as intended:

### 9.1 Check env vars and node modules in dev

```bash
# Attach shell to dev container
docker exec -it <dev-container-name> bash

printenv | grep NODE_ENV
# → NODE_ENV=development

ls node_modules | grep nodemon
# → nodemon (should exist in dev)
```

### 9.2 Check env vars and node modules in prod

```bash
# Attach shell to prod container
docker exec -it <prod-container-name> bash

printenv | grep NODE_ENV
# → NODE_ENV=production

ls node_modules | grep nodemon
# → (no output; nodemon should NOT be installed)
```

If `nodemon` doesn’t show up in prod, it means:

- The `ARG NODE_ENV=production` was passed correctly
- The Dockerfile `RUN` condition worked
- Only production dependencies were installed

---

## 10. Quick mental model

I keep this picture in my head:

- **Dockerfile**:
  - “How to bake a cake (image)”
  - `ARG NODE_ENV` decides which ingredients (deps) we put in.

- **docker-compose.yml**:
  - “Basic recipe card”
  - Shared instructions: **which oven, what temp, base settings**.

- **docker-compose.dev.yml**:
  - “When I bake at home (dev)”
  - Extra stuff: live reload, read-only mount from my laptop, extra tools.

- **docker-compose.prod.yml**:
  - “When I bake for customers (prod)”
  - Stripped down, only what’s needed, no dev overhead.

Run dev:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

Run prod:

```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Same project, same Dockerfile — just different “modes” driven by compose and build args.

---
