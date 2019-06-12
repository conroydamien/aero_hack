###Purpose
The purpose of this document is to demonstrate spinning up two containers, a `nginx` container and a `postgres` container, in four different ways.
1) Using docker
2) Using docker-compose
3) Using Kubernetes
4) Using Helm

The idea is to get started quickly and focus on the superficial similarities rather than the differences between these four approaches.

### Starting at the Start
If you don't know what [Docker](https://www.docker.com/) is go find out.

#### Starting nginx with Docker
`docker run -p 8000:80 nginx`
nginx starts in the container and the container's port 80 is mapped to port 8000 on the host.

Visit `http://localhost:8000/` in a browser.

#### Starting nginx with Docker Compose
Create a file called `docker-compose.yaml` with the following contents:
```
version: '3'
services:
  nginx:
    image: nginx
    ports:
      - "8000:80"
```
From the command line in the directory where the file is type `docker-compose up`.

Once you've checked out that it all worked `docker-compose down` will stop the container.

The nice thing about docker-compose is that one command takes care of everything. You can add a Postgres container to the file and the same command will bring both containers up.

```
version: '3'
services:
  nginx:
    image: nginx
    ports:
      - "8000:80"
  postgres:
    image: postgres
    ports:
      - "15432:5432"
```

Using the old (v 1.x) docker-compose containers can be named using the `link` attribute. This is deprecated in more recent versions but it can be used to demonstrate a very simple network with v 1.x. For example:

```
version: '3'
services:
  nginx:
    image: nginx
    ports:
      - "8000:80"
    links:
      - postgres
  postgres:
    image: postgres
    ports:
      - "15432:5432"
```
When the containers have been brought up we can connect to the nginx container using
`docker exec -it {containerId} /bin/bash`
and when connected (as root in the container) install `dnsutils`:
**`root@1265c607736b:/#`**`apt update; apt -y install dnsutils`
then check that the name has been mapped successfully:
**`root@1265c607736b:/#`**`host postgres` should return something like:
`postgres has address 172.20.0.2`

That's enough of that. There are better ways of networking with docker-compose. However the convenience of docker-compose as opposed to multiple docker command line invocations is obvious.

#### Kubernetes
[Kubneretes](https://kubernetes.io/) is similar to Docker/docker-compose in a number of ways. Let's consider those before looking at the differences.

1) Just like docker-compose requires Docker to be installed, Kubernetes requires a Kubernetes cluster to be installed. The most convenient to install locally is **[Minikube](https://kubernetes.io/docs/setup/minikube/)**. If you have not already installed Minikube install it now.

2) Bring up Minikube locally with `minikube start`. You'll need a VM installed.

3) Now we can deploy a Docker container to the Kubernetes cluster much like we did to Docker. Use the following commands:
  `$ kubectl run nginx --image=nginx`
  `$ kubectl run postgres --image=postgres`
  and to check that they are up and running:

     `$ kubectl get pods`

     ```
     NAME                        READY   STATUS    RESTARTS   AGE
     nginx-7cdbd8cdc9-mvvbt      1/1     Running   0          3m6s
     postgres-5995765dd4-wg5qk   1/1     Running   0          2m41s
     ```

4) Connect to the nginx pod as before, but this time we use:
    `$ kubectl exec -ti {pod name} /bin/bash`
5) Install the utils:
**`root@nginx-7cdbd8cdc9-mvvbt:/#`**`apt update; apt -y install dnsutils`
and check that the postgres pod can be seen:
**`root@nginx-7cdbd8cdc9-mvvbt:/#`**` host postgres`
`postgres has address 172.21.0.3`
6) Remove the pods:
`kubectl delete pod --all`
7) `$ kubectl get pods` shows that the deleted pods have been replaced.
8) `$ kubectl delete deployment --all` deletes the deployments.
9) `$ kubectl get pods` shows that the deleted pods have not been replaced.
####Kubernetes with a *docker-compose-esque* file
To get a similar effect to that achieved with docker-compose create a file called `app.yaml` with the following contents:

```
apiVersion: v1
kind: Pod
metadata:
   name: app
spec:
 containers:
   - name: nginx
     image: nginx
     ports:
     - containerPort: 80
   - name: postgres
     image: postgres
     ports:
     - containerPort: 5432
```

To start the containers in a pod run the following command:
`kubectl create -f app.yaml`
Check the results with `kubectl get pods` as before:
```
$ kubectl get pods
NAME   READY   STATUS              RESTARTS   AGE
app    0/2     ContainerCreating   0          21s
```
and
`kubectl get pods app -o jsonpath='{.spec.containers[*].name}'`
This will list our two containers `nginx` and `postgres` in the pod `app`.

We can connect to one of the containers (the nginx container in the app pod) as follows:
`kubectl exec -it app -c nginx /bin/bash`

We can install dnsutils and do the same checks as earlier.

So far there doesn't seem to be much difference between launching the containers with docker-compose and Kubernetes.

We can delete the `app` pod as follows:

`kubectl delete pod app`

####Helm (let's not worry about what it is yet...)
Helm requires us to take a different approach. We need a to create a directory for the project using `helm create`.
`helm create example` gives us:
```
example
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
```
We can dispense with some of the default files and copy in our `app.yaml` from the previous section as follows:

```
example
├── charts
├── Chart.yaml
├── templates
│   ├── app.yaml
│   └── _helpers.tpl
└── values.yaml
```

From the directory containing the `example` directory execute the following:

`helm install example/ --generate-name`

helm should report the following deployment message.

```
NAME: example-1559767568
LAST DEPLOYED: 2019-06-05 21:46:08.041105003 +0100 IST m=+0.102345500
NAMESPACE: default
STATUS: deployed
```

We can delete the deployment with helm:

`$ helm list --all`
```
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART        
example-1559767568      default         1               2019-06-05 21:46:08.041105003 +0100 IST deployed        example-0.1.0
```
`$ helm delete example-1559767568`
```
release "example-1559767568" uninstalled
```
`$ kubectl get pods`
```
No resources found.
```
####Helm with values extracted
We can move some values from the `app.yaml` file into the `values.yaml` file as follows:

`values.yaml`:

```
# Default values for example.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

nginx:
  image_name: nginx
  port: 80

postgres:
  image_name: postgres
  port: 5432
```

`app.yaml`

```
apiVersion: v1
kind: Pod
metadata:
   name: app
spec:
 containers:
   - name: nginx
     image: {{ .Values.nginx.image_name }}
     ports:
     - containerPort: {{ .Values.nginx.port }}
   - name: postgres
     image: {{ .Values.postgres.image_name }}
     ports:
     - containerPort: {{ .Values.postgres.port }}
```
