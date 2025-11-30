# Docker + MongoDB + DNS 
When the Node app container was finally running smoothly, it felt like the right time to make our project act like a real application. A real app doesn’t just respond — it *stores* things. For that, we needed a database. So I headed to Docker Hub, searched for **mongo**, clicked the official image, and grabbed the usage examples.

Inside my Docker Compose file, under `services:`, I created a second container. This time no custom Dockerfile — Mongo ships with everything we need. I used the `image:` field directly and added the required root username & password environment variables.

```yaml
services:
  node_app:
    # node app config here...

  mongo:
    image: mongo:latest
    environment:
      - MONGO_INITDB_ROOT_USERNAME=sanjeev
      - MONGO_INITDB_ROOT_PASSWORD=my_password
```

After saving, I spun everything up:

```bash
docker compose up -d
```

Now `docker ps` showed two containers running: **node_app** and **mongo**.

---

## Connecting Into Mongo and Writing Data

I wanted to inspect the database, so I jumped into the container:

```bash
docker exec -it mongo bash
```

Logged into Mongo using the credentials:

```bash
mongo -u sanjeev -p my_password
```

Mongo dropped me inside the default `test` DB. Creating my own DB was simple:

```bash
use mydb
```

I inserted my first document:

```bash
db.books.insert({ name: "Harry Potter" })
```

Confirmed it with:

```bash
db.books.find()
```

Everything looked perfect.

---

## The Shock: My Data Disappeared

Then I ran:

```bash
docker compose down -v
docker compose up -d
```

Logged in again, ran:

```bash
show dbs
```

…and **mydb was gone**.

Why?  
Because containers are temporary. When a container is deleted, its internal filesystem vanishes with it.

Mongo was storing all data *inside the container*, so wiping the container wiped the DB.

---

## Fixing It: Adding a Named Volume

To persist database data, we needed a **named volume**. Unlike an anonymous volume, a named one won’t get accidentally deleted — and I can easily identify it later.

I updated the Compose file:

```yaml
mongo:
  image: mongo:latest
  environment:
    - MONGO_INITDB_ROOT_USERNAME=sanjeev
    - MONGO_INITDB_ROOT_PASSWORD=my_password
  volumes:
    - mongo-db:/data/db

volumes:
  mongo-db:
```

Restarted everything:

```bash
docker compose down
docker compose up -d
```

Inserted data again.  
Brought containers down + up again.  

This time — the data stayed. Perfect.

---

## Cleaning Up Old Anonymous Volumes

Over time, Docker leaves behind lots of anonymous volumes.  
To clean them safely:

```bash
docker volume prune
```

This only removes volumes not currently used by any running or stopped container.

After pruning, running:

```bash
docker volume ls
```

showed only the named volume `mongo-db` plus a couple system ones.

---

## DNS Between Containers

Next I connected my Node app to Mongo using Mongoose.

Installed mongoose:

```bash
npm install mongoose
```

Rebuilt the Docker image and restarted Compose. Then updated `index.js`:

```js
const mongoose = require('mongoose');

mongoose.connect(
  'mongodb://sanjeev:my_password@mongo:27017/?authSource=admin'
)
  .then(() => console.log('Successfully connected'))
  .catch(err => console.log(err));
```

Notice the trick:  
Instead of hardcoding an IP address, I used **mongo** — the service name.  
Docker Compose networking automatically resolves service names to container IPs (DNS for free!), so:

```bash
ping mongo
```

from inside the node container worked instantly and resolved to the Mongo container’s IP.

That’s Docker DNS in action.

---

And that completed the entire chapter:  
- Added Mongo container  
- Connected Node → Mongo  
- Used named volumes to persist data  
- Cleaned anonymous volumes safely  
- Leveraged Docker DNS for internal networking
