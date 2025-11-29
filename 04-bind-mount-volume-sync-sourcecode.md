
# Hot-reloading our Node app inside Docker (bind mounts + nodemon)

Now that the app is happily running inside a Docker container and we can hit it from the browser, it’s time to run into the *next* very real problem:  
“Why doesn’t my code change show up?!”

---

## 1. The “why is nothing updating?” moment

We start with a working container. The browser shows:

> Hi there

Now we edit **index.js** locally:

- Change the response from **Hi there** to **Hi there!!** (add exclamation points)
- Save the file
- Go back to the browser and hit **Refresh**

Result: still just **Hi there**. No exclamation points.

Nothing is broken, there’s no error… it’s just stale.

---

## 2. Why the container still sees old code

Docker’s flow so far:

1. We wrote a **Dockerfile**
2. We built an image with **docker build .**
3. We ran a container from that image with **docker run ... node-app-image**

Important idea:  
The image is a snapshot of your project *at build time*.

When we first built the image, **index.js** did *not* have exclamation points.  
So:

- The image contains **old** index.js
- The container is running from that image
- Changing the local file later does **not** change the image

We can prove this inside the container:

1. Exec into the container:

   - **docker exec -it node-app bash**

2. List files:

   - **ls** → you see **index.js**, **package.json**, etc.

3. Print the file:

   - **cat index.js**

You’ll see the old version: no exclamation points.

So:

- Local **index.js** → new  
- Container’s **index.js** → still old (from image)

---

## 3. Fix 1 (slow): rebuild image and recreate container

The “pure Docker” way:

1. Delete the old container:

   - **docker rm node-app -f**

2. Rebuild the image:

   - **docker build -t node-app-image .**

   Since we only changed **index.js**, Docker’s cache kicks in:
   - Steps like **FROM node:15**, **WORKDIR /app**, **COPY package.json .**, **RUN npm install** are cached
   - Only the last step **COPY . .** reruns
   - This is why layer 5 re-executes and others say *Using cache*

3. Run the container again:

   - **docker run -d --name node-app -p 3000:3000 node-app-image**

4. Refresh the browser → now you see **Hi there!!**

So this works, but it’s a terrible dev workflow:

- Every tiny change = rebuild image + restart container  
- That’s fine for production, painful for development

We need something more like “normal local dev”: change → save → refresh.

---

## 4. The idea: sync local folder into the container (bind mount)

Docker has a feature called **volumes** for persisting/sharing data.

One special type is a **bind mount**:

> Take this folder on my machine and mirror it inside the container.

Perfect for dev:

- Use the image once as a base
- Mount the local project folder into **/app** inside the container
- So inside the container, Node reads files from a live view of our local code

### 4.1. Remove the current container

First clear the old one:

- **docker rm node-app -f**

### 4.2. Basic run command with bind mount

We reuse our old **docker run** command, but add a volume flag **-v**.

General pattern:

- **-v <path-on-host>:/app**

For example, on Windows (full path):

- **docker run -d --name node-app -p 3000:3000 -v C:\Users\you\path\to\node-docker:/app node-app-image**

On Linux/macOS:

- **docker run -d --name node-app -p 3000:3000 -v /home/you/node-docker:/app node-app-image**

This tells Docker:

> Whatever files are in my local project folder, show that same content at **/app** inside the container.

Since **WORKDIR /app**, Node inside the container now sees your real project files.

### 4.3. Shortcuts for the path (so we don’t type the full thing)

Typing the full absolute path is annoying. We can use “current directory” shortcuts.

Depending on your shell:

- **Windows Command Prompt (cmd.exe)**  
  Use: **%cd%**  
  Example: **docker run -d --name node-app -p 3000:3000 -v %cd%:/app node-app-image**

- **Windows PowerShell**  
  Use: **${PWD}**  
  Example: **docker run -d --name node-app -p 3000:3000 -v ${PWD}:/app node-app-image**

- **Linux / macOS (bash, zsh, etc.)**  
  Use: **$(pwd)**  
  Example: **docker run -d --name node-app -p 3000:3000 -v $(pwd):/app node-app-image**

On Windows, Docker may pop up a “file sharing” warning the first time. That’s just Docker asking permission to access your drive; you can usually allow it and move on.

Now our bind mount is active:

- Local changes in the project folder → instantly visible at **/app** inside the container

---

## 5. First bind-mount test: did the files sync?

With the bind-mount container running:

1. Open **index.js** locally and remove the exclamation points.
2. Save the file.
3. Exec into the container again:

   - **docker exec -it node-app bash**

4. Inside the container:

   - **ls** → you see **index.js**, etc.
   - **cat index.js** → you now see the version *without* exclamation points.

So the bind mount is working perfectly: the container sees the new file content.

But…

Refresh the browser and you still don’t see the change immediately.

Bind mount solved “files are stale,” but we still have a different problem.

---

## 6. Why the browser still doesn’t update: Node process is still old

Even though **index.js** changed, the Node process inside the container is:

- Still the same process that started earlier
- It loaded **index.js** at startup
- It doesn’t “re-read” the file every time you hit the endpoint

In plain words:

> The container’s filesystem is updated, but the running Node process doesn’t automatically restart.

For Node/Express dev, this is a classic problem.  
You usually solve it with **nodemon**.

---

## 7. Introducing nodemon for automatic restarts

**nodemon** watches your files and restarts Node when code changes.

We want:

- Container sees updated files (bind mount ✅)  
- Node restarts automatically on changes (nodemon ✅)

### 7.1. Install nodemon as a dev dependency

We do this on the **host**, just so **package.json** and **package-lock.json** get updated.  
The actual runtime install still happens inside the image via **npm install**.

On host, in project folder:

- **npm install nodemon --save-dev**

This adds **nodemon** to the **devDependencies** of **package.json**.

### 7.2. Add npm scripts

In **package.json**, under **"scripts"**, we add:

- **"start": "node index.js"**  
  (normal production-like start)

- **"dev": "nodemon index.js"**  
  (development mode with auto-restart)

On Windows, nodemon + Docker sometimes has file watching issues.  
The teacher mentions that on Windows you might need:

- **"dev": "nodemon -L index.js"**

Here **-L** = “legacy watch mode”, better for some file systems.

So final script might be:

- **"dev": "nodemon -L index.js"**

### 7.3. Rebuild the image (because package.json changed)

We changed **package.json**, which affects dependencies.  
Docker must re-run the image steps after **COPY package.json .**:

- **docker rm node-app -f**
- **docker build -t node-app-image .**

This time, build takes longer:

- Step 3 (**COPY package.json .**) changed  
- Step 4 (**RUN npm install**) must re-run  
- Step 5 (**COPY . .**) also re-runs

Good sign: our container will now have nodemon installed.

---

## 8. Updating the Dockerfile to use nodemon

Previously, our Dockerfile probably ended with:

- **CMD ["node", "index.js"]**

For dev, we want to use our **npm dev** script instead:

- **CMD ["npm", "run", "dev"]**

That tells Docker:

> When this container starts, run **npm run dev**, which runs **nodemon**, which runs **index.js** and restarts when files change.

So we edit Dockerfile and replace the CMD line:

- **CMD ["npm", "run", "dev"]**

Then rebuild again:

- **docker build -t node-app-image .**

Docker now caches up to the step before CMD. Rebuild ensures the final image has the updated entrypoint.

---

## 9. Running the container again with bind mount + nodemon

Now we combine everything:

- New image with nodemon
- Bind mount from host to **/app**
- Port mapping from host 3000 to container 3000

Run:

- **Windows cmd.exe**:  
  **docker run -d --name node-app -p 3000:3000 -v %cd%:/app node-app-image**

- **Windows PowerShell**:  
  **docker run -d --name node-app -p 3000:3000 -v ${PWD}:/app node-app-image**

- **Linux/macOS**:  
  **docker run -d --name node-app -p 3000:3000 -v $(pwd):/app node-app-image**

Now:

1. Open browser → **http://localhost:3000** → you see the current response.
2. Edit **index.js**:
   - Add or remove exclamation points
   - Or change the string entirely
3. Save the file
4. Refresh the browser

Result: you see the change immediately.

What actually happened behind the scenes:

- Bind mount synced the modified file into **/app/index.js** inside the container  
- **nodemon** noticed the file change  
- **nodemon** restarted the Node process with the new code  
- Your browser request hit the new version

No image rebuild.  
No container recreation.  
Just normal “save + refresh” experience, but fully inside Docker.

---

## 10. Summary of this chapter’s journey

We learned, in order:

1. **Why code didn’t update at first**  
   The container ran an old image; changing files on the host doesn’t modify the image.

2. **Fix 1: rebuild & rerun (slow)**  
   Delete container, rebuild image, run again. Works, but painful for development.

3. **Bind mounts for live code sync**  
   Use **-v <host-path>:/app** to mirror your project folder inside the container.

4. **Why even bind mount wasn’t enough**  
   Files updated inside container, but the Node process didn’t restart.

5. **nodemon for auto-restart**  
   Install with **npm install nodemon --save-dev**, add a **dev** script, update CMD to **["npm", "run", "dev"]**, rebuild image.

6. **Final dev setup**  
   - One Docker image with proper dependencies  
   - Bind mount for live code  
   - nodemon for automatic restarts  
   - Port 3000 mapped with **-p 3000:3000**

You now have a realistic, Docker-first Node development workflow:  
Code *lives* on your host, the app *runs* inside Docker, and you still get instant feedback like a normal local dev setup.
