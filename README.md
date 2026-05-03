# vCluster Bare Metal with KubeVirt

Run vCluster's metal3 bare metal node provider locally using KubeVirt VMs as fake bare metal servers.

## Prerequisites

- Docker
- [vcluster CLI](https://www.vcluster.com/docs/getting-started/setup)
- kubectl
- helm
- A host with KVM/hypervisor support and enough resources for the VMs (~16GB RAM, 4+ CPU cores)

## Setup

- **KubeVirt VMs** with a Redfish BMC shim (`virtbmc`) acting as fake bare metal servers
- A **Linux bridge** (`br0`, `192.168.100.0/24`) on the node as the shared provisioning and tenant network
- A **metal3 NodeProvider** that auto-deploys Multus, Metal3 + Ironic, and a DHCP server into the host cluster
- A **NodeEnvironment** with IP range `192.168.100.10-192.168.100.20`, gateway `192.168.100.1`, DNS `1.1.1.1`
- An Ubuntu 24.04 **OSImage** and a static **SSHKey**
- **BareMetalHost** resources pointing at the VMs' BMC endpoints

## Quick Start

```bash
# Create a vcluster-in-docker cluster
make vind-up

# Install everything (cert-manager, kubevirt, bridge, platform, node provider, etc.)
# You will be prompted for a valid platform license token.
make install

# Wait for the NodeProvider to deploy Metal3/Ironic (check the platform UI or
# kubectl get statefulset metal3 -n default), then create VMs + BMH resources.
make create-vms
```

## Usage

There are two ways to provision and attach nodes to vCluster. Either perform a manual BareMetalHosts provisioning using the `NodeClaim` resource or use the vCluster auto-nodes option. The second will use Karpener to scale-up and down available bare metal nodes.

### Provision BareMetalHosts manually

After `make create-vms`, the BareMetalHost resources appear in the platform UI.
Wait for them to become `available` before provisioning — Ironic needs to inspect each host first, which takes a few minutes.
You can track progress in the UI or with `kubectl get baremetalhost -A`.

To provision a single machine manually:

```bash
make create-machine
```

You may also delete and re-create BareMetalHost resources through the UI.

### Create a vCluster with auto-nodes

This creates a VirtualClusterInstance that automatically provisions bare metal nodes via the metal3 provider:

```bash
make create-vcluster
```

The vCluster uses kube-vip on the bridge network and requests nodes from the metal3 NodeProvider. What happens next:

1. The platform creates a **NodeClaim** for the vCluster — check with `kubectl get nodeclaim -A`
2. A BareMetalHost is selected and **provisioned** (Ironic writes the OS image) — watch with `kubectl get baremetalhost -A`
3. The machine boots, runs cloud-init, and **joins** the vCluster as a node — `kubectl get nodes` against the vCluster

This takes several minutes end-to-end.

### SSH Provisioned Machine

Create a new ssh-keypair for the deployment and replace the `ssh-demo-key`, `ssh-demo-key.pub`, and the `ssh-key.yaml` files. To SSH to the baremetalhosts, we can deploy the `bridge-jump-pod` pod. Copy the required keys to the pod and perform an SSH connection to the relevant endpoint.

```bash
# Create an SSH key-pair
$ ssh-keygen -b 2048 -t rsa

# Copy SSH key details to bridge-jump-pod
$ kubectl cp ssh-demo-key default/bridge-jump-pod:/tmp/ssh-demo-key
$ kubectl cp ssh-demo-key.pub default/bridge-jump-pod:/tmp/ssh-demo-key.pub

# Exec to bridge-jump-pod
$ kubectl exec -it bridge-jump-pod -- bash
# ssh -i /tmp/ssh-demo-key ubuntu@192.168.100.100
```

**Note:** The baremetal host details after provisioning are coming from the `manifests/node-environment.yaml` file and property `metal3.vcluster.com/network-ip-range: 192.168.100.100-192.168.100.120`. To check the IP address assigned, execute `kubectl describe baremetalhost bare-metal01 | grep -i "metal3.vcluster.com/ip-address"` on the management/controller cluster.

### Individual targets

| Target                      | Description                                      |
|-----------------------------|--------------------------------------------------|
| `make vind-up`              | Create the vcluster-in-docker host cluster       |
| `make vind-down`            | Tear down the host cluster                       |
| `make install`              | Install all infrastructure components            |
| `make create-vms`           | Deploy KubeVirt VMs + BMC StatefulSets           |
| `make create-bmh`           | Create/recreate BareMetalHost resources          |
| `make create-machine`       | Create a Machine/NodeClaim (manual provisioning) |
| `make create-vcluster`      | Create a vCluster with auto-nodes                |
| `make reset-admin-password` | Reset the platform admin password                |

## Tear Down

```bash
make vind-down
```

This deletes the vcluster-in-docker cluster and removes the local kubeconfig file. All state (VMs, BMHs, platform data) lives inside the cluster and is gone with it.
