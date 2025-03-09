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
Based on experience, installing Istio using `istioctl` is recommended, as shown in the [getting started guide](https://istio.io/latest/docs/setup/getting-started/#download):

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

This method is equivalent to installing **only `istio-base` and `istiod` via Helm**, as done in this repository under `/istio/system`.  
The [Deployment Profiles Documentation](https://istio.io/latest/docs/setup/additional-setup/config-profiles/#deployment-profiles) clarifies the components included in each profile.

### **References**
- [Istio Helm Installation](https://istio.io/latest/docs/setup/install/helm/)
- [Istio Base Helm Chart](https://artifacthub.io/packages/helm/istio-official/base)
- [Istiod Helm Chart](https://artifacthub.io/packages/helm/istio-official/istiod)
- [Installing Istio with Flux](https://trstringer.com/install-istio-flux/)

---

## **Features Explored**

### **1️⃣ gRPC Load Balancing**
This demo is based on a **Hello World gRPC application** consisting of a gRPC server and client.  
You can use the [gRPC quickstart guide](https://grpc.io/docs/languages/go/quickstart/) for any language and **dockerize** it, as demonstrated in [this repository](https://github.com/anvilco/load-balance-grpc-k8s).

Instead of building the application from scratch, I used a **pre-built, dockerized version** for testing:

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
To enable **Envoy proxy-based load balancing**, label the namespace as defined in `demos/grpc-loadbalancing/namespace.yaml`:

```sh
kubectl label namespace grpcdemo istio-injection=enabled
```

Then, **restart the gRPC pods** to trigger sidecar injection:

```sh
kubectl delete pods -n grpcdemo --all
```

Once the new pods start **with both the Envoy proxy and gRPC application**, running the same test:

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

### **2️⃣ Circuit Breaking (To Be Added)**
🛠️ **Coming Soon...**
