The usecases here are 2 fold. 
a) An operations view where the operations team needs a single pane of glass to visualize the cluster and application health for all namespaces. 
b) A localized view for every namespace where every application/customer team needs access to view metrics of the current application instance

This setup deploys prometheus in every namespace (hard tenancy). It also deploys the thanos sidecar for every prometheus and a single querier to query all the sidecars in every namespace. This querier is then used as a datasource for the centrally deployed grafana for the Ops view.


- Create a kind cluster. (preferrably a multi-node one.)
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: worker
- role: worker
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "worker=true"
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "control=true"
```

- Create a namespace called `monitoring`

```
kubectl create ns monitoring
```

- Clone this repository in a directory 
```
```

- Deploy kube-prometheus from the bitnami chart in the monitoring namespace. This chart deploys the operator in the previously created monitoring namespace alongwith a prometheus statefulset deployment. This will also enable the thanos sidecar for every prometheus. This prometheus scrapes all the service monitors in all application/customer namespaces. 

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install prometheus bitnami/kube-prometheus --namespace monitoring -f prometheus-per-ns/kube-prometheus/values.yaml
```

- Deploy the application and service monitors
```
kubectl apply -f kube-prometheus/01-example-app.yaml
kubectl apply -f kube-prometheus/02-prometheus-app1.yaml
kubectl apply -f kube-prometheus/03-prometheus-app2.yaml
kubectl apply -f kube-prometheus/04-servicemonitors.yaml
```
This deploys a Prometheus statefulset in both the namespaces `app1` and `app2` which are created in the steps above. 

- Deploy Thanos in the monitoring namespace
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install thanos bitnami/thanos --namespace monitoring -f prometheus-per-ns/thanos/values.yaml
```
> Note: This deploys the Thanos querier component with hardcoded endpoints for the `app1` and `app2` Thanos sidecars earlier. 
The values yaml also has endpoints for the prometheus sidecars from the app namespaces. 

- Deploy Grafana in the monitoring namespace

```
helm install grafana bitnami/grafana -f prometheus-per-ns/grafana/values.yaml
```

Add the Thanos querier as the datasource for this Grafana. Expose an ingress and you should be able to fulfil usecase 1. 
For localized views, deploy Grafana in every namespace with the datasource of that grafana pointing only to the prometheus within that namespace. 