# Setup Nginx Ingress Controller for cross namespace ingress resources

## Pre-requisites
* As part of setting this up, I tested this locally using minikube and on Google Kubernetes Engine (GKE)

## Setup
### Nginx Ingress Controller
* We will be installing Nginx Ingress controller as a helm chart from [kubernetes/charts/stable/nginx-ingress](https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress)
* Since recent Kubernetes versions have RBAC enabled by default, we will need to install it with RBAC in mind.
  * You can check if RBAC is enabled in your Kubernetes cluster by running `kubectl api-versions | grep rbac`
  * _**Note**: Steps to disable RBAC are outside of the scope of this walkthrough_
  * Following output indicates RBAC is enabled
    ```text
    rbac.authorization.k8s.io/v1
    rbac.authorization.k8s.io/v1beta1
    ```

### Nginx Ingress Controller Installation
#### Minikube
* Minikube comes with out of the box support for nginx ingress controller
* To enable default nginx ingress controller, run `minikube addons enable ingress`
* Check the status of nginx ingress by running `minikube addons list | grep ingress`

#### GKE (Google Kubernetes Engine) on Google Cloud Platform
##### Update `kubectl` context
* Check current context by running `kubectl config current-context`
* If you do not see desired kubernetes cluster in output of above command, you need to set appropriate context.
* To set a GKE cluster in context:
  * Get cluster/zone name: `gcloud container clusters list`
  * Update context: `gcloud container clusters get-credentials <cluster_name> --zone <zone_name>`
  * Replace `<cluster_name>` with actual cluster name and `<zone_name>` with actual zone name
  * Example: `gcloud container clusters get-credentials cluster-20062018-121027 --zone asia-south1-a`
  * Expected Output:
    ```text
    Fetching cluster endpoint and auth data.
    kubeconfig entry generated for cluster-20062018-121027.
    ```

##### Install Helm and Tiller
###### Helm
* Run GCloud Shell from Google Cloud platform Console
* Install helm
```bash
curl -o get_helm.sh https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get
chmod +x get_helm.sh
./get_helm.sh
```
* Expected Output:
```text
Downloading https://kubernetes-helm.storage.googleapis.com/helm-v2.9.1-linux-amd64.tar.gz
Preparing to install into /usr/local/bin
helm installed into /usr/local/bin/helm
Run 'helm init' to configure helm.
```

###### Tiller
* Create a service account for `tiller` in `kube-system` namespace.
```
kubectl create serviceaccount --namespace kube-system tiller
```
* Create a ClusterRoleBinding for `tiller` assigning it the role of `cluster-admin` and linking it with the service
account we created for tiller in `kube-system` namespace
```
kubectl create clusterrolebinding tiller-cluster-role --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```
* Create deployment for tiller by the way of initiating helm
```
helm init --service-account tiller --upgrade
```

##### Install Nginx Ingress Controller on GKE
* Since we checked previously that RBAC is enabled on kubernetes cluster, lets install nginx-ingress helm chart with RBAC rules
```
helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true
```
* Somewhere in output under heading `v1/Service` you should see:
```
==> v1/Service
NAME                           TYPE          CLUSTER-IP    EXTERNAL-IP  PORT(S)                     AGE
nginx-ingress-controller       LoadBalancer  10.35.246.69  <pending>    80:30505/TCP,443:31687/TCP  0s
nginx-ingress-default-backend  ClusterIP     10.35.241.12  <none>       80/TCP                      0s
```
* When the `<pending>` under `EXTERNAL-IP` column changes to an actual IP address, your GCP load balancer is ready.
  * Use `watch kubectl get svc` to constantly keep checking if the LoadBalancer is up (i.e. IP address is allocated to `nginx-ingress-controller` service)

Now go ahead and deploy your ingress resources in respective namespaces. Checkout file named `cross-ns-resources.yaml` in this repository.
