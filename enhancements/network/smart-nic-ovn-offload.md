---
title: smartnic-ovn-offload
authors:
  - "@zshi-redhat"
reviewers:
  - "@Billy99"
  - "@bnemeth"
  - "@danwinship"
  - "@erig0"
  - "@fabrizio8"
  - "@pliurh"
  - "@trozet"
approvers:
  - "@dcbw"
  - "@knobunc"
creation-date: 2021-04-13
last-updated: 2021-04-13
status: implementable
---

# SmartNIC OVN Offload

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Operational readiness criteria is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

Open vSwitch (OVS) Hardware Offload with NVIDIA ConnectX-5 improves OVS
dataplane performance by offloading processing of OVS OpenFlow flows to
hardware NIC. By integrating NVIDIA BlueField2 SmartNIC support in OpenShift,
additional capabilities are added to accelerate and secure the OVN control
plane by offloading OVS and OVN to the SmartNIC ARM processors.

## Terminology

- SmartNIC: A combined Network Interface Card (NIC) and CPU hardware which
offloads processing tasks that the system CPU would normally handle
- SmartNIC host(s): Baremetal servers where SmartNICs are plugged to
- SmartNIC ARM cores: The embedded aarch64 system running on SmartNIC CPU(s)
- SmartNIC node(s): SmartNIC ARM cores provisioned as an OpenShift node
- Regular OpenShift node(s): x86 server provisioned as an OpenShift node
- Host-Network Pods: Pods that use networking on the host. OpenShift CNIs are
not involved in configuring the networking for these pods. This allows the pods
to access the host loopback device, services listening on localhost, and could
be used to snoop on network activity of other pods on the same node.
- SR-IOV: Single Root I/O Virtualization, a NIC technology that allows
isolation of PCIe resources
- SR-IOV Pods: SR-IOV Pods are pods bound to a VF which is used to send and
receive packets on.
- PF: Physical Function, the physical ethernet controller that supports SR-IOV
- VF: Virtual Function, the virtual PCIe device created from PF. A given PF
only supports a limited number of VFs (hardware dependant) and is in the order
to 128 or 256 VFs per PF with NVIDIA BlueField2.
- VF representor: The port representor of the VF. VF and VF representor are
created in pairs where VF netdev appears in SmartNIC host and VF representor
appears in SmartNIC ARM cores

## Motivation

Encryption, Encapsulation, and Virtual switching (IPsec, Geneve, VXLAN, OVS,
Connection Tracking) to and from containers or VMs are basic building blocks
of the Hybrid cloud and particularly of OCP. These functions are very compute
intensive and can consume up to 40 cores from the server’s motherboard CPU
for terminating 25Gigs of traffic. Terminating 100Gigs without hardware offload
is not feasible at all. Most of our customers are demanding all of the above
without much use of the CPU on the motherboard of the server.

### Goals

- Only support NVIDIA BlueField2 because other implementations of SmartNIC are
unknown for now
- OVN-Kubernetes control plane including OVS and OVN controller are offloaded
to SmartNIC node, other CNIs are not currently planned
- SR-IOV pods run on SmartNIC host using VF as pod default route interfaces
- Manage SmartNIC host offload configurations in OpenShift Operators

### Non-Goals

- Provisioning and managing the OVN-Kubernetes components on SmartNIC ARM cores
is discussed in [Installing OCP on ARM-Based Smart NICs](https://github.com/openshift/enhancements/pull/738)
- Support multiple SmartNICs on one OpenShift baremetal worker node
- Support SmartNICs on OpenShift baremetal master nodes

## Proposal

- User installs SR-IOV Network Operator (OLM managed)
- User creates one or more custom MachineConfigPools whose nodes are installed
with SmartNIC hardware
- User specifies NodeSelector labels in SR-IOV Network Operator CRD to match on
the custom MachineConfigPools
- SR-IOV Network Operator renders and attaches SmartNIC MachineConfigs to the
selected custom MachineConfigPools
- MachineConfigOperator reboots nodes in custom MachineConfigPool while
applying MachineConfigs
- SR-IOV Network Operator configures SmartNIC devices and exposes SR-IOV VFs
as extended kubernetes resources
- Nodes in custom MachineConfigPools become ready and start serving SR-IOV pod
creation request
- User creates SR-IOV pod by specifying SR-IOV network request in pod
annotations
- SR-IOV pod starts with VF device as pod default route interface on SmartNIC
host

### User Stories

### Implementation Details/Notes/Constraints

SmartNIC OVN offload will be enabled with OVN-Kubernetes CNI only.
It is expected that the following changes will be made:

#### OVN-Kubernetes

There are three modes introduced in OVN-Kubernetes for running ovnkube-node:
- **full**: No change to current ovnkube-node logic, runs on Regular OpenShift nodes
- **smart-nic**: ovnkube-node on SmartNIC ARM cores, plumbs VF representor device
- **smart-nic-host**: ovnkube-node on SmartNIC host, runs cniserver and plumbs VF device

```html
                                                                         +--------------------------+
                                                                         |     Baremetal worker     |
                                                                         |     (SmartNIC host)      |
+--------------------------+                                             |                          |
|    Baremetal worker      |                                             | +----------------------+ |
|   (non-SmartNIC host)    |                                             | |  ovnkube-node in     | |
|                          |                     +-----------------------+-+                      | |
| +----------------------+ |                     |                       | |  smart-nic-host mode | |
| |  ovn-controller      | |      +--------------+---------------+       | +----------------------+ |
| +----------------------+ |      | Master       |               |       |                          |
|                          |      |              |               |       +------------+-------------+
| +----------------------+ |      |    +---------v-----------+   |                    |
| |  ovnkube-node in     | |      |    |                     |   |       +------------+-------------+
| |                      +-+------+---->   K8s APIServer     |   |       |        SmartNIC          |
| |  full mode           | |      |    |                     |   |       |                          |
| +----------------------+ |      |    +---------^-----------+   |       | +----------------------+ |
|                          |      |              |               |       | |  ovn-controller      | |
+------------+-------------+      +--------------+---------------+       | +----------------------+ |
             |                                   |                       |                          |
             |                                   |                       | +----------------------+ |
+------------+-------------+                     |                       | |  ovnkube-node in     | |
|       Non-SmartNIC       |                     +-----------------------+-+                      | |
+--------------------------+                                             | |  smart-nic mode      | |
                                                                         | +----------------------+ |
                                                                         |                          |
                                                                         +--------------------------+
```

On SmartNIC ARM cores, ovn-controller and ovnkube-node are running as
containers, OvS is running as system service.
On SmartNIC host, ovnkube-node is running as a container.
On non-SmartNIC host, ovn-controller and ovnkube-node are running as
containers, OvS is running as system service.

#### Cluster Network Operator

A new label `network.operator.openshift.io/ovn-offload-host` will be introduced
in Cluster Network Operator (CNO) to allow proper scheduling of OVN-Kubernetes
components on SmartNIC hosts and non-SmartNIC hosts.

Ovnkube-node-smart-nic-host daemonset will be added in CNO ovn-kubernetes
manifests, which executes ovnkube-node in `smart-nic-host` mode.
ovnkube-node-smart-nic-host daemonset runs only on SmartNIC hosts.

Ovnkube-node-smart-nic-host pod will be mounted with the config map
`ovn-management-ports` to get the ovn management port information of the node
it is running on. The ovn management port information is provided by SR-IOV
Network Operator and is used as argument to start ovnkube-node in
`smart-nic-host` mode. ovnkube-node-smart-nic pod waits until the
`ovn-management-ports` config map is created, then starts. The management
port is an SR-IOV VF device that will be configured to `ovn-k8s-mp0` port by
ovnkube-node.

Daemonsets of OVN-Kubernetes components other than ovnkube-node-smart-nic-host
will be modified with `nodeAffinity` to run on non-SmartNIC hosts.

#### SR-IOV Network Operator

SmartNIC offload will be enabled for nodes in custom MachineConfigPool.

A new field `OVNHardwareOffload` will be added in `SriovOperatorConfig` CRD
in SR-IOV Network Operator:

```go
// SriovOperatorConfigSpec defines the desired state of SriovOperatorConfig
// +k8s:openapi-gen=true
type SriovOperatorConfigSpec struct {
        // NodeSelector selects the nodes to be configured
        ConfigDaemonNodeSelector map[string]string `json:"configDaemonNodeSelector,omitempty"`
        // Flag to control whether the network resource injector webhook shall be deployed
        EnableInjector *bool `json:"enableInjector,omitempty"`
        // Flag to control whether the operator admission controller webhook shall be deployed
        EnableOperatorWebhook *bool `json:"enableOperatorWebhook,omitempty"`
        // OVNHardwareOffload describes the OVN HWOL configuration for selected MachineConfigPool
        OVNHardwareOffload []OVNHardwareOffloadConfig `json:"ovnHardwareOffload,omitempty"`
}

type OVNHardwareOffloadConfig struct {
	// NodeSelector matches on Labels defined in MachineConfigPoolSpec.NodeSelector
	// OVS HWOL MachineConfigs are generated and applied to Nodes in MachineConfigPool
	// Labels in NodeSelector are ANDed when matching on MachineConfigPoolSpec.NodeSelector
	NodeSelector map[string]string `json:"nodeSelector,omitempty"`
}
```

SR-IOV Network Operator will apply SmartNIC MachineConfigs by calling Machine
Config Operator APIs. SmartNIC MachineConfigs may include service that reverts
default OVN-Kubernetes ovs-configuration, and dropin service that disables Open
vSwitch services (openvswitch, ovs-vswitchd, ovsdb-server). Alternative is to
not install the ovs-configuration service on SmartNIC hosts when the node first
gets provisioned. This can be achieved by moving ovs-configuration to a
MachineConfig in CNO and having part of CNO run at bootstrap time to create the
correct MachineConfig before MCO ever starts. Refer to the [Improve install-time
network config usage](https://issues.redhat.com/browse/SDN-1615) for details about the alternative.

API to remove ovs-configuration service and its configuration.
have part of CNO run at bootstrap time to create the correct MachineConfigs
before MCO ever starts,

SR-IOV Network Operator will create `SriovNetworkNodePolicy` instances
automatically for nodes in the `MachineConfigPool` selected by
`OVNHardwareOffloadConfig.NodeSelector`. The creation of
`SriovNetworkNodePolicy` instances will trigger the provision of SR-IOV devices
on those nodes and advertisement of SR-IOV devices (except VF 0) to kubernetes
node status as extended resource. VF with index 0 will be used by OVN-Kubernetes
as management port `ovn-k8s-mp0`. The user can also create or edit the
`SriovNetworkNodePolicy` instances, so SR-IOV Network Operator will need to
ensure VF with index 0 is not added to the generated `sriov-device-plugin`
configMap used to advertise the SR-IOV devices.

SR-IOV Network Operator will configure [DNS Operator placement API](https://github.com/openshift/api/blob/release-4.8/operator/v1/types_dns.go#L64) based on
`MachineConfigPoolSelector` to isolate DNS pods from running on SmartNIC
hosts (see [Open Questions](#open-questions) for why removing DNS pods from
SmartNIC hosts).

SR-IOV Network Operator will apply label
`network.operator.openshift.io/ovn-offload-host` (defined in CNO) to nodes
selected by `MachineConfigPoolSelector` which drives ovn-controller, ovn-ipsec
 and ovs-node daemonset pods away from SmartNIC hosts and allows scheduling of
ovnkube-node-smart-nic-host daemonset pods onto SmartNIC hosts.

A new config map `ovn-management-ports` will be created by SR-IOV Network
Operator and mounted to CNO ovnkube-node-smart-nic daemonset to pass node ovn
management port information which contains node specific SR-IOV interface names
that are created and discovered by SR-IOV Network Operator.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ovn-management-ports
  namespace: openshift-ovn-kubernetes
data:
  smart-nic-worker-0: '{"ovnkube-node-mgmt-port":"eth0v0"}'
  smart-nic-worker-1: '{"ovnkube-node-mgmt-port":"eth0v0"}'
```

There are three supported modes with NVIDIA SmartNIC BlueField2:
- **Separated host (default)**: The embedded ARM system and the function
exposed to the host are both symmetric. Each one of those functions has its
own MAC address and is able to send and receive packets.
- **Embedded CPU**: The embedded ARM system controls the NIC resources and
data path. A network function is still exposed to the host, but it has limited
privileges.
- **Restricted**: Also referred to as isolated mode, for security and isolation
purposes, it is possible to restrict the host from performing operations that
can compromise the DPU.

BlueField2 needs to be configured to **Embedded CPU** or **Restricted** mode
when enabling OVN offload. SR-IOV network config daemon, a daemonset managed
by SR-IOV Network Operator, will be updated to use mstflint version >=
`4.15.0-1.el8` which supports BlueField2 mode configuration.

SR-IOV Network Operator identifies the NIC using its device ID, the NVIDIA BlueField2
device ID will be added support in SR-IOV Network Operator.

SR-IOV Network Operator will move to use systemd service for configuring
SmartNIC SR-IOV devices vs configuring SR-IOV devices in SR-IOV config daemonset
pods. This ensures that SR-IOV device with VF index 0 (which is used as ovn
management port) becomes available when ovnkube-node-smart-nic pod starts.

SR-IOV Network Operator will apply udev rule to SmartNIC VF0 to ensure the VF
interface name is consistant across rebooting and kernel upgrade.

### Risks and Mitigations

#### Setting up initial SmartNIC connectivity

It is required to establish an initial transparent network connection between
SmartNIC host and external network through SmartNIC ARM cores in order to
deploy SmartNIC host as an OpenShift worker node. This is currently done
manually by user outside of OpenShift management.

#### BlueField2 OOB interface driver not in RHEL

Accessing to BlueField2 ARM cores via Out-Of-Band (OOB) interface requires
additional [mlxbf_gige driver](https://bugzilla.redhat.com/show_bug.cgi?id=1803489) which is not available in RHEL-8.4.
This is currently done through ssh or console from SmartNIC host using rshim
[userspace driver](https://bugzilla.redhat.com/show_bug.cgi?id=1744737).
Rshim userspace driver needs to be made availabe in RHCOS image or downloadable
in Universal Base Image (UBI).

#### OpenShift and RHEL support for aarch64

Management of SmartNIC ARM cores (aarch64) as individual node in a separate
OpenShift cluster requires OpenShift and RHEL support for aarch64.

#### Manage SmartNICs and SmartNIC hosts in separate clusters

This enhancement makes the assumption that both SmartNIC nodes and SmartNIC
hosts belong to the same OpenShift cluster, where SmartNIC hosts are managed by
OpenShift and SmartNIC nodes are managed by user. Any future changes that beak
this assumption will need an update in the enhancement.

## Design Details

### Open Questions

#### Cluster Pods on SmartNIC host

With current OVN-Kubernetes design, it is only supported to run SR-IOV pods on
the SmartNIC host. Regular pods whose networks require virtual **veth** interfaces
cannot be plugged to OVN network because there will be no Open vSwitch running
on the SmartNIC host.

To create a SR-IOV pod, it is needed to request the SR-IOV resource explicitly
in pod spec, for example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sriov-pod
spec:
  containers:
  - name: sriov-container
    image: centos
    resources:
      requests:
        openshift.io/<sriov-resource-name>: 1
      limits:
        openshift.io/<sriov-resource-name>: 1
```

In the above pod yaml file:

`openshift.io/<sriov-resource-name>` is the full resource name that follows
the [extended resource naming scheme](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources),
it is advertised by SR-IOV Network Device Plugin and published in Kubernetes
node status. The SR-IOV Network Device Plugin is a component that discovers
and manages the limited SR-IOV devices on the SmartNIC host. The managed
SR-IOV devices, AKA VFs, are created from SmartNIC PF.

`1` is the number of SR-IOV devices requested for the `sriov-pod`. When the
requested number of SR-IOV devices are not available on the target node, the
`sriov-pod` will be in `Pending` state until resource become available again
(e.g. other SR-IOV pods release the VF device on the target node).

Given that regular **veth** pods cannot be created on the SmartNIC host,
failure would happen if some cluster pods or infra pods using **veth**
interfaces are scheduled on the SmartNIC host, examples of cluster or infra
pods are as below:

- ingress-canary in openshift-ingress-canary namespace
- image-pruner in openshift-image-registry namespace
- network-metrics-daemon in openshift-multus namespace
- network-check-source & network-check-target in openshift-network-diagnostics
- Pods in openshift-marketplace & openshift-monitoring namespaces

There are several possible ways to avoid the failure:

- Modify cluster pod manifests to run with `hostNetwork: true` which doesn't
require OVN-Kubernetes CNI to create pod network, this has drawback that it
gives cluster pod additional privileges which may not be necessarily needed.
- Modify cluster pod manifests to select non-SmartNIC host, this may have
issues for pods that need to run on every worker node, including the SmartNIC
host. This also implies that additional non-SmartNIC worker nodes need be
available
- Find a way for the cluster pods to consume SR-IOV device by default when they
are scheduled on the SmartNIC host. We cannot simply add SR-IOV resource request
in cluster pod manifests, since the cluster pods are usually daemonsets or
deployments, adding SR-IOV resource request in daemonsets or deployments will
result in their pods be scheduled only on nodes that advertising the SR-IOV
resources, this doesn't work for non-SmartNIC host
- Use mutation admission controller to inject SR-IOV resource request in pod
manifest so that pods can use SR-IOV device as their default interface.
The problem with this approach is that admission controller runs before the
pod is scheduled, it cannot know whether the pod is going to end up on a
SmartNIC host or Regular OpenShift node.
- Find a way to enable Open vSwitch for non-SR-IOV pods on SmartNIC host
- Taint the SmartNIC hosts so that pods without corresponding toleration won't
be scheduled to SmartNIC hosts

### Test Plan

It is expected that the initial testing environment will be set up manually
since SmartNIC ARM cores are not managed by an OpenShift cluster yet.

Current e2e test cases may need to be modified to run on SmartNIC hosts.

Extra e2e test cases may need to be developed for ovnkube-node running in
`smart-nic-host` mode.

### Graduation Criteria

#### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation
- Gather feedback from users and developers

#### Tech Preview -> GA

#### Removing a deprecated feature

### Upgrade / Downgrade Strategy

### Version Skew Strategy

Ovnkube-node on SmartNIC hosts communicates to ovnkube-node on SmartNIC nodes
through Kubernetes APIs (by annotating pod metadata), version skew can occur
when the two ovnkube-node are using different annotation format. We will have
to make sure OVN-Kubernetes has backward compatibility on pod annotations used
for passing SmartNIC device information.

SR-IOV, CNO, MCO and DNS Operators coordinate on SmartNIC configurations,
version skew may occur when CNO, MCO or DNS has API changes that are used by
SR-IOV Network Operator for inter-Operator communication. SR-IOV Network
Operator needs to have backward compatibility when communicating with CNO, MCO
and DNS.

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed

N/A
