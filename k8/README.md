
## 🚀 Running on Minikube

This project can be run locally on a Kubernetes cluster using [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download).

### Prerequisites
- At least **2 CPUs**
- At least **2 GB RAM**
- At least **20 GB free disk space**
- Internet connection
- A container or VM manager (Docker, Podman, VirtualBox, KVM, VMware, etc.)

### Install Minikube (Linux x86-64)
```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
````

### Start the Cluster

On Ubuntu 24.04, you can quickly install Docker and use it as the driver:

```bash
sudo apt update
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
````


```bash
minikube start --driver=docker
```

### Interact with the Cluster

If you already have `kubectl` installed:

```bash
kubectl get po -A
```

Or use the bundled version:

```bash
minikube kubectl -- get po -A
```

Optional alias for convenience:

```bash
alias kubectl="minikube kubectl --"
```

### Deploy a Sample App

```bash
kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```

Access the service:

```bash
minikube service hello-minikube
```

Or via port-forward:

```bash
kubectl port-forward service/hello-minikube 7080:8080
```

Now the app should be available at [http://localhost:7080](http://localhost:7080).





