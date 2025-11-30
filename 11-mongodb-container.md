# Docker + MongoDB: Adding a Database Container and Persisting Data

So far I’ve only had one container in this little project: the **Node/Express** app.  
That’s cute, but real apps almost always have at least a database sitting next to them.

In this chapter I wire in **MongoDB** as a second container, connect to it, then fix the “oops my data disappeared” problem using **named volumes**.

---

## 1. Adding a MongoDB container to `docker-compose`

I already have a `docker-compose` setup with a `node-app` service.

Now I want a second service for **MongoDB**.

On Docker Hub, the official image is simply called **`mongo`**. The docs show examples like:

```bash
docker run mongo:latest
```

But instead of `docker run`, I’m going to describe this container in `docker-compose.yml` as another **service**.

```yaml
version: "3"

services:
  node-app:
    # ... existing config for the Node/Express container ...
    # build, ports, volumes, env, etc.

  mongo-db:
    image: mongo            # use the official mongo image
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev   # root admin user
      - MONGO_INITDB_ROOT_PASSWORD=myPassword  # root admin password
```

A few key points:

- Under **`services`**, *each key* (`node-app`, `mongo-db`) represents one container.
- For **`node-app`** I use `build: .` because I have a custom `Dockerfile`.
- For **`mongo-db`** I *don’t* need a custom Dockerfile; the official `mongo` image already has everything I need, so I just use **`image: mongo`**.
- MongoDB needs a root user and password on startup, so I pass those in with `environment` variables as shown above.

Now I bring everything up:

```bash
docker-compose up -d
# -d = detached mode (run in the background)
```

Then I check what’s running:

```bash
docker ps
```

I expect to see something like:

- One container for `node-app`
- One container for `mongo-db` (using the `mongo` image)

So far so good: two containers, one `docker-compose` file.

---

## 2. Connecting into the MongoDB container

I want to talk to Mongo **inside** the container, so I exec into it.

First I grab the Mongo container’s name from `docker ps`, then:

```bash
docker exec -it <mongo-container-name> bash
# docker exec  = run a command in a running container
# -i           = keep STDIN open
# -t           = allocate a pseudo-TTY (interactive shell)
# bash         = start a Bash shell inside the container
```

Now I’m inside the container’s Linux filesystem.

The `mongo` image ships with the Mongo shell client, so I can connect like this:

```bash
mongo -u sanjeev -p
# mongo   = Mongo shell client
# -u      = username (must match MONGO_INITDB_ROOT_USERNAME)
# -p      = prompt for password (will ask interactively)
```

I enter `myPassword` when prompted. Now I’m logged into MongoDB as the root user defined by those env vars.

---

## 3. Playing with databases and collections

Inside the **Mongo shell**:

```text
> db
test
```

- `db` shows which database I’m currently using.
- By default it puts me in a `test` database (which may not show in the list yet).

I want to create my own database called `mydb`:

```text
> use mydb
switched to db mydb
```

Now I list all databases:

```text
> show dbs
# mydb is NOT listed yet
```

This is a classic Mongo thing: **a database won’t show up until you actually store some data in it.**

So I insert one document into a collection called `books`:

```text
> db.books.insert({ name: "Harry Potter" })
WriteResult({ "nInserted" : 1 })
```

Now I query that collection:

```text
> db.books.find()
{ "_id" : ObjectId("..."), "name" : "Harry Potter" }
```

If I run `show dbs` again:

```text
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
mydb    0.000GB
```

Now **`mydb`** appears, because it has at least one document.

So far everything is in memory / container filesystem only.

---

## 4. Faster way to get into Mongo shell

I did this earlier:

```bash
docker exec -it <mongo-container> bash
mongo -u sanjeev -p
```

I can skip the intermediate Bash shell and go straight into Mongo:

```bash
docker exec -it <mongo-container> mongo -u sanjeev -p
# Runs `mongo` directly in the container
```

Same result, fewer steps.

---

## 5. The “my data disappeared” problem

Now comes the classic Docker “oh no” moment.

I have `mydb` created and a `books` document inserted. Now I tear everything down:

```bash
docker-compose down -v
# down = stop and remove all containers defined in docker-compose
# -v   = also remove related volumes (we’ll revisit this option later)
```

Then I bring it back up:

```bash
docker-compose up -d
```

After a few seconds, I connect back to Mongo again:

```bash
docker exec -it <mongo-container> mongo -u sanjeev -p
> show dbs
```

Result: **`mydb` is gone.**

What happened?

- `docker-compose down` **deleted the containers**.
- When I ran `docker-compose up` again, it **created new containers** from the image.
- Mongo’s data directory lives **inside** the container filesystem by default.
- When the container died, that filesystem died with it, so all data was wiped.

This is totally unacceptable for a real database — we need the data to **survive** container restarts.

Enter: **volumes**.

---

## 6. Choosing the right type of volume

I’ve already used volumes for the **Node app** in dev:

- A **bind mount** to sync code from my host into `/app`.
- An **anonymous volume** to protect `/app/node_modules` from being overridden.

But for Mongo, I care about **database persistence**. I want:

- Data to be stored **outside** the container (so container can be destroyed and recreated safely).
- A **human-readable volume name** so I know what the volume is for.
- A setup where I don’t accidentally delete the DB volume with some cleanup command.

So instead of:

- **Bind mount**: `./data:/data/db` (good if I want to browse files on host),
- **Anonymous volume**: `/data/db` (Docker generates a random volume name),

…I use a **named volume**:

```yaml
mongo-db:/data/db
```

This means:

- `/data/db` in the container is stored in a Docker-managed volume called **`mongo-db`**.
- That volume lives beyond the container’s life.

We just need to wire this into `docker-compose.yml` properly.

---

## 7. Adding a named volume for MongoDB

First I find out where Mongo stores its data. The Docker Hub docs for `mongo` say that the data directory is **`/data/db`**.

So in my `docker-compose.yml` I update the `mongo-db` service:

```yaml
version: "3"

services:
  node-app:
    # ... as before ...

  mongo-db:
    image: mongo
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev
      - MONGO_INITDB_ROOT_PASSWORD=myPassword
    volumes:
      - mongo-db:/data/db
      # named volume:
      #   host side:   mongo-db (Docker-managed volume name)
      #   container:   /data/db (Mongo's data directory)
```

Now, that alone **isn’t enough** for named volumes. Docker Compose complains if I run this right away:

```bash
docker-compose up
# ERROR: named volume "mongo-db" is used in service "mongo-db" but no declaration was found in the volumes section.
```

Compose wants me to **declare** the named volume at the top level, because:

- Multiple services *could* share the same named volume.
- It needs to know this is a managed volume, not a bind mount typo.

So at the **bottom** of `docker-compose.yml` I add:

```yaml
volumes:
  mongo-db: {}
  # Just declare the volume name so Compose knows about it.
  # The {} is an empty config (same as leaving it blank).
```

Full structure now:

```yaml
version: "3"

services:
  node-app:
    # ... as before ...

  mongo-db:
    image: mongo
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev
      - MONGO_INITDB_ROOT_PASSWORD=myPassword
    volumes:
      - mongo-db:/data/db  # named volume -> container path

volumes:
  mongo-db: {}              # declare the named volume
```

Now I start everything again:

```bash
docker-compose up -d
```

This will:

- Start `node-app` container.
- Start `mongo-db` container.
- Create a named volume called **`projectname_mongo-db`** (Compose prefixes project name).

I can see volumes with:

```bash
docker volume ls
```

One of them will be something like:

```text
local  <project>_mongo-db
```

That’s my database volume.

---

## 8. Verifying persistence with the named volume

Time to test if the DB finally survives a teardown.

1. Connect to Mongo:

   ```bash
   docker exec -it <mongo-container> mongo -u sanjeev -p
   ```

2. Inside Mongo shell, create data again:

   ```text
   > use mydb
   switched to db mydb

   > db.books.insert({ name: "Harry Potter" })
   WriteResult({ "nInserted" : 1 })

   > show dbs
   admin   0.000GB
   config  0.000GB
   local   0.000GB
   mydb    0.000GB
   ```

3. Exit Mongo and the container shell.

4. Now run **down without `-v`** (this is important):

   ```bash
   docker-compose down
   # NO -v here, otherwise it deletes named volumes too
   ```

5. Bring everything back up:

   ```bash
   docker-compose up -d
   ```

6. Connect again and check the data:

   ```bash
   docker exec -it <mongo-container> mongo -u sanjeev -p

   > show dbs
   admin   0.000GB
   config  0.000GB
   local   0.000GB
   mydb    0.000GB

   > use mydb
   > db.books.find()
   { "_id" : ObjectId("..."), "name" : "Harry Potter" }
   ```

Success: **the data survived** the cycle of `down` + `up` because it lived in the **named volume**, not inside the container filesystem.

---

## 9. Why I stopped using `docker-compose down -v`

Earlier, I got into the habit of:

```bash
docker-compose down -v
```

That was convenient when my volumes were just:

- **Anonymous dev volumes** (like the little `/app/node_modules` hack).
- Stuff I didn’t mind nuking every time.

But now I have a named volume holding **critical DB data**. The problem:

- `docker-compose down -v` deletes **all** volumes attached to those services:
  - Anonymous volumes ✅
  - Named volumes ❌ (oops, there goes the database)

So for a DB volume, I **must not** use `-v` casually anymore.

From now on:

- For this project I use:

  ```bash
  docker-compose down          # keep mongo-db volume
  docker-compose up -d         # containers come back, DB still there
  ```

- If I really want to clean *everything*, including DB, I can manually `docker volume rm` that named volume, or temporarily use `down -v` knowing I’ll wipe data.

Named volumes are deliberately persistent. That’s the point.

---

## 10. Cleaning up unused anonymous volumes safely

Even though I’ve got my named volume, I still generate **anonymous volumes** in other scenarios, and they tend to pile up.

To see all volumes:

```bash
docker volume ls
```

Over time this list can get huge with lots of random IDs. Many of those are unused.

To clean up **only unused** volumes, Docker provides:

```bash
docker volume prune
# Removes all volumes not used by any containers
```

I like to:

1. Make sure my important services are up (so their volumes are “in use”):

   ```bash
   docker-compose up -d
   ```

2. Then prune:

   ```bash
   docker volume prune
   # Confirm with 'y' when prompted
   ```

3. Check remaining volumes:

   ```bash
   docker volume ls
   ```

I should still see my named DB volume (e.g. `<project>_mongo-db`) because it’s attached to the running Mongo container, but a bunch of junk anonymous volumes will be gone.

---

## 11. Summary

- I added a **second container** to the project: a MongoDB database using the official `mongo` image.
- I passed in Mongo’s root username/password via `environment` variables in `docker-compose.yml`.
- I connected to Mongo using **`docker exec -it <mongo> mongo -u ... -p`**, created a database, and inserted a document.
- I discovered that **data disappears** when I destroy and recreate containers, because the data lived only in the container filesystem.
- I fixed this by adding a **named volume**:

  ```yaml
  services:
    mongo-db:
      image: mongo
      volumes:
        - mongo-db:/data/db

  volumes:
    mongo-db: {}
  ```

- I stopped using `docker-compose down -v` casually, because it also deletes named volumes (including DB data).
- I now use `docker volume prune` to clean up **unused** volumes without touching the volumes still in use by running containers.

Now the Mongo container behaves like a real database: containers can die, but the data lives on in the named volume.
