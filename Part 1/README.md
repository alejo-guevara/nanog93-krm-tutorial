### **Lab Guide: Evolving Traffic Generator Deployment with Kubernetes**

This lab demonstrates the evolution of deploying a traffic generator app (`iperf3`) in Kubernetes from manual manifests to Helm Charts, and finally to CRD-based declarative management using a custom controller.

---

### **Step 0: Setup Kubernetes Clusters**

1. **Set Up Two Kind Clusters**  
   - Create two Kubernetes clusters:
     ```bash
     kind create cluster --name kind-k8s1
     kind create cluster --name kind-k8s2
     ```

2. **Preload the iperf3 Docker Images**  
   - Load the iperf3 image into both clusters:
     ```bash
     kind load image-archive iperf3.tar --name kind-k8s1
     kind load image-archive iperf3.tar --name kind-k8s2
     ```

3. **Verify Cluster Access**  
   - Check cluster connectivity:
     ```bash
     kubectl cluster-info --context kind-kind-k8s1
     kubectl cluster-info --context kind-kind-k8s2
     ```

---

### **Step 1: Create Traffic Generator Manually with Manifests**

#### **1A: Deploy Server Pods on `kind-k8s2`**
1. Create `iperf3-server.yaml` with this content:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: iperf3-server
   spec:
     containers:
     - name: iperf3-server
       image: iperf3-server:0.1a
       ports:
       - containerPort: 5201
   ```
2. Apply the manifest:
   ```bash
   kubectl apply -f iperf3-server.yaml --context kind-kind-k8s2
   ```

#### **1B: Deploy Client Pods on `kind-k8s1`**
1. Create `iperf3-client.yaml`:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: iperf3-client
   spec:
     containers:
     - name: iperf3-client
       image: iperf3-client:0.1a
       args: ["-c", "172.254.102.101", "-p", "5201"]
   ```
2. Apply the manifest:
   ```bash
   kubectl apply -f iperf3-client.yaml --context kind-kind-k8s1
   ```

#### **Verify Traffic**
- Verify that traffic is flowing:
  ```bash
  kubectl logs iperf3-client --context kind-kind-k8s1
  ```

---

### **Step 2: Transition to Helm Charts**

#### **2A: Create Helm Chart for iperf3**

1. **Chart Directory Structure**
   ```
   iperf3-chart/
   ├── Chart.yaml
   ├── values.yaml
   ├── templates/
       ├── deployment-server.yaml
       ├── deployment-client.yaml
   ```

2. Define `values.yaml`:
   ```yaml
   server:
     enabled: true
     replicas: 10

   client:
     enabled: true
     replicas: 10
   ```

3. Apply the chart:
   ```bash
   helm install iperf3-chart ./iperf3-chart --context kind-kind-k8s1
   ```

#### **2B: Scale Instances**
1. Update `values.yaml` for 40 instances:
   ```yaml
   server:
     enabled: true
     replicas: 40

   client:
     enabled: true
     replicas: 40
   ```
2. Upgrade Helm release:
   ```bash
   helm upgrade iperf3-chart ./iperf3-chart --context kind-kind-k8s1
   ```

#### **Verify Deployment**
- Check that all pods are running:
  ```bash
  kubectl get pods --context kind-kind-k8s1
  kubectl get pods --context kind-kind-k8s2
  ```

---

### **Step 3: Transition to CRDs and Controller**

#### **3A: Create a Custom Resource Definition**
1. Define a `TrafficGenerator` CRD:
   ```yaml
   apiVersion: apiextensions.k8s.io/v1
   kind: CustomResourceDefinition
   metadata:
     name: trafficgenerators.example.com
   spec:
     group: example.com
     names:
       kind: TrafficGenerator
       listKind: TrafficGeneratorList
       plural: trafficgenerators
       singular: trafficgenerator
     scope: Namespaced
     versions:
     - name: v1
       served: true
       storage: true
       schema:
         openAPIV3Schema:
           type: object
           properties:
             spec:
               type: object
               properties:
                 replicas:
                   type: integer
                 bufferSize:
                   type: string
                 bandwidth:
                   type: string
   ```

2. Apply the CRD:
   ```bash
   kubectl apply -f trafficgenerator-crd.yaml
   ```

---

#### **3B: Create a Controller**

1. Implement a controller to watch `TrafficGenerator` resources.
2. Use the controller to:
   - Generate server and client deployments dynamically.
   - Automatically scale deployments based on `spec.replicas`.

---

#### **3C: Define a TrafficGenerator Resource**
1. Create a `TrafficGenerator` instance:
   ```yaml
   apiVersion: example.com/v1
   kind: TrafficGenerator
   metadata:
     name: iperf3-gen
   spec:
     replicas: 40
     bufferSize: "4K"
     bandwidth: "40K"
   ```

2. Apply the resource:
   ```bash
   kubectl apply -f trafficgenerator.yaml
   ```

---

### **Step 4: Monitor and Verify**

#### **Monitor TrafficGenerator Status**
- Check the CRD's status for active replicas:
  ```bash
  kubectl get trafficgenerator iperf3-gen -o yaml
  ```

#### **Verify Traffic**
- Ensure all pods are running and traffic flows:
  ```bash
  kubectl get pods
  kubectl logs iperf3-client
  ```

---

### **Summary**

- **Step 1**: Manual manifests to deploy server and client.  
- **Step 2**: Helm Charts for modularity and scalability.  
- **Step 3**: CRD with a custom controller for full declarative automation.  
- **Outcome**: Simplified management, scalability, and better monitoring using Kubernetes-native tools.
