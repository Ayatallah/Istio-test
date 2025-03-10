# **Istio Test**

This repository is dedicated to exploring Istio’s features in a Kubernetes environment.

---

## **Prerequisites**

### 🔹 Local K3d Cluster

Create a local **K3d cluster** with the following configuration:

```sh
k3d cluster create istiotest --api-port 6500 -p "8085:80@loadbalancer" --agents 2
```

### 🔹 Bootstrap FluxCD

To manage Istio configurations with **FluxCD**, bootstrap it with:

```sh
flux bootstrap github --branch=main --path=/flux --owner Ayatallah --repository Istio-test --token-auth
```

---

## **Istio Installation**

Istio supports multiple [installation methods](https://istio.io/latest/docs/setup/install/).  

In the [getting started guide](https://istio.io/latest/docs/setup/getting-started/#download), Istio is installed via `istioctl` as shown below:

```sh
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.25.0

export PATH=$PWD/bin:$PATH

cat <<EOF > ./operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: minimal
EOF

istioctl install -f operator.yaml
```
This method enables Istio components to be managed by the Istio Operator, which simplifies lifecycle management and automates tasks, making testing easier.

For testing in this repository, Istio components—specifically **`istio-base` and `istiod`**—are installed via Helm under `/istio/system`. This approach allows for direct installation of components without the Istio Operator, offering more fine-grained control. The [Deployment Profiles Documentation](https://istio.io/latest/docs/setup/additional-setup/config-profiles/#deployment-profiles) clarifies the components included in each profile.

In both installation methods—IstioOperator and Helm—we must ensure Kubernetes Gateway API CRDs are installed. In the [getting started guide](https://istio.io/latest/docs/setup/getting-started/#download), this is accomplished as follows:

```sh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.2.1" | kubectl apply -f -; }
```

For this repository, these CRDs are added under `/istio/crds`.

For **production environments**, the most suitable installation method for Istio components and CRD deployment should be carefully chosen, adhering to best practices for CRD management.

### **Helm Installation References**
- [Istio Helm Installation](https://istio.io/latest/docs/setup/install/helm/)
- [Istio Base Helm Chart](https://artifacthub.io/packages/helm/istio-official/base)
- [Istiod Helm Chart](https://artifacthub.io/packages/helm/istio-official/istiod)
- [Installing Istio with Flux](https://trstringer.com/install-istio-flux/)
- [Helm CRD Best Practices](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#some-caveats-and-explanations)
- [Kubernetes Gateway API Discussions](https://github.com/kubernetes-sigs/gateway-api/discussions/2655)
- [FluxCD Discussions](https://github.com/fluxcd/flux2/discussions/4457)

---

## **Features Explored**

### **1️⃣ gRPC Load Balancing**

This demo is based on a **Hello World gRPC application** consisting of a gRPC server and client.  
You can use the [gRPC quickstart guide](https://grpc.io/docs/languages/go/quickstart/) for any language and **dockerize** it, as demonstrated in [this repository](https://github.com/anvilco/load-balance-grpc-k8s).

🔹 Instead of building the application from scratch, a **pre-built, dockerized version** is used for testing:

```sh
docker pull ghcr.io/anvilco/load-balance-grpc-k8s:latest
```

#### **🔹 Import the Image into the Local K3d Cluster**
```sh
k3d image import ghcr.io/anvilco/load-balance-grpc-k8s:latest -c istiotest
```

#### **🔹 Deploy the gRPC App**
The gRPC application resources are deployed in `demos/grpc-loadbalancing` **without namespace labeling**:

```sh
kubectl exec -it greeter-client-6b8cdf6989-wcgqv -n grpcdemo -- sh
node greeter_client
```

📌 **Initial Observation:**  
All responses were handled by a **single** gRPC server pod, meaning no load balancing was occurring.

#### **🔹 Enable Istio Sidecar Injection**
To enable **Envoy proxy-based load balancing**, label the namespace as defined in `demos/grpc-loadbalancing/namespace.yaml`.

Then, **restart the gRPC pods** to trigger sidecar injection:

```sh
kubectl delete pods -n grpcdemo --all
```

Once the new pods start with two containers—**Envoy proxy and gRPC application**—repeat the same test:

```sh
kubectl exec -it greeter-client-6b8cdf6989-wcgqv -n grpcdemo -- sh
node greeter_client
```

👉 **Final Observation:**  
Load is now **evenly distributed** across all server pods, demonstrating gRPC load balancing via Istio.

#### **References**
- [Load Balancing gRPC in Kubernetes with Istio](https://www.useanvil.com/blog/engineering/load-balancing-grpc-in-kubernetes-with-istio/)
- [AnvilCo Load Balancing Example](https://github.com/anvilco/load-balance-grpc-k8s)

---

### **2️⃣ Traffic Management**

This demo is based on a simple **Bookinfo application** from the [getting started guide](https://istio.io/latest/docs/setup/getting-started/).

The application resources are configured in `demos/traffic-mngmt/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: bookinfo
resources:
  - namespace.yaml
  - resources.yaml
  - bookinfo-gateway.yaml
```

The namespace is labeled for **sidecar injection** in `demos/traffic-mngmt/namespace.yaml`.

Port-forward to access the gateway:
```sh
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
```

Now navigate to `http://localhost:8080/productpage`. As you refresh the page, book reviews and ratings should change due to requests being distributed across different versions of the **reviews** service.

**DestinationRule** added in `demos/traffic-mngmt/destination-rules.yaml` is then created to inform Istio that three distinct subsets of the reviews service exist using version label as the discriminator.

**VirtualService** added in `demos/traffic-mngmt/traffic-shift-v1.yaml` shifts all incoming traffic to reviews service **v1 only**. Refreshing the page will always show v1.

**VirtualService** added in `demos/traffic-mngmt/traffic-shift-v1-v3.yaml` shifts **50%** of incoming traffic to **v3**. Now, you will see red-colored star ratings approximately **50%** of the time.

This is just the most basic traffic management demo. There's much more to explore! We can shift traffic for **A/B testing** or **canary deployments**, route specific requests based on explicit users or paths, and simulate faulty services for **chaos testing**.

#### **References**
- [Istio Traffic Management Overview](https://istio.io/latest/docs/concepts/traffic-management/)
- [DestinationRule Documentation](https://istio.io/latest/docs/reference/config/networking/destination-rule/#DestinationRule)
- [VirtualService Documentation](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
- [Traffic Shifting](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)
- [Request Routing](https://istio.io/latest/docs/tasks/traffic-management/request-routing/)
- [Fault Injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)
- [Comprehensive Istio Lab Guide](https://dev.to/sre_panchanan/mastering-traffic-management-a-comprehensive-istio-lab-guide-245l)

---

### **3️⃣ Mutual TLS (mTLS)**

This demo continues using Bookinfo application.

Istio by default runs mLTS in the permissive mode, which allows a service to accept both plaintext traffic and mutual TLS traffic at the same time.

Curl any of the bookinfo application services:

```
kubectl get svc -n bookinfo # to get all services, pick productpage for example

kubectl run -it --rm curl --image=radial/busyboxplus:curl --restart=Never -- sh

curl 10.43.67.210:9080/api/v1/products
[{"id": 0, "title": "The Comedy of Errors", "descriptionHtml": "<a href=\"https://en.wikipedia.org/wiki/The_Comedy_of_Errors\">Wikipedia Summary</a>: The Comedy of Errors is one of <b>William Shakespeare's</b> early plays. It is his shortest and one of his most farcical comedies, with a major part of the humour coming from slapstick and mistaken identity, in addition to puns and word play."}]
```
📌 **Initial Observation:**
Response received successfully. :white_check_mark:

Then, update **mLTS mode** to **STRICT** as in `demos/mlts/peer-authentication.yaml` & try again:

```
kubectl run -it --rm curl --image=radial/busyboxplus:curl --restart=Never -- sh

root@curl:/ ]$ curl 10.43.67.210:9080/api/v1/products
curl: (56) Recv failure: Connection reset by peer
```
👉 **Final Observation:**  
Failure Received. Because, as mLTS mode is STRICT, istio side-car container require a certificate, and since this curl does not have a certificate, the request is rejected.

**mLTS mode** can be set for specific workload using `spec.selector.matchLabels` or for specific port using `spec.portLevelMtls` as shown in [docs](https://istio.io/latest/docs/reference/config/security/peer_authentication/).

Istio has its own Certificate Authority (Citadel or Istiod) for key and certificate management, which issues certificates. However, Istio integrates with cert-manager and can use its certificates for mLTS secure connections.

#### **References**
- [Mutual TLS Authentication](https://istio.io/latest/docs/concepts/security/#mutual-tls-authentication)
- [mLTS Migration](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
- [Istio Authentication Architecture](https://istio.io/latest/docs/concepts/security/#authentication-architecture)
- [Peer Authentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/)
- [Istio Security Peer Authentication & mLTS](https://harsh05.medium.com/understanding-istio-security-peer-authentication-and-mtls-for-microservices-d1fd1ef60d55)
- [Istio Cert Manager Integration](https://istio.io/latest/docs/ops/integrations/certmanager/)
- [Istio In Practice Cert Manager Integration](https://docs.tetrate.io/istio-subscription/istio-in-practice/cert-manager-integration)

### **4️⃣ Circuit Breaking**

This demo uses Httpbin as running istio service, and the steps are straight forward as shown in [docs](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)

Resources needed are all added in `demos/circuit-breaking` & then running commands below:

```
kubectl exec fortio-deploy-pod-id -c fortio -n httpbin -- /usr/bin/fortio curl -quiet http://httpbin:8000/get

kubectl exec fortio-deploy-pod-id -c fortio -n httpbin -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get

kubectl exec fortio-deploy-pod-id -c fortio -n httpbin -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning  http://httpbin:8000/get
```

& see how `istio-proxy` open the circuits for further requests and connections when failures occur whether due to 5xx errors returned due to faulty/missing heathy pods or the too strict trafficPolicy resulting in rejected requests/failures.

If **mLTS** is enabled, the DestinationRule must include the following trafficPolicy:
```
trafficPolicy:
  tls:
    mode: ISTIO_MUTUAL
```
As shown [here](https://istio.io/latest/docs/ops/common-problems/network-issues/#503-errors-after-setting-destination-rule)

#### **References**
- [Circuit Breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)
- [Istio Advanced Circuit Breaking & Chaos Engineering](https://ibrahimhkoyuncu.medium.com/istio-powered-resilience-advanced-circuit-breaking-and-chaos-engineering-for-microservices-c3aefcb8d9a9)
