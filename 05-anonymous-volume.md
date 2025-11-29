\=== CHAPTER: Bind Mount Bug and Protecting node\_modules with an Anonymous Volume ===

  

We’ve now got a nice Docker dev workflow:

  

-   Our app runs inside a container.
-   A bind mount keeps our code in sync between local machine and container.
-   nodemon restarts the Node process when files change.

  

  

Everything seems perfect… until we break it on purpose.

  

  

  

— Breaking the app on purpose (so we understand the bug) —

  

The teacher starts by deleting the running container:

  

-   docker rm node-app -f

  

  

Then, on the local machine, he deletes the node\_modules folder in the project:

  

-   (In Explorer / Finder / terminal) delete the node\_modules folder next to index.js and package.json.

  

  

DUMMY TIP: The whole point of using Docker was “run everything inside the container”, so technically we don’t need local node\_modules anymore. That’s why he deletes it.

  

Now he re-runs the container using the same docker run … -v …:/app command with the bind mount.

  

If we open the browser and hit the app (for example http://localhost:3000), the page just spins and then errors. So something’s wrong.

  

  

  

— First clue: docker ps vs docker ps -a —

  

We check if the container is running:

  

-   docker ps

  

  

The container is not listed. That’s suspicious.

  

NOTE: docker ps only shows running containers. If a container crashed and exited, it disappears from this list.

  

So we run:

  

-   docker ps -a

  

  

Now we see the container node-app, but its status says something like Exited (1) 30 seconds ago.

  

DUMMY TIP:

  

-   docker ps → only running containers.
-   docker ps -a → all containers (running + stopped + crashed).

  

  

  

  

— Second clue: checking the logs —

  

To see why it crashed, we check logs for that container:

  

-   docker logs node-app

  

  

We see an error message in the output, something like:

  

-   nodemon: command not found  
    or
-   nodemon not found

  

  

So inside the container, when it tries to run npm run dev, it can’t find nodemon.

  

But wait… we know we installed nodemon earlier and rebuilt the image. It was working fine before we deleted node\_modules on the host. So why is it gone now?

  

  

  

— What actually happened (bind mount overwrote node\_modules) —

  

Let’s remind ourselves how the image is built (simplified):

  

1.  COPY package.json .
2.  RUN npm install  
    

-   This creates node\_modules inside the image at /app/node\_modules.

4.    
    
5.  COPY . .  
    

-   Copies the rest of the source code (like index.js) into the image.

7.    
    

  

  

So the image really does contain nodemon, because npm install ran during the build.

  

Then, when we run the container for development, we use a bind mount:

  

-   \-v /path/to/project/on/host:/app

  

  

This means:

  

-   The container’s /app folder is “linked” to your local project folder.
-   Docker basically says: “Use the host folder as the content for /app inside the container.”

  

  

Now, think about what happened when we deleted local node\_modules:

  

1.  In the image, we still have /app/node\_modules from npm install.
2.  But at runtime, the bind mount replaces the container’s /app with your local folder contents.
3.  Your local folder no longer has node\_modules.
4.  So, when you mount it, the container’s /app folder now mirrors your local project without node\_modules.
5.  Result: /app/node\_modules disappears from inside the running container.

  

  

Now when the container tries to run:

  

-   npm run dev  
    which calls
-   nodemon index.js

  

  

There’s no nodemon in node\_modules anymore, so the container crashes with “nodemon not found”.

  

That’s the bug:

  

The bind mount is “too powerful” — it stomps on /app/node\_modules in the container when that folder doesn’t exist on the host.

  

  

  

— Fix idea: protect /app/node\_modules with an anonymous volume —

  

We want two things at the same time:

  

1.  Keep syncing our source code between local and container using a bind mount (-v host\_path:/app).
2.  Do not let the bind mount wipe out /app/node\_modules that was created during image build.

  

  

Docker has a nice behavior: volume paths are resolved by specificity.

  

-   A mount on /app is general.
-   A mount on /app/node\_modules is more specific.

  

  

If both try to mount something to paths that overlap, the more specific path wins for that subfolder.

  

So we keep our old bind mount:

  

-   \-v /path/to/project:/app

  

  

And we add another volume just for /app/node\_modules, but this time we don’t bind it to the host; we let Docker manage it as an anonymous volume:

  

-   \-v /app/node\_modules

  

  

Full run command now conceptually looks like:

  

-   docker run  
    \-p 3000:3000  
    \-v /path/to/project:/app        ← bind mount for code  
    \-v /app/node\_modules            ← anonymous volume for node\_modules  
    –name node-app  
    node-app-image

  

  

Because /app/node\_modules has its own volume, the bind mount on /app cannot override that subfolder anymore.

  

DUMMY TIP:

  

-   Think of /app as a big box.
-   Think of /app/node\_modules as a small box inside.
-   We tell Docker: “The big box is shared with the host, but the small box is special; hands off.”
-   Docker respects the small box because its path is more specific.

  

  

  

  

— Walking through the fix step by step —

  

1.  First, delete the broken container:  
    

-   docker rm node-app -f

3.    
    
4.  Re-run the container with both volumes:  
    

-   \-v /path/to/project:/app (bind mount for code)
-   \-v /app/node\_modules (anonymous volume to protect dependencies)

6.    
    
7.  Check that it’s running:  
    

-   docker ps

9.    
    The container should stay in a Up state now, not exit immediately.
10.  Test the app in the browser:  
     

-   Visit http://localhost:3000
-   The app responds correctly.

12.    
     
13.  Make a code change (for example, remove exclamation marks in index.js) and save.
14.  Refresh the page – nodemon restarts the Node process, bind mount syncs the new code, and the app updates instantly.

  

  

So we now have:

  

-   A dev workflow with live reload and sync,
-   And a protected node\_modules that survives even if we delete the local one.

  

  

  

  

— Why do we still need COPY . . in the Dockerfile if we have a bind mount? —

  

This is an important architectural point.

  

You might wonder:

  

“If in development we use a bind mount to sync code into /app, do we even need the COPY . . in the Dockerfile?”

  

Answer: Yes, absolutely. Here’s why:

  

-   The bind mount is only used in development.  
    

-   We want fast feedback, auto-reload, easy editing.

-     
    
-   In production, we do not mount our source code from a host folder.  
    

-   We usually just run the image as-is on a server.
-   There is no local project directory to bind mount from.

-     
    

  

  

In production, the container’s file system comes entirely from the image:

  

-   No bind mount
-   No host folder
-   No external code sync

  

  

So where does the production container get the application code?

  

-   From the Dockerfile steps:  
    

-   COPY package.json .
-   RUN npm install
-   COPY . .

-     
    

  

  

That’s why we keep the COPY . . step.

It’s for production images, so the built image contains everything it needs: code + dependencies, without relying on a dev-only bind mount.

  

DUMMY TIP:

  

-   Dev workflow = image + bind mount + nodemon.
-   Prod workflow = image only (no bind mount, no nodemon dev reload).
-   So the Dockerfile must still fully describe how to build a self-contained image.

  

  

  

  

That’s the full story of:

  

-   How deleting node\_modules on the host accidentally broke the container,
-   How bind mounts can override folders inside the container,
-   Why /app/node\_modules disappeared,
-   And how an anonymous volume on /app/node\_modules protects your dependencies while still letting you sync code.