# Docker Dev vs Prod with One Dockerfile and Multiple Compose Files (Story Mode)

By this point in the course, I’ve already got:

- A small **Node/Express app** (`index.js`)
- A **Dockerfile** that can build an image for it
- Volumes and bind mounts for live-reload in dev
- A basic **`docker-compose.yml`** that starts the app container

Now I want to do something more “real world”:

> **Run the same app in two different environments — dev and prod — without maintaining two completely separate Docker setups.**

I *could* just copy-paste and create separate Dockerfiles and separate compose files and keep editing them forever…  
but future-me would hate that. So instead, I’m going to:

- Keep **one Dockerfile**
- Use **three docker-compose files**:
  - `docker-compose.yml` → **shared** stuff for all environments
  - `docker-compose.dev.yml` → dev-only settings
  - `docker-compose.prod.yml` → prod-only settings
- Use **build args** + **`NODE_ENV`** to control how dependencies are installed.

Let’s walk through the whole journey like I’m sitting at the keyboard, messing with it step by step.

---

## 1. What’s different between dev and prod?

First, I think through the differences.

### In **development** I want:

- **Bind mount**: so code changes on my host instantly appear in the container.
- **`nodemon`**: automatically restart the app when files change.
- `NODE_ENV=development`
- Install **all dependencies**, including dev dependencies:
  - `npm install`

### In **production** I want:

- **No bind mount**: the container just runs a frozen image.
- No `nodemon`: no auto-restart, just a plain Node process.
- `NODE_ENV=production`
- Install **only production deps**:
  - `npm install --only=production`
- Run app with **`node index.js`** (or `npm start` if I want).

So the idea is:

> Same Dockerfile, but behave differently depending on whether I’m building for dev or prod.

---

## 2. Splitting the compose config: base + dev + prod

Instead of keeping everything in one giant `docker-compose.yml`, I split it into:

- **`docker-compose.yml`** → shared config between dev and prod
- **`docker-compose.dev.yml`** → extra stuff only for dev
- **`docker-compose.prod.yml`** → extra stuff only for prod

I also rename my old compose file to keep for reference:

```bash
mv docker-compose.yml docker-compose.backup.yml
```

Now I create a fresh **base** compose file.

### `docker-compose.yml` (base shared config)

This file only contains things that are the **same in dev and prod**:

- Which services exist (`node-app`)
- What image to build from the Dockerfile
- Shared ports
- Shared environment variables that don’t change

```yaml
# docker-compose.yml
version: "3"

services:
  node-app:
    build: .             # build using the Dockerfile in this directory
    ports:
      - "3000:3000"      # host 3000 -> container 3000
    environment:
      - PORT=3000        # Express uses process.env.PORT || 3000
```

So far this is pretty normal — this is like my original simple compose file.

---

## 3. Dev-specific compose file: bind mounts + nodemon

Now I add **`docker-compose.dev.yml`** and only put dev-specific things here:

- Bind mount for live-reload
- The anonymous volume for `node_modules` hack
- `NODE_ENV=development`
- Use `npm run dev` (which runs `nodemon`)

```yaml
# docker-compose.dev.yml
version: "3"

services:
  node-app:
    # Override/extend the base service
    volumes:
      - ./:/app:ro              # bind mount project folder (read-only from container)
      - /app/node_modules       # anonymous volume for node_modules

    environment:
      - NODE_ENV=development    # dev mode

    command: npm run dev        # use nodemon in dev

    build:
      context: .                # location of Dockerfile
      args:
        NODE_ENV: development   # build-arg passed to Dockerfile
```

The important part is that `build` section: here I pass a **build argument** (`NODE_ENV`) that the Dockerfile will later use to decide how to install dependencies.

---

## 4. Prod-specific compose file: no bind mount, prod command

Now for **`docker-compose.prod.yml`** — this is what changes for production:

- No volumes (no bind mount, no local file syncing)
- `NODE_ENV=production`
- `command: node index.js`
- `build.args.NODE_ENV=production`

```yaml
# docker-compose.prod.yml
version: "3"

services:
  node-app:
    environment:
      - NODE_ENV=production     # production mode

    command: node index.js      # run plain Node in prod

    build:
      context: .                # same Dockerfile
      args:
        NODE_ENV: production    # build-arg for prod build
```

So now:

- Both dev and prod **reuse the same service name** (`node-app`).
- Both inherit from **`docker-compose.yml`**.
- Dev overrides with its own volumes, command, env, and build args.
- Prod overrides with its own env, command, and build args.

---

## 5. Making the Dockerfile smart: dev vs prod install

Now comes the fun part: the **Dockerfile** needs to react to `NODE_ENV`.

At first, it probably looked like this:

```dockerfile
FROM node:15

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

CMD ["npm", "run", "dev"]   # or ["node", "index.js"]
```

But now I want:

- In **dev build**: install everything (including devDeps).
- In **prod build**: install only prod deps (`--only=production`).

To do that, I add:

1. A **build argument**: `ARG NODE_ENV`
2. An **if/else** shell script inside `RUN` to choose how to install.

Here’s the updated Dockerfile:

```dockerfile
# Dockerfile

FROM node:15

WORKDIR /app

# Copy only dependency manifests first (cache-friendly)
COPY package*.json ./

# Build argument from docker-compose (NODE_ENV)
ARG NODE_ENV

# Conditional install depending on NODE_ENV
RUN if [ "$NODE_ENV" = "development" ]; then       echo "Running dev install (all dependencies)";       npm install;     else       echo "Running production install (prod dependencies only)";       npm install --only=production;     fi

# Copy the rest of the source code
COPY . .

# Default command (can be overridden by docker-compose)
CMD ["node", "index.js"]
```

A couple of **important shell gotchas**:

- There must be spaces around `[` and `]`:
  - ✅ `if [ "$NODE_ENV" = "development" ]; then`
  - ❌ `if ["$NODE_ENV"="development"];then`
- Use `\` at end of each line to continue the command.
- Use `; \` between commands inside the `RUN` block.

Now dev vs prod behavior is fully controlled by **`NODE_ENV` build arg**.

---

## 6. Wiring up dev & prod commands (docker-compose with multiple files)

Now the question: how do I actually **run** this thing?

Docker Compose can merge multiple files. The order matters.

General pattern:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
docker-compose -f docker-compose.yml -f docker-compose.dev.yml down -v
```

For production:

```bash
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker-compose -f docker-compose.yml -f docker-compose.prod.yml down -v
```

Note:

- `-f docker-compose.yml` → base config (shared)
- `-f docker-compose.dev.yml` → overrides for dev
- `-f docker-compose.prod.yml` → overrides for prod
- `up -d` → start in detached mode
- `down -v` → stop and also remove volumes

### Why order matters

Docker Compose merges files **from left to right**:

- Start with `docker-compose.yml`
- Then merge `docker-compose.dev.yml` or `.prod.yml`, which can override values.

If you reversed the order, dev/prod wouldn’t be able to override the base correctly.

---

## 7. Docker Compose caching and stale images

Now, here’s a subtle trap.

When I first run:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

Compose:

- Builds the image (with name like `foldername_node-app`)
- Starts the container

Later, I modify the **Dockerfile** or something important like this:

```dockerfile
RUN if [ "$NODE_ENV" = "development" ]; then ...
```

I run the same compose command again… and it’s **instant**. Suspiciously instant.

Why? Because by default:

> Docker Compose will **reuse** an existing image if its name already exists.  
> It doesn’t automatically detect changes in your Dockerfile or code.

So to force a rebuild, I add `--build`:

```bash
# Dev, with rebuild
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   up -d --build

# Prod, with rebuild
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   up -d --build
```

Now Compose is forced to rebuild, honoring the updated Dockerfile and build args.

---

## 8. Cleaning up the Docker context: dockerignore fixes

At some point I exec into the container:

```bash
docker exec -it <container-name> bash
ls
```

And I notice something annoying:

- All my `docker-compose*` files are **inside the image**.

They’re not needed there; they’re only useful on the host side. So I add them to **`.dockerignore`**:

```dockerignore
node_modules
npm-debug.log
Dockerfile
docker-compose*
.dockerignore
.git
.gitignore
```

The important bit is:

```dockerignore
docker-compose*
```

The `*` means: “any file whose name starts with `docker-compose`” — that covers:

- `docker-compose.yml`
- `docker-compose.dev.yml`
- `docker-compose.prod.yml`
- Any future variants

That keeps the image smaller and cleaner.

---

## 9. Verifying that dev installs nodemon, prod does not

Big moment: confirm that the **conditional install** actually works.

### Step 1: Dev build

Run dev stack with rebuild:

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   up -d --build
```

Then exec into the container:

```bash
docker exec -it node-docker_node-app_1 bash
cd node_modules
ls
```

I should see:

- `nodemon` present
- Dev deps installed

Also, if I change `index.js`, `/app` is synced via bind mount, nodemon restarts, and the browser reflects the changes immediately.

So dev behaves exactly as expected.

Then bring it down:

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   down -v
```

### Step 2: Prod build

Now run prod with rebuild:

```bash
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   up -d --build
```

Exec into the container:

```bash
docker exec -it node-docker_node-app_1 bash

cd node_modules
ls
```

This time:

- There should **not** be a `nodemon` folder
- Only production deps are present

Also, if I:

- Edit `index.js` on my host
- Refresh the page

Nothing changes — because:

- No bind mount in prod
- No nodemon
- Code inside the container is frozen to what got built into the image

Finally, I check the env:

```bash
printenv | grep NODE_ENV
```

I should see:

```text
NODE_ENV=production
```

That confirms:

- `NODE_ENV` build arg was set correctly
- Conditional `RUN` in Dockerfile used the right branch
- Compose env and build args are in sync

---

## 10. Final mental model: how everything fits together

Here’s the “movie in my head” of what’s going on.

### Dev flow

1. I run:

   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
   ```

2. Compose:
   - Merges base + dev
   - Passes `build.args.NODE_ENV=development` to Dockerfile
3. Dockerfile:
   - `ARG NODE_ENV`
   - `if [ "$NODE_ENV" = "development" ]; then npm install; else npm install --only=production; fi`
   - Installs **all** deps, including `nodemon`
4. `docker-compose.dev.yml`:
   - Adds bind mounts (`.:/app`)
   - Adds anonymous volume (`/app/node_modules`)
   - Sets `NODE_ENV=development` in the running container
   - Sets `command: npm run dev`
5. Result:
   - Container has all deps + nodemon
   - File changes on host → mirrored in `/app`
   - Nodemon restarts on change → browser shows updates instantly

### Dev flow

1. I run:

   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
   ```

2. Compose:
   - Merges base + prod
   - Passes `build.args.NODE_ENV=production` to Dockerfile
3. Dockerfile:
   - Sees `NODE_ENV=production`
   - Runs `npm install --only=production`
   - **Does not** install dev deps like `nodemon`
4. `docker-compose.prod.yml`:
   - Didn’t define any volumes → no bind mounts
   - Sets `NODE_ENV=production` in container
   - Sets `command: node index.js`
5. Result:
   - Production image is smaller (no dev deps)
   - Container runs plain Node
   - Host file changes do nothing — as they should in prod

---

## 11. Handy command cheat sheet (dev vs prod)

Just to keep it all in one place:

### Dev

```bash
# Start dev environment (with rebuild)
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   up -d --build

# Stop dev and remove containers + volumes
docker-compose   -f docker-compose.yml   -f docker-compose.dev.yml   down -v
```

### Prod

```bash
# Start prod environment (with rebuild)
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   up -d --build

# Stop prod and remove containers + volumes
docker-compose   -f docker-compose.yml   -f docker-compose.prod.yml   down -v
```

---

If you want, next step we can:

- Add a MongoDB container and plug it into this setup, or
- Refine the dockerignore, or
- Draw a little mini diagram of how compose file merging works.

But this is the core: **single Dockerfile, dev & prod flows cleanly separated by compose files + `NODE_ENV` + build args**, all behaving the way a real-world app would.
