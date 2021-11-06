# Kubernetes Workshop PXL 2021

## Introduction

In this Kubernetes workshop we'll be looking at how we can isolate our applications with containers, deploy them on Kubernetes with Pods, scale them with ReplicaSets and Horizontal Pod Autoscalers, update them without downtime with Deployments and expose them with Services. Additionally we'll explore some basic ways to pass configurations and secrets, and how we can give our application access to a file system.

#### Example application

To situate the exercises in a more real-world scenario, we'll be using an example application `crypto-ticker`. It's a simple 1-tier Node application (EJS + Express) with two versions:

- Version 1 shows the latest Bitcoin price
- Version 2 shows the latest Bitcoin and Ethereum price

Both versions have extra functionality to observe some Kubernetes features that we'll be exploring. Containerized versions are already available to you on the Packages section of this [GitHub repository](https://github.com/driesvb-dtgc/kubernetes-workshop-pxl-2021/pkgs/container/kubernetes-workshop-pxl-2021%2Fcrypto-ticker). When running the container, the application is exposed on port 3000.

#### Assumptions

- When an exercises asks to look at a file starting with `/repo`, it means that you can find that file in the GitHub repository of this workshop.
- Often when asked to deploy a Kubernetes object using a YAML file, a template will be given with markers where you should add additional specification. Do not change the parts that are already there as it might break the continuity of the exercises.



## Containers

Have a look at the different steps in the `/repo/containers/Dockerfile` used to build the `crypto-ticker` app. 



**Exercises**

- Why are first the `package` files copied to the working directory separately and the npm packages installed first (line 6 and 8), only then to copy all our files to the working directory (line 10)? Couldn't we simply copy over all the files and then run the `npm install` step, skipping the first copy step? Is there a disadvantage?

- Run the `crypto-ticker` application as a Docker container with 

  `docker run -p 8080:3000 -d ghcr.io/driesvb-dtgc/kubernetes-workshop-pxl-2021/crypto-ticker:v1`

  The `-p` part port forwards our local port 8080 to port 3000 on the container. You should now be able to access the application on `http://localhost:8080`

- Stop the container with `docker stop <container ID>`

Notice how easy it was to run a Node application from source with multiple dependencies without having to install any of those on your machine, possibly conflicting with dependencies and Node versions that you already have.



## Pods

A Pod is the smallest deployable unit of computing deployable on Kubernetes. It contains one or more containers with shared storage and network resources. 

As a first step, let's deploy the `crypto-ticker` application (v1) as a single-container Pod and try access the application on our machine.



**Exercises**

- Deploy a Pod called `crypto-ticker` running container `crypto-ticker:v1` using the *imperative* approach (i.e. using a `kubectl` command) with

  `kubectl run crypto-ticker --image=ghcr.io/driesvb-dtgc/kubernetes-workshop-pxl-2021/crypto-ticker:v1`

  This submits a request to the Kubernetes API service which creates a Pod and schedules it on one of the cluster nodes

  - List the running pods with `kubectl get pods`. What is the status of the pod? What do you think the value under the `READY` column means?
  - Check the logs of the pod with `kubectl logs crypto-ticker`. What kind of logs are these? Kubernetes system logs? Application logs?
  - Check the details of the pod with `kubectl describe pod crypto-ticker`
    - What are the labels of the pod? What do you think these are useful for?
    - What is the cluster-internal IP of the pod? Try to browse to it on port `3000`. Are you able to reach it? Why not?
    - Check out the steps the pod went through to start the container under the `Events` section. These are useful for troubleshooting should your pod not start successfully.

  - The IP of the Pod is local to the cluster and hence not accessible from our machine. We can however forward our local port 3000 to port 3000 on the pod with `kubectl port-forward crypto-ticker 3000:3000`. This is useful for testing but certainly not the preferred way to expose your service. Try reaching the application on `localhost:3000` in your browser. Explore the application for a bit.

  - Delete the pod with `kubectl delete pod crypto-ticker`

- Deploy a pod running container `crypto-ticker:v1` using the *declarative* approach (i.e. by applying a YAML file with `kubectl apply -f <yaml file>`). You can find a template to start from at `repo/pods/crypto-ticker-pod.yaml`

  - Port-forward the pod to a local port and browse to the application.

  - In a new terminal, keep a watch on the status of the pod with `kubectl get pods --watch`

  - In the application, browse to "System control" and click the "crash application (container)" button. This will make the container exit with a non-successful code (-1). Keep an eye on the watch. What happens? Is the container recreated? What Kubernetes entity is responsible for this?

    This already demonstrates some self-healing capabilities of Kubernetes, even from a simple object such as a Pod.

  - In essence, a Pod is still running (at least one) containers. We can get a shell in a container with `kubectl exec --stdin --tty <pod name> -- sh`. Try this out for the `crypto-ticker` pod. Explore the file system for a bit (e.g. use `ls` to list the files). Can you see the application files? Notice that this file system is completely isolated and unaware from your local file system.
    You can exit the shell with `CTRL+C`

  - Delete the pod



## ReplicaSets

Pods are the foundation of a Kubernetes deployment. However, what if a single pod can't handle the load? Do we start new pods with `kubectl run` if our traffic increases, and delete them again if it decreases? What if a pod crashes, do we restart it ourselves? It's a job no-one wants to do manually.

Manually managing pods is clearly tedious work. Fortunately, we can let Kubernetes do it for us using ReplicaSets. A *ReplicaSet* is a *controller* that takes a Pod specification as a template and makes sure a certain number of pods are running at all times, no matter what happens to them individually. The controller will continuously check whether the requested amount of replicas is running successfuly, and if not add or remove individual Pods. Remark that a ReplicaSet *does not* have an IP and does not expose an entry point load balancing traffic over the Pods it controls. It simply oberves the Pods and adds or removes them as needed.

In practice you rarely directly use ReplicaSets and use Deployments as it has some additional capabilities. However it's useful to experiment with them to understand how a Deployment works later.



**Exercises**

1) Deploy a ReplicaSet `crypto-ticker-rs` that keeps 3 replicas of a `crypto-ticker` pod running. You can re-use the YAML you created during the Pod section as a ReplicaSet spec contains a Pod spec as a template. A RS spec template is given in `/repo/replica-sets/crypto-ticker-rs.yaml`

2) List the pods. How many pods are running?
3) List the pods with a watch (`kubectl get pods --watch`). In a separate terminal, delete one of the pods. What happens? 
4) Keeping the watch open, deploy a single `crypto-ticker` pod with the specification you created earlier. Can you explain what happens?
5) Delete the ReplicaSet with `kubectl delete rs crypto-ticker-rs`. What happens to the pods it managed?
6) Deploy a single `crypto-ticker` pod with the specification you created earlier
7) Redeploy the ReplicaSet
8) How many pods are running? Did you expect a different result? Can you derive from this what the ReplicaSet uses to identify the pods it should manage?
9) Rescale the ReplicaSet by setting the `spec.replicas` field in the YAML specification to `4`. Reapply the specification. How many pods are running now?
10) Delete the single manually deployed `crypto-ticker`pod but **keep the ReplicaSet deployed**



### Horizontal Pod Autoscaling

With ReplicaSets we are already a step closer to a self-healing system that can adapt its resources to the load, as we can easily scale the number of pods that are running up and down by simply adjusting a specification. However, what we really want is to *automatically* resize the ReplicaSet based on metrics such as CPU or memory utilization (so we can finally sleep better at night). It happens often that an application is used more at certain periods of the day or suddenly has high peak usage when a certain event occurs. You can't always see this coming. We need a way to auto scale the ReplicaSet.

The *Horizontal Pod Autoscaler* is another controller that continuously checks different configured metrics and automatically rescales the ReplicaSet it manages when needed. Default metrics are the CPU or memory utilization (both on pod and container level). Quite complex scaling strategies and behavior can be defined. You can also set up and use custom metrics so that you can use your knowledge about your application to let the HPA know its current utilization.

Note: You also have a Vertical Pod Autoscaler, that instead of adding/removing Pods, adds and removes resources allocated to the individual Pods.


**Exercises**

**Note:** Minikube can experience issues with HPA because of a faulty metric service. It's possible that it won't work for your installation.

1) Deploy a HPA for the `crypto-ticker-rs` ReplicaSet scaling at a CPU utilization rate of 60% with a minimum of 1 pod and a maximum of 3 pods (you can use a `kubectl` command for this instead of a YAML spec)
   `kubectl autoscale rc crypto-ticker-fs --min=<min amount of Pods> --max=<max amount of Pods> --cpu-percent=<percentage>`

2) How many pods are running? 

3) (Optional) Manually rescale the ReplicaSet to 4 replicas. 

4) (Optional) You can put a watch on the Pod list, port-forward to a pod and load test it to see the scaling behavior however this will depend on your machine.

5) Delete the HPA

   

## Services

Our application deployment is taking shape! It's now able to auto scale and can easily handle crashing containers and pods without impacting availability.   What is still missing is a better way to access it in a maintainable way, whether it is by external users or different kinds of Pod groups (e.g. a frontend Pod calling a backend Pod). We have a whole bunch of pods running on different nodes, each with different IP addresses which can change constantly as pods are added and removed by the controllers. It's hence not realistic to use those as an entry point. What would be better is a more static set of access points, or even better, one single point of access, abstracting away the pods underneath.

A way to accomplish this is through *Services*. Services are an abstraction of a logical set of pods. What pods belong to that set is determined by a label selector. In Kubernetes you have three types of Services: NodePort, ClusterIP and LoadBalancer. We will consider each one.



### ClusterIP

A ClusterIP service is the default service and provides one single cluster-internal logical point of access to a group of pods for **internal** cluster traffic. A scenario is a group of pods running an app backend being exposed by a single ClusterIP so that they can be accessed by the Pods running a frontend application on a single, static IP and port decoupled from how many Pods are behind that entry point.

Let's deploy a ClusterIP for our `crypto-ticker` pods.



**Exercises**

1) If it's not running anymore, redeploy the `crypto-ticker-rs` ReplicaSet so that we have some Pods running.
2) Deploy a ClusterIP Service `crypto-ticker-cluster-ip-service` targeting the Pods managed by the ReplicaSet. You can use the template in `/repo/services/crypto-ticker-node-port-service.yaml`
   Hint: look at the labels of the Pod and use those as a selector. 
   Make sure the service is exposed on port 80 (the default port for a HTTP application like ours) instead of port 3000 that is being exposed by the Pods (Hint: use `spec.ports.port` and `spec.ports.targetPort`)
3) Check out the service with `kubectl get services`. What is the Cluster IP? Can you access it in your browser on port 80?
4) As mentioned before, this type of Service is only accessible *locally inside the cluster*. Our local machine network (which is not local to the cluster) can't reach it, however a Pod should be able to as it is part of the local network of the cluster. Let's test this!
   1) Get a shell to one of the running pods with `kubectl exec --stdin --tty <pod name> -- sh`
   2) Try to get the index page of the application with `wget --server-response http://<service IP>`. Are you able to reach it?
   3) Notice that you are being served by one of the Pods targeted by the Service. What Pod is completely abstracted away. The Service forwards the request to a random Pod, essentially load balancing the requests over the different Pods.
5) Delete the Service



### NodePort

A ClusterIP is very useful for microservices needing to communicate with each other internally. However, some groups of Pods should also be accessible from outside the local cluster network (e.g. a group of Pods running a frontend application). For this we can use a NodePort Service.

A NodePort Service exposes a group of Pods by opening a static port on *each* of the nodes in the cluster which can be accessed by external traffic (however in practice you don't want to expose your nodes directly to the Internet as this makes your infrastructure more vulnerable to attacks). Any traffic going to this port is load balanced and forwarded to a target port on one of the selected Pods. As the node IPs *are* part of our local network we should be able to reach it from our machine. Let's deploy such a Service so that we can finally access our application using a static IP from our machine!



**Exercises**

1) Deploy a NodePort Service `crypto-ticker-node-port-service` starting from the template in `/repo/services/crypto-ticker-node-port-service.yaml`. Make sure the opened nodeport on each node is `30050`.
2) List the nodes and their internal IP addressess with `kubectl get nodes -o wide` . There will only be one node on Minikube. However, in practice there will be many nodes with each their own internal IP address. They will each have `30050` port open due to this NodePort Service so you could pick any node IP to reach a targeted Pod.
3) Browse to `http://<internal node IP>:30050`. Can you reach the application?
4) **Keep this service running, we will need it later**

With exposing the group of Pods on a port of each of the nodes, the set of IP addresses that we have to maintain is much more limited as nodes are less volatile than Pods (however a cluster can certainly also auto scale). In theory you could maintain DNS records for each of the node IPs so that the Service is reachable through a user-friendly domain name. However in any real system this will rapidly become impractical. We are still missing one piece of the puzzle to a highly available service: a Service that will maintain the list of IP addresses for us and instead expose one (or more) static IP address and port that will forward our request to one of the nodes.



### LoadBalancer

Queue in LoadBalancer Services. This type of Service will typically be implemented by a cloud provider such as Google Cloud Platform (e.g. an external HTTPS load balancer). By default a LoadBalancer will use a NodePort service in the backgroud and maintain the list of node IP addresses for us, exposing a single external IP address through which the application can be reached.


**Exercises**

It's possible to emulate this on Minikube, however it will be more interesting to see it in action on an actual cloud platform such as Google Cloud Platform. After this part of the workshop your host will show you a demo.



## Deployments

Awesome! Our application is now fully accessible to our users! With Bitcoin skyrocketing it's being used a lot. You do however notice demand coming from your users to also have a view on the price of another interesting cryptocurrency: Ethereum.  Being the pro coder you are you quickly add this feature to your codebase and build a version 2 container of the app.

Think about how we will deploy this new version on Kubernetes. A straightforward approach is changing the tag of the container to the new version in the ReplicaSet specification and redeploying it. However, this will first terminate all the existing pods and then spin up the Pods with the new version, possibly leaving no Pods running to serve incoming requests, leaving the application temporarily inaccessible to the users. In many systems today this is unacceptable.

To automate a more gradual update rollout, ensuring availability in the meanwhile, *Deployments* provide the magic we need. Deployments are yet another controller managing ReplicaSets to facilitate rolling updates of our application from one version to another, keeping a healthy amount of Pods available during the process. Upon an update, the Deployment will create an empty new ReplicaSet for the Pods running the updated version, step-by-step scale down the old ReplicaSet and scale up the new ReplicaSet, always ensuring Pods are available to serve traffic. In its default strategy configuration, this is called a **rolling update**. Parameters like `maxSurge` and `maxUnavailable` determine by how many Pods the ReplicaSet is allowed to deviate from its replication setting. As the ReplicaSets scale down and up, different users might temporarely reach different versions of the application depending to which Pod they are forwarded too by e.g. a service that is targeting the application Pods.

During deployments it's also possible to pause or roll back to  in case you notice the new version is causing issues. Even after a complete deployment it is possible to roll back to previous revisions with simple commands.



**Exercises**

1) Delete all ReplicaSets and Pods that might still be running. You can get a list of all resources with `kubectl get all`**. Keep the NodePort service running**
2) Deploy a Deployment starting from the template in `/repo/deployments/crypto-ticker-deployment.yaml` . Make sure the ReplicaSet specification within still creates Pods with the version 1 container. You can re-use the ReplicaSet specification you created earlier.
3) List all ReplicaSets and keep the command on watch. How many do you see? How many Pods are ready?
4) Delete the ReplicaSet (**not** the deployment). What happens? 
5) List all Pods. How many do you see? Notice that the situation is much the same as if we'd only deploy a vanilla ReplicaSet. The Deployment will kick in once we make changes to the ReplicaSet specification.
6) Make sure you can still reach your v1 application through the NodePort service you deployed earlier. Keep a tab in your browser open.
7) Update the application to version 2 by changing the container tag from `v1` to `v2` in the Deployment specification and re-applying the specification.
   - Keep an eye on the ReplicaSet watch. Notice that a new ReplicaSet is added and the old ReplicaSet gradually being scaled down as the new ReplicaSet is being scaled up. At all times there are Pods available to serve traffic. 
   - Try refreshing the application multiple times in your browser while the Deployment is running. At one time you might still see the old version while, more and more, you will see the new version, depending on what Pod the NodePort service routed you to. Once the old ReplicaSet is scaled down completely, all traffic will be routed to the Pods controlled by the new ReplicaSet.
   - While the Deployment is running you can see its status with `kubectl rollout status deployment/crypto-ticker-deployment`
8) Check the history of application revisions with `kubectl rollout history deployment/crypto-ticker-deployment`. How many are there?
9) Some Bitcoin maximalists are demanding you to roll back to the previous version only supporting Bitcoin... . Roll back the Deployment with `kubectl rollout undo deployment/crypto-ticker-deployment`. (Note: you can roll back to any previous revision by specifying the `--to-revision` parameter)
   

Note: In real-world systems, a Deployment is usually preferred to manage a ReplicaSet instead of directly deploying ReplicaSets yourself because of the additional rolling update capabilities and it only acting as a "wrapper" around the ReplicaSet concept. Should a ReplicaSet controller fail then the Deployment will also automatically recreate it. The same Horizontal Pod Autoscaling objects can also be applied to deployments. 



## Extra Topics

### Volumes

When containers run in the same pod, they can share the same file system volumes. The type of these volumes  and where they are mounted inside the containers is specified on the container level in the Pod specification. Some volume types provide persistent storage (i.e. storage that is persisted even upon a Pod crashing) while others provide ephemeril storage (i.e. storage that is wiped clean upon termination of the Pod). Persistent storage is often backed by a virtual drive on a cloud provider (e.g. Persistent Disk on GCP)  or by a more traditional file system (e.g. an implementation of NFS) so that it remains intact even if containers mounting it are destroyed and nodes are being shut down. Luckily all of this is abstracted away for the containers and application developers. From the point of view of the container, it is simply a directory or file mounted in its local file system, no matter where it actually resides.

For the exercises we'll limit us to mounting a single file in the containers of our `crypto-ticker` Pods.


**Exercises**

We will be first mounting a EmptyDir volume in the container of our `crypto-ticker` Pods. This type of volume is created and mounted when a Pod is started on a node and is initially empty. We will then be creating a file with our private Bitcoin key in that directory. 

1) Make sure at least one `crypto-ticker` Pod is up (e.g. using a single Pod deployment) and you are able to access the app from your browser (e.g. using `kubectl port-forward`).
2) Go to the "File system" page of the application. This page is looking for a file `/crytpo/wallet.dat` on the file system of the container. If it is found, the contents will be displayed. 

To provide the file we will be first creating an EmptyDir volume and mount it as `/crypto` in the container's file system. We will also add a start script to the Pod specification that is executed when the container is started that downloads our `wallet.dat` file from Google Cloud Storage (https://storage.googleapis.com/kubernetes-workshop-pxl-2021/wallet.dat) in the correct `/crypto` directory.

1) Adjust the specification to add an EmptyDir volume named `wallet-volume` to the Pod. You can start from the template in `/repo/volumes/crypto-ticker-pod.yaml`

2) Adjust the specification to mount the `wallet-volume`  volume at `/crypto` in the v1 container.

3) Adjust the specification to add a start script to download the `wallet.dat` file from GCS into the `/crypto` directory

4) Redeploy the Pod. You might have to delete the older Pod first as changes to the lifecycle of a Pod (here: the start script) can not be updated in-place

5) Get a shell in the Pod's container with `kubectl exec --stdin --tty crypto-ticker -- sh`. Explore the container's file system. Can you find the `/crypto` directory? Can you see the `wallet.dat`? Try printing its contents, is there a key in there?

6) Port-forward to the Pod and access the application in your browser. Can you now see the key on the File System page?

7) Keep the port forward running and adjust the contents of the `wallet.dat` file using the container's shell, e.g. with command:

   ```
   "Your content" > /crypto/wallet.dat
   ```

   Refresh the File System page. Has the content adjusted to what you wrote?



### ConfigMaps

It's a good software engineering practice to keep your application agnostic of the environment it is running in, but rather pass environment-dependent details as configurations to your container at runtime. This is possible with *ConfigMaps*. In your Pod specification, you can define key-value configuration pairs. They can be made available to the container in 4 ways:

- As a a container command
- As an environment variable
- As a read-only volume
- Through the Kubernetes API from within your application code

ConfigMaps are objects on their own and can be submitted to Kubernetes uncoupled from any other resources using them. Then, Pods can reference the ConfigMap objects in their specification and expose them in the mentioned ways. You can hence re-use the same configurations in different Pod specifications.



**Exercises**

- Create and deploy the specification for a ConfigMap object. You can start from the specification in `/repo/config-maps/crypto-ticker-config-map-dev.yaml`. You can choose your own key-value pairs.
- Adjust your Pod specification to expose one or more of the keys in the ConfigMap as an environment variable.
- Deploy your Pod and port-forward to it.
- Navigate to the "System Info" page of the application. This page shows all the environment variables visible for the container. Can you see your configuration's variables?



Note: ConfigMaps are not fit to store secrets such as API keys or passwords. A more suited object is a Secret. You can reference to a Secret object in your Pod and expose it to your containers much like a ConfigMap. By default, the secrets are stored in Kubernetes' API server's data store (etcd). However, in practice, once you have many nodes possibly spread over multiple clusters, it becomes hard to manage these secrets on the different clusters. That's why in production systems often a centralized secret manager outside of Kubernetes (e.g. provided by a cloud provider) is used and let the application itself use the API or SDK of that secret manager to retrieve the secret at runtime. An example is Google Cloud Secret Manager.



















