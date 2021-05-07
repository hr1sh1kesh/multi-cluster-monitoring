The usecases here are 2 fold. 
a) An operations view where the operations team needs a single pane of glass to visualize the cluster and application health for all namespaces. 
b) A localized view for every namespace where every application/customer team needs access to view metrics of the current application instance

This setup deploys prometheus in a single centralized namespace (soft tenancy) and then uses the prom-label-proxy component to seperate out metric visualization for different stakeholders.  

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
git clone https://github.com/hr1sh1kesh/multi-tenant-monitoring.git
```

- Deploy kube-prometheus from the bitnami chart in the monitoring namespace. This chart deploys the operator in the previously created monitoring namespace alongwith a prometheus statefulset deployment. This will also enable the thanos sidecar for every prometheus. This prometheus scrapes all the service monitors in all application/customer namespaces. 

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install prometheus bitnami/kube-prometheus -f single-prometheus-prom-label-proxy/kube-prometheus/values.yaml
```

- Deploy the application and service monitors
```
kubectl apply -f kube-prometheus/01-example-app.yaml
kubectl apply -f kube-prometheus/02-servicemonitors.yaml
kubectl apply -f kube-prometheus/03-prom-label-proxy-rbac.yaml
kubectl apply -f kube-prometheus/04-prom-label-proxy.yaml
```

This deploys the prom-label-proxy deployment in the `monitoring` namespace. It also deploys just the service monitoring in the individual `app1` and `app2` namespaces. 

- Deploy Grafana in each application namespaces `app1` and `app2`. The datasource configuration would require a `namespace=app1` extra parameter when configuring it. These grafanas solve the usecase of a localized dashboard per team / customer. 

app1 and app2 namespaces are created in the above step.
```
helm install grafana -n app1 bitnami/grafana -f grafana/values.yaml
helm install grafana -n app2 bitnami/grafana -f grafana/values.yaml
```


A centralized grafana in the monitoring namespace can be deployed to satisfy usecase 1 described above. 

```
helm install grafana bitnami/grafana -n monitoring -f single-prometheus-prom-label-proxy/grafana/values.yaml
```
