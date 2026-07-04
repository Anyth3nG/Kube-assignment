# Kube-assignment

## Prerequisites

before starting, the following tools are needed:

- [Docker](https://docs.docker.com/get-docker/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

## How to run the the app

### 1. Clone the repository
```
git clone https://github.com/Anyth3nG/Kube-assignment
cd Kube-assignment
```

### 2. start minikube
```
minikube start
```

### 3. deploy all manifests
```
kubectl apply -f manifests
kubectl apply -f manifests
```
run the command twice. the first run may fail the ConfigMap if the namespace isnt ready yet.

### 4. verify everything is running
```
kubectl get all -n webapp
```
all pods should show "Running" and services should be listed.

### 5. access the webapp
```
minikube service webapp-svc -n webapp --url
```
open the web browser and paste the given URL, you should see the nginx web app page.

using the ingress:
```
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts
```
the browser url for the app will be `http://webapp.local`

## Proof of deployment



## Written questions

### 1. Why is a StatefulSet used for PostgreSQL instead of a Deployment? What would break if you used a Deployment?

deployments and statefulsets are both sets of instructions on how to run pod, the difference between them is that using a statefulset will provide the pods with predictable names (postgres-0) opposed to deployments which set random names (webapp-79bb94856-9982b), with a statefulset pods start in order meaning postgres-0 will always start before postgres-1, and each pod gets its own dedicated PersistentVolumeClaim that follows if it restarts.
the reason we would use a statefulset for a database is because it provides a stable identity, stable storage and ordered operations which databases depend on.
if we were to use a deployment for our database it would be unreliable, we would risk data loss, data corruption and pods that cant find each other.

### 2. What is the difference between a ConfigMap and a Secret in Kubernetes? Why should credentials not be stored in a ConfigMap?

ConfigMap and Secret are both set to provide variable to be recalled throughout the manifests the difference is that ConfigMap is used to store non-sensitive variables like the name of the database, it is usually in plain text (it can be push to version control tools like GitHub), Secret on the other hand contains variables that are sensitive like passwords for example, they should never be hardcoded into any file, secrets are base64 encoded, anyone can decode it so its not a security measure but rather just a formatting requirement.
The security comes from: never pushing to tools like GitHub, common practice is to put the file into .gitignore (for the assignment Secret.yaml was purposfuly pushed to git), and to use secret managers like AWS Secrets Manager.

### 3. Your web app Deployment has 2 replicas. A customer reports that after running kubectl apply with a new image, they briefly saw errors. What Kubernetes settings could prevent this?

What happens when you update the pods with a new image is that in order for the image to be applied to the pods they need to be rebuilt, the reason why they briefly saw errors was because the pods where deleted and then built back up again without having a fallback.
The setting that could prevent that is a strategy type called RollingUpdate which replaces pods gradually, for it to work properly you have to set a maxSurge which means how many extra pods above the replica count can run during the update, and maxUnavailable which means how many pods can be unavailable during the update, a setting of 0 means no pods can go down until a new one is ready. 
Combined with a readinessProbe, kubernetes wont route traffic to the new pod until its confirmed healthy.

### 4. Explain what happens at the network level when the web app pod connects to postgres-svc:5432. How does Kubernetes resolve that hostname?

The pod will send a DNS query for postgres-svc to the cluster's internal DNS server, called CoreDNS which runs as a pod inside the cluster.
CoreDNS will resolve postgres-svc to its ClusterIP (we set that in postgres-service.yaml) and the webapp will send a request to its ClusterIP on port 5432.
The service will use the selector to find the postgres pod and will forward traffic towards that pod.


### 5. A pod is stuck in CrashLoopBackOff. Walk through the kubectl commands you would run to diagnose the issue.

CrashLoopBackOff is a way to try and avoid hammering a broken container, if a container keeps crashing each time kubernetes tries to restart it it will wait longer between each restart.
first you would want to find which pod is crashing with:
```
kubectl get pods -n webapp
```
a status called CrashLoopBackOff will show and how many times the pod has crashed.
next you would want to get more details aobut what happened to the pod:
```
kubectl describe pod <pod-name> -n webapp
```
this will show the events of the pod and there it will describe what went wrong, missing secrets for example.
check the logs:
```
kubectl logs <pod-name> -n webapp
```
this will show what the container printed before it crashed.
if the pod has already restarted and the logs are from the new instance, add --previous to see logs from the crashed container.