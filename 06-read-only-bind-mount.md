# Docker Volumes: Read-Only Bind Mounts (Protecting Your Source Code)

Now that we’ve spent time working with volumes and bind mounts, the teacher introduces one final tweak — a small but *very* practical improvement.  
This is one of those “oh… I didn’t know containers could do that” moments.

The goal in this chapter is simple:  
**Stop the Docker container from accidentally modifying your source code.**

---

## Entering the Container One More Time

The teacher already has the container running, with:

- a bind mount syncing the project folder  
- an anonymous volume protecting **/app/node_modules**

He drops into the container again:

```bash
docker exec -it app bash
# docker exec   → run a command inside an existing container
# -it           → interactive terminal
# app           → container name
# bash          → command to run inside the container
```

Inside, he runs:

```bash
ls
# Lists the files inside /app, which is the working directory
```

Because of the bind mount, **everything inside your local folder is also visible inside /app**.

---

## Understanding the Two-Way Street (The Problem)

To demonstrate this, he creates a file *from the host* and sees it appear inside the container:

```bash
# From your host machine
touch myfile
```

Then inside the container:

```bash
ls
# myfile appears here
```

Okay — makes sense.

But now he shows the opposite direction:  
Creating a file **inside** the container:

```bash
touch testfile
ls
# testfile appears inside container
```

And guess what?  
It also appears on your host system.

### Why this is dangerous

Your Docker container — which should be a safe, isolated environment — now has the ability to:

- create files in your local project  
- modify files  
- delete files  

This is rarely what you want during development.  
Most applications should **never** be able to alter your source code.

Think of this as letting strangers into your house and giving them keys to all the rooms.  
Not good.

---

## Making the Bind Mount Safe (Read-Only)

So the teacher exits the container:

```bash
exit
```

Then removes the active container:

```bash
docker rm app -f
# -f → force remove even if running
```

Now he re-runs the container, **but with one small addition**.

On Windows Command Prompt (what the teacher was using):

```bash
docker run ^
  -p 3000:3000 ^
  -v %cd%:/app:ro ^
  -v /app/node_modules ^
  --name app ^
  node-app-image
```

On PowerShell:

```bash
docker run `
  -p 3000:3000 `
  -v ${PWD}:/app:ro `
  -v /app/node_modules `
  --name app `
  node-app-image
```

On macOS / Linux:

```bash
docker run   -p 3000:3000   -v $(pwd):/app:ro   -v /app/node_modules   --name app   node-app-image
```

Let’s break down the important pieces:

```text
-v $(pwd):/app:ro
|   |        └──────── /app inside the container (project folder in container)
|   └───────────────── $(pwd) = current directory on the host (your project)
└───────────────────── -v means “create a volume / bind mount”

:ro
└─ “read-only” flag → the container can *read* from /app, but cannot write to it
```

We still keep the anonymous volume for **/app/node_modules**:

```bash
-v /app/node_modules
# Separate volume just for node_modules so it isn’t overridden by the bind mount
```

So now we have:

- **Bind mount (read-only)** for the source code at **/app**
- **Anonymous volume (read-write)** for **/app/node_modules**

The container can:

- read your source files (to run the app)
- write into **/app/node_modules** (so npm install works)

But it **cannot**:

- create new files in your project folder
- delete or edit your local source code

---

## Verifying the Read-Only Behaviour

The teacher starts the container with the new command, then drops back into bash:

```bash
docker exec -it app bash
```

He tries to create a new file inside **/app**:

```bash
touch newfile
# Error: “Read-only file system”
```

This is exactly what we want.

- Docker is now syncing your source into the container (thanks to the bind mount).
- But the container itself is not allowed to mess with your files.

This tiny change (**:ro**) turns your bind mount into a **safe, read-only view** of your project, which is a much better default for day-to-day development.

You still get live code reloads (because of bind mount + nodemon),  
but your laptop’s source folder is protected from any accidental writes from inside the container.

---
