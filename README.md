# NVIDIA-Networking

## Frameworks

Two frameworks:
- Infiniband: proprietary technology credit-based. Credits are control by hardware (some kind of LLC2 at hardware). Support for Scalable Hierarchical Aggregation and Reduction Protocol (SHARP) for in-network compute
- SpectrumX: Ethernet 

## NICs

### ConnectX
RDMA-capable but no DPU

### BlueField
DPU-capable NICs (mellanox).
DOCA Platform Framework (DPF) to provision and orchestrate BlueField DPUs in cloud environments.

#### Check Status
-	ibv_devinfo
-	doca-info
  
#### Management

Use Kubernetes NicClusterPolicy CRD to LCM driver and device plugin states. 
- rmashareddeviceplugin
  - The HCA handles memory registration, QP (Queue Pair) management, and DMA directly in hardware, bypassing the CPU/kernel for data transfers
rdmaHcaMax: 63 means up to 63 pods can share RDMA contexts on a single physical HCA
- doca-ofed: drivers for BF NICs
  - 
```
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    image: doca-driver
    repository: nvcr.io/nvidia/mellanox
    version: "24.10-0.7.0.0-0"
    forcePrecompiled: false

  rdmaSharedDevicePlugin:
    image: k8s-rdma-shared-dev-plugin
    repository: ghcr.io/mellanox
    version: latest
    resources:
      - name: rdma_shared_device_a
        rdmaHcaMax: 63
        selectors:
          vendors: ["15b3"]

  sriovDevicePlugin:
    image: sriov-network-device-plugin
    repository: ghcr.io/k8snetworkplumbingwg
    version: latest
    resources:
      - name: sriov_rdma
        vendors: ["15b3"]
        isRdma: true

  ibKubernetes:
    image: ib-kubernetes
    repository: ghcr.io/mellanox
    version: latest
    pKeyGUIDPoolRangeStart: "02:00:00:00:00:00:00:00"
    pKeyGUIDPoolRangeEnd: "02:FF:FF:FF:FF:FF:FF:FF"
    ufmSecret: ufm-secret

  nvIpam:
    image: nvidia-k8s-ipam
    repository: ghcr.io/mellanox
    version: latest
    enableWebhook: false

  nicFeatureDiscovery:
    image: nic-feature-discovery
    repository: ghcr.io/mellanox
    version: latest

  docaTelemetryService:
    image: doca_telemetry
    repository: nvcr.io/nvidia/doca
    version: latest
```
