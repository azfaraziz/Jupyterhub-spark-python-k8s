# Setup of Jupyterhub + all-spark-notebook + Kubernetes (k8s) Locally

The goal of this "code" is to set up [Jupyterhub](https://github.com/jupyterhub/jupyterhub) within a local Kubenetes using [Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/) that will run the [all-spark-notebook]( https://github.com/jupyter/docker-stacks/tree/master/all-spark-notebook)

These steps were done on a MacOS running High Sierra. This assumes you have these installed:

- Docker
- VirtualBox
- At least 10GB and 4 Cpus

## Step 1: Setup Minikube

These steps followed the steps in [Getting Started Guides](https://kubernetes.io/docs/getting-started-guides/minikube/) from the Kubernetes

### Install

1. Install [Virtualbox](https://www.virtualbox.org/)
2. Install kubectl which is the K8s command-line tool

``` bash
brew install kubectl
```

3. Install minikube v0.27

``` bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.27.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

### Start up

This will execute minikube through virtualbox with 8GB of memory and 2 CPUs

```bash
minikube --memory 8192 --cpus 2 start
```

To ensure proper setup, spin up a simple "Hello World" container

```bash
kubectl run hello-minikube --image=k8s.gcr.io/echoserver:1.4 --port=8080

kubectl expose deployment hello-minikube --type=NodePort
```

### Dashboard
The K8s dashboard is a great tool to see what is going on with the "cluster". start it up, executed the following

```bash
minikube dashboard
```

This will open up the GUI in your default browser.

If this fails to start up, you may have to manually download and start it up. Execute the following:

```bash
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

kubectl proxy &
```

Then open a browser, and go this link: <http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/overview?namespace=default>

## Step 2: Setup Jupyterhub with all-spark-noteboook on Minikube

### Installation

This step follows the tutorial [Zero To Jupyterhub](http://zero-to-jupyterhub.readthedocs.io/en/latest/index.html)

0. The K8s is Minikube
1. Set up Helm. Execute the following lines

```bash
# Get cmd line
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

# Create service account
kubectl --namespace kube-system create serviceaccount tiller

# Enable RBAC
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

# Set up Helm on the cluster
helm init --service-account tiller

# Verify
helm version

# Secure Helm
kubectl --namespace=kube-system patch deployment tiller-deploy --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```

2. Set up JupyterHub

Use the existing ```config.yaml``` file. This contains the details for the all-spark-notebook by default. This can be changed in the next section.

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/

helm repo update

helm install jupyterhub/jupyterhub \
    --version=v0.6 \
    --name=poc-jupyterhub \
    --namespace=poc-jupyterhub \
    --set rbac.enabled=false \
    -f config.yaml
```

### Access JupyterHub

With minikube, there is no external proxy possible which prevents jupyterhub from being able get the proxy-public pod to fully run. It will be active, but the load balancer is not properly set up. To get the URL, you will need to access one of the endpoints. To get this, run 
```bash
minikube service list
```
Then take one of the of IPs from the public-proxy.


## Step 3: Test Pyspark

Log into Jupyterhub with any username and password. There is no security and each login user will align to a single container in k8s. Once the server is up, import the ```pyspark-test.ipynb``` notebook and run it. If successful, that means that pyspark is properly running.

# Additional Items

## Run Jupyter within Docker

To run Jupyter locally within docker, follow this documentation [jupyter-docker-stacks](http://jupyter-docker-stacks.readthedocs.io/en/latest/)

Esssentially, you need to execute:
```bash
docker run --rm -p 10000:8888 -v "$PWD":/home/jovyan/work jupyter/all-spark-notebook
```

This will spawn a single jupyter server locally

## Change the docker image in Jupyterhub

This follows the instructions found in [Zero-To-Jupyterhub: User Environment](http://zero-to-jupyterhub.readthedocs.io/en/latest/user-environment.html)

1. Modify the config.yaml with the specific image

```yaml
singleuser:
  image:
    name: jupyter/all-spark-notebook
    tag: 4ebeb1f2d154
```

2. Run a helm upgrade 

```bash 
helm upgrade poc-jupyterhub jupyterhub/jupyterhub --version=v0.6 --set rbac.enabled=false -f config.yaml
```


# Additional resources

[Jupyter and Jupyterhub on AWS EMR](https://aws.amazon.com/blogs/big-data/running-jupyter-notebook-and-jupyterhub-on-amazon-emr/)