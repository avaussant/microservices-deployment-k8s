## Microservices deployment on k8s

### Description: This is demo project where I deployed a Flask-based microservice (along with Postgres and React.js) to a Kubernetes cluster. Using different kubernetes components & objects the app is successfully deployed on local kubernetes like k3d and minikube. 

#### _Find out the demo app frontend written in React.js [here](https://github.com/faysalmehedi/reading-list-frontend) and backend wriiten in Flask [here](https://github.com/faysalmehedi/reading-list-backend)_

#### Features:
- Configure a Kubernetes cluster to run locally with Minikube and k3d
- Frontend is written in React framework and used multi-stage build with Nginx for containerization. 
- Backend written in flask framework and dockerized. 
- Postgresql Database is used for storing data So I used PersistentVolume and PersistentVolumeClaim for this purpose
- Secrets objects are used to manage sensitive information
- Expose Flask app and React app to external users via an Ingress

#### Setting up local kubernetes cluster
For this demo I used most popular tool `minikube` which quickly sets up a local Kubernetes cluster. I also tested with another polpular tool `k3d(based on k3s` and it worked fine for me too!

Minikube or k3d can be installed on any machine with any virtualization tool; even only docker(deploys as container) is enough for the setup. 

Got to [minikube documentation](https://minikube.sigs.k8s.io/docs/) or [k3d documentation](https://k3d.io/) for details. It's very easy!

`kubectl` is the basic command line utility for managing kubernetes cluster. So, this needs to be installed also. Go to [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) for the details.

#### Start up minikube
Start the minikube cluster with two worker nodes and one master nodes
```bash
# this will start master node with two worker nodes
minikube start --nodes 3

# check the nodes using kubectl
kubectl get nodes

# check the kube-system namespaces for the kubernetes components running
kubectl get pods -n kube-system
```

#### Persistent Volume
Next step is to create a persistent volume for the storage for database and claim the volume. They will be on separate on files.

```
Apply them to the cluster

```bash
# create pv and pvc
kubectl apply -f postgres-persistent-volume.yaml
kubectl apply -f postgres-persistent-volume-claim.yaml
# check the status
kubectl get pv
kubectl get pvc
```

#### Secrets
Create a secret object for managing the sensitive information for postgres. Values from secrets will be reffered on different k8s objects later.

```bash
# secrets.yaml 'data' fields are base64 encoded strings
echo -n "postges" | base64

# apply the yaml file
kubectl apply -f secrets.yaml
kubectl get secrets
```

#### Postgresql pod deployment and services

Now it's time to create deployment for the postgresql database with the volume and database credentials set up in the cluster.

```bash
# apply the postgres-deployment.yaml
kubectl apply -f postgres-deployemt.yaml
# check the deployments and pods
kubectl get deployments
kubectl get pods
```

A service is needed to work as reliable endpoint for the backend api call. Services will monitor the pods and will send traffic by load balancing in random alogorithm. This time postgres service will be ClusterIP which is default.

```bash
# create postgresql service using postgres-service.yaml
kubectl apply -f postgres-service.yaml
# check the status 
kubectl get svc

# get all the status of services, deployments and pods
kubectl get all
```
#### Flask (Backend) microservices deployment

In the flask microservices there is two api right now one is for getting all the books registered and on is to add the book on the list. These api will use the postgres pods as their db server. So, the service name defined earlier inpostgres-service.yaml file will enough to use as POSTGRES_HOST is enough to connect them.

```bash
# create deployment of backend microservices
kubectl apply -f rl-backend-deployment.yaml

# check the deployments and pods
kubectl get deployments
kubectl get pods
```

Now it's time to expose the the backend api to cluster or to the outside. If we just want to expose the api privately only on cluster then default ClusterIP is enough.

```bash
# create service(cluster; not outside of cluster) of backend microservices
kubectl apply -f rl-backend-service.yaml

# check the deployments and pods
kubectl get svc
```
**_but if we want to expose the api to the outside of the cluster we need to NodePort service._**

```bash 
# create Service (NodePort) of backend microservices
kubectl apply -f rl-backend-service-nodeport.yaml 
# Check the services
kubectl get svc
```
_Now we can call the backend api from the outside of the cluster like from browser or terminal._
```bash
# get the minikube ip
minikube ip
# now access the ip with the port assigend in NodePort yaml file as 'nodePort'
curl 192.168.49.2:30050
```

#### React (Frontend) microservices deployment
As of now we deployed postgres and backend succesfully in minikube cluster, now it's time to deploy frontend on the cluster. React app is binded with Nginx in docker container using multi-stage build. This helps me to handle the CORS issue which costs a lot hour to debug!!!

**One thing to notice, please make a change to the deployment file; API_HOST. I didn't find out why backend service name wasn't working(I will figure the solution later)**

```bash
# get the endpoints(back-node-port) of the k8s cluster
kubectl get endpoints
# now place the IP as API_HOST and port as API_PORT
# in rl-frontend-deployment.yaml file

# create deployment of frontend app 
kubectl apply -f rl-frontend-deployment.yaml
# check the pod and deployment status
kubectl get pods
kubectl get deployments
```

Now we are going to create a service for the frontend service. It will also use the NodePort Service to expose outside the cluster.

```bash
# create the service using rl-frontend-service.yaml
kubectl apply -f rl-frontend-service.yaml
# check the status
kubectl get svc

# access the frontend using browser
minikube ip
# use the port defined on rl-frontend-service.yaml
curl 192.168.49.2:30080
```

#### Ingress
will be added soon


#### Scaling

Kubernetes makes it easy to scale, adding additional Pods as necessary, when the traffic load becomes too much for a single Pod to handle. It can handle using kubectl utilty directly but it will not be saved on the yaml file or anyone can make the changes on yaml file and can use `apply -f`on the yaml file. It will be permanent.
```bash
# from command line; it will not be saved
kubectl scale deployment back-node-port --replicas=2

# change the 'replicas' field on the deployment file then apply the yaml
kubectl apply -f rl-backend-service.yaml

# now check the pods
kubectl get pods
# you will pods are creating or terminating based on ReplicaSet changes
```
