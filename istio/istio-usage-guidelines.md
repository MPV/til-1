# Istio Usage Guidelines

This til provides information regarding Istio installation and usage.

## Istio Installations

This section provides step by step guideline regarding istio installation.

* Create a namespace using the manifest given below:

```json
{ 
  "kind": "Namespace", 
  "apiVersion": "v1",
  "metadata": { 
    "name": "istio-system", 
    "labels": { 
      "name": "istio-system" 
    } 
  } 
}
```

Create the above namespace using the command given below:

```bash
$ sudo kubectl apply -f namespace-creation-script.yaml
```

* There are two ways to install istio in kubernetes cluster:

  * Method 1: Follow the guidelines given in this [link](https://istio.io/docs/setup/kubernetes/install/kubernetes/) to install `istio` using kubernetes manifests. 

  * Method 2: In this method `HelmRelease` will be used for installation, `HelmRelease` manifest is given below:
  
  
```yaml
apiVersion: flux.weave.works/v1beta1
kind: HelmRelease
metadata:
  name: istio
  namespace: istio-system
spec:
  releaseName: istio
  chart:
    repository: https://raw.githubusercontent.com/IBM/charts/master/repo/stable
    name: ibm-istio
    version: 1.0.5
  values:
    global:
      enableTracing: true
    tracing:
      enabled: true
    jaeger:
      ingress:
        enabled: true
    pilot:
      traceSampling: 1
```
Currently, I am using [IBM's helm chart for istio](https://github.com/IBM/charts/tree/master/stable/ibm-istio).



## Enable Tracing
The above manifest will enable the istio tracing. To check whether tracing is enabled or not we will use the sample app provided by istio.

* Follow these [guidelines](https://istio.io/docs/examples/bookinfo/) to deploy the sample application.

* Deploy the xposer for enabling ingress, ask `ali` for its kubernetes manifest for xposer.

* Use the manifest given below to enable jeager ingress because the default configuration(using the helm chart value parameters) are not working correctly due to invalid documentation: 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:    
    ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/ingress.class: external-ingress
  name: jaeger-query
spec:
  rules:
  - host: jaeger-query.stakater.com
    http:
      paths:
      - backend:
          serviceName: jaeger-query
          servicePort: 16686
```

* Ingress will take sometime before it is enabled.

* Each of the application service needs to be visible in the jeager services drop down as well as in the dependencies graph.

### Nginx Ingress Controller

The reason for using nginx-ingress is that our other services are already using it to create ingress (a load balancer is attached with nginx-ingress) but istio creates a new service of type LoadBalancer to expose its services and we don't want to use it. An [issue](https://github.com/istio/istio/issues/14328) has been created regarding how to use nginx-ingress controller with istio.


## Proxy(sidecar) Container Injection
There are two ways to inject sidecar(proxy) container:

* By default istio inserts sidecar containers automatically, to disable it set this `sidecarInjectorWebhook.enabled` flag to false.

* To manually inject sidecar use the instruction given on this [link](https://istio.io/docs/setup/kubernetes/additional-setup/sidecar-injection/#manual-sidecar-injection).

## NOTES
These notes are regarding the issue that might come up during istio deployment:

* Sometimes due to ungraceful deletion of istio helm release CRDs will not be removed properly. There are two ways to delete remaining CRDs. 

  * `Method-1`: Use the command given below to get all the CRDs and delete them one by one:
  ```bash
  $ sudo kubectl get crd

  $ sudo kubectl delete crd <crd-name>
  ```

  * `Method-2`: In this method we will use the manifest for crd creation to delete all the corresponding CRDs. First of all down the istio release. Move inside this folder (istio-X/install/kubernetes/helm/istio-init/files/) and run the command given below on each file:
  ```bash
  $ sudo kubectl delete -f <filename>.yaml
  ```

* Before enabling tracing for an application following requirements needs to be fulfilled:
  * Pods and services [requirements](https://istio.io/docs/setup/kubernetes/prepare/requirements/)
  * Application code needs to modified a little bit(example can be found in this [link](https://github.com/istio/istio/blob/master/samples/bookinfo/src/productpage/productpage.py#L130) of istio sample application) so that it can handle the trace information that is part of the request. Details can be found on this [link](https://github.com/istio/istio/issues/14094)  
