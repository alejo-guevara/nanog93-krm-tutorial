### **Lab Guide: Evolving Traffic Generator Deployment with Kubernetes**

This lab demonstrates the evolution of deploying a traffic generator app (`iperf3`) in Kubernetes from manual manifests to Helm Charts, and finally to CRD-based declarative management using a custom controller.

## Infrastructure setup

Following are the steps to setup the environment.

Get into the workspace folder for this part of the lab

```shell
# change into Part 1 directory
cd /workspaces/nanog93-krm-tutorial/part1
```

Build iperf docker images
```shell
docker build -t iperf3-client:0.1a -f iperf-images/Dockerfile.iperf3client .
docker build -t iperf3-server:0.1a -f iperf-images/Dockerfile.iperf3server .
```

### Load pre-cached container images

```shell
# import the locally cached sr-linux container image
docker image load -i /var/cache/srlinux.tar
```

---

### **Setup Kubernetes Kind App**

#### **What is Kind?**
**Kind** is a opensource tool for running local Kubernetes clusters using Docker container "nodes." It’s great for learning, testing, or developing on Kubernetes.

#### **Kind Installation**

1. Import the locally cached kind node container image
    ```shell
    docker image load -i /var/cache/kindest-node.tar
    ```

2. Create docker network for kind and set iptable rule
    ```shell
    # pre-creating the kind docker bridge. This is to avoid an issue with kind running in codespaces. 
    docker network create -d=bridge \
      -o com.docker.network.bridge.enable_ip_masquerade=true \
      -o com.docker.network.driver.mtu=1500 \
      --subnet fc00:f853:ccd:e793::/64 kind
    
    # Allow the kind cluster to communicate with the later created containerlab topology
    sudo iptables -I DOCKER-USER -o br-$(docker network inspect -f '{{ printf "%.12s" .ID }}' kind) -j ACCEPT
    ```


3. **Start ContainerLab Topology**  
   - Create two Kubernetes clusters:
      ```shell
      # deploy Nokia SRL containers via containerlab
      cd clab-topology
      sudo containerlab deploy
      cd ..
      ```

4. **Kubernestes Contexts**
  - You should be able to see both contexts, and '*' showing the current one.
    ```
    sudo kubectl config get-contexts
    CURRENT   NAME         CLUSTER      AUTHINFO     NAMESPACE
              kind-k8s01   kind-k8s01   kind-k8s01   
    *         kind-k8s02   kind-k8s02   kind-k8s02  
    ```

4. **Preload the iperf3 Docker Images to Kind Kubernetes**  
   - Load the iperf3 image into both clusters:
     ```bash
     kind load docker-image iperf3-client:0.1a --name k8s01
     kind load docker-image iperf3-server:0.1a --name k8s02
     ```


5. **Set srlinux dev1 basic configuration**
  - Get into dev1 and set a basic configuration
    ```shell
    docker exec -ti dev1 sr_cli
    ```
  - You should see something like this inside the srlinux interface:
    ```
      ❯ docker exec -ti dev1 sr_cli
      Using configuration file(s): []
      Welcome to the srlinux CLI.
      Type 'help' (and press <ENTER>) if you need any help using this.
      --{ + running }--[  ]--
      A:dev1#
    ```

  - Now, paste the following:
    ```
    enter candidate
        network-instance default {
          interface ethernet-1/10.0 {
          }
          interface ethernet-1/11.0 {
          }
      }
      interface ethernet-1/10 {
          admin-state enable
          subinterface 0 {
              admin-state enable
              ipv4 {
                  admin-state enable
                  address 172.254.101.1/24 {
                  }
              }
          }
      }
      interface ethernet-1/11 {
          admin-state enable
          subinterface 0 {
              admin-state enable
              ipv4 {
                  admin-state enable
                  address 172.254.102.1/24 {
                  }
              }
          }
    commit now
    quit      
    ```
  - dev1 should be configured and ready to communicate the iperf instances

---

### **Step 1: Create Traffic Generator Manually with Manifests**

#### **1A: Deploy Server Pods on `kind-k8s02`**
1. Use `iperf3-server.yaml` in folder `manifests` with this content:
   ```yaml
    # Pod definition for iperf3-server
    apiVersion: v1
    kind: Pod
    metadata:
      name: iperf3-server
      labels:
        app: iperf3-server
    spec:
      containers:
      - name: iperf3-server
        image: iperf3-server:0.1a
        ports:
        - containerPort: 5201
    
    ---
    
    # Service definition for iperf3-server
    apiVersion: v1
    kind: Service
    metadata:
      name: iperf3-server-service
    spec:
      type: NodePort
      selector:
        app: iperf3-server
      ports:
      - protocol: TCP
        port: 5201
        targetPort: 5201
        nodePort: 30001
   ```
2. Apply the manifest:
   ```bash
   sudo kubectl apply -f iperf3-server.yaml --context kind-k8s02
   ```

3. Uses `sudo kubectl get pods  --context kind-k8s02` to check if the pod is running    

#### **1B: Deploy Client Pods on `kind-k8s01`**
1. Uses  `iperf3-client.yaml` now:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: iperf3-client
   spec:
     containers:
     - name: iperf3-client
       image: iperf3-client:0.1a
       args: ["-c", "172.254.102.101", "-p", "30001"]
   ```
2. Apply the manifest:
   ```bash
   sudo kubectl apply -f iperf3-client.yaml --context kind-k8s01
   ```

#### **Verify Traffic**
- Verify that traffic is flowing:
  ```bash
  sudo kubectl logs iperf3-client --context kind-kind-k8s01
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
