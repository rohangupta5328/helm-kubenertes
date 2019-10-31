# helm-kubenertes
This is a small application running nginx web server for illustrating how helm and kubernetes relates


What you can learn is:
   1) creating a deployment that will create two pods (two instances of the nginx app) and run it on one node
   2) creating a service that will enable users reach out to the two instances as if it was one with the help of the ingress that will create a load balancer
   3) creating an ingress, ingress will specify routes and creates a loadbalancer to balance requests between the various instances
   the routes specified in the ingress helps us have only one DNS name for our several services running in the cluster. For now its only one service but i've tried to show how that would be done


setup already done for this project:
    1)  `helm init` - initializes helm in your machine
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

`kubectl create serviceaccount --namespace kube-system tiller`

`kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller`

`kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'`