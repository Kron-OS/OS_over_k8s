# OS_over_k8s

## 0 - Installation

### Packages

In order to perform the folwin tests, we have to setup the environnement. To use OpenSearch, and mainly k8s, we need a few packages :
- **kubectl** : The k8s managing tool;
  ```bash
  sudo pacman -S kubectl
  ```

- **minikube** : To run a single node Kubernetes cluster locally;
  ```bash
  sudo pacman -S minikube
  docker context use default
  ```
- **container runtime** : Docker or container-d.
  ```bash
  sudo pacman -S docker
  sudo systemctl start docker.service  # start now
  sudo systemctl enable docker.service # start on boot
  sudo usermod -aG docker $USER # to not need to run it as root
  ```
- **go** : To install OpenSearch Kubernetes Operator
  ```bash
  sudo pacman -S go
  ```

### Verification

Firstly, set up a local Kubernetes cluster : `minikube start --force`:
```text
😄  minikube v1.37.0 on Arch 
❗  minikube skips various validations when --force is supplied; this may lead to unexpected behavior
✨  Automatically selected the docker driver. Other choices: none, ssh
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.48 ...
💾  Downloading Kubernetes v1.34.0 preload ...
    > index.docker.io/kicbase/sta...:  488.51 MiB / 488.52 MiB  100.00% 4.97 Mi
    > preloaded-images-k8s-v18-v1...:  337.07 MiB / 337.07 MiB  100.00% 3.19 Mi
❗  minikube was unable to download gcr.io/k8s-minikube/kicbase:v0.0.48, but successfully downloaded docker.io/kicbase/stable:v0.0.48@sha256:7171c97a51623558720f8e5878e4f4637da093e2f2ed589997bedc6c1549b2b1 as a fallback image
🔥  Creating docker container (CPUs=2, Memory=4900MB) ...
🐳  Preparing Kubernetes v1.34.0 on Docker 28.4.0 ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Then, verify the Kubernetes cluster with `minikube kubectl -- get pods -A`:
```text
    > kubectl.sha256:  64 B / 64 B [-------------------------] 100.00% ? p/s 0s
    > kubectl:  57.75 MiB / 57.75 MiB [-------------] 100.00% 7.51 MiB p/s 7.9s
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-66bc5c9577-fz5d8           1/1     Running   0             2m3s
kube-system   etcd-minikube                      1/1     Running   0             2m10s
kube-system   kube-apiserver-minikube            1/1     Running   0             2m8s
kube-system   kube-controller-manager-minikube   1/1     Running   0             2m10s
kube-system   kube-proxy-bpxlj                   1/1     Running   0             2m3s
kube-system   kube-scheduler-minikube            1/1     Running   0             2m10s
kube-system   storage-provisioner                1/1     Running   1 (93s ago)   2m7s
```

Finally, we will enable th eminikube dashboard with `minikube dashboard`:
```text
🔌  Enabling dashboard ...
    ▪ Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
    ▪ Using image docker.io/kubernetesui/dashboard:v2.7.0
💡  Some dashboard features require the metrics-server addon. To enable all features please run:

        minikube addons enable metrics-server

🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:45793/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

You can get the address again with `minikube dashboard --url`. Your dashboard should open on your default webbrowser automatically.


## 1 - Minikube & OpenSearch Operator setup

According to the OpenSearch documentation, and as in a classic docker installation, we have to adjust the `max_map_count`, so first run the following command:
- ```bash
  minikube ssh 'sudo sysctl -w vm.max_map_count=262144'
  ```

### OpenSearch Kubernetes Operator

This open-source kubernetes operator helps automate the deployment.


1. Run minikube and enable addons:
  ```bash
  minikube start
  minikube status

  minikube addons enable storage-provisioner
  minikube addons enable default-storageclass
  ```
2. Add the helm repo:
  ```bash
  helm repo add opensearch-operator https://opensearch-project.github.io/opensearch-k8s-operator/
  helm repo update
  helm repo list
  ```
3. Create a namespace:
  ```bash
  kubectl create namespace opensearch-operator-system
  kubectl get namespaces
  ```

3. Install OpenSearch Kubernetes Operator:
  ```bash
  helm install opensearch-operator opensearch-operator/opensearch-operator --namespace opensearch-operator-system
  kubectl get pods -n opensearch-operator-system --watch
  kubectl logs -n opensearch-operator-system -l control-plane=controller-manager --tail=50
  kubectl get crd | grep opensearch

  ```


## 2 - OpenSearch Cluster first Deployment

Now, we are ready to make our first OpenSearch cluster Deployment. From the example section of the documentatio, we can deploy a single OpenSearch pod with following procedure.

### Cluster configuration

From the documentation, we can find this simple configuration file:
```yaml
apiVersion: opensearch.opster.io/v1
kind: OpenSearchCluster
metadata:
  name: my-first-cluster                # used to identify resources in Kubernetes
  namespace: default                    # where the cluster will be deployed
spec:
  # Security configuration for the cluster
  security:                             
    config:                             # used to specify security config (TLS certificates, roles, ...)
    tls:
       http:
         generate: true                 # generate TLS certificates for HTTP
       transport:
         generate: true                 # generate TLS certificates for transport
         perNode: true                  # unique certificate for each node
  # General settings for whole cluster
  general:
    httpPort: 9200                      # default OpenSearch REST API port
    serviceName: my-first-cluster       # name of the Kubernetes service
    version: 2.14.0
    pluginsList: ["repository-s3"]      # OpenSearch plugins to install
    drainDataNodes: true
  # Dashboards configuration
  dashboards:
    tls:
      enable: true                      # enable tls
      generate: true
    version: 2.14.0
    enable: true
    replicas: 1                         # cound of pods ("OpenSearch Node")
    resources:                          # resource for Dashboards pod
      requests:
         memory: "512Mi"
         cpu: "200m"
      limits:
         memory: "512Mi"
         cpu: "200m"
  nodePools:                            # core node pool
    - component: masters                # name of the group
      replicas: 3                       # count of replicas -> change to 1 if your test lab has ressources constraints
      resources:                        # ressources per Pod (per OpenSearch node)
         requests:
            memory: "4Gi"
            cpu: "1000m"
         limits:
            memory: "4Gi"
            cpu: "1000m"
      roles:                            # roles assigned to "OpenSearch nodes"
        - "data"
        - "cluster_manager"
      persistence:
         emptyDir: {}                   # ephemeral storage

```

> Note that *"node"* in OpenSearch context refers to a Kubernetes *"pod"*, and not a Kubernetes "node".

### Cluster Deployment

To deploy the cluster, we now have to run the following command:
```bash
kubectl apply -f opensearch-cluster.yaml
```

Then, you can check your deployment with `minikube kubectl get svc`:
```text
NAME                                                      READY   STATUS    RESTARTS       AGE
my-first-cluster-bootstrap-0                              1/1     Running   0              8m39s
my-first-cluster-dashboards-6469b98fd7-mk9s7              0/1     Running   2 (107s ago)   8m28s
my-first-cluster-masters-0                                0/1     Running   0              8m39s
my-first-cluster-securityconfig-update-2tq2w              1/1     Running   0              8m40s
opensearch-operator-controller-manager-6d6c6b88c8-x52dw   2/2     Running   2 (27m ago)    30m
```
The master can take some time to start, during this time the dashboard will not be able to start.

To connect to your cluster, use the `port-forward` command accordingly to the last command:
```bash
kubectl port-forward svc/my-first-cluster-dashboards 5601
```