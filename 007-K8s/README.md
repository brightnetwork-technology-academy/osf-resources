# Kubernetes Lab

## Prerequisite

1. Install [Docker](https://docs.docker.com/get-docker/)
2. Install [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) the Kubernetes Commandline tool
3. Install [Minikube](https://minikube.sigs.k8s.io/docs/start/) to allow use to create a cluster locally.

## Activity

### Start Minikube

The first thing we need to do is start up [Minikube](https://minikube.sigs.k8s.io). This will give us a local Kubernetes 
cluster to deploy application to. The minikube cluster runs inside a docker container.

```shell
minikube start
```


Once minikube is up and running you can load the Kubernetes dashboard. This command will start the dashboard and open it 
in your browser. (**Note** this command blocks the terminal so all following commands will need to run in a new terminal
window).

```shell
minikube dashboard
```

Once the dashboard has loaded check on the Pods page. There should not be any pods running yet.

### Imperative Commands

First we are going to deploy and scale an application using imperative commands. We are going to tell Kubernetes exactly 
what we want it to do. 

Using `kubectl` command to tell Kubernetes exactly what containers to spin up, how many replicas to run and how to 
expose the Service. This is great for a demonstration but trying to maintain a large number of high demand microservices 
doing this not maintainable.

#### Create a Deployment

We now want to deployment within the Kubernetes cluster. Today we are going to deploy the Docker container we created 
during the [container lab](../006-containers/containerDemo/README.md) and deploy it to the Minikube Kubernetes cluster.

We can do this using the `kubectl`. This is a local commandline tool that interacts with the Kubernetes API running 
inside our cluster. The command below will create a deployment within our cluster called `demo-node`. It will pull the
container from your Docker Hub (Container Registry). 

```shell
kubectl create deployment demo-node --image={docker_hub_username}/demo-app:1.0.0
```

**Note** If you have not completed the[container lab](../006-containers/containerDemo/README.md) before starting this 
one you can use an instance I created on my account `braddle/demo-app:1.0.0`

##### Check the deployment

We can now check the Deployment page on the dashboard. You should now see a Deployment listed for the demo-app. It 
should look something like this. This page show that there is a deployment call `demo-node` using the image 
`braddle/demo-app:1.0.0` and it is running healthy on a single Pod.

![The deployments page of the Kubernetes dashboard](docs/deployments.png)

You can also check the deployments using the `kubectl` command line tool.

```shell
kubectl get deployments
```

The output should look like this:

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
demo-node   1/1     1            1           2m55s
```

##### Check Pods

We can also check the Pods page on the dashboard. You should now see a Pod listed for the demo-app. 

![The Pods page of the Kubernetes dashboard](docs/pods.png)

You can also check the running Pods using the kubectl

```shell
kubectl get pods
```

The output should look like this:

```
NAME                         READY   STATUS    RESTARTS   AGE
demo-node-6b584b4f6c-nfjsp   1/1     Running   0          7m22s
```

#### Accessing The App

At the moment the app is only accessible by in IP within the Kubernetes cluster. We want the app to be available to the 
outside world. To do this we need to expose the Pod as a Kubernetes Service. We can do this with the following command

```shell
kubectl expose deployment demo-node --type=LoadBalancer --port=8080
```

The `--type=LoadBalancer` flag indicates that you want to expose your Service outside the cluster.

##### check Service

The service should now be listed on the services page of the dashboard

![The Services page of the Kubernetes dashboard](docs/services.png)

You can also get a list of the Service using the following kubectl command

```shell
kubectl get services
```

The output should look like the following:

```
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
demo-node    LoadBalancer   10.101.96.95   <pending>     8080:31647/TCP   4m48s
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          134m
```

On cloud providers that support load balancers, an external IP address would be provisioned to access the Service. On 
minikube, the LoadBalancer type makes the Service accessible through the minikube service command.

```shell
minikube service demo-node
```

This command will load the app in the web browser, and you should see `Hello Docker World`.

#### Scaling

At the moment we have a single Pod running for our application. But what if we need to handle more load. Kubernetes 
allows us to scale the number of Pods we want running for a deployment. We can set the number of Pods for the desired 
state using the `scale` command of the `kubectl` command line tool. Use the command below to set the number of Replicas 
or Pods to 3.

```shell
kubectl scale deployment demo-node --replicas=3
```

If you now check the Pods page on the dashboard. You should now see 3 Pods listed for the `demo-node` app. Kubernetes has 
reacted to the change in the desired state and increased the number of Pods running for the `demo-node` app 

![The Pods page of the Kubernetes dashboard](docs/three-pods.png)

You can also check the running Pods using the kubectl

```shell
kubectl get pods
```

The output should look like this:

```
NAME                         READY   STATUS    RESTARTS   AGE
demo-node-6b584b4f6c-hptqw   1/1     Running   0          2m45s
demo-node-6b584b4f6c-kx5jp   1/1     Running   0          2m45s
demo-node-6b584b4f6c-zn9x7   1/1     Running   0          5m54s
```

The deployment should now be aware of the three pods and routing traffic to all of them. Check the Deployments page on 
the dashboard. You should now see 3 Pods listed for the demo-node.

![The Pods page of the Kubernetes dashboard](docs/deployments-scaled.png)

you can also check this via kubectl, using the following command:

```shell
kubectl get deployments
```

The output should look like this:

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
demo-node   3/3     3            3           14m
```

#### Clean up

Now you can clean up the resources you created in your cluster:

```shell
kubectl delete service demo-node
kubectl delete deployment demo-node
```

### Imperative Configuration

So far we have configured using declarative commands to manually configure what we want Kubernetes to do. We need to know
what the current start of the cluster is to make changes to it.

Alternatively we can use YML files to provide Kubernetes with our Desired State. Kubernetes will make any changes to the
cluster when we `apply` these files to the cluster.

These files can be stored within your project repository and run by your pipelines to automate deployments and service 
configuration.

#### Create a deployment

Add the following YML configuration to `deployment/demo-node.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-node
spec:
  replicas: 1
  selector:
    matchLabels:
      app: "demo-node"
  template:
    metadata:
      labels:
        app: demo-node
        track: stable
        version: 1.0.0
    spec:
      containers:
        - name: demo-node
          image: "braddle/demo-app:1.0.0"
          ports:
            - containerPort: 8080
```

Now you can use kubectl to apply the configuration within the file using the apply command

```shell
kubectl apply -f deployments/demo-node.yml
```

Expected output `deployment.apps/demo-node created`

Re-running the apply command at this point should not change anything in the cluster. Expected output 
`deployment.apps/demo-node unchanged`

Update the number of replicas to 3 and re-run the apply command. Expected output `deployment.apps/demo-node configured`

#### Create a Service

`services/demo-node.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: "demo"
spec:
  selector:
    app: "demo-node"
  ports:
    - port: 8080
      targetPort: 8080
      protocol: "TCP"
```

```shell
kubectl apply -f services/demo-node.yml
```

To access the running service you need to run the following command

```shell
minikube service demo
```

Expected output `Hello Docker World`

once you have seen the service running correctly you can stop minikube exposing the service using `ctrl c`

#### Service Discovery

When we are working with microservice we need the APIs we deploy to communicate with each other. When we are using 
Kubernetes we will not now the IP address assigned our the pods and services we create. Kubernetes creates a hostname for 
our services from the name we provide in the YAML. We are going to use this to spin up a new application that will convert
the response from our demo app to upper case.

##### Deploying Upper Case Application

Add the following YAML to `deployments/upper.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upper-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upper-app
  template:
    metadata:
      labels:
        app: upper-app
        track: stable
        version: 1.0.0
    spec:
      containers:
        - name: upper-app
          image: braddle/upper-case-app:1.0.0
          env:
            - name: TEXT_HOST
              value: http://demo:8080
          ports:
            - containerPort: 8080
```

This configuration is very similar to what we have created in `deployments/demo-node.yml`, we are using different names, 
label and image. The major different is we have added an `env` section, this enables us to provide environment variables
to the container when Kubernetes brings it up.

We have configured an environment variable called `TEXT_HOST`. The application that we are deploying uses they 
environment variable to make call to the provided hostname. We know that the hostname of the demo service and application
is `http://demo:8080` because of the configuration in `services/demo-node.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  selector:
    app: demo-node
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
```

Kubernetes will provide a hostname based on the `name` of service in this case it is `demo` and we now that the service 
is exposed on `port` `8080`

We now need to apply the configuration of the `upper-app` to a Kubernetes cluster, by running the following command

```shell
kubectl apply -f deployments/upper.yml
```

Expected output `deployment.apps/upper-app created`

We can now check our running deployments 

```shell
kubectl get deployments
```

and should see something like this:

```
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
demo-node   1/1     1            1           4h36m
upper-app   1/1     1            1           115m
```

Showing that both our demo and upper application are running. We now need to expose the upper application via a service, 
so we can access it. 

##### Creating Upper Caser Service

Add the follow YAML configuration to `services/upper.yml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: upper
spec:
  selector:
    app: upper-app
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
```

Other than names and selectors this is the same setup as `services/demo-node.yml`. We can apply this configuration to 
our cluster using the following command

```shell
kubectl apply -f services/upper.yml
```

Expected output `service/upper created`

We should now see two services available when we check them with the follow command

```shell
kubectl get services
```

you should see the following output

```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
demo         ClusterIP   10.106.162.237   <none>        8080/TCP   4h44m
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    109d
upper        ClusterIP   10.109.92.228    <none>        8080/TCP   37s
```

To access the running service you need to run the following command

```shell
minikube service upper
```

expected output `HELLO DOCKER WORLD`

once you have seen the service running correctly you can stop minikube exposing the service using `ctrl c`


### Clean up

Now you can clean up the resources you created in your cluster:

```shell
kubectl delete service demo-node
kubectl delete deployment demo-node
```

Optionally, stop the Minikube virtual machine (VM):

```shell
minikube stop
```

Optionally, delete the Minikube VM:

```shell
minikube delete
```