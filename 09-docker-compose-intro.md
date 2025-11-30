# Docker Compose for Dev: Replacing Long `docker run` Commands

These notes follow the teacher’s **story flow**, including his “mistakes” and thought process, not just a cheat‑sheet. Idea is: when you re‑read later, you remember *why* each thing exists.

---

## 1. The problem: our `docker run` monster

Up to now, we’ve been starting our dev container with a **huge `docker run` command**:

```bash
docker run   -p 3000:3000 \          # map localhost:3000 -> container:3000
  -v ./:/app:ro \         # bind mount project -> /app (read-only)
  -v /app/node_modules \  # anonymous volume so bind mount can't delete node_modules
  --name node-app \       # name the container
  node-app-image          # image to run
```

For **one** Node/Express container, it’s already long but manageable.

But in a real project you might also have:

- a **database** container (Postgres / Mongo / MySQL)
- a **Redis** container
- maybe **Elasticsearch**, etc.

Each one would need its own `docker run` command with correct flags:

- port mappings
- volumes
- env vars
- image names
- container names

Doing that **every time** is painful and error‑prone.

So we want a way to:

- describe all containers **once**
- start *everything* with **one command**
- stop *everything* with **one command**

That’s what **Docker Compose** gives us.

---

## 2. What is Docker Compose (at our level)?

Very simply:

- You create a file: **`docker-compose.yml`**
- Inside you describe your **services** (each service = one container)
- You write:
  - what image to build/use
  - which ports to map
  - which volumes to attach
  - env vars to set
- Then you run:
  - **`docker-compose up -d`** → builds images (if needed) and starts containers
  - **`docker-compose down`** → stops containers and cleans up

You no longer remember a huge `docker run …` command.  
You remember just:

```bash
docker-compose up -d
docker-compose down -v
```

---

## 3. Creating the `docker-compose.yml` file

He creates **`docker-compose.yml`** in the project root.

### 3.1 Compose version

At the top we specify the Compose file format version:

```yaml
version: "3"
```

Notes:

- There are multiple versions (2, 2.4, 3, 3.8, etc.).
- Each version supports different features.
- For our simple use case, `"3"` is perfectly fine.

He even mentions that Docker docs have a matrix of features per version, but we don’t need that now.

### 3.2 The `services` section

In Compose:

- **Each container** is defined as a **service**.
- We list services under a top‑level key: `services:`

We currently have only **one** container: our Node app.  
So our basic skeleton becomes:

```yaml
version: "3"

services:
  node-app:  # name of the service (you choose this)
    # config goes here
```

Important detail (from the teacher): **YAML is indentation‑sensitive**.

- Keys under `services:` need to be indented.
- Keys under `node-app:` need another level of indentation.
- Wrong spacing → file becomes invalid.

He uses tabs visually, but in real YAML you should prefer **spaces** (2 is common). VS Code can convert tabs to spaces for you.

---

## 4. Translating our old `docker run` into Compose

Let’s carefully migrate each piece of the long `docker run` into `docker-compose.yml`.

Our old run command (conceptually) was:

```bash
docker run   -p 3000:3000   -v ./:/app:ro   -v /app/node_modules   --name node-app   node-app-image
```

We’ll capture this under the single `node-app` service.

### 4.1 Build the image (`build:`)

Instead of manually doing `docker build -t node-app-image .`,  
Compose can **build the image for us**.

So inside `node-app` we add:

```yaml
services:
  node-app:
    build: .
```

Explanation:

- `build: .` means:
  - Use the **current directory** (where `docker-compose.yml` lives)
  - Look for a `Dockerfile` there
  - Run `docker build` with that as the build context

Compose will **auto‑generate** an image name internally (we’ll see this shortly).

### 4.2 Ports mapping (`ports:`)

We previously had:

- `-p 3000:3000` → host 3000 → container 3000

In Compose:

```yaml
services:
  node-app:
    build: .
    ports:
      - "3000:3000"
```

This is the same semantics as `-p 3000:3000` in `docker run`.

You can also open multiple ports, e.g.:

```yaml
    ports:
      - "3000:3000"
      - "4000:4000"
```

We only need **one** for this app.

### 4.3 Volumes (`volumes:`)

We used two volumes:

1. A **bind mount** of the project into `/app` **read‑only**
2. An **anonymous volume** mounted on `/app/node_modules` to protect dependencies

In Compose:

```yaml
services:
  node-app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - "./:/app:ro"          # bind mount project -> /app (read-only)
      - "/app/node_modules"   # anonymous volume for node_modules
```

Line by line:

- `./:/app:ro`
  - `./` → project directory on host (where `docker-compose.yml` is)
  - `/app` → path inside container (our `WORKDIR` in `Dockerfile`)
  - `ro` → read‑only (container can’t modify our source files)
- `/app/node_modules`
  - No host path given → **anonymous volume**
  - Because volume paths are **more specific** than `/app`, this overrides the bind mount for `node_modules` and prevents it from being deleted when the host folder is missing `node_modules`

This recreates our manual hack from earlier, but **cleaner**.

### 4.4 Environment variables (`environment:` or `env_file:`)

Earlier with `docker run` we had something like:

- `--env PORT=3000`
- Or we used `--env-file .env`

In Compose we have **two options** as well:

#### Option A – Inline environment values

```yaml
services:
  node-app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - "./:/app:ro"
      - "/app/node_modules"
    environment:
      - PORT=3000
```

This is equivalent to doing `--env PORT=3000` on the CLI.

#### Option B – Load from `.env` file

If we have a `.env` file in the project root:

```env
PORT=3000
SOME_SECRET=value
```

We can load this via:

```yaml
services:
  node-app:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - "./:/app:ro"
      - "/app/node_modules"
    env_file:
      - .env
```

He shows this and then *comments it out* because we only had one variable, but it’s good to know for later.

In the video, he ultimately chooses the **inline `environment:`** form for simplicity.

---

## 5. Final `docker-compose.yml` for this chapter

Here is what the final Compose file looks like (for dev):

```yaml
version: "3"

services:
  node-app:
    build: .               # build image from the Dockerfile in current directory
    ports:
      - "3000:3000"        # host 3000 -> container 3000
    volumes:
      - "./:/app:ro"       # bind mount code (read-only) into /app
      - "/app/node_modules" # anonymous volume for node_modules
    environment:
      - PORT=3000          # env var visible as process.env.PORT inside Node
```

You can copy‑paste this into `docker-compose.yml` in your project.

---

## 6. Using Docker Compose: `up` and `down`

With the file written and saved, he runs:

```bash
docker-compose up -d
```

Breakdown:

- **`docker-compose`** → the Compose CLI
- **`up`** → create and start containers in the file
- **`-d`** → detached mode (don’t attach the terminal to logs)

What happens the **first time** you run `docker-compose up -d`:

1. **Builds the image** using `build: .`
   - You’ll see the familiar Dockerfile steps (Step 1/5, 2/5, etc.)
2. **Creates a network** for this project
   - Name is usually `<folder>_default`
   - This allows services to talk via DNS names (e.g. `node-app`, `db`) later
3. **Creates and starts the container** for `node-app`

You can verify with:

```bash
docker ps
```

You should see something like:

- container name: `node-docker_node-app_1`
  - `node-docker` → project directory name
  - `node-app` → service name
  - `_1` → first instance; if you scaled there might be `_2`, `_3`, etc.

And to test:

- Open browser → `http://localhost:3000`
- You should see your Express “Hi there” response
- Try editing `index.js` to add/remove exclamation marks and refreshing to confirm your bind mount + nodemon still work

### 6.1 Bringing everything down

To stop and clean up the containers created by Compose:

```bash
docker-compose down
```

This will:

- stop the containers in the Compose project
- remove the containers
- remove the network created for this project

**Important for volumes:**  
By default, `docker-compose down` **does not delete named/anonymous volumes** (similar to how `docker rm` doesn’t automatically delete volumes).

If you also want to delete volumes created by the services (like our anonymous `/app/node_modules` volumes), you can add `-v`:

```bash
docker-compose down -v
```

This is equivalent to:

- stop and remove containers
- remove the network
- remove associated volumes

He does exactly this: `docker-compose down -v` to fully clean up.

---

## 7. How Docker Compose names images and containers

After running `docker-compose up -d` with `build: .`, run:

```bash
docker image ls
```

You should see a built image with a name like:

- `node-docker_node-app`

Pattern is roughly:

- `<project-folder>_<service-name>`

Where:

- `project-folder` = name of your directory that has `docker-compose.yml`
- `service-name` = the key under `services:` (`node-app` in our file)

This matters later when we talk about rebuild behaviour.

---

## 8. Re-running `docker-compose up -d`: does it always rebuild?

This part is subtle and important.

### 8.1 What *you might* expect

You might assume:

- “If I change the Dockerfile and then run `docker-compose up -d` again, it will rebuild the image.”

### 8.2 What actually happens by default

What Compose actually does:

- It **checks if an image** with the expected name exists  
  (e.g. `node-docker_node-app:latest`)
- If it exists, **it does NOT rebuild** the image
- It just creates/starts containers from that **existing** image

So if you:

1. Build once with `docker-compose up -d`
2. Change your `Dockerfile` (e.g. change default PORT or CMD)
3. Run `docker-compose up -d` again

→ It will **NOT** rebuild the image.  
You will now be using a **stale image**.

He demonstrates this exactly:

- Changes a line in `Dockerfile`
- Runs `docker-compose up -d` again
- Build step is skipped; only container start happens

### 8.3 Forcing a rebuild: `--build`

To tell Compose: “Rebuild the image even if one already exists”, use:

```bash
docker-compose up -d --build
```

Now the steps are:

1. Rebuild image from Dockerfile
2. Recreate container(s)
3. Start them in detached mode

He shows this flow:

1. `docker-compose down` (or `down -v`)
2. Edit `Dockerfile` (e.g. change default `ENV PORT`)
3. Run:
   ```bash
   docker-compose up -d --build
   ```
4. See build logs again; image gets updated
5. App runs with new behaviour

This is a key habit:  
**Any time you change the Dockerfile (or anything in the build context that matters), run Compose with `--build`.**

---

## 9. Where this is going next

At the end, the teacher points out an important limitation:

Right now, our `docker-compose.yml` is **development‑focused**:

- We run **`npm run dev`** (nodemon) inside the container
- We use a **bind mount** (`./:/app:ro`) to sync code
- This is perfect for dev, but **not** what we want in production

In production we usually want:

- No bind mount (image contains the built app; containers are immutable)
- A different command, e.g.:
  - `npm start`
  - or `node index.js`
- Possibly different env vars (DB URLs, secrets, etc.)
- Maybe different port mapping or scaling

He says: in the **next section**, they will:

- Split dev and prod configuration
- Use separate Compose files (e.g. `docker-compose.yml` for dev + `docker-compose.prod.yml` for production overrides)
- Ensure prod uses proper commands & settings

So this chapter is **Compose basics**, focusing mainly on:

- converting the long `docker run` into a `docker-compose.yml`
- starting/stopping everything with simple commands
- understanding image naming and rebuild behaviour (`--build` flag)
- preparing for separate dev vs prod compose setups

---

## 10. Quick mental summary

If you only remember a few things from this chapter, remember these:

1. **`docker-compose.yml`** defines your containers as **services**.
2. Our dev file:

   ```yaml
   version: "3"

   services:
     node-app:
       build: .
       ports:
         - "3000:3000"
       volumes:
         - "./:/app:ro"
         - "/app/node_modules"
       environment:
         - PORT=3000
   ```

3. Bring everything up (build + start):

   ```bash
   docker-compose up -d
   ```

4. Tear everything down, including volumes:

   ```bash
   docker-compose down -v
   ```

5. If you **change the Dockerfile** and want a fresh image:

   ```bash
   docker-compose up -d --build
   ```

That’s the whole “Docker Compose for dev” chapter in one place, aligned with how the teacher explained and debugged it.
