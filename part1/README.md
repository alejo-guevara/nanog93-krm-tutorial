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

3. Uses `sudo kubectl get pods  --context kind-k8s02` to check if the pod is running and `sudo kubectl get svc --context kind-k8s02` for the service.    

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
  sudo kubectl logs iperf3-client --context kind-k8s01
  ```
- Output should be something like this:
  ```
  Connecting to host 172.254.102.101, port 30001
  [  5] local 10.244.0.6 port 59900 connected to 172.254.102.101 port 30001
  [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
  [  5]   0.00-1.00   sec  1.25 MBytes  10.5 Mbits/sec    2   1.41 KBytes       
  [  5]   1.00-2.00   sec   768 KBytes  6.29 Mbits/sec   36   1.41 KBytes       
  [  5]   2.00-3.00   sec  0.00 Bytes  0.00 bits/sec    1   1.41 KBytes       
  [  5]   3.00-4.00   sec   896 KBytes  7.34 Mbits/sec   20   1.41 KBytes       
  [  5]   4.00-5.00   sec  1.00 MBytes  8.39 Mbits/sec    8   1.41 KBytes       
  [  5]   5.00-6.00   sec  0.00 Bytes  0.00 bits/sec    1   1.41 KBytes       
  [  5]   6.00-7.00   sec   896 KBytes  7.35 Mbits/sec    8   1.41 KBytes       
  [  5]   7.00-8.00   sec  1.00 MBytes  8.39 Mbits/sec    8   1.41 KBytes       
  [  5]   8.00-9.00   sec  0.00 Bytes  0.00 bits/sec    1   1.41 KBytes       
  [  5]   9.00-10.00  sec   896 KBytes  7.34 Mbits/sec   17   1.41 KBytes       
  - - - - - - - - - - - - - - - - - - - - - - - - -
  [ ID] Interval           Transfer     Bitrate         Retr
  [  5]   0.00-10.00  sec  6.62 MBytes  5.56 Mbits/sec  102             sender
  [  5]   0.00-10.25  sec  6.25 MBytes  5.11 Mbits/sec                  receiver
  ```
#### **Cleaning up**  
- Remove pods and services via:
  ```shell
  sudo kubectl delete -f iperf3-client.yaml --context kind-k8s01
  sudo kubectl delete -f iperf3-server.yaml --context kind-k8s02
  ```
---

### **Step 2: Transition to Helm Charts**

Helm is a package manager for Kubernetes, simplifying application deployment and management. A Helm chart is a collection of YAML templates and configurations that define a Kubernetes application, enabling versioned and repeatable installations. Charts make it easier to deploy complex apps with predefined configurations.

#### **2A: Create Helm Chart for iperf3**

1. **Chart Directory Structure**
   ```
   iperf3-chart/
   ├── Chart.yaml
   ├── values.yaml
   ├── templates/
       ├── deployment-server.yaml
       ├── deployment-client.yaml
       ├── service-server.yaml
   ```

2. We'll use the `20-instances-20K-values.yml` file for the parameters. Use `code 20-instances-20K-values.yml` to open it and check it out.

3. Install Helm tool:
   ```bash
   cd /workspaces/nanog93-krm-tutorial/part1
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
   chmod 700 get_helm.sh
   ./get_helm.sh
   ```

4. Create the chart package:
   ```bash
   sudo helm package iperf3-chart 
   ```

5. Install iPerf server instances:
   ```bash
   sudo kubectl config use-context kind-k8s02
   sudo helm install my-iperf3-server iperf3-chart --set client.enabled=false -f 20-instances-20K-values.yml
   ```

6. Check the status of the pods
   ```bash
   sudo kubectl get pods --context kind-k8s02
   ```
   You should see something like
   ```bash
    /workspaces/nanog93-krm-tutorial/part1 ·main* 19:02:54
    ❯ sudo kubectl get pods --context kind-k8s02
    NAME                                READY   STATUS    RESTARTS   AGE
    iperf3-server-1-79fbb7b58d-hfk8s    1/1     Running   0          9s
    iperf3-server-10-5d6595f5c8-nc99b   1/1     Running   0          11s
    iperf3-server-11-679f49d7c-7jcvk    1/1     Running   0          11s
    iperf3-server-12-765bb46bd8-pk294   1/1     Running   0          11s
    iperf3-server-13-86c8d7c6f5-bwk6l   1/1     Running   0          11s
    iperf3-server-14-f88c6994b-d9l52    1/1     Running   0          11s
    iperf3-server-15-6d69d89669-sdgk8   1/1     Running   0          11s
    iperf3-server-16-79bc47fbc6-n9km8   1/1     Running   0          11s
    iperf3-server-17-64ffcb4d67-sjm25   1/1     Running   0          9s
    iperf3-server-18-5696b48858-rnt6n   1/1     Running   0          10s
    iperf3-server-19-6ff7859745-zjmvx   1/1     Running   0          9s
    iperf3-server-2-c9774dc4f-4nlww     1/1     Running   0          11s
    iperf3-server-20-7986d8d4fc-wgkfw   1/1     Running   0          10s
    iperf3-server-3-6977f48f6b-j4ksx    1/1     Running   0          10s
    iperf3-server-4-84575987f-89wj7     1/1     Running   0          10s
    iperf3-server-5-59488f565b-wg6bv    1/1     Running   0          10s
    iperf3-server-6-84dcc76c49-9647n    1/1     Running   0          9s
    iperf3-server-7-5f88f767cd-smn5n    1/1     Running   0          9s
    iperf3-server-8-667cf5cdff-hkqsj    1/1     Running   0          11s
    iperf3-server-9-7bbdf5f6b4-qlzdz    1/1     Running   0          11s
   ```
7. Now, let's start the client instances:
   ```bash
   sudo kubectl config use-context kind-k8s01
   sudo helm install my-iperf3-client iperf3-chart --set server.enabled=false -f 20-instances-20K-values.yml
   ```
8. Check the status of the clients:
    ```bash
    sudo kubectl get pods
    NAME                                READY   STATUS    RESTARTS   AGE
    iperf3-client-1-5844cf645b-sw5cx    1/1     Running   0          12s
    iperf3-client-10-7cb8787cdd-tbr9b   1/1     Running   0          12s
    iperf3-client-11-5fcd844c88-jmpnd   1/1     Running   0          12s
    iperf3-client-12-7487c4fbbd-xszvx   1/1     Running   0          12s
    iperf3-client-13-5cdbd5d8fb-n8fcc   1/1     Running   0          12s
    iperf3-client-14-77685b8d88-s6tnm   1/1     Running   0          10s
    iperf3-client-15-7f9b9854dd-9swz2   1/1     Running   0          11s
    iperf3-client-16-5b6f7fb64-886c9    1/1     Running   0          10s
    iperf3-client-17-7fccdfcc7b-8xr7l   1/1     Running   0          11s
    iperf3-client-18-6f47d84b96-5cswb   1/1     Running   0          11s
    iperf3-client-19-7b7f5c4bd4-zn5hm   1/1     Running   0          12s
    iperf3-client-2-5c46dbb584-rtb47    1/1     Running   0          12s
    iperf3-client-20-7c9b694bcb-nwcmd   1/1     Running   0          12s
    iperf3-client-3-788b8d5f44-lbckz    1/1     Running   0          12s
    iperf3-client-4-67ffc894b8-xz8n5    1/1     Running   0          11s
    iperf3-client-5-7cfd79ccbd-fw8nb    1/1     Running   0          12s
    iperf3-client-6-78cbfd49c6-7vg9r    1/1     Running   0          12s
    iperf3-client-7-6cf9df8c44-d8zd6    1/1     Running   0          10s
    iperf3-client-8-f86645d9b-6qbzx     1/1     Running   0          10s
    iperf3-client-9-8dd769c56-z6r7p     1/1     Running   0          11s
    ```
    Let's check the activity in one of the instances:
    ```bash
    sudo kubectl logs iperf3-client-15-7f9b9854dd-9swz2                                  
    Connecting to host 172.254.102.101, port 30015
    [  5] local 10.244.0.21 port 41166 connected to 172.254.102.101 port 30015
    [  7] local 10.244.0.21 port 41182 connected to 172.254.102.101 port 30015
    [  9] local 10.244.0.21 port 41192 connected to 172.254.102.101 port 30015
    [ 11] local 10.244.0.21 port 41194 connected to 172.254.102.101 port 30015
    [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
    [  5]   0.00-1.15   sec  4.00 KBytes  28.6 Kbits/sec    0   14.1 KBytes       
    [  7]   0.00-1.15   sec  4.00 KBytes  28.6 Kbits/sec    0   14.1 KBytes       
    [  9]   0.00-1.15   sec  4.00 KBytes  28.6 Kbits/sec    0   14.1 KBytes       
    [ 11]   0.00-1.15   sec  4.00 KBytes  28.6 Kbits/sec    0   14.1 KBytes       
    [SUM]   0.00-1.15   sec  16.0 KBytes   114 Kbits/sec    0             
    - - - - - - - - - - - - - - - - - - - - - - - - -
    [  5]   1.15-2.02   sec  2.00 KBytes  18.9 Kbits/sec    0   14.1 KBytes       
    [  7]   1.15-2.02   sec  2.00 KBytes  18.9 Kbits/sec    0   14.1 KBytes       
    [  9]   1.15-2.02   sec  2.00 KBytes  18.9 Kbits/sec    0   14.1 KBytes       
    [ 11]   1.15-2.02   sec  2.00 KBytes  18.9 Kbits/sec    0   14.1 KBytes 
    ```
9. Let's see the stat activity in the switch:
    ```bash
    docker exec -ti dev1 sr_cli
    ```
    then inside the switch you might use:
    ```shell
    watch show interface ethernet-1/10 detail | grep Octets
    ```    
10. Cleaning up
    ```shell
    sudo kubectl config use-context kind-k8s01
    sudo helm delete  my-iperf3-client
    sudo kubectl config use-context kind-k8s02
    sudo helm delete  my-iperf3-server
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
