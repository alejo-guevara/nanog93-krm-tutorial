# Leveraging KRM for Declarative Network Automation - Tutorial - Part 2

## Overview

This second part of the workshop is mainly concerned with [SDC](https://docs.sdcio.dev/) (Schema Driven Configuration).

## Infrastructure setup

### **Set Up Your Kubernetes Cluster (Kind)**

#### **What is Codespaces?**
GitHub Codespaces is a cloud-based development environment that lets you code directly in the cloud using Visual Studio Code. It provides pre-configured containers with your code, dependencies, and tools, allowing you to start coding instantly without setting up a local development environment.

#### **Start your Codespaces instance**
To start your lab, use this link: https://codespaces.new/cloud-native-everything/nanog93-krm-tutorial
Then Check the configuration before continue. Ensure the repo name is: `cloud-native-everything/nanog93-krm-tutorial`
Make sure you are using the 4-core instance

There’s a free usage limit (120 core hours), which you can check under the Codespaces section
https://github.com/settings/billing/summary.

   **Important:** after using it, stop it and remove it at https://github.com/codespaces or it will keep using your free usage quota.

Click on **Create codespace**

Wait a few minutes for codespaces to load and go to the **Terminal** Section

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

---

### **Create your kubernetes cluster**
1. **Create Kind Cluster for GitHub CodeSpaces**
   Create one-node kubernetes cluster
   ```bash
   kind create cluster
   ```
2. **preload images**
   ```shell
   # Load the local images into the kind cluster
   kind load image-archive /var/cache/data-server.tar
   kind load image-archive /var/cache/config-server.tar
   ```

## **Prepping SDCIO Setup**   
Following are the steps to setup the environment.

```shell
# change into Part 2 directory
cd /workspaces/nanog93-krm-tutorial/part2
```

### Load pre-cached container images

```shell
# import the locally cached sr-linux container image
docker image load -i /var/cache/srlinux.tar
```

### Nokia SRLinux based containerlab topology

```shell
# deploy Nokia SRL containers via containerlab
cd clab-topology
sudo containerlab deploy
cd -
```

## SDC

Let's install SDC.

### Installation

The installation of SDC is rather simple. We just need to apply the following yaml file.

It contains all the resources required for SDC to run in your kubernetes environment.

- **Deployment:**
  A resource that manages the creation, scaling, and updating of a set of Pods with a defined containerized application, ensuring the desired state is maintained.
- **Services:**
  An abstraction that defines a stable network endpoint to expose and route traffic to a set of Pods, enabling communication within or outside the cluster.
- **CustomResourceDefinitions:**
  Extends the Kubernetes API, allowing users to define and manage custom resource types to meet specific application needs.
- **APIService:**
  Registers an API endpoint with the Kubernetes API server, enabling custom or aggregated APIs to be served alongside core Kubernetes APIs.
- **PersistentVolumes:**
  A storage resource provisioned in the cluster, abstracting underlying storage systems and providing persistent data storage for Pods independent of their lifecycle.
- **ServiceAccount:**
  Provides an identity for processes running in a Pod, enabling them to authenticate to the Kubernetes API and access cluster resources securely.
- **RoleBindings:**
  Grant specific permissions defined in a Role or ClusterRole to a user, group, or ServiceAccount within a namespace (or across the cluster for ClusterRoleBindings).

```shell
# install sdcio
kubectl apply -f sdcio.yaml
```

Check the pods are running

``` shell
> kubectl get pods -n network-system
```

The output should look like:
```shell
NAME                             READY   STATUS    RESTARTS       AGE
config-server-6964c6484b-8chv8   2/2     Running   2 (8m7s ago)   13h
```

Checking the api-registrations exist.

```shell
kubectl get apiservices.apiregistration.k8s.io | grep "sdcio.dev\|NAME"
```

The following two services should be `AVAILABLE == True`. This might take a second, since pods need to start and register themselfes.

```shell
NAME                                   SERVICE                        AVAILABLE   AGE
v1alpha1.config.sdcio.dev              network-system/config-server   True        16s
v1alpha1.inv.sdcio.dev                 Local                          True        16s
```

### Setup

SDC relies on different kinds of resource.

- **Yang Schema:** Defining the schema of the device configuration.
- **Connection Profile:** Defining connection parameters like protocol, encoding, port etc. for configuration writes to the devices.
- **Sync Profile**: Defining connection parameters like protocol, encoding, port etc. for configuration synchronisation / reads from the devices.
- **Secret:** Authentication information against the devices.
- **Discovery Rule:** Definition of devices (targets). Four options as of today (static, ip based, k8s pod, k8s service).

Next we will take a look at them and apply them afterwards.

```shell
# inspect the different artifact files
batcat artifacts/schema-nokia-srl-24.7.2.yaml artifacts/target-conn-profile-proto.yaml artifacts/target-sync-profile-gnmi.yaml artifacts/secret-srl.yaml artifacts/discovery_address.yaml
```

Let's apply the schema resource.

```shell
# Nokia SR Linux Yang Schema
kubectl apply -f artifacts/schema-nokia-srl-24.7.2.yaml
```

Verify that the schema is downloaded and ready to be used.

```shell
> kubectl get schemas.inv.sdcio.dev srl.nokia.sdcio.dev-24.7.2
NAME                         READY   PROVIDER              VERSION   URL                                            REF
srl.nokia.sdcio.dev-24.7.2   True    srl.nokia.sdcio.dev   24.7.2    https://github.com/nokia/srlinux-yang-models   v24.7.2
```

The other resources do not have any specific status. So we apply them in bulk.

```shell
# Connection Profile
kubectl apply -f artifacts/target-conn-profile-proto.yaml

# Sync Profile
kubectl apply -f artifacts/target-sync-profile-gnmi.yaml

# SRL Secret
kubectl apply -f artifacts/secret-srl.yaml

# Discovery Rule
kubectl apply -f artifacts/discovery_address.yaml
```

As a result of the discovery rule you shoud now see two targets created.
The discovery will also determine additional information like the discovered `ADDRESS`, the `PROVIDER` in use, `PLATFORM`, `SERIALNUMBER` and chassis `MACADDRESS`.

```shell
> kubectl get targets.inv.sdcio.dev 
NAME   READY   REASON   PROVIDER              ADDRESS        PLATFORM       SERIALNUMBER     MACADDRESS
dev1   True             srl.nokia.sdcio.dev   172.21.0.200   7220 IXR-D2L   Sim Serial No.   1A:EB:00:FF:00:00
dev2   True             srl.nokia.sdcio.dev   172.21.0.201   7220 IXR-D2L   Sim Serial No.   1A:F8:01:FF:00:00
```

### Usage

Let's interact with the network device through kubernetes now.

#### Retrieve Configuration

SDC will sync the device configurations according to the sync profile and allow you to query the device config via kubectl.

```shell
kubectl get runningconfigs.config.sdcio.dev dev1 -o yaml
# or in json format
kubectl get runningconfigs.config.sdcio.dev dev1 -o json
```

The output is quite extensive so Let's just take a look at the network-instance configuration and pretty-print it via `jq`.

```shell
kubectl get runningconfigs.config.sdcio.dev dev1 -o jsonpath="{.status.value.network-instance}" | jq
```

#### Apply Configuration

Verify interface config for interface `system0`.

```shell
# via docker
docker exec dev1 sr_cli "info interface system0"

# or via ssh 
ssh dev1 "info interface system0"
```

This will result in no output, since the interface `system0` is not yet configure.
So let's get to it.

```shell
batcat configs/system0.yaml
kubectl apply -f configs/system0.yaml
```

Verify that the config is properly applied.

```shell
> kubectl get configs.config.sdcio.dev
NAME           READY   REASON   TARGET         SCHEMA
dev1-system0   False   Failed   default/dev1   
```

You will see the resource is not `READY`, it failed to apply.

Let's investigate why that is.

```shell
# either reference by name
kubectl get configs.config.sdcio.dev dev1-system0 -o yaml
# or via the filename again
kubectl get -f configs/system0.yaml -o yaml
```

```json
...
status:
  conditions:
  - lastTransitionTime: "2024-11-11T13:57:36Z"
    message: 'rpc error: code = Unknown desc = value "shutdown" does not match enum
      type "admin-state", must be one of [disable, enable]'
    reason: Failed
    status: "False"
    type: Ready
```

The data-server indicated, that for the `admin-state` field, the allowed values are either `enable` or `disable`. These allowed field values it did take from the schema, that defined the allowed value space.

Let's set the value to `disable`.

```shell
kubectl apply -f configs/system0_disable.yaml
```

Let's check the status again.

```shell
# either reference by name
kubectl get configs.config.sdcio.dev dev1-system0 -o yaml
# or via the filename again
kubectl get -f configs/system0_disable.yaml -o yaml
```

```json
...
status:
  appliedConfig:
    config:
    - path: /
      value:
        interface:
        - admin-state: disable
          description: Ethernet-1/3 Interface Description
          name: ethernet-1/3
    lifecycle: {}
    priority: 10
  conditions:
  - lastTransitionTime: "2024-11-11T14:19:51Z"
    message: |-
      rpc error: code = Unknown desc = cumulated validation errors:
      error must-statement ["((. = 'enable') and starts-with(../srl_nokia-if:name, 'system0')) or not(starts-with(../srl_nokia-if:name, 'system0'))"] path: interface/system0/admin-state: admin-state must be enable
    reason: Failed
    status: "False"
    type: Ready
```

Now we hit a must-statement, also a YANG construct, that allows to define fine grained rules on field in the configuration. The must statement is even provided as part of the output.

```text
"((. = 'enable') and starts-with(../srl_nokia-if:name, 'system0')) or not(starts-with(../srl_nokia-if:name, 'system0'))"
```

Now that we now, system0 must always be up, we can set its admin-state to up and config will be applied.

Let's set the value to `enable`.

```shell
kubectl apply -f configs/system0_enable.yaml
```

Verify that configs status.

```shell
> kubectl get configs.config.sdcio.dev 
NAME           READY   REASON   TARGET         SCHEMA
dev1-system0   True    Ready    default/dev1   srl.nokia.sdcio.dev/24.7.2
```

Verify on the device, that the config is applied.

```shell
docker exec dev1 sr_cli "info interface system0"
# or via kubernetes again
kubectl get runningconfigs.config.sdcio.dev dev1 -o jsonpath="{.status.value.interface}" | jq '.[] | select(.name | test("system0"))'
```

#### Config Drift and Remediation

What if the config we set via the the intent is being changed on the device.

Let's change the description of interface `system0`.

```shell
# set interface system0 description to 
docker exec -it dev1 sr_cli -ec -- set interface system0 description "foobar description"
```

Let's see whats happens to the interface description over time.

```shell
# verify the change is committed on the device.
watch ssh dev1 -- info interface system0
```

#### Apply ConfigSet

The ConfigsSet resource is a config snippet that is not bound to a single target, as the config.config.sdcio.dev resource was.
A selector labels are used to determine which targets should inherit the config in the ConfigSet.

Verify interface config for ethernet-1/1 and ethernet-1/2.

```shell
# via docker
docker exec dev1 sr_cli "info interface ethernet-1/{1,2}"

# or via ssh 
ssh dev1 "info interface ethernet-1/{1,2}"
```

Notice no Vlans are actually configured.
Now take a look at `configs/vlans.yaml` and apply it.

```shell
batcat configs/vlans.yaml
kubectl apply -f configs/vlans.yaml
```

Verify the device config.

```shell
# via docker
docker exec dev1 sr_cli "info interface ethernet-1/{1,2}"

# or via ssh 
ssh dev1 "info interface ethernet-1/{1,2}"
```

The `ConfigSet` controller will have created `Config` resources based on the `ConfigSet` information for all the matching `Targets`.

```shell
> kubectl get configs.config.sdcio.dev
NAME                READY   REASON   TARGET         SCHEMA
dev1-system0        True    Ready    default/dev1   srl.nokia.sdcio.dev/24.7.2
vlan-configs-dev1   True    Ready    default/dev1   srl.nokia.sdcio.dev/24.7.2
vlan-configs-dev2   True    Ready    default/dev2   srl.nokia.sdcio.dev/24.7.2
```

Let's quickly trace back the label.

> Consult the DiscoveryRule again to see where / how the labels where applied to the targets.
>
> ```shell
> > kubectl get discoveryrules.inv.sdcio.dev basic-usage -o yaml
> apiVersion: v1
> items:
> - apiVersion: inv.sdcio.dev/v1alpha1
>   kind: DiscoveryRule
>  
>   spec:
>     ...
>     targetTemplate:
>       labels:
>         sdcio.dev/region: us-east
>     ...
> ```
>
> And check how they are applied to the `Target`:
>
> ```shell
> > kubectl get targets.inv.sdcio.dev dev1 -o yaml
> apiVersion: inv.sdcio.dev/v1alpha1
> kind: Target
> metadata:
>   annotations:
>     inv.sdcio.dev/discovery-rule: basic-usage
>   labels:
>     inv.sdcio.dev/discovery-rule: basic-usage
>     sdcio.dev/region: us-east
>   name: dev1
>   namespace: default
> ...
> ```

Change the `configs/vlans.yaml` file. Change `ethernet-1/1` to `ethernet-1/2`.
If you like add or remove vlans, descriptions, whatever and reapply the config.

```shell
# change
sed -i 's/ethernet-1\/1/ethernet-1\/2/g' configs/vlans.yaml
# check
batcat configs/vlans.yaml
# apply 
kubectl apply -f configs/vlans.yaml
```

The device config should reflect the changes. It will have removed the initial config on `ethernet-1/1` and applied it to `ethernet-1/2` actual changes.

The changes can be verified via above commands again, but also via kubernetes itself:

```shell
kubectl get runningconfigs.config.sdcio.dev dev1 -o jsonpath="{.status.value.interface}" | jq
## or more specific again just ethernet-1/1 and ethernet-1/2
kubectl get runningconfigs.config.sdcio.dev dev1 -o jsonpath='{.status.value.interface}' | jq '.[] | select(.name | test("ethernet-1/{1,2}"))'
```


### How to generate yaml config snippets on SRLinux

1. Login to the container
  
```shell
# ssh
ssh dev1
# or via docker
docker exec -it dev1 sr_cli
```

2. Build sample config on device

```shell
# enter configuration context
> enter candidate

# change config
> set interface ethernet-1/3 description "MyDescription"

# show diff in yaml format
> diff | as yaml
+ interface:
+   - name: ethernet-1/3
+     description: MyDescription

# NOTE: plus signes need ot be removed, but that is the actual yaml based config of the change.

# use discard now to discard changes, to make them apply via k8s
> discard now
```

3. Build k8s resource

```yaml
apiVersion: config.sdcio.dev/v1alpha1
kind: Config
metadata:
  name: MyIntent                # << Adjust Intenet Name 
  namespace: default
spec:
  priority: 10
  config:
  - path: /                     # if the intent covers just a subpath it should be defined here. e.g. /interface[name=ethernet-1/1] 
    value:
      <INSERT CONFIG HERE>      # << Add config here
```

## How to get the yaml config structure from yang

gnmic provides the ability to generate yaml file that contains all the config flags defined in a yang model under the given path.

```shell
git clone -b v24.7.2 https://github.com/sdcio/yang.git srl-yang

gnmic generate --file srl-yang/srl_nokia/models/ --dir srl-yang/ --exclude .tools. --path /interface/subinterface > srl.yaml

batcat srl.yaml
```

If you take that file, remove all the entries you do not need and adjust the values you're interested in, you have a working intent snippet at hand. (Path given to gnmic should match the path provided in the `config` resource)

For details on the `gnmic generate` command see: https://gnmic.openconfig.net/cmd/generate/
