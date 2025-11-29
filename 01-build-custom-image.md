# Building a Custom Docker Image for Our Express App

In this chapter, I’m taking the simple Express app we already built and moving it into a Docker container by creating our **own custom image** based on the official **Node** image.

---

## Starting point

- I already have a simple Express app working locally:
  - Main file: **index.js**
  - Config file: **package.json**
  - Dependency: **express** (installed with **npm install express**)
- The app is already tested by running **node index.js** and visiting **http://localhost:3000** to see **“Hi there”**.

Now I want to run this app **inside Docker**, not directly on my machine.

---

## Step 1 – Make sure Docker is installed

- I’m assuming Docker is already installed on my machine.
- If not, I would:
  - Go to the official Docker website.
  - Download and install Docker for my OS (Windows / macOS / Linux).
  - Confirm installation by running **docker --version**.

Once Docker is installed and running, I can start working with images and containers.

---

## Step 2 – Choosing a base image from Docker Hub

I don’t want to install Node manually inside some OS image. Instead, I’ll reuse an existing **Node** image.

1. I go to **hub.docker.com**.
2. In the search bar, I type **node**.
3. I pick the **official Node image** (from the Node team).
4. On that page I can see:
   - Different tags/versions like **node:15**, **node:14**, **node:10**, etc.
   - Basic instructions for using the image.

This **Node** image already contains Node.js, so my custom image only needs to add:

- My **source code**.
- My **dependencies** (from **package.json**).

---

## Step 3 – Creating a `Dockerfile`

To define my custom image, I create a file called **Dockerfile** (capital D, no extension) in the **root of the project** (same folder as **index.js** and **package.json**).

This file is a list of instructions that Docker will follow to build the image.

### 3.1 – Base image: `FROM`

First instruction in **Dockerfile**:

- **FROM node:15**

Meaning:

- “Start from the official **node** image, version **15**.”
- I could also use **node:14** or **node:latest**. I just picked **15** to be explicit.
- This becomes **Layer 1** of my image.

### 3.2 – Working directory: `WORKDIR`

Next instruction:

- **WORKDIR /app**

Meaning:

- Inside the container’s filesystem, my “home folder” for the app will be **/app**.
- Any later commands like **COPY**, **RUN**, **CMD** will run **inside `/app`** by default.
- This becomes **Layer 2**.

This keeps things organized instead of dumping everything in **/**.

---

## Step 4 – Copying `package.json` and installing dependencies

Now I want to install **express** inside the image.

### 4.1 – Copy only `package.json` first

Instruction:

- **COPY package.json .**

Because **WORKDIR /app** is set, the **.** here means **/app** inside the container.

So this copies the **package.json** file from my host project folder into **/app/package.json** in the image.

This becomes **Layer 3**.

### 4.2 – Install dependencies: `RUN npm install`

Instruction:

- **RUN npm install**

Meaning:

- Inside the image (in **/app**), run **npm install**.
- This looks at **package.json** and installs all dependencies (like **express**) into **/app/node_modules**.
- This becomes **Layer 4**, and it is often the **slowest** layer (because it downloads and installs packages).

---

## Step 5 – Copying all source code

Now I want to copy my actual code.

Instruction:

- **COPY . .**

Because **WORKDIR /app** is set:

- Left **.** = “current folder on my host” (my project folder).
- Right **.** = “current folder inside the container” ( **/app** ).

So this copies **all files** (except ones ignored by **.dockerignore**, which we’ll add later in another chapter):

- **index.js**
- Any other source files
- Any config files

This becomes **Layer 5**.

### Why copy `package.json` first and then `COPY . .`?

This is an important optimization:

- Docker builds the image **layer by layer**.
- Each instruction (**FROM**, **WORKDIR**, **COPY**, **RUN**) creates a **layer**.
- Docker **caches** each layer.

Layers in our case:

1. **FROM node:15**
2. **WORKDIR /app**
3. **COPY package.json .**
4. **RUN npm install**
5. **COPY . .**

What Docker does when rebuilding:

- If **nothing changed** in an earlier step, Docker reuses the **cached** result for that layer and everything after until something changes.

Example:

- When I’m developing, I mostly change my **source code** (like **index.js**).
- I **rarely change `package.json`** (I don’t add new dependencies every 2 minutes).

Because we split copy into two steps:

- As long as **package.json** hasn’t changed:
  - **Layer 3** is reused (no need to copy again).
  - **Layer 4** is reused (no need to run **npm install** again).
- Only **Layer 5** has to be rebuilt (copying updated source code).

If we had a single **COPY . .** before **RUN npm install**:

- Changing **any file** (even just **index.js**) would invalidate that copy layer.
- Docker would then have to re-run **npm install** every time (slow!).

So splitting **COPY package.json .** and then **COPY . .** is a **speed optimization** for rebuilds.

---

## Step 6 – Exposing the port (documentation only)

We know our app listens on port **3000**:

- In **index.js** we have something like:
  - **const port = process.env.PORT || 3000**

So in the **Dockerfile** we add:

- **EXPOSE 3000**

Important: this **does not actually open the port to the host**.  

It’s **documentation** inside the image that says: “this container expects traffic on port 3000”.  

The **real port mapping** is done later with **docker run -p**, which we’ll see in the next chapter.

---

## Step 7 – Defining the startup command: `CMD`

Finally, we tell Docker how to **start** our app when a container is created from this image.

Instruction:

- **CMD ["node", "index.js"]**

Meaning:

- When the container starts, run **node index.js** inside **/app**.
- This is the **runtime command**, not a build-time command.

Difference:

- **RUN** = command during **build** (install deps, compile things).
- **CMD** = command during **container start** (run the app).

---

## Step 8 – Full Dockerfile (mental picture)

Without code formatting, our **Dockerfile** conceptually has these lines:

- **FROM node:15**
- **WORKDIR /app**
- **COPY package.json .**
- **RUN npm install**
- **COPY . .**
- **EXPOSE 3000**
- **CMD ["node", "index.js"]**

That’s enough to create a working image (ignoring **.dockerignore** for now).

---

## Step 9 – Stopping local Node and building the image

Before building the image, I should stop any **locally running** Node server (the one started with **node index.js**) so I know traffic later is going to the **container**, not my host process.

- Stop local Node:
  - Go to the terminal where **node index.js** is running.
  - Press **Ctrl + C**.

### 9.1 – Build the Docker image

From the project folder (where **Dockerfile** lives), run:

- **docker build -t node-app-image .**

Explanation:

- **docker build** = build a new image.
- **-t node-app-image** = give the image a **name/tag** (here: **node-app-image**).
- **.** = build context is the **current directory** (where Dockerfile and code live).

On first build, you’ll see output like:

- Step 1/5: FROM node:15
- Step 2/5: WORKDIR /app
- Step 3/5: COPY package.json .
- Step 4/5: RUN npm install
- Step 5/5: COPY . .
- Successfully built <IMAGE_ID>

If you run the same build command again without changing files, you’ll see **“Using cache”** on most steps.

### 9.2 – Check images

To see the images:

- **docker image ls**

You’ll see at least:

- The base **node:15** image.
- Your custom **node-app-image**.

If you built once **without** the **-t** name, you might also see an image with `<none>` as a name. You can safely remove that with:

- **docker image rm IMAGE_ID**

---

## Step 10 – Running a container from the image

Now we run a container:

- **docker run -d --name node-app node-app-image**

Explanation:

- **docker run** = create and start a new container.
- **-d** = run in **detached** mode (in the background).
- **--name node-app** = name the container **node-app** so it’s easy to refer to.
- **node-app-image** = the name of the image we just built.

To see the running container:

- **docker ps**

You should see a container with **IMAGE** = **node-app-image** and **NAMES** = **node-app**.

---

## Step 11 – Testing (and why it “hangs” for now)

Now I try to visit the app in the browser:

- Open **http://localhost:3000**

What happens:

- The page just **spins** and eventually fails.

Why?

- Our app **inside the container** is listening on **port 3000**.
- But we **did not tell Docker** to forward port 3000 from the **host** to the **container**.
- **EXPOSE 3000** in the Dockerfile is only documentation. It does not open anything to the host.

So at this point:

- The container is up.
- Node app is running inside.
- But the outer machine cannot reach it yet.

We will fix this in the **next chapter** using **port mapping** with **docker run -p** (like **-p 3000:3000**).

For now, this chapter’s goal was:

- Understand **how to build a custom image** from the Node base image.
- Understand **layers** and **caching**.
- Understand **Dockerfile** structure and the meaning of **FROM**, **WORKDIR**, **COPY**, **RUN**, **EXPOSE**, **CMD**.
- Successfully **build and run a container**, even if the networking part isn’t wired up yet.

That’s the complete set of notes for this chapter.