# helm-kubenertes
This is a small application running nginx web server for illustrating how helm and kubernetes relates


What you can learn is:
   1) creating a deployment that will create two pods (two instances of the nginx app) and run it on one node
   2) creating a service that will enable users reach out to the two instances as if it was one with the help of the ingress that will create a load balancer
   3) creating an ingress, ingress will specify routes and creates a loadbalancer to balance requests between the various instances
   the routes specified in the ingress helps us have only one DNS name for our several services running in the cluster. For now its only one service but i've tried to show how that would be done


setup already done for this project:
    1)  `helm init` - initializes helm in your cluster
        `helm create kubelab-chart` generates the folder structure and necessary files
           chart.yml - contains the chart metadata, version and name
           requirements.yml - would contain a declaration of any dependencies your app depends on e.g. mongoDb
           values.yml - key pair values / variables that would be interpolated into your templates
           _helpers.tpl - contains functions that would create and interpolate values to the templates
           templates/ - a folder containing kubernetes deployment templates
           running `helm search <package_name>` would give you versions of that package, 
                   `helm dep list` - shows you a list of dependensies declared but not downloaded
                   `helm dependency update .` - downloads dependencies and places them in the chart folder
     

steps to follow:
    1) create your kubernetes cluster
    2) running  `helm template .` - will show you what will be applied to tiller
                `helm install .` - will then apply the deployments in the cluster, so in the wood, tiller would run a command like: `kubectl apply -f <generated_file>`
                running `helm install --name=nginx-v1 .` will ovveride the name


Incase of the error : forbidden: User "system:serviceaccount:kube-system:default" cannot get namespaces in the namespace "default, then run the following commands


// create service account in the tiller with full sudo rights / permissions that will manage resources e.g.
delete any resource, create any resource. Ofcourse not ideal for production where you need to create a role with 
only specific tasks tiller is allowed to perform.
    `kubectl create serviceaccount --namespace kube-system tiller`
    `kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`
    `kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'`

other commands:
    `helm list` - lists applications installed in your cluster
    `helm status` - status of application deployment
    `helm upgrade` -
    `helm rollback` - go back to previous deploment
    `helm delete (--purge)` - deletes deployment, with purge deletes history
    `helm reset (--force)` - deletes the tiller client

AWS
It's much easier to run kubernetes in google but incase you switch to AWS, follow the following procedure to setup environments:

1.) create cluster
2.) create worker nodes by lauching worker node in desired region: 
    for more info: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html
3.) link the two by running below: kubectl apply -f aws-auth-cm.yaml

aws-auth-cm.yaml:
```
    apiVersion: v1 #for connecting aws nodes to the cluster
    kind: ConfigMap
    metadata:
    name: aws-auth
    namespace: kube-system
    data:
    mapRoles: |
        - rolearn: <ARN of instance role (not instance profile)>
        username: system:node:{{EC2PrivateDNSName}}
        groups:
            - system:bootstrappers
            - system:nodes
```

4.) to switch to AWS eks in local machine, update-kubeconfig by running - 
`aws eks --region <region> update-kubeconfig --name <cluster name>`

5.) Incase you are running kubernetes clusters in different providers e.g. aws, minikube and gcp then you can get all contexts and switch between them using below commands:
     -`kubectl config get-contexts`
     -`kubectl config use-context CONTEXT_NAME`
     
     
     
**more on kubernetes**
- creating a pod : ```kubectl run <pod name> --image=nginx:alpine```
- When a pod is brought to live, it’s given a cluster IP address which is only accessible within the cluster, to access a pod outside of the cluster(e.g. from a browser), then use port-forward : ```kubectl port-forward <podname> <external port>:<internal port>```
- ```Kubectl delete pod``` will delete the pod but will spin a new one (auto-recovery). If you want to completely delete a pod and should never come back then use ```kubectl delete deployment```
- Yml files is a collection of maps and lists, declarative approach
- ```Kubectl create -f file.yml``` will throw an error if the resource already exists, ```kubectl apply -f file.yml``` will update the resource
- You can edit a resource either with set, edit or patch commands
- ```kubectl exec -it <pod-name> ls /``` is used to execute a shell command on a pod. It is same as: ```docker exec -it --user root jenkinsnode bash -c "ls"``` used to execute command inside docker container as root
- to get a shell session with a pod, do:
```
kubectl exec <pod name> -it sh
> apk add curl // to add a package e.g. curl
> curl -s http://podIP or podName // to test connection to another pod
```

**pod health**
- kubernetes relies on probes to determine the health of pods
- probes are just diagnostics performed periodically by the kubelet in a container
- you can write the probes in code as a developer or in yml file
- we have 2 types of probes, liveness probs and readiness probes
 
 **creating probes in yml file**
 1.) HTTP liveness probe
```
apiVersion: v1
kind: Pod

...

spec:
  containers:
  - name: my-nginx
    image: nginx:alpine
    livenessProbe: // define readiness probe
      httpGet:
        path: /index.html // check index page
        port: 80
      initialDelaySeconds: 15 // wait 15 seconds
      timeoutSeconds: 2 // timeout after 2 seconds
      periodSeconds: 5 // check every 5 seconds
      failureThreshold: 1 // allow 1 failure before failing the pod
```
2.) Exec liveness probe, straight from documentation
```
apiVersion: v1
kind: Pod

...

spec:
  containers:
  - name: my-nginx
    image: k8s.gcr.io/busybox
    
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30;
      rm -rf /tmp/healthy; sleep 600 // to test failure

    livenessProbe: // define readiness probe
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5 // wait 5 seconds
      periodSeconds: 5 // check every 5 seconds
```
- the above pod will be alive untill the file is removed then it will fail

3.) Readiness probes
- determines when should traffic start routing to the container
```
apiVersion: v1
kind: Pod

...

spec:
  containers:
  - name: my-nginx
    image: nginx:alpine
    readinessProbe: // define readiness probe
      httpGet:
        path: /index.html // check index page
        port: 80
      initialDelaySeconds: 2 // wait 2 seconds
      periodSeconds: 5 // check every 5 seconds untill the request is successful
```

**difference between a pod and a deployment**
- Both Pod and Deployment are full-fledged objects in the Kubernetes API. Deployment manages creating Pods by means of ReplicaSets. What it boils down to is that Deployment will create Pods with spec taken from the template. It is rather unlikely that you will ever need to create Pods directly for a production use-case
- A deployment is a declarative way to manage Pods using a replicaSet:
```
deployment -> replicaSet -> Pod
```

**functions of replicaSets**
- brings a self healing mechanism
- ensures we have the righ number of pods
- provide fault-tolerance
- can be used to scale pods
- relies on pod template
- no need to create pods directly
- used by deployments

**role of deployment**
- pods are managed using replicaSets
- scales replicaSets which scales pods
- supports zero-downtime updates by creating and destroying replicaSets
- provides rollback functionality
- creates a unique label that is assigned to the replicaSets and generated pods 

* the nice thing about creating a deployment is that you don't need to create a replicaSet, that will be handled automatically behind the scenes

- ```kubectl get deployment --show-labels``` show a list of deploments and labels
- ```kubectl get deployment -l switch``` show a list of deploments with label switch
- ```kubectl scale deployment <deployment name> --replicas=5``` creates 5 pods
or scale by referncing the yml file ```kubectl scale -f file.deployment.yml --replicas=5```
or 
```
spec:
  replicas: 3
  selector:
    tier: frontend
```

**resources**
- It's important to keep a limit to the resources (memory and cpu usage) so that incase of anything an individual container does not bring down the whole node
```
apiVersion: app/v1 # deployment
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: my-nginx
spec: # replica set
  replicas: 4
  minReadySeconds: 10 # wait for the pod to make sure the container hasn't crashed for 10 seconds before getting traffic. Useful for cases when a pod 1st starts, a container might crash and will have to be rescheduled.
  selector:
    matchLabels:
      app: my-nginx
  template: # pod
    metadata:
      labels:
        app: my-nginx
    spec:
      containers:
      -name: my-nginx
       image: nginx:alpine
       ports:
       -containerPort: 80
       resources:
         limits:
           memory: '128Mi' #128mb
           cpu: '200m' #200 millicpu (.2 cpu or 20% of the cpu)
```

**deployment options**
- one of the kubernetes is zero downtime deployments, that means it can bring up new pods and once they are ready kill the old ones

**options**
- rolling updates (default)
- blue green deployments - have multiple env running at thesame time and once proven you switch to the desired one
- canary deployments - a very small amount of traffic goes to the new deployment and once proven to work then you switch all traffic
- rollbacks - going back to previous deployment


**services**
- same as port-forward but a little more official :-)
- a service provides a single point of entry for accessing one or more pods
- we can't rely on the IP address of a pod because they change alot hence the ip address change also
- also pods scale horizontally and each with it's own IP
- pod's ip is assigned after scheduling and no way to know in advance

- labels are used to associate pods with a service
- services loadbalances traffic between pods
- node's kube-proxy creates a virtual IP for services
- this uses layer 4 (tcp/udp over ip)
- services are not emphemeral (short lived)
- creates endpoints
- browsers will keep thesame k8s service connection (session)

**service types**
- ClusterIP (default) - service IP is exposed internally within the cluster (expose the service on a cluster internal IP)
                      - only pods within the cluster can talk with each other or to other pods
- NodePort - expose the service on each node's IP at a static port (range: 30000 - 32767) (where we have an IP address for a node and a static port to access it)
           - each node proxies the allocated port
- LoadBalancer - sits in front of our different nodes and provision an external IP to act as a loadbalancer to call into the nodes and then pods
               - exposes a cluster externally
               - is combined with cloud provider load balancer
               - nodePort and clusterIP services are created as well
               - each node proxies the allocated port
- ExternalName service - that maps a service to a DNS name
                       - service that acts as an alias for an external service
                       - allows a service to act as the proxy for an external service
                       - external service details are hidden from cluster and makes it easier change

**port-forwarding**
```kubectl port-forward pod/<pod name> 8080:80``` # listen on port 8080 locally and forward to port 80 in pod
```kubectl port-forward deployment/<deployment name> 8080:80``` # listen on port 8080 locally and forward to deployment's pod
```kubectl port-forward service/<service name> 8080:80``` # listen on port 8080 locally and forward to service's pod


services with different names in the metadata section will be allocated different dns entries
e.g.
```
apiVersion: v1
kind: Service
metadata:
  name: frontend
```

```
apiVersion: v1
kind: Service
metadata:
  name: backend
```

- these 2 will be assigned different dns entries
- a frontend pod can access a backend pod using `backend:<port>`

**external name**
- instead of having to reference an external dns name all over your pods, they instead call an external name e.g.
```
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.george.komen
  ports:
  - port: 9000

```
- in essence it's creating an alias or a proxy of a name to the dns entry. It makes it easier to manage such external dependencies that might change

**node port**
- when using nodePort service, the IP address is that of the host machine. So if running k8s locally then you can access the service through `localhost:<nodePort>`, the node port selected is the external port
- the hops for a nodeport service would be `nodePort -> port -> targetPort`

**load balancer**
- this does some magic because it creates the localhost loopback address so you can access the port through : `localhost:<port>`
- great for development because of the localhost loopback
- loopback address. An address that sends outgoing signals back to the same computer for testing. In a TCP/IP network, the loopback IP address is 127.0. 0.1, and pinging this address will always return a reply unless the firewall prevents it.
- Broadcast address is the last address in the network, and it is used for addressing all the nodes in the network at the same time.