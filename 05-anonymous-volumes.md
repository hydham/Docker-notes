CHAPTER — Fixing the “nodemon not found” problem (bind mount deleting node\_modules)

  

Up to this point everything was working perfectly: the Docker container was running our Node app, the bind mount was syncing our local source code into the container, and nodemon was watching for code changes and restarting automatically.

  

Now the teacher intentionally breaks things to show a very common real-world trap.

  

  

  

  

1.  Reproducing the bug

  

  

We begin by deleting the running container:

  

docker rm node-app -f

(-f forces removal even if the container is running)

  

Then he deletes the local node\_modules folder from the host machine. This seems harmless since we “don’t need node\_modules locally anymore” because development has moved inside Docker.

  

After that, he redeploys the container using the same docker run command that includes the bind mount and tries refreshing the browser. The app spins and then crashes.

  

Everything was working before, so what broke?

  

  

  

  

2.  Checking the container status

  

  

We check running containers:

  

docker ps

  

The container is missing. That means it must have crashed immediately.

  

So we list all containers, including those that exited:

  

docker ps -a

  

Now we see our node-app container is in “Exited (1)” state. That means it started, hit an error, and shut down.

  

  

  

  

3.  Checking the logs

  

  

We inspect the logs:

  

docker logs node-app

  

The important line appears:

  

nodemon: not found

  

This is strange because nodemon absolutely existed earlier inside the container. The only thing we changed was deleting local node\_modules. How does removing a local folder impact the container?

  

This is the key moment.

  

  

  

  

4.  Understanding what actually happened with bind mounts

  

  

We need to mentally replay how the image and container are built.

  

Inside the Dockerfile:

  

-   We copy package.json into /app
-   We run npm install inside the image → node\_modules is created inside the image
-   We copy the rest of the project files into /app
-   Our final image now contains /app/node\_modules with nodemon installed

  

  

This “baked-in node\_modules” lives inside the image, not on your local machine.

  

But…

  

When we start the container using a bind mount:

  

\-v /path/to/project/on/host:/app

  

Docker overlays (mounts) the host folder on top of /app inside the container.

  

This means:

Everything inside /app from the image is HIDDEN by whatever exists in the host folder.

  

If the host folder contains:

  

index.js

package.json

(no node\_modules folder)

  

Then inside the container, /app will appear as:

  

index.js

package.json

(no node\_modules folder)

  

So even though the image originally contained /app/node\_modules, the bind mount overwrote it.

  

This is why deleting node\_modules on the host unintentionally deleted it inside the container.

  

As a result:

  

nodemon cannot be found

npm scripts fail

the container exits immediately

  

This explains the error perfectly.

  

  

  

  

5.  The fix: preserve node\_modules using a second anonymous volume

  

  

We still want:

  

-   Source code synced from host to container
-   But node\_modules should belong exclusively to the container

  

  

The solution is to mount a second volume just for node\_modules:

  

\-v /app/node\_modules

  

This creates an anonymous Docker-managed volume at /app/node\_modules inside the container.

Because Docker volumes follow a specificity rule, the more specific mount path wins.

  

Two mounts:

  

1.  Host bind mount:  
    host-folder → /app
2.  Anonymous volume:  
    (docker-managed) → /app/node\_modules

  

  

Because /app/node\_modules is a deeper path than /app, it overrides only that folder.

  

Result inside the container:

  

/app/index.js → from host

/app/package.json → from host

/app/node\_modules → from anonymous container volume

  

This ensures:

  

-   Code comes from host
-   node\_modules stays inside container
-   nodemon exists again
-   the app no longer crashes

  

  

After running the command with both volumes, refreshing the browser works, and nodemon auto-reloads the app when code changes.

  

  

  

  

6.  Why we still need “COPY . .” in the Dockerfile

  

  

You might now wonder:

If development uses bind mounts, do we still need the COPY . . step in the Dockerfile?

  

Yes, absolutely.

  

Because:

  

During development:

  

-   Bind mount overrides the image filesystem
-   node\_modules is preserved via anonymous volume
-   You can edit code without rebuilding the image

  

  

But in production:

  

-   You do NOT use a bind mount
-   The code in the image must already contain everything
-   Production container starts from the baked image exactly as built

  

  

So the Dockerfile’s COPY . . step ensures:

  

-   Code is embedded in the production image
-   node\_modules created during docker build remains intact
-   No development shortcuts leak into production

  

  

Bind mount is only a development convenience.

  

COPY . . is required for production.