# Ditching Docker-Compose for Kubernetes?

Usually I incorporate Docker Compose into my local development workflow: Bringing up supporting containers needed to run databases, reverse proxies, other applications, or just to see how the container I'm developing works. Given that [Docker Desktop](https://www.docker.com/products/docker-desktop) comes with a [single node] Kubernetes (K8s) cluster and I usually end up deploying my containers to a Kubernetes cluster, I thought it would be good to figure out if I can switch from Docker-Compose to Kubernetes for local development. On top of that it's a good place to work the kinks out of my Kubernetes manifests or Helm charts without disrupting any shared environments.

To validate this I need to know how I'm going to handle the following:
* Building an image locally and running it on the K8s cluster.
* Making changes to an image and running that newly updated image in K8s.
* Running applications on K8s that require persistent data (volume mounts).
* Running applications on K8s that can communicate with applications running on the host OS.
* Have a app on the host OS that can communicate with images running on K8s.

I'm going to setup the following applications to validate all of this:
1. An API Server that keeps count of the number of requests. The count will be saved into a file so this is where we can get a handle on the persistence. This will also let us try out communication from a process on the host OS to an app running on local K8s. Finally this is where we can experiment with making updates to an image and redeploying. Let's call it the Request Count Server.
1. A Webapp that makes a call to the Request Count Server and displays the results. This will be the app will be a host-os process, so we can try out making a call from an app on the host OS to an app on K8s (the Request Count Server). We're going to with a Single Page Application (SPA) that is also Server Side Rendered (SSR) so we have a reason to throw in a ...
1. Reverse Proxy. This will run on kubernetes and direct some http requests to the Webapp and some http requests to the Request Count Server. This should showcase an app running in k8s able to communicate with an app running on the host OS as well as typically inter-cluster communication we expect from k8s.

So we should see the flow of information like this:

<!-- TODO: Fix diagram. Maybe use an image -->
Browser --> Reverse Proxy (K8s) --> Webapp (Host OS)
                         \--> Request Count Server (K8s)

An understanding of Docker and Kubernetes is required for this article. If you are following along from the [code base](https://github.com/PaulDMooney/Local-K8s-Dev-Blog/) make sure that Kubernetes is enabled in your Docker Desktop instance.


## The Request Count Server
This will be where a big chunk of the experiment is because it's where we're going to cover building an image and running it in k8s and, making changes, and persistence.

For a recap of what this is: It's a Node/express server that keeps count of and responds back the number of requests and saves it to a file, like a cheap database. It listens on port 3000.
The code can be found [here](https://github.com/PaulDMooney/Local-K8s-Dev-Blog/tree/main/request-count-server).

<!-- TODO: Keep this?? -->
We're going to pretend like this is something that we actively make changes to but would be too hard to run outside of a container (which is not true, but let's pretend).

### Initial Setup

The repo already contains the application, and the Dockerfile. The first thing we need to do is build the docker image:

`npm ci && docker build --tag request-count-server:local .`

This will build and store the image into Docker's local image cache, which is the same image cache K8s will use because it's the same docker instance. This means we don't need to worry about anything fancy like setting up a local [Docker Registry](https://docs.docker.com/registry).

Now let's deploy the application. My deployment.yaml looks like this:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: request-count-server
  labels:
    app.kubernetes.io/name: request-count-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: request-count-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: request-count-server
    spec:
      containers:
        - name: request-count-server
          image: "request-count-server:local"
          imagePullPolicy: Never
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
```
There are two important things here:
1. The value of the `image` field exactly matches the `--tag` value from our `docker build` command. For local development I advise using a tag that would only exist locally. 
1. The `imagePullPolicy` is set to `Never`. We want to avoid the value `Always` for local development because it will make docker think it needs to pull the image down from a remote registry like [Docker Hub](https://hub.docker.com/). Since the image does not exist on a remote registry it will get stuck on an image pull error. A value of `IfNotPresent` should work as well.

We will also need a `NodePort` type service.yaml to expose this application both inside and outside the cluster:
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: request-count-server
  labels:
    app.kubernetes.io/name: request-count-server
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: http
      protocol: TCP
      name: http
      nodePort: 30001 
  selector:
    app.kubernetes.io/name: request-count-server
```
Note that you don't have to use a `NodePort` service, you could use [kubectl port-forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#forward-a-local-port-to-a-port-on-the-pod) instead but you'll be running that command alot.

Now we can create our deployment and service:
`kubectl create -f deployment.yaml && kubectl create -f service.yaml`

<!-- TODO: Show output of running pods and services -->

And we should be able to access our application on the nodeport at localhost:
`curl http://localhost:30001/request-count`

response: `1`

### Redeploying with Changes

<!-- TODO: Rewrite this section for webapp-proxy since it makes more sense in terms of why we would do it? -->

So now that we have an built locally and running on our local K8s cluster. We can see what the experience of making changes and redeploying is going to be like.

Currently the application has hardly any output. I want to update that so it logs the request count to the console every time it increments. I'm just going to add a `console.log("Count", newCount)` after the response is sent, and I want to get those changes up into the app running on the cluster.

<!-- TODO: Show new application code? -->

Once I make the change to the application, I build the image the same way I did the first time: `npm ci && docker build --tag request-count-server:local .`

This unfortunately doesn't get the new version of the application up and running yet. If we run `kubectl get pods` we can see our `request-count-server-*` pod has been running for a while and if we run `kubectl logs request-count-server-*` we don't see our new output when invoking the service.

To get the new version of the application running in kubernetes we just need to get the pod restarted. Two ways to do this:
1. Scale the deployment down to 0 and back up again: `kubectl scale deployment/request-count-server --replicas=0` and then `kubectl scale deployment/request-count-server --replicas=1` (or how ever many replicas you want). This approach is good if you're running multiple replicas.
1. If you only have one replica then it's probably easiest to just delete the pod: `kubectl delete pod request-count-server-*`. The deployment will automatically start a new one. 

Once the new version of the application is running, we can follow the logs of the new pod: `kubectl logs -f request-count-server-*` and every time we invoke the service (`curl http://localhost:30001/request-count`) we should see a log output.
<!-- TODO: show log output -->

But the count has reset since we restarted the pod. It's starting from 1 again. We need it to persist.

### Persistence

The kubernetes way of persistence is through [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) (PV). To take advantage of that in Docker Desktop's kubernetes we can make a [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath) PV. Knowing that Docker Desktop is running in a virtual machine, if I used a hostPath where does it go? How do I get access to it to see what was created or clean it up after? Is this going to be a path somewhere into this magical virtual machine's disk? Probably. Luckily Docker Desktop has some file sharing with the host OS. We can seen and alter these paths in the dashboard under settings/preferences -> Resources -> File Sharing. 

I'm going to setup a persistent volume and a [persistent volume claim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims):
```yaml
# pv-local.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: request-counter-volume
spec:
  storageClassName: counter-volume
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: "/Users/Shared/request-counter-volume"
    type: DirectoryOrCreate
```
```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: request-count-server-claim
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: counter-volume
```
Note that my hostPath.path is a subpath of /Users which is one of the directories setup for file sharing in Docker Desktop. On Windows the paths will be different so the hostPath needs to be different, eg: `/host_mnt/c/path/to/file` or `/mnt/wsl/mnt/some/funny/loc` if running in WSL 2 mode. If you're sharing these persistent volume definitions with the team I recommend agreeing on what shared directories should be setup beforehand in Docker Desktop and also probably having separate PV definitions for MacOS and Windows.

We can create the Persistent Volume and Persistent Volume Claim:
`kubectl create -f pv-local.yaml && kubectl create -f pvc.yaml`

And then we can update the deployment mount the persistent volume claim. By default the request-count-server stores its data in a file called `/data/countfile.txt`, so we want the mount the pvc there:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: request-count-server
  labels:
    app.kubernetes.io/name: request-count-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: request-count-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: request-count-server
    spec:
      containers:
        - name: request-count-server
          image: "request-count-server:local"
          imagePullPolicy: Never
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          volumeMounts:
            - name: data-volume
              mountPath: /data
      volumes:
        - name: data-volume
          persistentVolumeClaim:
            claimName: request-count-server-claim
```

We apply these changes: `kubectl apply -f deployment.yaml`.
Now if we invoke `curl http://localhost:30001/request-count` we will see the count has started over from 1 again, but if we restart the pod (by deleting it: `kubectl delete pod request-count-server-*`) and invoke that endpoint again the count starts from where we left off. Persistence! We can even see and manipulate the file by going to `/Users/Shared/request-counter-volume` on the host OS.

## The Webapp

The webapp in this scenario will be what is truly in active development. It requires constant updates and automatic incremental rebuilds so I won't be deploying to kubernetes. It will be the application that runs on the host OS but it requires containerized applications running in Kubernetes to support it.

The application can be found [here](https://github.com/PaulDMooney/Local-K8s-Dev-Blog/tree/main/webapp). It is a simple [Angular](https://angular.io/) application that retrieves the latest count from the request-count-server and displays it. It can run with server side rendering turned on (via `npm ci && npm run dev:ssr` command). When it renders a page server side it can talk directly to the request-count-server at http://localhost:30001/request-count. I've demonstrated this can work previously when we invoked that same service using the `curl http://localhost:30001/request-count` command. However when it runs in the browser it cannot access the request-count-server because of [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS). So we need to a way around that. One way to solve this is with a [the Reverse Proxy](#the-reverse-proxy).

## The Reverse Proxy

The reverse proxy will be an nginx server that can direct traffic either to request-count-server or to the [webapp](#the-webapp) depending on the URL. Because I don't want to install nginx on my host OS (I rarely want to install anything anymore) I will run it as a containerized application in Kubernetes. Since this communicating with the webapp which is running on the host OS this nicely demonstrates running an application on Kubernetes that can communicate with applications running on the host OS.



## Helm

## Scripting

## Comparing to Docker Compose

### Building
### Persistence / Volumes
### Scripting

## Final Thoughts