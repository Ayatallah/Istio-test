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
🚧 **To be added**

### **4️⃣ Circuit Breaking**
🚧 **To be added**
