Publishing this site
====================

This site is made from sphinx-docs.  Here's how I get these docs published to a website running on kubernetes.

When you run sphinx to build your restructured text documents into html (out of scope for this doc), it takes the source directory of ./source and publishes to ./build/html.  I went to the build/html folder and initialized a local git repository by running ``git init``.

Then we add everything to the local repo by running ``git add .``  After that we commit all added files to the local repository with a comment:  ``git commit -a -m "initial commit"``

Next we create a repo on github.com, and tell our local git repository it has a remote repo to sync to: ``git remote add origin master https://github.com/cstevens-io/cstevens-io.github.io.git``

Next we'll create a Dockerfile in the build/html/ directory with the following contents:

.. code-block:: text
   :caption: Dockerfile
   
   FROM nginx:alpine
   WORKDIR '/usr/share/nginx/html'
   COPY . ./
   EXPOSE 80

This instructs docker to download the latest nginx alpine image.  It will copy the contents of the build/html/ folder to /usr/share/nginx/html inside the nginx image, which is where nginx will read files from the documentroot.  Finally it tells docker to expose port 80.

Then we will run ``docker build . -t splooge/docs-cstevens-io`` which will execute the above and create a local image called ``splooge/docs-cstevens-io:latest`` where ``splooge`` is my docker hub username, ``docs-cstevens-io`` is my image name, and ``latest`` is a tag or a version number.

Next we will add a .travis.yml file (note the preceding period) in the build/html/ folder and put the following contents in it:

.. code-block:: text
   :caption: .travis.yml
   
   services:
     - docker
   
   script:
     - docker build . -t splooge/docs-cstevens-io
   
   after_success:
     - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_ID" --password-stdin
     - docker push splooge/docs-cstevens-io

This will instruct travis to build the docker image, just like we did locally, and then push the image to docker hub, where we can download it from later.

You'll need to log into travis-ci.org and setup your account to read your github repos for your automated builds.  You can log into travis-ci.org with your github account.  There's a sync button you can press when you first log into travis-ci, it will read your repos on github and populate them in travis-ci.org.  One thing we need to do is define the DOCKER_ID and DOCKER_PASSWORD variables in your repo settings.  This is your docker hub username and password that will be used to push your images to the docker hub registry.

Next, in your build/html/ folder, check in the new Dockerfile and .travis.yml files, again with ``git add .``, ``git commit -a -m "add Dockerfile and .travis.yml"`` and ``git push``.  travis-ci will notice you have a new .travis.yaml file, follow the instructions, checkout your repo, build the container image and then push it to docker hub.

Now that the image is on a registry (hub.docker.com) you can continue onto setting up kubernetes, which we will configure to pull the image from the registry.

TODO: instead of having the same name for all resources (docs-cstevens-io) it might be easier to read if I made them more unique to make a little more sense.

First I created a new namespace in kubernetes for my docs website.  Namespaces are a way of partitioning your kubernetes cluster for ease of management.

.. code-block:: text
   :caption: namespace.yaml

   apiVersion: v1
   kind: Namespace
   metadata:
     name: docs-cstevens-io

Next we will create a deployment.  A deployment will handle creating and scaling your kubernetes pods.  The first spec (specification) is the deployment spec, the 2nd is the container spec.

.. code-block:: text
   :caption: deploy.yaml

   apiVersion: apps/v1
   kind: Deployment
   metadata:
     labels:
       app: docs-cstevens-io
     name: docs-cstevens-io
     namespace: docs-cstevens-io
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: docs-cstevens-io
     strategy: {}
     template:
       metadata:
         labels:
           app: docs-cstevens-io
       spec:
         containers:
           - image: splooge/docs-cstevens-io
             name: docs-cstevens-io
             imagePullPolicy: Always

Now we'll create a service.  A service gives us an entrypoint to our deployment and will handle load balancing amongst those pods

.. code-block:: text
   :caption: service.yaml

   apiVersion: 1
   kind: Service
   metadata:
     labels:
       app: docs-cstevens-io
     name: docs-cstevens-io
     namespace: docs-cstevens-io
   spec:
     ports:
     - port: 80
       protocol: TCP
       targetPort: 80
     selector:
       app: docs-cstevens-io

Finally we'll create an ingressroute, which is Traefik's version of a kubernetes ingress, but not really within the scope of this document.  In this case, this ingressroute will match for the host headers "docs.cstevens.io" and forward traffic to the docs-cstevens-io service (which then forwards to the docs-cstevens-io deploy, which then forwards to the docs-cstevens-io pods)

.. code-block:: text
   :caption: ingressroute.yaml
   
   apiVesion: traefik.containo.us/v1alpha1
   kind: IngressRoute
   metadata:
     name: docs-cstevens-io
     namespace: docs-cstevens-io
   spec:
     entryPoints:
       - web
     routes:
       - match: Host(`docs.cstevens.io`)
         kind: Rule
         services:
           - name: docs-cstevens-io
             port: 80

We can then apply the configs by running

.. code-block:: text

   kubectl create -f namespace.yaml -f deploy.yaml -f service.yaml -f ingressroute.yaml

You can view the resources by running

.. code-block:: text

   kubectl -n docs-cstevens-io get ingressroute,service,deploy,rs,pod

You'll notice that kubernetes has downloaded the custom nginx image we created (splooge/docs-cstevens-io) from docker hub that has our documentation already installed on it.

You can scale up/down the number of pods behind the service, then monitor them by running:

.. code-block:: text

   kubectl -n docs-cstevens-io scale deploy docs-cstevens-io --replicas 5
   kubectl -n docs-cstevens-io get all
   kubectl -n docs-cstevens-io scale deploy docs-cstevens-io --replicas 2
   kubectl -n docs-cstevens-io get all

The site http://docs.cstevens.io should be available now.  You should now also be able to see the relevant information (routers, services) in the traefik dashboard.

When you next update your html, do a checkin, and let travis-ci build and publish your new image to docker hub, you can schedule kubernetes to download and run the latest image by running the following:

.. code-block:: text

   kubectl -n docs-cstevens-io rollout restart deployment
   kubectl -n docs-cstevens-io rollout status deployment docs-cstevens-io

Watch the replicaset spin up new pods and spin down old ones:

.. code-block:: text

   deployment.apps/docs-cstevens-io restarted
   Waiting for deployment "docs-cstevens-io" rollout to finish: 0 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 0 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 0 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 3 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 5 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 6 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 8 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 out of 10 new replicas have been updated...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 3 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 3 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 3 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 2 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 1 old replicas are pending termination...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 8 of 10 updated replicas are available...
   Waiting for deployment "docs-cstevens-io" rollout to finish: 9 of 10 updated replicas are available...
   deployment "docs-cstevens-io" successfully rolled out
