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