# Deploying Red Hat OpenShift with Assisted Installer using Cilium (CNI)

Note:  Red Hat does not currently support a third party CNI.  This is not an official document.

## Red Hat Assisted Installer

1. Before proceeding, create an account on [Red Hat Hybrid Cloud Console](https://console.redhat.com)
2. Log into the Hybrid Cloud Console
3. Create a new OpenShift cluster    
4. Select "OpenShift"
5. Select "Cluster List" on the left
6. Select "Create Cluster"
7. Select "Datacenter"
8. Select "Create Cluster"
9. Be sure to select the "Include custom manifests" option

Complete all the fields to continue the installation.

## Custom Manifests
Currently, the Assisted Installer will deploy the RH OVN CNI by default.  There is not an option to disable this during the installation.  To override this, a custom manifest will need to be added to both disable the OVN and add Cilium or another third party CNI as the default.

## Disable OVN
Below is an example manifest to disable the default network provider.

```code block

apiVersion: operator.openshift.io/v1
kind: Network
metadata:
  name: cluster
spec:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  defaultNetwork:
    type: None  # Disable OVNKubernetes
    
```
Add this as the first custom manifest.  The file name can be anything.  The name 99-cni-configuration.yaml was used for this example.  

Once added, continue with the installation.

## Enable Cilium as the replacement CNI

When installation completes, log into the OpenShift web console.
1. Navigate to the OperatorHub
2. Search for Cilium
3. Install the Cilium CNI onto the openshift-network-operator namespace
4. Configure Cilium (Below is a sample configuration yaml file)

```code block

kind: CiliumConfig
apiVersion: cilium.io/v1alpha1
metadata:
  name: cilium-openshift-default
  namespace: openshift-network-operator
spec:
  # Enable the CNI plugin
  cni:
    enabled: true
    customConf: true  

  # Define Cluster and Service Networks to match OpenShift
  cluster:
    name: # name of the openshift cluster
    pool:
      podCIDR: "10.128.0.0/14"      # Pod network CIDR
      serviceCIDR: "172.30.0.0/16"  # Service network CIDR

  # Enable specific configurations for OpenShift compatibility
  endpointHealthChecking: true
  hostReachableServices: true

  # Additional Cilium features and configurations
  ipv4:
    enabled: true  # Enable IPv4 support (required for OpenShift)
  ipv6:
    enabled: false # Disable IPv6 if not required

  # Enable Hubble for visibility and troubleshooting (optional)
  hubble:
    enabled: true
    ui: true  # Enable Hubble UI (optional)
    metrics:
      - dns
      - drop
      - tcp
      - flow
      - icmp
      - http

  # Cilium Network Policies (CNP) and Network Policy enforcement
  policyEnforcementMode: default

  # Enable BPF (Berkeley Packet Filter) options for better performance
  bpf:
    nodePort:
      enabled: true
    preallocation: true

  # Configure MTU based on your network infrastructure
  mtu: 1400

```
