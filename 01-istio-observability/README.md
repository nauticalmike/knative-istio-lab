# knative-istio-lab
Quick lab on how to use Istio + Knative

## Prerequisites

- k8s cluster
- kubectl
- istioctl

## Steps to install knative + Istio demo

1. Install the knative operator either by using the version on the attached file 'operator.yaml':
    ```
    kubectl apply -f operator.yaml
    ```
    or by using:
    ```
    kubectl apply -f https://github.com/knative/operator/releases/download/knative-v1.2.0/operator.yaml
    ```

1. Verify the knative operator deployment:
    ```
    kubectl get deployment knative-operator
    ```
    ```
    kubectl logs -f deploy/knative-operator
    ```

1. Create the knative serving custom resource:
    ```
    kubectl apply -f knative-serving.yaml
    ```

1. Verify the knative deployment:
    ```
    kubectl get deployment -n knative-serving
    ```

1. For the networking layer we are going to use Istio, specifically we want to use the `getmesh` cli tool to install the Tetrate Istio Distribution by using:
    ```
    curl -sL https://istio.tetratelabs.io/getmesh/install.sh | bash
    ```
    Lets validate the cli tool:
    ```
    getmesh version
    ```

1. Having our k8s cluster running, we can install Istio using getmesh:
    ```
    getmesh istioctl install -y
    ```
    While the resources go up, lets label the default and knative-serving ns for istio injection:
    ```
    kubectl label namespace default knative-serving istio-injection=enabled
    ```
    When all the Istio pods are ready then monitor the pods on the `istio-system` ns and when they are ready, validate the install by issuing:
    ```
    getmesh config-validate
    ```

1. Now lets fetch the Istio ingress gateway external IP to be used later on this lab:
    ```
    export ISTIO_GW=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
    ```
    ```
    echo $ISTIO_GW
    ```

1. For the purpose of this Lab, lets just use Magic DNS (sslip.io) as the DNS suffix:
    ```
    kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-default-domain.yaml
    ```
    To validate everything is configured properly you should be able to find the $ISTIO_GW external IP in the knative `config-domain` configmap:
    ```
    kubectl get cm config-domain --namespace knative-serving -ojson | grep "sslip.io"
    ```

1. Finally, if everything is properly configured you should see a `READY` status when running:
    ```
    kubectl get KnativeServing knative-serving -n knative-serving
    ```

1. Let's deploy the example service `Hello world`:
    ```
    kubectl apply -f hello-world-svc.yaml
    ```

1. Let's also deploy the istio sleep service to test the `hello-world` service:
    ```
    kubectl apply -f sleep.yaml
    ```

1. Capture the name of the sleep pod to an environment variable:
    ```
    export SLEEP_POD=$(kubectl get pod -l app=sleep -ojsonpath='{.items[0].metadata.name}')
    ```

1. Now lets test accessing from sleep service to `hello-world` service directly wihin the mesh:
    ```
    kubectl exec $SLEEP_POD -it -- curl hello.default
    ```
    You should see the `Hello World!` output.
    This proves knative succesfully configured the `hello-world` service and gateway's instance to access the service within the cluster.

1. Now lets perform the same test but now accessing from the ingress:
    ```
    curl http://hello.default.$ISTIO_GW.sslip.io
    ```

14. Now lets install the observability tools:
    ```
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/prometheus.yaml
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/grafana.yaml
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/extras/zipkin.yaml
    kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/kiali.yaml
    ```
    If you get the following error No matches for kind "MonitoringDashboard" in version "monitoring.kiali.io/v1alpha1" when installing Kiali, make sure you re-run the command again.

1. Now that we have Prometheus, Grafana, Zipkin and Kiali installed, let generate some load to our `hello-world` service using a load generator tool like `siege`:
    ```
    siege --delay=3 --concurrent=3 --time=5M http://hello.default.$ISTIO_GW.sslip.io
    ```
    If you don't want to use `siege`, you can always just do a loop like:
    ```
    while true; \
    do curl http://hello.default.$ISTIO_GW.sslip.io; \
    sleep 1;done
    ```

1. Now that we are generating traffic to the `hello-world` service lets see how this looks on kiali:
    ```
    istioctl dashboard kiali
    ```
    - A new tab should open on your browser, if not just open one an go to the URL printed in the console output.
    - The istioctl dashboard command also blocks. Leave it running until you're finished using the dashboard, at which time pressing Ctrl+C can interrupt the process and put you back at the terminal prompt.
    - In the Kiali dashboard select the Graph section from the sidebar.
    - Under Select Namespaces (at the top of the page), select the default namespace, the location where the application's pods are running.
    - From the third "pulldown" menu, select App graph.
    - From the Display "pulldown", toggle on Traffic Animation.
    - From the footer, toggle the legend so that it is visible. Take a moment to familiarize yourself with the legend.
    - Observe the visualization and note the following:
        - We can see traffic coming in through the ingress gateway to the `hello-world` service as well the istio-ingress gateway. This is because Kiali doesn't recognize the DNS sslip.io job and it marks it as unknown.
        - The lines connecting the services are green, indicating healthy requests
    - Such visualizations are helpful with understanding the flow of requests in the mesh, and with diagnosis.
    - Feel free to spend more time exploring Kiali.
    - Close the Kiali dashboard. Interrupt the istioctl dashboard kiali command by pressing Ctrl+C.

1. Launch the Zipkin dashboard:
    ```
    istioctl dashboard zipkin
    ```
    - When the Zipkin dashboard displays, click on the red '+' button and select serviceName.
    Select the service named `hello-world.default` and click on the Run Query button (lightblue) to the right.

    - A number of query results will display. Each row is expandable and will display more detail in terms of the services participating in that particular trace.

    - Click the Show button to the right of one of the traces having two (2) spans. Looking closely you can see the `hello-world` span is being "called" by the `istio-ingressgateway`.

    - The resulting view shows spans that are part of the trace, and more importantly how much time was spent within each span. Such information can help diagnose slow requests and pin-point where the latency lies.

    - Distributed tracing also helps us make sense of the flow of requests in a microservice architecture.

    - Close the Zipkin dashboard. Interrupt the istioctl dashboard zipkin command with Ctrl+C.

1. Start the prometheus dashboard:
    ```
    istioctl dashboard prometheus
    ```

    - In the search field enter the metric named `istio_requests_total`, and click the Execute button (on the right).

    - Select the tab named Graph to obtain a graphical representation of this metric over time.

    - Note that you are looking at requests across the entire mesh, i.e. this includes both requests to `hello-world` and to the `ingress`.

    - As an example of Prometheus' dimensional metrics capability, we can ask for total requests having a response code of 200:
    ```
    istio_requests_total{response_code="200"}
    ```

    - With respect to requests, it's more interesting to look at the rate of incoming requests over a time window. Try:
    ```
    rate(istio_requests_total[5m])
    ```

    - There's much more to the Prometheus query language (this may be a good place to start).

    - Grafana consumes these metrics to produce graphs on our behalf.

    - Close the Prometheus dashboard and terminate the corresponding istioctl dashboard command.

1. Launch the Grafana dashboard:
    ```
    istioctl dashboard grafana
    ```
    
    - From the sidebar, select Dashboards → Manage, then click on the folder named Istio to reveal pre-designed Istio-specific Grafana dashboards.

    - Explore the Istio Mesh Dashboard. Note the Global Request Volume and Global Success Rate.
    
    - Explore the Istio Service Dashboard. First select the service `hello-world-private.default.svc.cluster.local` and inspect its metrics.
    
    - Explore the Istio Workload Dashboard. Select the `hello-world` workload. Look at Outbound Services and note the outbound requests to the customers service. 

    - Feel free to further explore these dashboards.

    - To clean up, terminate the istioctl dashboard command (Ctrl+C).