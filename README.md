# Triton Inference Server and NGINX+ Ingress Controller
This repository provide a working example of how NGINX Plus Ingress Controller can provide secure external access -as well as load balancing- to an [NVIDIA Triton Inference Server cluster](https://www.nvidia.com/en-us/ai-data-science/products/triton-inference-server/).  The repository is forked from the NVIDIA [Triton Inference Server repo](https://github.com/triton-inference-server/server) and includes a Helm chart along with instructions for installing NVIDIA Triton Inference Server and NGINX+ Ingress Controller in an on-premises or cloud-based Kubernetes cluster.  

This guide assumes you already have Helm installed (see [Installing Helm](#installing-helm) for instructions). Note the following requirements:

* To deploy Prometheus and Grafana to collect and display Triton metrics, your cluster must contain sufficient CPU resources to support these services.

* To use GPUs for inferencing, your cluster must be configured to contain the desired number of GPU nodes, with
support for the NVIDIA driver and CUDA version required by the version
of the inference server you are using.

* To enable autoscaling, your cluster's kube-apiserver must have the [aggregation layer
enabled](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/).
This will allow the horizontal pod autoscaler to read custom metrics from the prometheus adapter.

For more information on Helm and Helm charts, visit the [Helm documentation](https://helm.sh/docs/).

## Quick Deploy Instructions

First, clone this repository to a local machine. Then, execute the following commands:

### Install helm

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

#### Deploy Prometheus and Grafana

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```
### Create a new TLS secret named tls-secret
```
  kubectl create secret tls tls-secret --cert=path/to/tls.cert --key=path/to/tls.key
```

#### Deploy Triton with default settings

```
cd /triton-server-ngxin-plus-ingress
helm install example .
```


<!-- The steps below describe how to set-up a model repository, use Helm to
launch the inference server, and then send inference requests to the
running server. You can access a Grafana endpoint to see real-time
metrics reported by the inference server. -->


## Installing Helm

### Helm v3

If you do not already have Helm installed in your Kubernetes cluster,
executing the following steps from the [official Helm install
guide](https://helm.sh/docs/intro/install/) will
give you a quick setup.

If you are currently using Helm v2 and would like to migrate to Helm v3,
see the [official migration guide](https://helm.sh/docs/topics/v2_v3_migration/).

## Model Repository
If you already have a model repository, you may use that with this Helm
chart. If you do not have a model repository, you can check out a local
copy of the server source repository to create an example
model repository:

```
$ git clone https://github.com/f5devcentral/triton-server-ngxin-plus-ingress.git
```

Triton Server needs a repository of models that it will make available
for inferencing. For this example, we are using an existing NFS server and
placing our model files there. 

Following the [QuickStart](../../docs/getting_started/quickstart.md), download the
example model repository to your system and copy it onto your NFS server.
Then, add the url or IP address of your NFS server and the server path of your
model repository to `values.yaml`.


## Deploy Prometheus and Grafana

The inference server metrics are collected by Prometheus and viewable
through Grafana. The inference server Helm chart assumes that Prometheus
and Grafana are available so this step must be followed even if you
do not want to use Grafana.

Use the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart to install these components. The
*serviceMonitorSelectorNilUsesHelmValues* flag is needed so that
Prometheus can find the inference server metrics in the *example*
release deployed in a later section.

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```

Then port-forward to the Grafana service so you can access it from
your local browser.

```
$ kubectl port-forward service/example-metrics-grafana 8080:80
```

Now you should be able to navigate in your browser to localhost:8080
and see the Grafana login page. Use username=admin and
password=prom-operator to log in.

An example Grafana dashboard is available in dashboard.json. Use the
import function in Grafana to import and view this dashboard.

## Enable Autoscaling
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

## Enable Load Balancing
To enable load balancing, ensure that the loadBalancing tag in `values.yaml`
is set to `true`. This will do two things:

1. Deploy a Traefik reverse proxy through the [Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart).

2. Configure two Traefik [IngressRoutes](https://doc.traefik.io/traefik/providers/kubernetes-crd/),
one for http and one for grpc. This will allow the Traefik service to expose two
ports that will be forwarded to and balanced across the Triton pods.

To choose the port numbers exposed, or to disable either http or grpc, edit the
configured variables in `values.yaml`.

## Deploy the Inference Server

Deploy the inference server, autoscaler, and load balancer using the default
configuration with the following commands.

Here, and in the following commands we use the name `example` for our chart.
This name will be added to the beginning of all resources created during the helm
installation.

```
$ cd <directory containing Chart.yaml>
$ helm install example .
```

Use kubectl to see status and wait until the inference server pods are
running.

```
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
example-triton-inference-server-5f74b55885-n6lt7   1/1     Running   0          2m21s
```

There are several ways of overriding the default configuration as
described in this [Helm
documentation](https://helm.sh/docs/using_helm/#customizing-the-chart-before-installing).

You can edit the values.yaml file directly or you can use the *--set*
option to override a single parameter with the CLI. For example, to
deploy a cluster with a minimum of two inference servers use *--set* to
set the autoscaler.minReplicas parameter.

```
$ helm install example --set autoscaler.minReplicas=2 .
```

You can also write your own "config.yaml" file with the values you
want to override and pass it to Helm. If you specify a "config.yaml" file, the
values set will override those in values.yaml.

```
$ cat << EOF > config.yaml
namespace: MyCustomNamespace
image:
  imageName: nvcr.io/nvidia/tritonserver:custom-tag
  modelRepositoryPath: gs://my_model_repository
EOF
$ helm install example -f config.yaml .
```



## Using Triton Inference Server

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
## Cleanup

After you have finished using the inference server, you should use Helm to
delete the deployment.

```
$ helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
mytest  default         1               2024-04-15 19:01:31.772857 -0700 PDT    deployed        triton-inference-server-1.0.0   1.0        

$ helm uninstall mytest
```