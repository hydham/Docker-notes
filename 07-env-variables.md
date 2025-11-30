# Docker Environment Variables with Express & Node

These notes follow the teacher’s flow, including the small “mistakes” and corrections, so it reads like a story of how we learned it, not just a dry cheat sheet.

We already have:

- A simple Express app in **`index.js`**.
- A Docker image that runs that app.
- A running container we can hit from the browser.

Now we want to **control the port using environment variables** in Docker.

---

## 1. Reminder: Express app reads `process.env.PORT`

In the video, we already had an Express app like this:

```js
const express = require("express");
const app = express();

// Use PORT from environment if it exists, otherwise fall back to 3000
const port = process.env.PORT || 3000;

app.get("/", (req, res) => {
  res.send("<h2>Hi there</h2>");
});

app.listen(port, () => {
  console.log(`Listening on port ${port}`);
});
```

Key idea:

- The server does **not** hard‑code the port.
- It first checks **`process.env.PORT`**.
- If **`PORT`** isn’t set, it defaults to **`3000`**.

So Docker’s job now is to **set that `PORT` env variable inside the container**.

---

## 2. Set a default `PORT` in the Dockerfile with `ENV`

The teacher starts by wiring a default port inside the image itself.

In **`Dockerfile`** he adds an `ENV` instruction after copying dependencies:

```dockerfile
FROM node:15
# ^ Base image: official Node 15 image from Docker Hub

WORKDIR /app
# All subsequent commands will run in /app inside the container

COPY package.json .
# Copy only package.json first (layer + caching optimization)

RUN npm install
# Install dependencies from package.json

ENV PORT=3000
# Set default environment variable PORT inside the image
# Any container created from this image will see PORT=3000
# In Node this appears as process.env.PORT

EXPOSE ${PORT}
# Document that this image expects PORT to be open
# (EXPOSE does NOT actually open ports; it’s for humans / tooling)
# Using ${PORT} keeps this in sync if we change the ENV above

COPY . .
# Copy the rest of the app source into /app

CMD ["npm", "run", "dev"]
# At container runtime, run "npm run dev"
# (which we previously configured to start nodemon / the app)
```

Important points from this step:

- `ENV PORT=3000` sets a **default**.
- If we do nothing else, the app will listen on `3000` inside the container.
- Our earlier Express code’s `process.env.PORT || 3000` will now see `PORT=3000` and use that.

The teacher makes it clear: **this alone doesn’t change behavior** compared to before, because we were *already* effectively using port `3000`. Now it’s just explicit and configurable.

---

## 3. Rebuild the image after changing the Dockerfile

Because we changed the Dockerfile (added `ENV` and changed `EXPOSE`), we must rebuild:

```bash
docker build -t node-app-image .
# docker          -> Docker CLI
# build           -> builds an image from a Dockerfile
# -t node-app-image -> tag (name) for the built image
# .               -> build context is current directory (where Dockerfile lives)
```

Notes, aligned with the teacher’s earlier caching explanation:

- Docker sees that instructions changed, so it may invalidate some layers.
- It re-runs the necessary steps (especially anything after a changed line).
- Once done, the image now **bakes in** `ENV PORT=3000`.

---

## 4. Running the container with the default `PORT`

If we run the container **without overriding** the environment variable, it behaves the same as before:

```bash
docker run -d --name node-app -p 3000:3000 node-app-image
# docker run          -> create & start container
# -d                  -> run in detached mode (in background)
# --name node-app     -> name for the container
# -p 3000:3000        -> hostPort:containerPort (forward host 3000 -> container 3000)
# node-app-image      -> image to run (the one we just built)
```

- Inside the container:
  - `PORT` = `3000` (from `ENV` in the Dockerfile).
  - Express uses `process.env.PORT` → `3000`.
  - Our `-p 3000:3000` forwards host `3000` → container `3000`.

Result: visiting **`http://localhost:3000`** still works like before.

This step feels “boring” on purpose: nothing (visibly) changes yet. The real power comes when we **override** the environment variable at runtime.

---

## 5. Overriding `PORT` at runtime using `--env` / `-e`

The teacher’s next move is: “Let’s change the port the app listens on **without changing the code or Dockerfile again**.”

We override `PORT` when running the container:

```bash
docker run -d   --name node-app   -p 3000:4000   -e PORT=4000   node-app-image
# -p 3000:4000 -> host 3000 -> container 4000
# -e PORT=4000 -> override PORT env var inside the container
```

What this does:

- The container’s **`PORT`** is now `4000` (overriding the Dockerfile’s `ENV PORT=3000`).
- Express sees `process.env.PORT = "4000"` → listens on `4000` **inside** the container.
- `-p 3000:4000` sends **host port 3000** → **container port 4000**.

So:

- Browser sends request to: `http://localhost:3000` (host).
- Docker forwards that to container `4000`.
- Express is listening there, so it responds.

The teacher also shows that you can use either:

- `--env PORT=4000`
- or the short form: `-e PORT=4000`

They mean the same thing.

---

## 6. Verifying the environment variable inside the container

To confirm that `PORT` is actually set to `4000` inside the container, we can exec into it:

```bash
docker exec -it node-app bash
# docker exec  -> run a command in a running container
# -it          -> interactive + TTY (so we get a usable shell)
# node-app     -> name of the container
# bash         -> the command to run (bash shell)
```

Now inside the container, the teacher uses:

```bash
printenv | grep PORT
# printenv -> prints all environment variables
# | grep PORT -> filter to only lines containing "PORT"
```

He sees:

```text
PORT=4000
```

This confirms that:

- The Dockerfile’s `ENV PORT=3000` was the default.
- The `-e PORT=4000` on `docker run` **overrode** that when the container started.

---

## 7. Passing many environment variables with an `.env` file

The teacher then points out a pain point:

- Real apps have **many** environment variables.
- Doing `-e VAR1=... -e VAR2=... -e VAR3=...` becomes annoying and error-prone.

So he introduces the idea of an **env file**, usually named `.env`.

### 7.1 Create `.env` file

On the host (same folder as Dockerfile), create a `.env` file:

```text
PORT=4000
# In real apps, you would also add things like:
# DB_HOST=db
# DB_USER=myuser
# DB_PASS=super-secret
# NODE_ENV=production
```

This is a simple **key=value** file, one per line.

The teacher uses just `PORT=4000` for the demo.

### 7.2 Run container using `--env-file`

Instead of passing `-e PORT=4000` on the command line, we can load from `.env`:

```bash
docker run -d   --name node-app   -p 3000:4000   --env-file ./.env   node-app-image
# --env-file ./.env -> load all key=value pairs from this file
```

Now Docker:

- Reads `.env`.
- Sets each key/value as environment variables inside the container.
- Express again sees `process.env.PORT = "4000"`.

### 7.3 Verify inside container (again)

As usual, we can check:

```bash
docker exec -it node-app bash
printenv | grep PORT
```

We expect to see:

```text
PORT=4000
```

Which matches the `.env` file.

---

## 8. Summary of what we learned in this chapter

This chapter’s “story” in plain words:

1. **Express already supported `process.env.PORT`**, defaulting to `3000`.
2. In the **Dockerfile**, we added:
   - `ENV PORT=3000` → a default inside the image.
   - `EXPOSE ${PORT}` → a documentation hint using the same value.
3. We **rebuilt** the image using `docker build -t node-app-image .`.
4. We ran the container:
   - First without overrides, still listening on `3000`.
   - Then with `-e PORT=4000` and `-p 3000:4000` to change the container’s port.
5. We **verified** `PORT` inside the container using `docker exec` + `printenv`.
6. Finally, we learned to use an **`.env` file** + `--env-file` when we have many env vars.

The deeper idea:

- The code doesn’t care where the port comes from.
- The Dockerfile sets a **sensible default** (`ENV PORT=3000`).
- At runtime, we can override that via:
  - single `-e` flags, or
  - a shared `.env` file with `--env-file`.
- Environment variables are the standard way to pass configuration into containers, without hard-coding secrets or environment-specific values into the source code.
