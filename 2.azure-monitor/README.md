# Istio Monitoring on Azure Monitor

## Prerequisites
- Azure k8s cluster
- Azure Monitor enabled on your AKS instance
- Istio + Knative
- Workload with metrics exposed

## Explanation

TL:DR

## Steps

1. Azure monitor supports Prometheus like monitoring by using the same annotations used on pods to report metrics used in a traditional Prometheus setup. Your pod needs to expose the endpoints to be scraped for monitoring and they can be discovered using the following annotations:
```
prometheus.io/scrape: 'true'
prometheus.io/path: '/data/metrics'
prometheus.io/port: '80'
```

Using our sleep pod example we can `describe` the pod and observe the following annotations:
```
prometheus.io/path: /stats/prometheus
prometheus.io/port: 15020
prometheus.io/scrape: true
```

If you remote shell into your sleep pod you can see the stats being exposed on the path mentioned above:
```
kubectl exec sleep-557747455f-tdbjk -it -- /bin/sh
```
```
curl localhost:15020/stats/prometheus
```
You should see different key-value pairs being printout corresponding to the metrics exposed at the envoy proxy level.

1. Now that we now our workload exposes metrics "Prometheus" style and having Monitor enabled on your AKS cluster, lets validate we have the conrainer insights pods running:
```
kubectl get pods -n kube-system
```
You should see and `omsagent` pod running:
![](../images/omsagent-pod.png)

    If you don't see the pod, probably Monitor is not enabled on your AKS cluster, follow this [article](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard) to set it up. 

1. Azure monitoring configuration is managed by the K8s ConfigMap named `container-azm-ms-agentconfig`, you can also find a copy of it [here](https://github.com/microsoft/OMS-docker/blob/ci_feature_prod/Kubernetes/container-azm-ms-agentconfig.yaml) in case yours comes empty. 

In this ConfigMap `prometheus-data-collection-settings` we can specify how the metrics are going to be scraped, you can declare specific URLs, pods, ports etc. In our case we are going to enable the cluster-wide monitoring and the port and path used by our `sleep` and `hello-world` pods:
```
prometheus.io/path: /stats/prometheus
prometheus.io/port: 15020
monitor_kubernetes_pods = true
```

Now you can apply this ConfigMap:
```
kubectl apply -f container-azm-ms-agentconfig.yaml
```

Now list all the pods on the `kube-system` ns and look for the pod name starting `omsagent-rs-` which should be restarting with the new settings and check its logs when up:
```
kubectl logs omsagent-rs-5c5f869c9c-7sbnr -n kube-system
```
You should see logs for FluentBit and Telegraf starting.

## How to view monitoring metrics on Azure Monitor

 