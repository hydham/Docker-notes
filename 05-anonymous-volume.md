
# Docker Dev Workflow: Bind Mount Bug & Protecting `node_modules` with an Anonymous Volume

We now have a nice Docker dev workflow:

- Our app runs inside a container.
- A bind mount keeps our code in sync between local machine and container.
- **nodemon** restarts the Node process when files change.

Everything seems perfect… until we *intentionally* break it to understand a subtle problem.

---

## Breaking the App on Purpose

First, we **delete the running container**:

```bash
# Remove the container named "node-app" even if it's running
docker rm node-app -f
```

Then, **on the local machine**, we delete the `node_modules` folder in the project:

- Delete the local `node_modules` folder that lives next to `index.js` and `package.json`.

> Idea: Since our dependencies are installed inside the Docker image, we *think* we no longer need local `node_modules` for development.

Now we **re-run the container** using the same bind mount command as before (something like):

```bash
# Example dev run command (conceptual)
docker run \
  -p 3000:3000 \              # forward host 3000 -> container 3000
  -v /path/to/project:/app \  # bind mount local project folder into /app
  --name node-app \
  node-app-image
```

If we open the browser and hit the app (e.g. `http://localhost:3000`), the page spins and eventually errors.

Something is broken.

---

## First Clue: `docker ps` vs `docker ps -a`

We check if the container is running:

```bash
# Show only running containers
docker ps
```

The container **does not** appear. That’s suspicious.

> Reminder: `docker ps` only shows containers that are currently **running**.

To see *all* containers (running, stopped, crashed):

```bash
# Show all containers (running + exited)
docker ps -a
```

Now we see `node-app` with a status like:

> `Exited (1) 30 seconds ago`

So the container started, crashed, and exited.

---

## Second Clue: Reading Container Logs

To see why it crashed, we inspect the logs:

```bash
# Show logs for the node-app container
docker logs node-app
```

We see an error similar to:

> `nodemon: command not found`  
> or  
> `nodemon not found`

So inside the container, when it tries to run `npm run dev`, it can’t find **nodemon**.

But earlier everything worked fine. We installed `nodemon` and rebuilt the image. The only thing we changed was **deleting local `node_modules`**.

So why is `nodemon` now missing *inside* the container?

---

## How the Image Was Built

Let’s remind ourselves how the image is built (simplified Dockerfile):

```dockerfile
FROM node:15

WORKDIR /app

# 1. Copy package metadata
COPY package.json .

# 2. Install dependencies (this creates /app/node_modules inside the image)
RUN npm install

# 3. Copy the rest of the source code into /app
COPY . .

# 4. Start command (for dev)
CMD ["npm", "run", "dev"]
```

Key point:  
After the image is built, the image *does* contain `/app/node_modules` with `nodemon` installed.

So the bug must come from what happens **at runtime**, not at build time.

---

## How the Bind Mount Overwrites `node_modules`

In development we run the container with a **bind mount**:

```bash
docker run \
  -p 3000:3000 \
  -v /path/to/project:/app \
  --name node-app \
  node-app-image
```

What does this mean?

- `-v /path/to/project:/app` says:
  - Take the local folder `/path/to/project` on the **host**
  - Mount it **into the container** at `/app`
  - So the container’s `/app` now shows whatever is in your local project folder.

Now replay the sequence:

1. The image contains `/app/node_modules` from the `RUN npm install` step.
2. We start the container **with a bind mount** on `/app`.
3. The bind mount makes `/app` inside the container mirror the host folder.
4. On the host, we **deleted `node_modules`**.
5. Therefore, inside the container, `/app/node_modules` also disappears (because `/app` is now just mirroring the host project directory, which no longer has that folder).

End result:

- `node_modules` is gone **inside** the container.
- When the container runs `npm run dev`, it tries to execute `nodemon`, but `nodemon` doesn’t exist anymore.
- Container crashes → `nodemon not found`.

> The bind mount is “too strong”: by replacing `/app` with the host folder, it overwrote the `node_modules` that was baked into the image.

---

## Using an Anonymous Volume to Protect `/app/node_modules`

We want **two things** at the same time:

1. A bind mount for our source code so that changes on the host are instantly reflected in the container:
   - `-v /path/to/project:/app`

2. Protection for `/app/node_modules` so it **stays** inside the container and is **not** overwritten when the host has no `node_modules`:
   - some volume on `/app/node_modules` that is independent of the host.

Docker has a helpful behavior:

> When multiple volumes target overlapping paths, the **more specific** path wins.

Examples:

- `/app` → generic
- `/app/node_modules` → more specific (a deeper subfolder)

So the trick is:

- Keep the bind mount on `/app` for code.
- Add **another volume** specifically on `/app/node_modules` so that path is protected.

We do that by adding another `-v`:

```bash
docker run \
  -p 3000:3000 \
  -v /path/to/project:/app \     # bind mount for code
  -v /app/node_modules \         # anonymous volume just for node_modules
  --name node-app \
  node-app-image
```

Notes:

- The second `-v /app/node_modules` **does not** specify a host path.
- That means Docker creates and manages an **anonymous volume** for `/app/node_modules`.
- Because `/app/node_modules` is **more specific** than `/app`, the anonymous volume wins for that subfolder, and the bind mount cannot wipe it out.

Think of it like this:

- `/app` = big box mounted from host
- `/app/node_modules` = smaller box inside big box, controlled by Docker
- Docker says: “For the smaller, more specific path, I’ll use this other volume instead of the host.”

So:

- `/app` comes from your host project directory (code, config, etc.).
- `/app/node_modules` comes from the container volume (dependencies installed during `npm install`).

---

## Fixing the Crash Step by Step

1. **Delete the broken container:**

   ```bash
   docker rm node-app -f
   ```

2. **Re-run the container** with **both** volumes:

   ```bash
   docker run \
     -p 3000:3000 \
     -v /path/to/project:/app \
     -v /app/node_modules \
     --name node-app \
     node-app-image
   ```

3. Check that it’s running:

   ```bash
   docker ps
   ```

   You should see `node-app` with a `Up` status (not `Exited`).

4. Test in the browser:

   - Go to `http://localhost:3000`
   - The app should load normally.

5. Change the code in `index.js` (e.g. add/remove exclamation marks), save, and refresh:

   - The bind mount syncs the new code into `/app`.
   - `nodemon` (from `/app/node_modules`) restarts the Node process.
   - The browser shows the updated response.

Now we have a dev setup where:

- Code is live-synced via bind mount.
- Dependencies live inside the container via anonymous volume.
- Deleting local `node_modules` no longer breaks the container.

---

## Why We Still Need `COPY . .` in the Dockerfile

You might ask:

> “If in dev I mount the project folder into `/app`, do I still need `COPY . .` in the Dockerfile?”

**Yes.** Here’s why:

- The bind mount is a **development-only** trick:
  - It’s for fast iteration, live reload, simple editing.
- In **production**, we usually do **not** bind mount our source code:
  - We run containers purely from the image.
  - There is no local project directory to mount.

So in production:

- The container’s filesystem is exactly what the image contains.
- No bind mounts, no dev-only volumes for syncing code.

Therefore, your image must already include all application code and dependencies. That’s the job of:

```dockerfile
COPY package.json .
RUN npm install
COPY . .
```

These lines ensure:

- All dependencies are installed in the image.
- All source files are baked into the image.

Dev path:

- Image + bind mount + nodemon + anonymous volume on `/app/node_modules`.

Prod path:

- Image only (or composed via Docker Compose/Swarm/Kubernetes).
- No bind mount for code.
- No dev auto-reload.

So `COPY . .` stays, because it’s critical for **production**.

---

## Summary

In this chapter we learned:

- **Bind mounts** are powerful but can accidentally **overwrite** files inside the container, including `node_modules`.
- Deleting local `node_modules` and then using a bind mount on `/app` causes `/app/node_modules` to disappear inside the container, breaking `nodemon`.
- Docker resolves multiple volumes by **specificity**; a path like `/app/node_modules` is more specific than `/app`.
- We can protect dependencies by using an **anonymous volume** on `/app/node_modules` while still using a bind mount on `/app` for code.
- Even with bind mounts in dev, we still need `COPY . .` in the Dockerfile so the image is self-contained for production.

This gives us a safe and ergonomic dev setup:
- Live code changes via bind mount,
- Auto-restart via nodemon,
- Stable dependencies inside the container via an anonymous volume on `/app/node_modules`.
