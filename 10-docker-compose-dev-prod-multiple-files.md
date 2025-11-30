# Docker Dev & Prod Environment


## 1. Introduction

This is the final big chapter where we take everything we’ve done so far with Docker — building images, running containers, bind mounts, anonymous volumes, environment variables, Docker Compose — and we now evolve it into a **real development vs production workflow**.

The goal is simple:

- Use **one Dockerfile** for both dev & prod  
- Use **two Docker Compose files**: one for dev, one for prod  
- Allow different behaviors depending on environment (nodemon in dev, regular node in prod)  
- Make Docker smart enough to install dev dependencies only in dev, and skip them in prod  
- Learn how arguments get passed into Dockerfile during build  
- Learn how Docker Compose merges multiple YAML files  
- Learn the final commands for spinning up dev or prod environments  

This is the final professional setup — what real companies do.

---

## 2. Why we need two environments

In real software development:

- **Development mode**  
  Code changes every few seconds.  
  We need bind mounts, nodemon, auto-reloads.

- **Production mode**  
  Code is frozen.  
  No bind mounts.  
  No nodemon.  
  Only production dependencies.  
  The container must be clean and optimized.

You *can* create two Dockerfiles, but maintaining two files is annoying, so instead:

- We keep **one Dockerfile**
- But we give it the intelligence to behave differently depending on environment

For that, we will use:

- **Docker args** → passed during build  
- **Docker env vars** → available at runtime  
- **Conditional logic inside the Dockerfile** → using bash IF statements  

---

## 3. Preparing our three Compose files

We create **three files**:

1. **docker-compose.yml** → Shared settings  
2. **docker-compose.dev.yml** → Dev-only overrides  
3. **docker-compose.prod.yml** → Prod-only overrides  

We temporarily rename our old compose file:

```
docker-compose.backup.yml
```

This is just for reference.

---

## 4. Create the shared compose file

`docker-compose.yml` contains settings that both dev and prod share.

Example:

- Same image name
- Same ports
- Same environment variable for port
- Same build context

So the file looks like this:

```yaml
version: "3"

services:
  nodeapp:
    build: .
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
```

This file does **not** contain anything dev-specific (bind mounts)  
or production-specific (special commands).  
It is purely the shared, common ground.

---

## 5. Create docker-compose.dev.yml

This file adds:

- Bind mount for live code syncing  
- Anonymous volume hack to protect node_modules  
- NODE_ENV=development  
- The command: **npm run dev** (nodemon)

```yaml
version: "3"

services:
  nodeapp:
    volumes:
      - "./:/app"
      - "/app/node_modules"
    environment:
      - NODE_ENV=development
    command: npm run dev
```

Remember:

- Bind mount for reading local code  
- Anonymous volume prevents node_modules from being overwritten  
- Nodemon auto restarts  
- This environment **will see live code changes**  

---

## 6. Create docker-compose.prod.yml

Production has:

- NO bind mount  
- NO anonymous volumes  
- NO nodemon  
- NODE_ENV=production  
- Command: `node index.js`

```yaml
version: "3"

services:
  nodeapp:
    environment:
      - NODE_ENV=production
    command: node index.js
```

Production should:
- Run cleanly  
- Never pick up code changes  
- Never auto-reload  
- Never install dev dependencies  

---

## 7. Updating the Dockerfile for environment awareness

We now modify the Dockerfile to support environment‑based behavior.

Two major changes:

### **A. Add an argument called NODE_ENV**

```dockerfile
ARG NODE_ENV
```

This allows Docker Compose to pass the environment into the Dockerfile **during build**.

### **B. Replace simple npm install with an IF statement**

```dockerfile
RUN if [ "$NODE_ENV" = "development" ] ; then       npm install ;     else       npm install --only=production ;     fi
```

This is real Bash running inside Docker during the build.

### **C. Set a default CMD (can be overridden by Compose)**

```dockerfile
CMD ["node", "index.js"]
```

Docker Compose (dev/prod versions) will override this.

---

## 8. Passing args from Docker Compose

To send build args into Dockerfile, we update both dev and prod compose files:

### docker-compose.dev.yml

```yaml
build:
  context: .
  args:
    NODE_ENV: development
```

### docker-compose.prod.yml

```yaml
build:
  context: .
  args:
    NODE_ENV: production
```

Now Docker knows:

- When building dev images → include dev dependencies  
- When building prod images → exclude dev dependencies  

---

## 9. Running the development environment

Command:

```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

Order matters:

1. First file → base config  
2. Second file → overrides base settings  

This brings up the full dev environment:

- nodemon enabled  
- bind mounts active  
- node_modules protected  
- live reload works  

Try editing code → browser updates instantly.

---

## 10. Running the production environment

Command:

```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Differences:

- No bind mount  
- No node_modules overwritten  
- No nodemon  
- No code reload  
- Only production dependencies installed  

Try editing code → **nothing changes**, which is correct.

---

## 11. Verifying production behavior

Inside the container:

```
docker exec -it <container> bash
cd node_modules
ls
```

You will NOT see nodemon.

This proves:

- The if‑statement in Dockerfile worked  
- Build args were passed correctly  
- Production dependencies were installed only  

---

## 12. Stopping containers (dev or prod)

Use:

```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml down -v
```

or

```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml down -v
```

`-v` cleans up temporary volumes automatically.

---

## 13. Summary of all important commands

### Start dev
```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

### Stop dev
```
docker-compose -f docker-compose.yml -f docker-compose.dev.yml down -v
```

### Start prod
```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

### Stop prod
```
docker-compose -f docker-compose.yml -f docker-compose.prod.yml down -v
```

### View logs
```
docker logs <container>
```

### Enter container
```
docker exec -it <container> bash
```

---

## 14. Why this system is the real-world standard

This setup mimics what real companies do:

- One Dockerfile  
- Multiple Compose files  
- Build args to control environment  
- Zero duplication of shared settings  
- Predictable dev & prod parity  
- Same image can promote through CI/CD stages  

This is a perfect foundation for deploying:

- To a cloud VM  
- To Docker Swarm  
- To Kubernetes later  

This completes the philosophical end of the Docker basics journey.

---

## End of full long‑form chapter.
