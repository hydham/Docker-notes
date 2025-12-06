# Connecting Express + Mongoose to MongoDB in Docker (with Docker DNS & Networks) — A1 Notes

In this chapter I’m wiring up the Express app (inside one container) to the MongoDB database (inside another container), using **Mongoose** and **Docker’s internal DNS** instead of hard‑coding container IPs.

I’ll walk through exactly what’s happening, including the mistakes and the “wait, how do I get the IP?” moments.

---

## 1. Install Mongoose in the Node app

First thing: I want a nicer way to talk to MongoDB than using a bare driver, so I install **mongoose** in my Node project.

From my app folder (on the host):

```bash
npm install mongoose
```

This updates:

- **`package.json`** → adds `mongoose` to dependencies  
- **`package-lock.json`** → records the exact version + sub‑dependencies

Because I added a new dependency, the image that was built earlier is now **stale**. The container we’re running does **not** magically get the new `node_modules`.

So I:

1. Bring docker‑compose down (without `-v`, because we want to keep the DB volume!).
2. Bring it up again with a forced build.

```bash
# Stop and remove containers (but keep volumes!)
docker compose down

# Start everything again, rebuilding images because deps changed
docker compose up -d --build
```

- `down` → stops containers and removes them  
- **No `-v`** here, because that would delete the **named MongoDB volume**, and we’d lose data.  
- `up -d --build` → recreate containers in detached mode and **rebuild** any images referenced by `build:`.

Now the new image includes the updated `node_modules` with **mongoose** installed.

---

## 2. Require Mongoose in `index.js`

Inside my Express entry file (say `index.js`), I pull mongoose in:

```js
const mongoose = require('mongoose');
```

This doesn’t connect yet — it just loads the library so I can call `mongoose.connect(...)`.

---

## 3. Build the MongoDB connection string (the “IP address headache”)

Mongoose expects a MongoDB connection URI in this form:

```text
mongodb://USERNAME:PASSWORD@HOST:PORT/?authSource=admin
```

Rough structure:

- `mongodb://` → protocol  
- `USERNAME` → must match `MONGO_INITDB_ROOT_USERNAME` from the mongo container  
- `PASSWORD` → must match `MONGO_INITDB_ROOT_PASSWORD`  
- `HOST` → how to reach the MongoDB container (for now we’ll use IP, later we’ll replace it with DNS)  
- `PORT` → MongoDB port (default `27017`)  
- `authSource=admin` → tells Mongo to authenticate against the `admin` DB (where that root user lives)

So I start with something like:

```js
const mongoose = require('mongoose');

mongoose
  .connect(
    'mongodb://sanjeev:myPassword@172.25.0.2:27017/?authSource=admin'
  )
  .then(() => {
    console.log('Successfully connected to database');
  })
  .catch((err) => {
    console.error('Error connecting to database:', err);
  });
```

But of course, I don’t magically know `172.25.0.2`. That’s where Docker’s networking comes in.

---

## 4. Use `docker inspect` to find the MongoDB container IP (the ugly way)

At first, I do this the clumsy way: grab the container IP using `docker inspect`.

List running containers:

```bash
docker ps
```

I’ll see something like:

```text
CONTAINER ID   IMAGE               NAMES
abc123...      node_docker-node    node_docker-node-app-1
def456...      mongo               node_docker-mongo-1
```

Now inspect the **MongoDB** container:

```bash
docker inspect node_docker-mongo-1
```

This prints a giant JSON blob. I scroll down to the `NetworkSettings` section:

```json
"NetworkSettings": {
  "Networks": {
    "node_docker_default": {
      "IPAddress": "172.25.0.2",
      "Gateway": "172.25.0.1",
      ...
    }
  }
}
```

- `node_docker_default` → the custom Docker network that `docker compose` created for this project.
- `IPAddress` → the internal IP for that mongo container (e.g. `172.25.0.2`).

I plug that into the Mongoose URL:

```js
mongoose
  .connect(
    'mongodb://sanjeev:myPassword@172.25.0.2:27017/?authSource=admin'
  )
  .then(() => console.log('Successfully connected to database'))
  .catch((err) => console.error('Error connecting to database:', err));
```

Then I check logs from the Node container:

```bash
docker ps
# find the node app container name, e.g. node_docker-node-app-1

docker logs node_docker-node-app-1
```

If everything is good, I’ll see:

```text
Successfully connected to database
```

So: it works. But it’s **gross**.

Problems with hardcoding the IP:

- If the containers are recreated, the IP might change.
- I’d have to run `docker inspect` each time and edit my code.
- It’s brittle and not portable.

So we need something better.

---

## 5. Docker networks + DNS: stop caring about IPs

Docker uses **networks** to group containers. Run:

```bash
docker network ls
```

You’ll see something like:

```text
NETWORK ID     NAME                    DRIVER    SCOPE
...            bridge                  bridge    local
...            host                    host      local
...            node_docker_default     bridge    local
```

- `bridge` and `host` → default networks created by Docker
- **`node_docker_default`** → a custom network auto‑created by `docker compose` for this project

The magic: on **custom networks** like `node_docker_default`, Docker gives you **built‑in DNS**.

That means:

- Each service defined in `docker-compose.yml` gets a **DNS name** equal to its **service name**.
- Containers can talk to each other using that service name instead of an IP.

In our compose file, we probably have something like:

```yaml
services:
  node-app:
    build: .
    ...

  mongo:
    image: mongo
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev
      - MONGO_INITDB_ROOT_PASSWORD=myPassword
    ...
```

Here:

- Service name for the Node container: **`node-app`**
- Service name for the Mongo container: **`mongo`**

Inside the same Docker network, the Node app can reach Mongo at host **`mongo`**, on port `27017`.

So I can rewrite the Mongoose connect string as:

```js
mongoose
  .connect(
    'mongodb://sanjeev:myPassword@mongo:27017/?authSource=admin'
  )
  .then(() => console.log('Successfully connected to database'))
  .catch((err) => console.error('Error connecting to database:', err));
```

No IPs. Just the service name.

---

## 6. Proving that Docker DNS works (ping the service name)

To convince myself this isn’t magic, I hop into the **Node container** and ping `mongo`.

First get the container name:

```bash
docker ps
```

Then:

```bash
docker exec -it node_docker-node-app-1 bash
```

Now I’m inside the Node container’s shell.

Ping the mongo service:

```bash
ping mongo
```

I’ll see something like:

```text
PING mongo (172.25.0.2): 56 data bytes
64 bytes from 172.25.0.2: icmp_seq=0 ttl=64 time=0.123 ms
64 bytes from 172.25.0.2: icmp_seq=1 ttl=64 time=0.089 ms
...
```

So `mongo` got resolved to the correct IP by Docker’s internal DNS.

Key point:

- This DNS resolution (service name → container IP) **only works on custom networks** (like `node_docker_default`).
- It does **not** apply to the built‑in `bridge` network in the same way.
- `docker compose` created `node_docker_default` for us automatically and attached both containers to it.

---

## 7. Inspecting the custom Docker network

Just to see everything wired up nicely, I can inspect the custom network:

```bash
docker network inspect node_docker_default
```

In the output:

- Under `"IPAM"` I’ll see the subnet / gateway:

  ```json
  "IPAM": {
    "Config": [
      {
        "Subnet": "172.25.0.0/16",
        "Gateway": "172.25.0.1"
      }
    ]
  }
  ```

- Under `"Containers"` I’ll see both containers:

  ```json
  "Containers": {
    "abc123...": {
      "Name": "node_docker-mongo-1",
      "IPv4Address": "172.25.0.2/16",
      "MacAddress": "02:42:ac:19:00:02"
    },
    "def456...": {
      "Name": "node_docker-node-app-1",
      "IPv4Address": "172.25.0.3/16",
      "MacAddress": "02:42:ac:19:00:03"
    }
  }
  ```

That confirms:

- Both containers share the same network (`node_docker_default`).
- Each has its own IP inside that subnet.
- They can reach each other via their **service names**: `mongo`, `node-app`, etc.

---

## 8. Final mental model to remember

When I’m inside this project:

1. **Mongoose** gives me a nice API to talk to Mongo.
2. My Node container and Mongo container are on the same custom Docker network created by `docker compose`.
3. On that network, Docker runs a little DNS service:
   - It maps **service names** (`mongo`, `node-app`) to **container IPs**.
4. In my Node code I should **never** hardcode container IPs.
   - Instead I use the **service name** (`mongo`) in the connection string:

   ```js
   mongoose.connect(
     'mongodb://sanjeev:myPassword@mongo:27017/?authSource=admin'
   );
   ```

5. If containers are recreated and their IPs change, I don’t care — DNS keeps resolving `mongo` to the correct IP.

That’s the core story: we start by fumbling around with `docker inspect` and fixed IPs, then we graduate to using **Docker’s internal DNS + service names**, which is what you’ll always want in real projects.

