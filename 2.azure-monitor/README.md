# Istio Observability on Azure Monitor

The purpose of this lab is to explore the current options available to export Istio known observability mechanisms to Azure Monitor, including concerns like: logging, distributed tracing, metrics, dashboards and service mesh consoles.  

## Service Mesh Console => Kiali

![](../images/kiali-app.png)

Kiali is the management console for Istio service mesh, it could be installed as an add-on and it could be a trusted part of your production environment. 

At the time of writing, Kiali is the default and only visualization tool for Istio and it could be installed on a production setting (https://kiali.io/docs/installation/installation-guide/). Also at the time of writing, Azure Monitor doesn't include a Kiali-like console for Istio service mesh management.

There is a discussion taking place to use Kiali for OSM which could open the door for other providers to offer a standardized way for service mesh management (https://github.com/kiali/kiali/discussions/4166).

The recommendation in this case is if you are going to use Istio and need a visual management console, then just use Kiali as there are no other "turn-around" consoles to use for the same purpose and creating one on Azure Monitor could be a significant endeavor.

***

## Centralized Dashboards => Grafana

Grafana is an open source analytics and interactive visualization web application for centralized dashboards. Azure Monitor has a similar idea implemented under the `Application Insights` umbrella, where you can centralize metrics and dashboards from different data sources into one logical entity. 

Both Grafana and Insights provide powerful features for diagnostics, alerting and failure management under a "single pane of glass" having Azure Monitor telemetry features a much larger footprint than Grafana.

When it comes to Istio concerns, Grafana being an open source project provides a set of loaded dashboards customized for Istio (https://gcsweb.istio.io/gcs/istio-release/releases/1.12.5/grafana/), including:

- Mesh dashboard
- Performance dashboard
- Service dashboard
- Workload dashboard
- Extension dashboard
- Pilot dashboard

The recommendation in this case is to use Grafana's dashboards instead of trying to recreate the same on Azure Monitoring, which could be done by "reverse engineering" the dashboards and importing the same data sources used into Azure Monitor. This as expected, could require a significant amount of time to accomplish.

The recommendation is to use the out of the box Grafana-Istio dashboards and in the case of having a Grafana instance is not something feasible, then analyze the gap and create custom Azure monitor dashboards.

***

## Logging => Access Logging

Logging is one of the most important and traditional observability concerns. AKS comes built in with container logging monitoring when Azure AKS monitoring is enabled on the cluster. This means `stdout` and `stderr` is automatically collected by insights. For more information read [here.](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-agent-config)

The simplest kind of Istio logging is Envoy’s access logging. Envoy proxies print access information to their standard output. The standard output of Envoy’s containers can then be printed by the kubectl logs command.

The recommended way to enable access logging is using the Telemetry API, see the file named `access-logging.yaml` in this folder which enables mesh wide access logging using envoy as log provider.

Similar configuration can also be applied on an individual namespace, or to an individual workload, to control logging at a fine grained level.

The following short lab presents a quick way to enable Istio access logging and observe it using Azure monitor.

***
## Lab Prerequisites

- AKS cluster
- Monitoring enabled on your AKS instance
- Workload with metrics exposed

## LAB: Istio Access Logging on Azure Monitor

The Istio Telemetry API exposes a `Telemetry` resource that can be used to configure the mesh telemetry settings. The following can be used to enable envoy as the access log provider across the mesh:

```
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
```

Apply the resource to your cluster:

```
kubectl apply -f access-logging.yaml
```

In addition to applying this resource, you need to make sure the `istio` ConfigMap on the `istio-system` namespace has the `meshConfig` path for Envoy's access logs: `/dev/stdout`.

You can accomplish this by adding it to the ConfigMap:

![](../images/istio-cm.png)

You can also add the config when using the Istio Operator:

```
spec:
  meshConfig:
    accessLogFile: /dev/stdout
```

Or when using the `istioctl install` command:

```
istioctl install <flags-you-used-to-install-Istio> --set meshConfig.accessLogFile=/dev/stdout
```

Now that you have all the configuration in place, generate some load from the `sleep` service to the `hello-world` service:

```
kubectl exec $SLEEP_POD -it -- curl hello.default
```

Now you can inspect the logs for each service `istio-proxy` containers and you should see the request source and destination:

`sleep`:
```
kubectl logs sleep-557747455f-8f4xc -c istio-proxy
```
expect a log like:
```
[2022-03-17T19:39:58.072Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 13 2445 2444 "-" "curl/7.82.0-DEV" "a2a491d6-4e30-455f-a81d-8029a4dc5f31" "hello.default" "10.150.0.186:8012" outbound|80||hello-world.default.svc.cluster.local 10.150.1.21:46372 10.0.28.127:80 10.150.1.21:43916
```

Now on the `hello-world`:
```
kubectl logs hello-world-deployment-cc647cf6d-bmpcp -c istio-proxy
```
expect a log like:
```
[2022-03-17T19:40:29.074Z] "GET /metrics HTTP/1.1" 200 - via_upstream - "-" 0 49 0 0 "-" "Go-http-client/1.1" "07162f64-94fe-4c42-8542-711f9b805151" "10.150.0.139:9090" "10.150.0.139:9090" inbound|9090|| 127.0.0.6:39535 10.150.0.139:9090 10.150.0.249:44576 - default
```

Take a note on the "long" id shared on the log, those are the `X-REQUEST-ID` of the transaction, save both for a later search.

For more information about the `x-request-id` header read [here.](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-request-id)

You can customize how Envoy's access logging works, for more information read [here.](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)

If everything was properly configured then by just exposing these logs over `stdout` we can query them over Azure Monitor logs. On the UI go to your AKS cluster -> Monitoring -> Logs and run a query similar to the following one, replacing the `x-request-id`:

```
let FindString = "a2a491d6-4e30-455f-a81d-8029a4dc5f31";
ContainerLog 
| where LogEntry has FindString 
|take 100
```

You should expect to see something similar to this:

![](../images/access-logging.png)

***
NOTE:
Take into account Azure Monitor agents takes  more time than other agents in this space to report logs and it could take up to 20 minutes for them to appear on the monitoring side.
***

***
NOTE: If you don't want to use Azure built-in agents to "ship" the logs to Azure, there are other alternatives like using FluentBit directly. Note too that these Azure agents are actually based on FluentBit and Telegraf.
***

***

## Metrics => Prometheus

Prometheus is a time series DB used as a metrics-based monitoring system. Azure Monitor provides the ability out of the box to scrape prometheus-like metrics exposed from workloads into Insights. 

This happens when you enable Monitoring on your AKS cluster and configure its agents to scrape the endpoints of your workloads in a similar way a Prometheus instance would do.

The following lab presents the steps on how to enable this metrics integration.

***

### Lab Prerequisites

- AKS cluster
- Monitoring enabled on your AKS instance
- Workload with metrics exposed
### LAB: Prometheus Style Metrics

Azure monitor supports Prometheus like monitoring by using the same annotations used on pods to report metrics used in a traditional Prometheus setup. Your pod needs to expose the endpoints to be scraped for monitoring and they can be discovered using the following annotations:
```
prometheus.io/scrape: 'true'
prometheus.io/path: '/data/metrics'
prometheus.io/port: '80'
```

For our lab example we are going to deploy the `sleep` sample app and describe its pod:
```
kubectl apply -f sleep.yaml
```

Using our sleep pod example we can `describe` the pod:

```
kubectl describe pod sleep-557747455f-8f4xc
```

Now observe the following annotations:
```
prometheus.io/path: /stats/prometheus
prometheus.io/port: 15020
prometheus.io/scrape: true
```

This annotations are used to "tell" Prometheus if this pod should be scrapped and the path/port where to do it. Azure Monitor integrations leverages these same annotations without the need to have a Prometheus instance running.

If you remote shell into your sleep pod you can see the stats being exposed on the path mentioned above:

```
kubectl exec sleep-557747455f-8f4xc -it -- curl localhost:15020/stats/prometheus
```

You should see different key-value pairs being printout corresponding to the metrics exposed.

Now that we know our workload exposes "prometheus-style" metrics and having Azure monitoring enabled on your AKS cluster, lets validate we have the insights agent pods running:

```
kubectl get pods -n kube-system | grep oms
```

You should see a ReplicaSet with the name `omsagent` running:

![](../images/omsagent-pod.png)

***
NOTE: If you don't see the pod, probably Monitor is not enabled on your AKS cluster, follow this [article](https://docs.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-onboard) to set it up.
*** 

Azure monitoring configuration is managed by the K8s ConfigMap named `container-azm-ms-agentconfig`, you can find a copy of it [here](https://github.com/microsoft/OMS-docker/blob/ci_feature_prod/Kubernetes/container-azm-ms-agentconfig.yaml). 

In this ConfigMap, you can find a section called `prometheus-data-collection-settings` where we can configure how the metrics are going to be scraped. In our case we are going to enable the cluster-wide monitoring by doing:

```
monitor_kubernetes_pods = true
```

Now you can apply this ConfigMap:

```
kubectl apply -f container-azm-ms-agentconfig.yaml
```

Now list all the pods on the `kube-system` ns and look for the pod name starting `omsagent-rs-` which should be restarting with the new settings. In order to make sure the new configuration took effect, you need to inspect the logs after the pod has restarted:

```
kubectl logs omsagent-rs-5c5f869c9c-7sbnr -n kube-system
```

***
NOTE:
Take into account Azure Monitor agents takes  more time than other agents in this space to report logs and it could take up to 20 minutes for them to appear on the monitoring side.
***

You should see logs for processing the Prometheus configuration like these:

```
****************Start Prometheus Config Processing********************
config::configmap container-azm-ms-agentconfig for settings mounted, parsing values for prometheus config map
config::Successfully parsed mounted prometheus config map
config::Successfully passed typecheck for config settings for replicaset
config::Starting to substitute the placeholders in telegraf conf copy file for replicaset
config::Successfully substituted the placeholders in telegraf conf file for replicaset
config::Successfully created telemetry file for replicaset
****************End Prometheus Config Processing********************
```

and you should see logs for FluentBit and Telegraf starting:

```
Fluent Bit v1.7.8
* Copyright (C) 2019-2021 The Fluent Bit Authors
* Copyright (C) 2015-2018 Treasure Data
* Fluent Bit is a CNCF sub-project under the umbrella of Fluentd
* https://fluentbit.io

Routing container logs thru v2 route... 
waitforlisteneronTCPport found listener on port:25226 in 1 secs
Telegraf 1.20.3 (git: HEAD 7a4e60aa)
2022-03-15T15:57:52Z I! Starting Telegraf 1.20.3
td-agent-bit 1.7.8
stopping rsyslog...
 * Stopping enhanced syslogd rsyslogd
   ...done.
getting rsyslog status...
 * rsyslogd is not running
2022-03-15T15:57:52.1866550Z: Onboarding success. Sucessfully registered certificate with OMS Homing service.
Onboarding success
```

***
NOTE: Make sure you track down and correlate the Telegraf timestamps to make sure your new configuration took effect. I experienced several times my agent pods not restarting or just not taking the configuration as expected.
***

### How to view monitoring metrics on Azure Monitor

On your Azure console go to your AKS cluster -> Monitoring -> Logs and run a query similar to the following one:

```
InsightsMetrics 
| where Name contains "istio_requests_total"
```

***
NOTE: The `istio_requests_total` is the same Istio metric we used on the previous section to showcase Prometheus.
***

You should be able to see the same metrics for `istio_requests_total` observed previously on Prometheus or when scraping the `sleep` or `hello-world` services:

![](../images/prometheus-metrics.png)

Notice all the metrics collected from all the workloads on the cluster using the "prometheus-style" annotations. These should include the `sleep`, `hello-world` workloads as well as the Istio's `ingressgateway`. 

You can filter the metrics even more by using tags as follows:

```
InsightsMetrics 
| where Name contains "istio_requests_total"
| where Tags contains "sleep"
```

You should see now only `istio_requests_total` metrics for the `sleep` service.
 
***

 ## Distributed Tracing => Zipkin / Jaeger / Lightstep

From Istio's documentation website: distributed tracing enables users to track a request through mesh that is distributed across multiple services. This allows a deeper understanding about request latency, serialization and parallelism via visualization.

Istio leverages [Envoy’s distributed tracing](https://www.envoyproxy.io/docs/envoy/v1.12.0/intro/arch_overview/observability/tracing) feature to provide tracing integration out of the box. Specifically, Istio provides options to install various tracing backend and configure proxies to send trace spans to them automatically. See [Zipkin](https://istio.io/latest/docs/tasks/observability/distributed-tracing/zipkin/), [Jaeger](https://istio.io/latest/docs/tasks/observability/distributed-tracing/jaeger/) and [Lightstep](https://istio.io/latest/docs/tasks/observability/distributed-tracing/lightstep/) task docs about how Istio works with those tracing systems.

Current efforts are trying to converge some of the presented observability concerns into an open standard to be used across the industry. [OpenTelemetry](https://opentelemetry.io/) is getting traction in this field and recently, Microsoft announced the adoption of the standard but there is a lot of work still to be done. See [here.](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/433)

In the `OpenTelemetry` world, a collector is a vendor-agnostic implementation with the purpose to receive, process and export telemetry data.

If the intention is to use Istio's out of the box distributed tracing configuration then please use Zipkin or Jaeger to explore further, but if the intention is to use Azure Monitor, then some integration using `OpenTelemetry` is needed.  

The reason is because Istio's tracing is based on Envoy's tracing which uses `B3` headers to propagate and send the span information to be correlated into a single trace. At the time of writing, Azure Monitor's tracing is not compatible with `B3` headers and instead it uses `W3C` style headers. For this reason we need to use an `otel` (OpenTelemetry) collector to receive the Istio `B3` style headers and process them to be exported using and Azure compatible exporter with the purpose to get those traces to Azure Monitor's collector. 

Using an `otel` collector, the tracing telemetry would flow like:

`Envoy Sidecar -> Zipkin-style span data -> OTel -> azure span data -> Azure collector`

The following lab brings hands-on experience in how to use this instrumentation.

***

### Lab Prerequisites

- AKS cluster
- Monitoring enabled on your AKS instance
- Istio Service Mesh

### LAB: Otel Collector to Azure Monitor

Take a look at the file named `otel-collector.yaml` where we have four resources, a couple ConfigMaps, a DaemonSet and a Deployment.