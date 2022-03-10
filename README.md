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

2. Verify the knative operator deployment:
```
kubectl get deployment knative-operator
```
```
kubectl logs -f deploy/knative-operator
```

3. Create the knative serving custom resource:
```
kubectl apply -f knative-serving.yaml
```

4. Verify the knative deployment:
```
kubectl get deployment -n knative-serving
```

5. For the networking layer we are going to use Istio, specifically we want to use the `getmesh` cli tool to install the Tetrate Istio Distribution by using:
```
curl -sL https://istio.tetratelabs.io/getmesh/install.sh | bash
```
Lets validate the cli tool:
```
getmesh version
```

6. Having our k8s cluster running, we can install Istio using getmesh:
```
getmesh istioctl install -y
```
While the resources go up, lets label the default and knative-serving ns for istio injection:
```
kubectl label namespace default knative-serving istio-injection=enabled
```
Monitor the pods on the `istio-system` ns and when they are ready, validate the install by issuing:
```
getmesh config-validate
```

7. Now lets enable mTLS:
```
kubectl apply -f kantive-peer-auth.yaml
```

5. Now lets fetch the Istio ingress gateway external IP to be used later on this lab:
```
export ISTIO_GW=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
```
```
echo $ISTIO_GW
```

6. Lets use Magic DNS (sslip.io) as the DNS suffix:
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.2.0/serving-default-domain.yaml
```
To validate everything is configured properly you should be able to find the $ISTIO_GW external IP in the knative `config-domain` configmap:
```
kubectl get cm config-domain --namespace knative-serving -ojson | grep "sslip.io"
```

7. Finally, if everything is properly configured you should see a `READY` status when running:
```
kubectl get KnativeServing knative-serving -n knative-serving
```

8. Let's deploy the example service `Hello world`:
```
kubectl apply -f hello-world-svc.yaml
```

9. Let's also deploy the istio sleep service to test the `hello world` service:
```
kubectl apply -f sleep.yaml
```

10. Now lets test accessing from sleep to hello-world:
```
kubectl exec $SLEEP_POD -it -- curl hello.default.$ISTIO_GW.sslip.io
```
You should see the `Hello World!` output.
This proves knative succesfully configured the and instance of an Istio gateway for the hello world service.

11. Now lets install the observability tools:
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/grafana.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/extras/zipkin.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.10/samples/addons/kiali.yaml
```
If you get the following error No matches for kind "MonitoringDashboard" in version "monitoring.kiali.io/v1alpha1" when installing Kiali, make sure you re-run the command again.

12. WIP
