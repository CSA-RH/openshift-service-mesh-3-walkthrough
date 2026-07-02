# OpenShift Service Mesh 3.3: Sidecar and Ambient Mode Walkthrough

This guide covers the installation of Red Hat OpenShift Service Mesh 3.3 (via the Sail Operator), Kiali, and demonstrates both traditional Sidecar injection and the new Ambient mesh architecture with the bookinfo application from the Istio project.

## Installation

### Prerequisites

Install the **OpenShift Service Mesh 3.3 Operator** and the **Kiali Operator** via the OpenShift OperatorHub before proceeding. Additionally, we will need: 

- An OpenShift cluster with cluster-admin privileges
- The `oc` CLI installed and logged in
- The `istioctl` CLI (installation steps included below)

>*NOTE*: The scripts provided here have been executed in an OpenShift Web Terminal, available via the installation of the **Web Terminal** operator

### Install and Configure `istioctl`
 
Obtain the download URL from the OpenShift Console or the OSSM documentation, then:
 
```bash
# Download the AMD64 binary
curl -L -O <DOWNLOAD_URL>
 
# Extract the archive
tar xzvf <FILENAME>.tar.gz
 
# Add to PATH (Web Terminal, trasient state)
export PATH=$PATH:~/istioctl-linux-amd64
```

<details>
<summary>✅ Validation</summary>

```bash
istioctl version --short
```

Expected output:
```
client version: 1.28.5
```

</details>

### Kiali instance
 
Create a default Kiali instance from the OperatorHub-installed Kiali Operator in the `kiali` namespace.

Firstly, create the namespace. 

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: kiali
spec: {}
EOF
```

Secondly, create the Kiali instance, directly from the `Ecosystem / Installed` Operators menu option. Accept all the defaults. 

>**NOTE**: If you postpone Kiali configuration until after the application is deployed, you can observe exactly what each step does in real-time. Application monitoring will fail in various ways until all Kiali components and RBAC permissions are correctly configured.

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n kiali
```

Expected output:
```
NAME                    READY   STATUS    RESTARTS   AGE
kiali-xxxxx-xxxxx       1/1     Running   0          XXs
```

```bash
oc get kiali -n kiali
```

Expected output:
```
NAME    AGE
kiali   XXs
```

</details>

 
### Enable User Workload Monitoring
 
> **Reference:** https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.20/html/configuring_user_workload_monitoring/preparing-to-configure-the-monitoring-stack-uwm#configurable-monitoring-components_preparing-to-configure-the-monitoring-stack-uwm
 
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n openshift-user-workload-monitoring
```

Expected output (wait a minute for pods to come up):
```
NAME                                   READY   STATUS    RESTARTS   AGE
prometheus-user-workload-0             6/6     Running   0          XXs
prometheus-user-workload-1             6/6     Running   0          XXs
thanos-ruler-user-workload-0           4/4     Running   0          XXs
thanos-ruler-user-workload-1           4/4     Running   0          XXs
```

</details>

### Point Kiali to Thanos Querier
 
Patch the Kiali CR (`kiali` instance in the `kiali` namespace) to point at the in-cluster Thanos Querier endpoint:
 
```bash
oc patch kiali kiali -n kiali --type=merge -p '
spec:
  external_services:
    prometheus:
      url: "https://thanos-querier.openshift-monitoring.svc:9091"
      auth:
        type: "bearer"
        use_kiali_token: true
'
```

<details>
<summary>✅ Validation</summary>

```bash
oc get kiali kiali -n kiali -o jsonpath='{.spec.external_services.prometheus.url}'
```

Expected output:
```
https://thanos-querier.openshift-monitoring.svc:9091
```

</details>
 
### Fix 403 Error — Grant Kiali Monitoring Access
 
After updating the endpoint you may see a **403 Forbidden** error. Grant the Kiali service account the required cluster role:
 
```bash
oc adm policy add-cluster-role-to-user cluster-monitoring-view \
  -z kiali-service-account -n kiali
```

<details>
<summary>✅ Validation</summary>

```bash
oc adm policy who-can get pods --subresource=prometheus-metrics -n openshift-monitoring | grep kiali-service-account
```

Or verify with:
```bash
oc get clusterrolebinding -o wide | grep kiali-service-account
```

Expected output (should show the `cluster-monitoring-view` binding):
```
cluster-monitoring-view-xxxxx   ClusterRole/cluster-monitoring-view   ...   kiali/kiali-service-account
```

</details>

## A. Sidecar mode

> **Reference:** https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.3/html-single/installing/index#ossm-sidecar-injection

### A.1 Components installation 

#### A.1.1 Install Istio CNI plugin 

The CNI plugin handles network configuration for sidecar injection at the node level.

```bash
# Create istio-cni project
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: istio-cni
spec: {}
---
kind: IstioCNI
apiVersion: sailoperator.io/v1
metadata:
  name: default
spec:
  namespace: istio-cni
  version: v1.28.5
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get istiocni default -o jsonpath='{.status.state}'
```

Expected output:
```
Healthy
```

```bash
oc get pods -n istio-cni
```

Expected output (one pod per node):
```
NAME                   READY   STATUS    RESTARTS   AGE
istio-cni-node-xxxxx   1/1     Running   0          XXs
istio-cni-node-xxxxx   1/1     Running   0          XXs
...
```

</details>

#### A.1.2 Istio Control Plane

```bash
# Create the control plane namespace and 
#  the Istio resource with a discovery selector
#  (only namespaces labelled istio-discovery=enabled will be managed)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: istio-system
spec: {}
---
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  namespace: istio-system
  values:
    meshConfig:
      discoverySelectors:
        - matchLabels:
            istio-discovery: enabled
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get istio default -o jsonpath='{.status.state}'
```

Expected output:
```
Healthy
```

```bash
oc get pods -n istio-system
```

Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxx-xxxxx        1/1     Running   0          XXs
```

</details>

### A.2 Application deployment

In sidecar mode, an Envoy proxy container is injected alongside each application container in a pod. Watch the container count change after enabling injection.
 
#### A.2.1 Deploy the Bookinfo Application

```bash
# Create the application namespace
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: bookinfo
spec: {}
EOF

# Deploy the Bookinfo sample app (without sidecars yet)
oc apply -f https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n bookinfo
```

Expected output (1 container per pod, no sidecar yet):
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-xxxxx-xxxxx            1/1     Running   0          XXs
productpage-v1-xxxxx-xxxxx        1/1     Running   0          XXs
ratings-v1-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v1-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v2-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v3-xxxxx-xxxxx            1/1     Running   0          XXs
```

</details>

Optionally, expose the app directly to verify it works before mesh injection:

```bash
# Expose the productpage deployment
oc expose deployment/productpage -n bookinfo
 
# Create an edge-terminated route
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: productpage
    service: productpage
  name: productpage
  namespace: bookinfo
spec:
  port:
    targetPort: http
  tls:
    termination: edge
  to:
    kind: ""
    name: productpage
EOF
 
# Print the route URL
echo https://$(oc get -n bookinfo route productpage -ojsonpath='{.spec.host}')
```

You can also verify individual microservices from within the cluster (For instance, from the Web Terminal):

```bash
# Details service
curl -s http://details.bookinfo.svc:9080/details/0 | jq
 
# Reviews service
curl -s http://reviews.bookinfo.svc:9080/reviews/0 | jq
```

#### A.2.2 Enable Sidecar Injection
 
Label the namespace to enable both mesh discovery and automatic sidecar injection:
 
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: bookinfo
  labels:
    istio-discovery: enabled
    istio-injection: enabled
EOF
```

Restart all workloads so the sidecars are injected, then watch the pods come back up:
 
```bash
# Trigger a rolling restart
oc rollout restart deployments -n bookinfo
 
# Watch pods — you should see 2 containers per pod (app + istio-proxy)
oc get pod -n bookinfo -w
```

<details>
<summary>✅ Validation</summary>

```bash
oc get namespace bookinfo --show-labels | grep istio
```

Expected output (should contain both labels):
```
bookinfo   Active   XXm   istio-discovery=enabled,istio-injection=enabled,...
```

```bash
oc get pods -n bookinfo -o custom-columns="NAME:.metadata.name,READY:.status.containerStatuses[*].ready,CONTAINERS:.spec.containers[*].name"
```

Expected output (2 containers per pod — app + istio-proxy):
```
NAME                              READY        CONTAINERS
details-v1-xxxxx-xxxxx            true,true    details,istio-proxy
productpage-v1-xxxxx-xxxxx        true,true    productpage,istio-proxy
ratings-v1-xxxxx-xxxxx            true,true    ratings,istio-proxy
reviews-v1-xxxxx-xxxxx            true,true    reviews,istio-proxy
reviews-v2-xxxxx-xxxxx            true,true    reviews,istio-proxy
reviews-v3-xxxxx-xxxxx            true,true    reviews,istio-proxy
```

</details>
 
#### A.2.3 Configure Prometheus Scraping
 
Once the 403 is resolved, you may notice that no metrics appear. This is because Prometheus has no `PodMonitor` configured to scrape the Envoy sidecar metrics endpoint (`/stats/prometheus`).
 
Create a `PodMonitor` for all sidecar-injected pods in the `bookinfo` namespace:
 
```bash
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: bookinfo-proxies-monitor
  namespace: bookinfo
  labels:
    k8s-app: istio
spec:
  selector:
    matchExpressions:
    - key: security.istio.io/tlsMode
      operator: Exists
  podMetricsEndpoints:
  - path: /stats/prometheus
    port: http-envoy-prom
    interval: 15s
    relabelings:
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_container_name]
      regex: "istio-proxy"
    - action: replace
      sourceLabels: [__meta_kubernetes_pod_label_app]
      targetLabel: app
    - action: replace
      sourceLabels: [__meta_kubernetes_pod_label_version]
      targetLabel: version
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get podmonitor -n bookinfo
```

Expected output:
```
NAME                       AGE
bookinfo-proxies-monitor   XXs
```

```bash
oc get pods -n openshift-user-workload-monitoring -l app.kubernetes.io/name=prometheus -o jsonpath='{.items[0].status.phase}'
```

Expected output (Prometheus is running and will pick up the new PodMonitor):
```
Running
```

</details>
 
### A.3. Security and Networking
 
#### A.3.1 Enforce mTLS

Apply a `PeerAuthentication` policy to enforce mutual TLS (mTLS) for all workloads in the `bookinfo` namespace:
 
```bash
cat <<EOF | oc apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```
 
<details>
<summary>✅ Validation</summary>

```bash
oc get peerauthentication -n bookinfo
```

Expected output:
```
NAME      MODE     AGE
default   STRICT   XXs
```

Verify mTLS is enforced (the direct route should now fail):
```bash
curl -sk https://$(oc get route productpage -n bookinfo -o jsonpath='{.spec.host}')/productpage
```

Expected output:
```
(empty response or 502 Bad Gateway)
```

</details>

> **Note:** Once STRICT mTLS is active, the direct `productpage` route created earlier will return a **502 Bad Gateway**, because the OpenShift Router cannot complete the mTLS handshake. An Istio Ingress Gateway is required — see Section 3.2.
 
#### A.3.2 Deploy the Ingress Gateway
 
Deploy the Envoy-based ingress gateway workload (Service, Deployment, RBAC):
 
```bash
oc apply -f https://raw.githubusercontent.com/istio-ecosystem/sail-operator/main/chart/samples/ingress-gateway.yaml -n bookinfo
```
 
Create the Istio `Gateway` and `VirtualService` resources for Bookinfo:
 
```bash
oc apply -f https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/networking/bookinfo-gateway.yaml -n bookinfo
```
 
Expose the ingress gateway via an OpenShift Route:
 
```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: istio-ingressgateway
  namespace: bookinfo
spec:
  port:
    targetPort: http2
  tls:
    termination: edge
  to:
    kind: Service
    name: istio-ingressgateway
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n bookinfo -l istio=ingressgateway
```

Expected output:
```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-xxxxx-xxxxx        2/2     Running   0          XXs
```

```bash
oc get route istio-ingressgateway -n bookinfo -o jsonpath='{.spec.host}'
```

Expected output:
```
istio-ingressgateway-bookinfo.apps.<cluster-domain>
```

Test the application through the ingress gateway:
```bash
curl -sk https://$(oc get route istio-ingressgateway -n bookinfo -o jsonpath='{.spec.host}')/productpage | grep -o "<title>.*</title>"
```

Expected output:
```
<title>Simple Bookstore App</title>
```

</details>
 
#### A.3.3 Inspect Certificate / SPIFFE Identity
 
Use `istioctl` to inspect the certificate of any sidecar-injected pod:
 
```bash
# Set the pod name first
POD_TO_INSPECT=<pod-name>
 
istioctl proxy-config secret $POD_TO_INSPECT -n bookinfo -o json | \
  jq -r '.dynamicActiveSecrets[]? | select(.name=="default") | .secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | openssl x509 -text -noout
```

<details>
<summary>✅ Validation</summary>

The certificate output should contain a SPIFFE URI in the Subject Alternative Name:
```
X509v3 Subject Alternative Name: critical
    URI:spiffe://cluster.local/ns/bookinfo/sa/<service-account-name>
```

</details>
 
#### A.3.4 Test with a Sleep Pod
 
Deploy a curl-based pod with sidecar injection enabled to test connectivity from within the mesh:
 
```bash
oc run sleep \
  --image=curlimages/curl \
  --restart=Never \
  -n bookinfo \
  --labels="sidecar.istio.io/inject=true" \
  -- sleep 3600
```
 
Exec into the pod (e.g. via the OpenShift Web Terminal) and test access to the Reviews service:
 
```bash
curl -I -X GET reviews:9080/reviews/0
# Expected: HTTP 200 OK
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pod sleep -n bookinfo -o jsonpath='{.status.containerStatuses[*].name}'
```

Expected output (pod running with sidecar):
```
sleep istio-proxy
```

From inside the sleep pod:
```bash
oc exec sleep -n bookinfo -c sleep -- curl -s -o /dev/null -w "%{http_code}" reviews:9080/reviews/0
```

Expected output:
```
200
```

</details>
 
#### A.3.5 Authorization Policy
 
Restrict access to the `reviews` service so that only `productpage` (identified by its SPIFFE/mTLS identity) is permitted:
 
```bash
cat <<EOF | oc apply -f -
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: reviews-identity-enforcement
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: reviews
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]
EOF
```
 
After applying this policy, the `sleep` pod should receive a **403 Forbidden** when attempting to reach `reviews`, while `productpage` continues to work normally.

<details>
<summary>✅ Validation</summary>

```bash
oc get authorizationpolicy -n bookinfo
```

Expected output:
```
NAME                           AGE
reviews-identity-enforcement   XXs
```

Verify that the sleep pod is now denied access to reviews:
```bash
oc exec sleep -n bookinfo -c sleep -- curl -s -o /dev/null -w "%{http_code}" reviews:9080/reviews/0
```

Expected output:
```
403
```

Verify that productpage can still reach reviews (app still works end-to-end):
```bash
curl -sk https://$(oc get route istio-ingressgateway -n bookinfo -o jsonpath='{.spec.host}')/productpage | grep -o "<title>.*</title>"
```

Expected output:
```
<title>Simple Bookstore App</title>
```

</details>

### A.4. Cleanup

```
oc delete namespace bookinfo
oc delete istio default
oc delete istiocni default
oc delete namespace istio-system
oc delete namespace istio-cni
```

---

## B. Ambient mode

>*NOTE*: https://docs.redhat.com/en/documentation/red_hat_openshift_service_mesh/3.3/html-single/installing/index#ossm-istio-ambient-mode
We need to configure the cluster CNO. We need to make sure that the field Networks.operator.spec.defaultNetwork.ovnKubernetesConfig.gatewayConfig.routingViaHost is set to true. 

### B.1 Components installation

For scoping the service mesh with discovery selector to limit the scope of the OSSM in Istio ambient mode. The configuration controls which namespaces the control plane discovers based on label selectors. 

#### B.1.1. Install ZTunnel

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: istio-ztunnel
spec: {}
---
apiVersion: sailoperator.io/v1
kind: ZTunnel
metadata:
  name: default
  namespace: istio-ztunnel
  labels:
    istio-discovery: enabled
spec:
  namespace: istio-ztunnel
  profile: ambient
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get ztunnel default -n istio-ztunnel -o jsonpath='{.status.state}'
```

Expected output:
```
Healthy
```

```bash
oc get pods -n istio-ztunnel -l app=ztunnel
```

Expected output (one pod per node):
```
NAME              READY   STATUS    RESTARTS   AGE
ztunnel-xxxxx     1/1     Running   0          XXs
ztunnel-xxxxx     1/1     Running   0          XXs
...
```

</details>

#### B.1.2. Install Istio CNI plugin

We install the Istio CNI in ambient mode as well

```bash
# Create istio-cni project and IstioCNI
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: istio-cni
  labels:
    istio-discovery: enabled
spec: {}
---
apiVersion: sailoperator.io/v1
kind: IstioCNI
metadata:
  name: default
spec:
  namespace: istio-cni
  profile: ambient
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get istiocni default -o jsonpath='{.status.state}'
```

Expected output:
```
Healthy
```

```bash
oc get pods -n istio-cni -l k8s-app=istio-cni-node
```

Expected output (one pod per node):
```
NAME                   READY   STATUS    RESTARTS   AGE
istio-cni-node-xxxxx   1/1     Running   0          XXs
istio-cni-node-xxxxx   1/1     Running   0          XXs
...
```

</details>

#### B.1.3. Install Istio control plane

We install the istio resource in ambient mode: 

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: istio-system
  labels:
    istio-discovery: enabled
spec: {}
---
apiVersion: sailoperator.io/v1
kind: Istio
metadata:
  name: default
spec:
  namespace: istio-system
  profile: ambient
  values:
    pilot:
      trustedZtunnelNamespace: istio-ztunnel
  meshConfig:
    discoverySelectors:
    - matchLabels:
        istio-discovery: enabled
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get istio default -o jsonpath='{.status.state}'
```

Expected output:
```
Healthy
```

```bash
oc get pods -n istio-system
```

Expected output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-xxxxx-xxxxx        1/1     Running   0          XXs
```

</details>

## B.2 Deploy the bookinfo app

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:  
  name: bookinfo
  labels:
    istio-discovery: enabled
    istio.io/dataplane-mode: ambient
spec: {}
EOF
oc apply -n bookinfo -f https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo.yaml
oc apply -n bookinfo -f https://raw.githubusercontent.com/openshift-service-mesh/istio/release-1.24/samples/bookinfo/platform/kube/bookinfo-versions.yaml
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n bookinfo
```

Expected output (1 container per pod — no sidecar in ambient mode):
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-xxxxx-xxxxx            1/1     Running   0          XXs
productpage-v1-xxxxx-xxxxx        1/1     Running   0          XXs
ratings-v1-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v1-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v2-xxxxx-xxxxx            1/1     Running   0          XXs
reviews-v3-xxxxx-xxxxx            1/1     Running   0          XXs
```

```bash
oc get namespace bookinfo --show-labels | grep ambient
```

Expected output (should contain the ambient label):
```
bookinfo   Active   XXs   istio-discovery=enabled,istio.io/dataplane-mode=ambient,...
```

</details>

Confirm that Ztunnel proxy has successfully opened listening sockets in the pod network namespace by running the following command:

```bash
istioctl ztunnel-config workloads --namespace istio-ztunnel
```

<details>
<summary>✅ Validation</summary>

The output should list all bookinfo workloads with their IP addresses and `PROTOCOL: HBONE`:
```
NAMESPACE   POD NAME                          IP           NODE        WAYPOINT   PROTOCOL
bookinfo    details-v1-xxxxx-xxxxx            10.x.x.x    worker-0               HBONE
bookinfo    productpage-v1-xxxxx-xxxxx        10.x.x.x    worker-0               HBONE
bookinfo    ratings-v1-xxxxx-xxxxx            10.x.x.x    worker-1               HBONE
bookinfo    reviews-v1-xxxxx-xxxxx            10.x.x.x    worker-1               HBONE
...
```

</details>

Install the Gateway

```bash
cat <<EOF | oc apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  labels:
    istio.io/waypoint-for: service
  name: waypoint
  namespace: bookinfo
spec:
  gatewayClassName: istio-waypoint
  listeners:
  - name: mesh
    port: 15008
    protocol: HBONE
EOF
```

Enroll the bookinfo namespace to use the waypoint

```bash
oc label namespace bookinfo istio.io/use-waypoint=waypoint
```

<details>
<summary>✅ Validation</summary>

```bash
oc get gateway waypoint -n bookinfo -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
```

Expected output:
```
True
```

```bash
oc get pods -n bookinfo -l gateway.networking.k8s.io/gateway-name=waypoint
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
waypoint-xxxxx-xxxxx        1/1     Running   0          XXs
```

Check enrollment:
```bash
istioctl ztunnel-config svc --namespace istio-ztunnel
```

Expected output (services should show the waypoint address):
```
NAMESPACE   SERVICE NAME   SERVICE VIP   WAYPOINT          PROTOCOL
bookinfo    details        10.x.x.x      10.x.x.x:15008   HBONE
bookinfo    productpage    10.x.x.x      10.x.x.x:15008   HBONE
bookinfo    ratings        10.x.x.x      10.x.x.x:15008   HBONE
bookinfo    reviews        10.x.x.x      10.x.x.x:15008   HBONE
...
```

</details>

For scrapping the metrics 

```bash
cat <<EOF | oc apply -f - 
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: ztunnel-monitor
  namespace: istio-ztunnel
spec:
  selector:
    matchLabels:
      app: ztunnel
  podMetricsEndpoints:
  - port: ztunnel-stats
    path: /metrics
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: waypoint-monitor
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      gateway.networking.k8s.io/gateway-name: waypoint
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 15s
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get podmonitor -n istio-ztunnel
```

Expected output:
```
NAME               AGE
ztunnel-monitor    XXs
```

```bash
oc get podmonitor -n bookinfo
```

Expected output:
```
NAME                AGE
waypoint-monitor    XXs
```

</details>

### B.3. Security and Networking

We can expose the app via a Gateway (k8s): 

```bash
# Create the Gateway
cat <<EOF | oc apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bookinfo-ingress-gateway
  namespace: bookinfo
  annotations:
    networking.istio.io/service-type: ClusterIP
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get gateway bookinfo-ingress-gateway -n bookinfo -o jsonpath='{.status.conditions[?(@.type=="Programmed")].status}'
```

Expected output:
```
True
```

```bash
oc get pods -n bookinfo -l gateway.networking.k8s.io/gateway-name=bookinfo-ingress-gateway
```

Expected output:
```
NAME                                              READY   STATUS    RESTARTS   AGE
bookinfo-ingress-gateway-istio-xxxxx-xxxxx        1/1     Running   0          XXs
```

</details>

After the gateway is deployed, we will see the envoy proxy that has been spinned up by the previous CRD (gatewayClassName istio). We can now create an HTTPRoute object that will inject the traffic into the mesh

```bash
cat <<EOF | oc apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookinfo-entry-route
  namespace: bookinfo
spec:
  parentRefs:
  - name: bookinfo-ingress-gateway
  rules:
  - matches:
    - path:
        type: Exact
        value: /productpage
    - path:
        type: PathPrefix
        value: /static
    - path:
        type: Exact
        value: /login
    - path:
        type: Exact
        value: /logout
    - path:
        type: PathPrefix
        value: /api/v1/products
    backendRefs:
    - name: productpage
      port: 9080
EOF
```

Now, we can create a route to the gateway. 

```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: main
  namespace: bookinfo
spec:
  to:
    kind: Service
    name: bookinfo-ingress-gateway-istio
    weight: 100
  port:
    targetPort: 80 # The port defined in your Gateway listener
  tls:
    termination: edge # Optional: Let the OpenShift Router handle external TLS
    insecureEdgeTerminationPolicy: Redirect
EOF
```

<details>
<summary>✅ Validation</summary>

```bash
oc get httproute -n bookinfo
```

Expected output:
```
NAME                   HOSTNAMES   AGE
bookinfo-entry-route               XXs
```

```bash
oc get route main -n bookinfo -o jsonpath='{.spec.host}'
```

Expected output:
```
main-bookinfo.apps.<cluster-domain>
```

Test the application through the route:
```bash
curl -sk https://$(oc get route main -n bookinfo -o jsonpath='{.spec.host}')/productpage | grep -o "<title>.*</title>"
```

Expected output:
```
<title>Simple Bookstore App</title>
```

</details>

We check that we can reach the pod from outside the mesh (Web Terminal, for instance) 

```bash
# It reteurns the info about the author (William Shakespeare) and the publication
curl http://details.bookinfo.svc:9080/details/0 
```

At this point, if we didn't restart the pods, the new ip/nftables are not handled by the Ambient mode, so we won't see anything in the Kiali Graph view. If we perform an application restart and generate some traffic, we will able to see it. 

Restart all workloads so the sidecars are injected, then watch the pods come back up:
 
```bash
# Trigger a rolling restart
oc rollout restart deployments -n bookinfo
 
# Watch pods — you should see 2 containers per pod (app + istio-proxy)
oc get pod -n bookinfo -w
```

<details>
<summary>✅ Validation</summary>

```bash
oc get pods -n bookinfo -l app=productpage -o jsonpath='{.items[0].status.phase}'
```

Expected output:
```
Running
```

Generate some traffic and verify Kiali can see the graph:
```bash
for i in $(seq 1 5); do curl -sk https://$(oc get route main -n bookinfo -o jsonpath='{.spec.host}')/productpage > /dev/null; done
```

</details>

We apply the PeerAuthentication CRD to enable mTLS at namespace level

```bash
cat <<EOF | oc apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

Now a cURL from the web terminal will not succeed. 

```bash
# Now, we get an error
curl http://details.bookinfo.svc:9080/details/0 
```

We can explore, then, the gateway logs: 

```bash
# Will show the blocked request after applying the PeerAuthorization CRD at namespace level.  
oc logs -n istio-ztunnel -l app=ztunnel -c istio-proxy --tail=100 | grep details
```

<details>
<summary>✅ Validation</summary>

```bash
oc get peerauthentication -n bookinfo
```

Expected output:
```
NAME      MODE     AGE
default   STRICT   XXs
```

Verify mTLS enforcement (plain HTTP from outside the mesh should fail):
```bash
oc exec -n openshift-console deployment/console -c console -- curl -s -o /dev/null -w "%{http_code}" http://details.bookinfo.svc:9080/details/0
```

Expected output (connection refused or reset):
```
000
```

Check ztunnel logs for denied connections:
```bash
oc logs -n istio-ztunnel -l app=ztunnel -c istio-proxy --tail=20 | grep -i "denied\|RBAC\|details"
```

Expected output (should show denied/blocked entries):
```
... inbound connection from ... denied ...
```

</details>

### B.4. Cleanup

```
oc delete istio default
oc delete istiocni default
oc delete ztunnel default
oc delete namespace bookinfo
oc delete namespace istio-system
oc delete namespace istio-cni
oc delete namespace istio-ztunnel
```
