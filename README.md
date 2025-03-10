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

In [getting started guide](https://istio.io/latest/docs/setup/getting-started/#download), Istio is installed via `istioctl` as shown below:

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
This way allows Istio components to be managed by the Istio Operator, which provides simplified lifecycle management with built-in automation that makes testing easier.

For testing in this repo, Istio components **only `istio-base` and `istiod`** are installed via helm under `/istio/system`. This allow directly installing components without Operator installing/managing them providing more fine-grained control. The [Deployment Profiles Documentation](https://istio.io/latest/docs/setup/additional-setup/config-profiles/#deployment-profiles) clarifies the components included in each profile.

In both methods, IstioOperator or Helm, we will need to ensure Kubernetes Gateway API CRDs are installed. In [getting started guide](https://istio.io/latest/docs/setup/getting-started/#download), this is ensured as below:
```
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.2.1" | kubectl apply -f -; }
```
In this repo, these CRDs are added under `/istio/crds`

For production environment, the suitable installation method for Istio components and CRDs deployment to be chosen carefully, taking into consideration best practices regarding CRDs management.

### **Helm Installation References**
- [Istio Helm Installation](https://istio.io/latest/docs/setup/install/helm/)
- [Istio Base Helm Chart](https://artifacthub.io/packages/helm/istio-official/base)
- [Istiod Helm Chart](https://artifacthub.io/packages/helm/istio-official/istiod)
- [Installing Istio with Flux](https://trstringer.com/install-istio-flux/)
https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#some-caveats-and-explanations https://github.com/kubernetes-sigs/gateway-api/discussions/2655
https://github.com/fluxcd/flux2/discussions/4457


---

## **Features Explored**

### **1️⃣ gRPC Load Balancing**
This demo is based on a **Hello World gRPC application** consisting of a gRPC server and client.  
You can use the [gRPC quickstart guide](https://grpc.io/docs/languages/go/quickstart/) for any language and **dockerize** it, as demonstrated in [this repository](https://github.com/anvilco/load-balance-grpc-k8s).

🔹 Instead of building the application from scratch, I used a **pre-built, dockerized version** for testing:

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

Once the new pods start with two containers **Envoy proxy and gRPC application**, repeat the same test:

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
