# Triton Inference Server and NGINX+ Ingress Controller
This repository provide a working example of how NGINX Plus Ingress Controller can provide secure external access -as well as load balancing- to an [NVIDIA Triton Inference Server cluster](https://www.nvidia.com/en-us/ai-data-science/products/triton-inference-server/).  The repository is forked from the NVIDIA [Triton Inference Server repo](https://github.com/triton-inference-server/server) and includes a Helm chart along with instructions for installing NVIDIA Triton Inference Server and NGINX+ Ingress Controller in an on premises or cloud-based Kubernetes cluster.  

<img src="images/archdiag.png" alt="Flowers" >

This guide assumes you already have Helm installed (see [Installing Helm](#installing-helm) for instructions).  For more information on Helm and Helm charts, visit the [Helm documentation](https://helm.sh/docs/).  Please note the following requirements:

* The Triton server requires access to a models repository via and external NFS server.  If you already have an NFS server to host the model repository, you may use that with this Helm chart. If you do not, an NFS server (k8s manifest) is included which may be deployed and loaded with the included model repository.

* To deploy Prometheus and Grafana to collect and display Triton metrics, your cluster must contain sufficient CPU resources to support these services.

* Triton-server works with both CPUs and GPUs.  To use GPUs for inferencing, your cluster must be configured to contain the desired number of GPU nodes, with support for the NVIDIA driver and CUDA version required by the version of the inference server you are using.

* To enable autoscaling, your cluster's kube-apiserver must have the [aggregation layer
enabled](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/).
This will allow the horizontal pod autoscaler to read custom metrics from the prometheus adapter.


# Deployment Instructions

First, clone this repository to a local machine. 
```
git clone https://github.com/f5devcentral/triton-server-ngxin-plus-ingress.git
cd Triton-Server-NGINX-Plus-Ingress-Controller
```
### Create a new NGINX private registry secret
You will need to use your NGINX Ingress Controller subscription [JWT token](https://docs.nginx.com/nginx-ingress-controller/installation/nic-images/using-the-jwt-token-docker-secret/) to get the NGINX Plus Ingress Controller image. Create a secret that will be referenced by the NGINX Ingress Controller deployment allowing for automatic image access and pulling.

```
kubectl create secret docker-registry regcred --docker-server=private-registry.nginx.com --docker-username=<JWT Token> --docker-password=none [-n nginx-ingress]
```
### Create a new TLS secret named tls-secret
```
kubectl create secret tls tls-secret --cert=<path/to/tls.cert> --key=<path/to/tls.key>
```
### Model Repository
If you already have a model repository, you may use that with this Helm chart. If you do not have a model repository, you can make use of the local repo copy located in the at **_/model_repository_** to create an example
model repository:

Triton Server needs a repository of models that it will make available for inferencing. For this example, we are using an existing NFS server and placing our model files there.  Copy the local _model_repository_ directory onto your NFS server.  Then, add the url or IP address of your NFS server and the server path of your
model repository to `values.yaml`.  

If you do not have an NFS currently available, you can deploy a NFS server (k8s manifest) which may be deployed and loaded with the included model repository.
```
cd Triton-Server-NGINX-Plus-Ingress-Controller
kubectl apply -f nfs-server.yaml
```
Connect to the NFS server pod, clone the repo onto the container and move the model_repository directory.
``` _
kubectl exec <nfs-server POD name> --stdin --tty -- /bin/bash
cd /
git clone https://github.com/f5devcentral/Triton-Server-NGINX-Plus-Ingress-Controller.git
mv /Triton-Server-NGINX-Plus-Ingress-Controller/model_repository /exports
exit
```

### Deploy Prometheus and Grafana

The inference server metrics are collected by Prometheus and viewable
through Grafana. The inference server Helm chart assumes that Prometheus
and Grafana are available so this step must be followed even if you
do not want to use Grafana.

Use the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart to install these components. The
*serviceMonitorSelectorNilUsesHelmValues* flag is needed so that
Prometheus can find the inference server metrics in the *example*
release deployed in a later section.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```

### Enable Autoscaling
To enable autoscaling, ensure that autoscaling tag in `values.yaml`is set to `true`.
This will do two things:

1. Deploy a Horizontal Pod Autoscaler that will scale replicas of the triton-inference-server
based on the information included in `values.yaml`.

2. Install the [prometheus-adapter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-adapter) helm chart, allowing the Horizontal Pod Autoscaler to scale
based on custom metrics from prometheus.

The included configuration will scale Triton pods based on the average queue time,
as described in [this blog post](https://developer.nvidia.com/blog/deploying-nvidia-triton-at-scale-with-mig-and-kubernetes/#:~:text=Query%20NVIDIA%20Triton%20metrics%20using%20Prometheus). To customize this,
you may replace or add to the list of custom rules in `values.yaml`. If you change
the custom metric, be sure to change the values in autoscaling.metrics.

If autoscaling is disabled, the number of Triton server pods is set to the minReplicas
variable in `values.yaml`.

#### Updating the `values.yaml` file
Before deploying the Inference server and NGINX+ Ingress Controller update the `values.yaml` specifying your modelRepositoryServer IP and path (*default is '/'*), service FQDNs, and autoscaling preference, (see below).

<img src="images/img1.png" alt="Flowers" width="60%">

### Deploy the Inference Server
Deploy the inference server and NGINX Plus Ingress Controller using the default configuration with the following commands. Here, and in the following commands we use the name _mytest_ for our chart. This name will be added to the beginning of all resources created during the helm installation.  With the `values.yaml` file updated, you are ready to deploy the Helm Chart.
```
cd <directory containing Chart.yaml>
helm install mytest .
```
Use kubectl to see status and wait until the inference server pods are running.

```
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
mytest-triton-inference-server-5f74b55885-n6lt7   1/1     Running   0          2m21s
```

### Using Triton Inference Server

Now that the inference server is running you can send HTTP or GRPC
requests to it to perform inferencing.

```
$ kubectl get svc
NAME                                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
kubernetes                               ClusterIP      10.0.0.1       <none>         443/TCP                      10d
mytest-nginx-ingress-controller          LoadBalancer   10.0.179.216   20.252.89.78   80:31336/TCP,443:31862/TCP   39m
mytest-triton-inference-server           ClusterIP      10.0.231.100   <none>         8000/TCP,8001/TCP,8002/TCP   39m
mytest-triton-inference-server-metrics   ClusterIP      10.0.21.98     <none>         8080/TCP                     39m
nfs-service                              ClusterIP      10.0.194.248   <none>         2049/TCP,20048/TCP,111/TCP   123m...

```
Enable port forwarding from the the Grafana service so you can access it from
your local browser.

```
kubectl port-forward service/example-metrics-grafana 8088:80
```
Now you should be able to navigate in your browser to 127.0.0.1:8088
and see the Grafana login page. Use username=admin and
password=prom-operator to log in.

An example Grafana dashboard is available -*dashboard.json*- in the repo. Use the
import function in Grafana to import and view this dashboard, (see below).

<img src="images/img3.png" alt="Flowers">

Enable port forwarding from the /NGINX Ingress Controller pod to view service access metrics.
```
kubectl port-forward *<NGINX ingress controller pod name>* 8080:8080
```
The NGINX+ dashboard can be reached at 127.0.0.1/dashboard.html, (see below).

<img src="images/img2.png" alt="Flowers">

### Run a couple sample queries
If the included sample models are loaded, you can test connectivity to the Triton Inference server(s) by running the included  *simple_http_infer_client.py* python script.  After running the script a few times, you can return to the NGINX+ and Grafana dashboards to monitor.
```
python3 simple_http_infer_client.py -u *<triton server http URL>> --ssl --insecure
```
_**Example:** python3 simple_http_infer_client.py -u triton-http.f5demo.net --ssl --insecure_

## Cleanup

After you have finished using the inference server, you should use Helm to
delete the deployment.  If you deployed the NFS server use *Kubectl* to remove.

```
helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
mytest  default         1               2024-04-15 19:01:31.772857 -0700 PDT    deployed        triton-inference-server-1.0.0   1.0        

helm uninstall mytest

kubectl delete -f nfs-server.yaml
kubectl delete secret tls-secret
kubectl delete secret regcred
```