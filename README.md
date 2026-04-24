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

Kubernetes CRDs:

*NicClusterPolicy* to LCM driver and device plugin states (see snap below):
- doca-ofed: drivers for BF NICs
  - Deploys the DOCA/OFED kernel driver as a DaemonSet on each node.
- rmashareddeviceplugin
  - Deploys the RDMA device plugin, which advertises RDMA Host Channel Adapter (HCA = rough equivalent of a NIC in IB) to the K8s device API.
  - The HCA handles memory registration, QP (Queue Pair) management, and DMA directly in hardware, bypassing the CPU/kernel for data transfers
rdmaHcaMax: 63 means up to 63 pods can share RDMA contexts on a single physical HCA. rdmaHcaMax: 63 means up to 63 pods can share RDMA contexts on a single physical HCA (bit

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
*NicConfigurationTemplate* to configure NICs

Fields
- nicSelector.nicTypePCI device ID — mandatory, single type per template
- nodeSelector Limits which nodes the template applies to
- resetToDefault: true Nuke all config back to FW defaults (useful for cleanup)
- roceOptimized + pfcCritical for RoCEv2 lossless fabric — maps to DCQCN config
- gpuDirectOptimizedEnables PCIe peer-to-peer for GPU↔NIC GPUDirect RDMA
- multiplaneMode: hwplbConnectX-8 only — multi-rail hardware load balancing for Spectrum-X

```
#### Standard ConnectX-6 example

apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicConfigurationTemplate
metadata:
  name: connectx6-roce-config
  namespace: nvidia-network-operator
spec:
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"

  nicSelector:
    nicType: "101b"           # ConnectX-6 PCI device ID
    # pciAddresses:           # optional: pin to specific slots
    #   - "0000:03:00.0"
    # serialNumbers:          # optional: pin to specific cards
    #   - "MT2116X09299"

  resetToDefault: false       # if true, ignores template, resets FW to defaults

  template:
    numVfs: 4
    linkType: Ethernet        # or InfiniBand

    pciPerformanceOptimized:
      enabled: true
      maxAccOutRead: 44       # PCIe outstanding reads tuning

    roceOptimized:
      enabled: true
      qos:
        trust: dscp           # DSCP-based QoS (vs pcp)
        pfc: "0,0,0,1,0,0,0,0"  # PFC enabled on TC3

    gpuDirectOptimized:
      enabled: true
      env: Baremetal          # or "HPC"

#### SpectrumX / ConnectX

apiVersion: configuration.net.nvidia.com/v1alpha1
kind: NicConfigurationTemplate
metadata:
  name: spectrum-x-multiplane
  namespace: nvidia-network-operator
spec:
  nicSelector:
    nicType: "1023"           # ConnectX-8
  template:
    numVfs: 1
    linkType: Ethernet
    spectrumXOptimized:
      enabled: true
      version: "RA2.1"
      overlay: "none"
      multiplaneMode: "hwplb" # Hardware Packet Load Balancing
      numberOfPlanes: 4
```
