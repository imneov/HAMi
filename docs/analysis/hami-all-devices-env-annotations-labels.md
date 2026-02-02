# HAMi 全设备类型：环境变量、Annotation、Label、Resource 完整清单

HAMi 支持 14 种设备类型，每种设备通过 **Annotation** 传递调度信息，通过 **Resource** 声明资源需求，
部分设备通过 **环境变量** 控制运行时行为。本文档逐一列出每种设备的所有元数据。

---

## 通用 Annotation（所有设备共享）

**文件：** `pkg/util/types.go`

| Annotation Key | 常量名 | 值示例 | 用途 |
|---------------|--------|--------|------|
| `hami.io/vgpu-time` | `AssignedTimeAnnotations` | `1706000000` | 调度器分配设备的时间戳 |
| `hami.io/vgpu-node` | `AssignedNodeAnnotations` | `node-1` | 调度器选定的目标节点 |
| `hami.io/bind-time` | `BindTimeAnnotations` | `1706000001` | Pod 绑定时间 |
| `hami.io/bind-phase` | `DeviceBindPhase` | `allocating`/`failed`/`success` | 设备绑定阶段 |
| `hami.io/node-scheduler-policy` | `NodeSchedulerPolicyAnnotationKey` | `binpack`/`spread` | 节点调度策略覆盖 |
| `hami.io/gpu-scheduler-policy` | `GPUSchedulerPolicyAnnotationKey` | `binpack`/`spread`/`topology-aware` | GPU 调度策略覆盖 |

---

## 1. NVIDIA GPU

**文件：** `pkg/device/nvidia/device.go`, `pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go`

### Resource（Pod 资源请求）

| Resource Name | 可配置 | 用途 |
|--------------|--------|------|
| `nvidia.com/gpu` | ResourceCountName | GPU 数量 |
| `nvidia.com/gpumem` | ResourceMemoryName | 显存（MB） |
| `nvidia.com/gpumem-percentage` | ResourceMemoryPercentageName | 显存百分比（0-100） |
| `nvidia.com/gpucores` | ResourceCoreName | 算力百分比（0-100） |
| `nvidia.com/priority` | ResourcePriority | 任务优先级 |

### Annotation — 节点级

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/node-handshake` | `HandshakeAnnos` | 节点握手状态 |
| `hami.io/node-nvidia-register` | `RegisterAnnos` | 节点 GPU 设备列表（编码） |
| `hami.io/node-nvidia-score` | `RegisterGPUPairScore` | GPU 拓扑打分 |
| `hami.io/mutex.lock` | `NodeLockNvidia` | 节点互斥锁 |

### Annotation — Pod 级（调度分配）

| Annotation Key | 常量名/Map | 用途 |
|---------------|-----------|------|
| `hami.io/vgpu-devices-to-allocate` | `InRequestDevices["NVIDIA"]` | 待分配设备（调度器写入） |
| `hami.io/vgpu-devices-allocated` | `SupportDevices["NVIDIA"]` | 已分配设备（device-plugin 读取） |

### Annotation — Pod 级（用户指定过滤）

| Annotation Key | 常量名 | 值示例 | 用途 |
|---------------|--------|--------|------|
| `nvidia.com/use-gputype` | `GPUInUse` | `A100,H100` | GPU 类型白名单 |
| `nvidia.com/nouse-gputype` | `GPUNoUse` | `T4` | GPU 类型黑名单 |
| `nvidia.com/use-gpuuuid` | `GPUUseUUID` | `GPU-xxxx` | GPU UUID 白名单 |
| `nvidia.com/nouse-gpuuuid` | `GPUNoUseUUID` | `GPU-yyyy` | GPU UUID 黑名单 |
| `nvidia.com/numa-bind` | `NumaBind` | `true` | 强制 NUMA 绑定 |
| `nvidia.com/vgpu-mode` | `AllocateMode` | `hami-core,mig,mps` | 分配模式过滤 |

### 环境变量 — Device Plugin Allocate() 注入

| 环境变量 | 来源 | 值示例 | 用途 |
|---------|------|--------|------|
| `NVIDIA_VISIBLE_DEVICES` | `server.go:752` | `GPU-xxxx,GPU-yyyy` | 容器可见 GPU |
| `CUDA_DEVICE_MEMORY_LIMIT_0` | `server.go:535` | `4096m` | 第 0 号设备显存上限 |
| `CUDA_DEVICE_MEMORY_LIMIT_1` | `server.go:535` | `2048m` | 第 1 号设备显存上限（依次类推 `_N`） |
| `CUDA_DEVICE_SM_LIMIT` | `server.go:538` | `50` | SM 算力限制百分比 |
| `CUDA_DEVICE_MEMORY_SHARED_CACHE` | `server.go:539` | `/path/vgpu/<uuid>.cache` | vGPU 显存缓存文件 |
| `CUDA_OVERSUBSCRIBE` | `server.go:541` | `true` | 显存超分（DeviceMemoryScaling > 1） |
| `LIBCUDA_LOG_LEVEL` | `server.go:544` | `0`/`1`/`3`/`4` | 日志级别 |
| `GPU_CORE_UTILIZATION_POLICY` | `server.go:547` | `disable` | 算力限制策略 |
| `NVIDIA_GDRCOPY` | `server.go:638` | `enabled` | GPUDirect RDMA Copy |
| `NVIDIA_GDS` | `server.go:641` | `enabled` | GPUDirect Storage |
| `NVIDIA_MOFED` | `server.go:644` | `enabled` | Mellanox OFED |
| `NVIDIA_IMEX_CHANNELS` | `server.go:762` | `0,1,2` | IMEX 通道 |

### 环境变量 — Webhook MutateAdmission 注入

| 环境变量 | 来源 | 值示例 | 用途 |
|---------|------|--------|------|
| `CUDA_TASK_PRIORITY` | `nvidia/device.go:346` | 优先级值 | CUDA 任务优先级 |
| `GPU_CORE_UTILIZATION_POLICY` | `nvidia/device.go:354` | `force`/`disable`/`default` | 节点级算力策略 |
| `NVIDIA_VISIBLE_DEVICES` | `nvidia/device.go:373` | `none` | 屏蔽 GPU（OverwriteEnv=true 时） |

### 用户可设置的环境变量（HAMi 读取但不注入）

| 环境变量 | 来源 | 用途 |
|---------|------|------|
| `CUDA_DISABLE_CONTROL` | 用户在 Pod spec 设置 | `true` 时不挂载 ld.so.preload，禁用 vGPU 拦截 |

---

## 2. Hygon DCU

**文件：** `pkg/device/hygon/device.go`

### Resource

| Resource Name | 可配置 Flag | 用途 |
|--------------|------------|------|
| `hygon.com/dcunum` | `--dcu-name` | DCU 数量 |
| `hygon.com/dcumem` | `--dcu-memory` | DCU 显存（MB） |
| `hygon.com/dcucores` | `--dcu-cores` | DCU 算力百分比 |

### Annotation — 节点级

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/node-handshake-dcu` | `HandshakeAnnos` | 节点握手 |
| `hami.io/node-dcu-register` | `RegisterAnnos` | 节点 DCU 设备列表 |
| `hami.io/mutex.lock` | `NodeLockDCU` | 节点互斥锁 |

### Annotation — Pod 级（调度分配）

| Annotation Key | Map | 用途 |
|---------------|-----|------|
| `hami.io/dcu-devices-to-allocate` | `InRequestDevices["DCU"]` | 待分配 |
| `hami.io/dcu-devices-allocated` | `SupportDevices["DCU"]` | 已分配 |

### Annotation — Pod 级（用户过滤）

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hygon.com/use-dcutype` | `DCUInUse` | DCU 类型白名单 |
| `hygon.com/nouse-dcutype` | `DCUNoUse` | DCU 类型黑名单 |
| `hygon.com/use-gpuuuid` | `DCUUseUUID` | DCU UUID 白名单 |
| `hygon.com/nouse-gpuuuid` | `DCUNoUseUUID` | DCU UUID 黑名单 |

### 环境变量

**无** — Hygon DCU 不通过环境变量注入容器。

---

## 3. Cambricon MLU

**文件：** `pkg/device/cambricon/device.go`

### Resource

| Resource Name | 可配置 Flag | 用途 |
|--------------|------------|------|
| `cambricon.com/mlu` | `--cambricon-mlu-name` | MLU 数量 |
| `cambricon.com/mlu.smlu.vmemory` | `--cambricon-mlu-memory` | MLU 显存（256MB 单位） |
| `cambricon.com/mlu.smlu.vcore` | `--cambricon-mlu-cores` | MLU 算力百分比 |

### Annotation — Pod 级（调度分配）

| Annotation Key | Map | 用途 |
|---------------|-----|------|
| `hami.io/cambricon-mlu-devices-to-allocate` | `InRequestDevices["MLU"]` | 待分配 |
| `hami.io/cambricon-mlu-devices-allocated` | `SupportDevices["MLU"]` | 已分配 |

### Annotation — Pod 级（用户过滤）

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `cambricon.com/use-mlutype` | `MLUInUse` | MLU 类型白名单 |
| `cambricon.com/nouse-mlutype` | `MLUNoUse` | MLU 类型黑名单 |
| `cambricon.com/use-gpuuuid` | `MLUUseUUID` | MLU UUID 白名单 |
| `cambricon.com/nouse-gpuuuid` | `MLUNoUseUUID` | MLU UUID 黑名单 |
| `cambricon.com/dsmlu.lock` | `DsmluLockTime` | DSMLU 调度锁时间戳 |

### 环境变量常量（用于 Annotation key，非容器环境变量）

| 常量名 | 值 | 用途 |
|-------|-----|------|
| `MluMemSplitLimit` | `CAMBRICON_SPLIT_MEMS` | 显存分割限制 |
| `MluMemSplitIndex` | `CAMBRICON_SPLIT_VISIBLE_DEVICES` | 可见设备索引 |
| `MluMemSplitEnable` | `CAMBRICON_SPLIT_ENABLE` | 显存分割开关 |
| `DsmluProfile` | `CAMBRICON_DSMLU_PROFILE` | DSMLU 配置文件（idx_cores_memory） |
| `DsmluResourceAssigned` | `CAMBRICON_DSMLU_ASSIGNED` | DSMLU 资源分配状态 |

---

## 4. Iluvatar (天数智芯)

**文件：** `pkg/device/iluvatar/device.go`

> 注：Iluvatar 按芯片型号动态注册，`{cw}` = commonWord（如 `MR-V100`, `BI-V150`）

### Resource

| Resource Name 模式 | 示例 | 用途 |
|-------------------|------|------|
| `iluvatar.ai/{cw}-vgpu` | `iluvatar.ai/MR-V100-vgpu` | 设备数量 |
| `iluvatar.ai/{cw}.vMem` | `iluvatar.ai/MR-V100.vMem` | 显存 |
| `iluvatar.ai/{cw}.vCore` | `iluvatar.ai/MR-V100.vCore` | 算力 |

### Annotation — 节点级

| Annotation Key 模式 | 示例 | 用途 |
|--------------------|------|------|
| `hami.io/node-{cw}-register` | `hami.io/node-MR-V100-register` | 节点设备列表 |
| `hami.io/node-handshake-{cw}` | `hami.io/node-handshake-MR-V100` | 节点握手 |

### Annotation — Pod 级（调度分配）

| Annotation Key 模式 | Map | 用途 |
|--------------------|-----|------|
| `hami.io/{cw}-devices-to-allocate` | `InRequestDevices[cw]` | 待分配 |
| `hami.io/{cw}-devices-allocated` | `SupportDevices[cw]` | 已分配 |

### Annotation — Pod 级（用户过滤）

| Annotation Key 模式 | 用途 |
|--------------------|------|
| `hami.io/use-{cw}-uuid` | UUID 白名单 |
| `hami.io/no-use-{cw}-uuid` | UUID 黑名单 |

### 环境变量

| 环境变量 | 来源 | 值 | 用途 |
|---------|------|-----|------|
| `SOL_CONTINER_NAME` | `iluvatar/device.go:101` | 容器名 | 传递容器名给运行时（拼写错误保留向后兼容） |

### 支持的芯片型号

`MR-V100`, `MR-V50`, `BI-V150`, `BI-V100`

---

## 5. Ascend (华为昇腾)

**文件：** `pkg/device/ascend/device.go`

> 注：Ascend 按芯片型号动态注册，`{cw}` = commonWord（如 `Ascend910A`, `Ascend910B`）

### Resource

| Resource Name 模式 | 示例 | 用途 |
|-------------------|------|------|
| `huawei.com/{cw}` | `huawei.com/Ascend910A` | vNPU 数量 |
| `huawei.com/{cw}-memory` | `huawei.com/Ascend910A-memory` | vNPU 显存 |

### Annotation — 节点级

| Annotation Key 模式 | 用途 |
|--------------------|------|
| `hami.io/node-register-{cw}` | 节点设备列表（JSON） |
| `hami.io/node-handshake-{cw}` | 节点握手 |
| `hami.io/mutex.lock` | 节点互斥锁 |

### Annotation — Pod 级（调度分配）

| Annotation Key 模式 | Map | 用途 |
|--------------------|-----|------|
| `hami.io/{cw}-devices-to-allocate` | `InRequestDevices[cw]` | 待分配 |
| `hami.io/{cw}-devices-allocated` | `SupportDevices[cw]` | 已分配 |

### Annotation — Pod 级（用户过滤 + 厂商特有）

| Annotation Key | 用途 |
|---------------|------|
| `hami.io/use-{cw}-uuid` | UUID 白名单 |
| `hami.io/no-use-{cw}-uuid` | UUID 黑名单 |
| `predicate-time` | 分配决策时间戳 |
| `huawei.com/{cw}` | 华为厂商运行时信息（UUID + 模板名 JSON） |

### 环境变量

**无** — Ascend 不通过环境变量注入容器。

---

## 6. Mthreads (摩尔线程)

**文件：** `pkg/device/mthreads/device.go`

### Resource

| Resource Name | 可配置 Flag | 用途 |
|--------------|------------|------|
| `mthreads.com/vgpu` | `--mthreads-name` | vGPU 数量 |
| `mthreads.com/sgpu-memory` | `--mthreads-memory` | sGPU 显存 |
| `mthreads.com/sgpu-core` | `--mthreads-cores` | sGPU 算力 |

### Annotation — Pod 级（调度分配）

| Annotation Key | Map | 用途 |
|---------------|-----|------|
| `hami.io/mthreads-vgpu-devices-to-allocate` | `InRequestDevices["Mthreads"]` | 待分配 |
| `hami.io/mthreads-vgpu-devices-allocated` | `SupportDevices["Mthreads"]` | 已分配 |

### Annotation — Pod 级（用户过滤 + 厂商特有）

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `mthreads.ai/use-gpuuuid` | `MthreadsUseUUID` | UUID 白名单 |
| `mthreads.ai/nouse-gpuuuid` | `MthreadsNoUseUUID` | UUID 黑名单 |
| `mthreads.com/gpu-index` | `MthreadsAssignedGPUIndex` | 分配的 GPU 索引 |
| `mthreads.com/predicate-node` | `MthreadsAssignedNode` | 分配的节点名 |
| `mthreads.com/predicate-time` | `MthreadsPredicateTime` | 分配时间（纳秒） |
| `mthreads.com/request-gpu-num` | — | 请求的 GPU 数量（>1 时设置） |

### 环境变量

**无** — Mthreads 不通过环境变量注入容器。

---

## 7. Metax GPU (沐曦)

**文件：** `pkg/device/metax/device.go`, `pkg/device/metax/sdevice.go`, `pkg/device/metax/protocol.go`

### 7a. Metax 物理 GPU

#### Resource

| Resource Name | 可配置 Flag | 用途 |
|--------------|------------|------|
| `metax-tech.com/gpu` | `--metax-name` | GPU 数量 |

#### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/metax-gpu-devices-to-allocate` | `InRequestDevices["Metax-GPU"]` | 待分配 |
| `hami.io/metax-gpu-devices-allocated` | `SupportDevices["Metax-GPU"]` | 已分配 |
| `metax-tech.com/gpu.topology.losses` | `MetaxAnnotationLoss` | 节点拓扑损耗（JSON） |
| `metax-tech.com/gpu.topology.scores` | `MetaxAnnotationScore` | 节点拓扑打分（JSON） |

### 7b. Metax SGPU（虚拟化）

#### Resource

| Resource Name | 可配置 Flag | 用途 |
|--------------|------------|------|
| `metax-tech.com/sgpu` | `--metax-vcount` | SGPU 数量 |
| `metax-tech.com/vcore` | `--metax-vcore` | 虚拟算力 |
| `metax-tech.com/vmemory` | `--metax-vmemory` | 虚拟显存 |

#### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/metax-sgpu-devices-to-allocate` | `InRequestDevices["Metax-SGPU"]` | 待分配 |
| `hami.io/metax-sgpu-devices-allocated` | `SupportDevices["Metax-SGPU"]` | 已分配 |
| `hami.io/mutex.lock` | `MetaxNodeLock` | 节点互斥锁 |
| `metax-tech.com/node-gpu-devices` | `MetaxSDeviceAnno` | 节点 SGPU 设备列表 |
| `metax-tech.com/gpu-devices-allocated` | `MetaxAllocatedSDevices` | 已分配设备（厂商格式） |
| `metax-tech.com/predicate-time` | `MetaxPredicateTime` | 分配时间 |
| `metax-tech.com/use-gpuuuid` | `MetaxUseUUID` | UUID 白名单 |
| `metax-tech.com/nouse-gpuuuid` | `MetaxNoUseUUID` | UUID 黑名单 |
| `metax-tech.com/sgpu-qos-policy` | `MetaxSGPUQosPolicy` | QoS 策略（`best-effort`/`fixed-share`/`burst-share`） |
| `metax-tech.com/sgpu-topology-aware` | `MetaxSGPUTopologyAware` | 拓扑感知开关 |
| `metax-tech.com/sgpu-app-class` | `MetaxSGPUAppClass` | 应用类型（`online`/`offline`） |

### 环境变量

**无** — Metax 不通过环境变量注入容器。

---

## 8. Kunlun 物理 GPU (百度昆仑)

**文件：** `pkg/device/kunlun/device.go`

### Resource

| Resource Name | 用途 |
|--------------|------|
| 可配置 | GPU 数量 |

### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/kunlun-allocated` | `SupportDevices["kunlun"]` | 已分配 |
| `baidu.com/use-gpuuuid` | `KunlunUseUUID` | UUID 白名单 |
| `baidu.com/nouse-gpuuuid` | `KunlunNoUseUUID` | UUID 黑名单 |

### 环境变量

| 环境变量 | 常量名 | 用途 |
|---------|--------|------|
| `BAIDU_COM_DEVICE_IDX` | `KunlunDeviceSelection` | 分配的设备索引 |

---

## 9. Kunlun XPU（虚拟化）

**文件：** `pkg/device/kunlun/vdevice.go`

### Resource

| Resource Name | 用途 |
|--------------|------|
| `KunlunResourceVCount`（可配置） | XPU 数量 |
| `KunlunResourceVMemory`（可配置） | XPU 显存 |

### Annotation

| Annotation Key | Map/常量 | 用途 |
|---------------|---------|------|
| `hami.io/xpu-devices-to-allocate` | `InRequestDevices["XPU"]` | 待分配 |
| `hami.io/xpu-devices-allocated` | `SupportDevices["XPU"]` | 已分配 |
| `hami.io/node-handshake-xpu` | `HandshakeAnnos["XPU"]` | 节点握手 |
| `hami.io/node-register-xpu` | `RegisterAnnos` | 节点设备列表 |
| `hami.io/use-xpu-uuid` | `UseUUIDAnno` | UUID 白名单 |
| `hami.io/no-use-xpu-uuid` | `NoUseUUIDAnno` | UUID 黑名单 |
| `hami.io/mutex.lock` | `NodeLock` | 节点互斥锁 |

### 环境变量

**无**

---

## 10. Enflame (燧原)

**文件：** `pkg/device/enflame/device.go`, `pkg/device/enflame/gcu.go`

### 10a. Enflame 共享 vGCU

#### Resource

| Resource Name | 用途 |
|--------------|------|
| `enflame.com/shared-gcu` / `EnflameResourceNameVGCU`（可配置） | 共享 GCU |
| `enflame.com/gcu-count` / `CountNoSharedName` | 非共享 GCU 数量 |
| `EnflameResourceNameVGCUPercentage`（可配置） | GCU 百分比 |

#### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/enflame-vgpu-devices-allocated` | `SupportDevices["Enflame"]` | 已分配 |
| `enflame.com/use-gpuuuid` | `EnflameUseUUID` | UUID 白名单 |
| `enflame.com/nouse-gpuuuid` | `EnflameNoUseUUID` | UUID 黑名单 |
| `enflame.com/gcu-request-size` | `PodRequestGCUSize` | 请求的 GCU 大小 |
| `enflame.com/gcu-assigned-id` | `PodAssignedGCUID` | 分配的 GCU ID |
| `enflame.com/gcu-assigned` | `PodHasAssignedGCU` | 是否已分配 |
| `enflame.com/gcu-assigned-time` | `PodAssignedGCUTime` | 分配时间 |
| `enflame.com/gcu-shared-capacity` | `GCUSharedCapacity` | 共享容量 |

### 10b. Enflame 物理 GCU

#### Annotation

| Annotation Key | Map | 用途 |
|---------------|-----|------|
| `hami.io/enflame-gcu-devices-to-allocate` | `InRequestDevices["GCU"]` | 待分配 |
| `hami.io/enflame-gcu-devices-allocated` | `SupportDevices["GCU"]` | 已分配 |

### 环境变量

**无**

---

## 11. AWS Neuron

**文件：** `pkg/device/awsneuron/device.go`

### Resource

| Resource Name | 用途 |
|--------------|------|
| ResourceCountName（可配置） | Neuron 数量 |
| ResourceCoreName（可配置） | Neuron 算力 |

### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/aws-neuron-devices-allocated` | `SupportDevices["AWSNeuron"]` | 已分配 |
| `aws.amazon.com/neuron-index` | `AWSNeuronDeviceSelection` | 设备索引 |
| `aws.amazon.com/use-neuron-uuid` | `AWSNeuronUseUUID` | UUID 白名单 |
| `aws.amazon.com/nouse-neuron-uuid` | `AWSNeuronNoUseUUID` | UUID 黑名单 |
| `aws.amazon.com/predicate-node` | `AWSNeuronAssignedNode` | 分配的节点 |

### 环境变量

| 环境变量 | 常量名 | 用途 |
|---------|--------|------|
| `AWS_NEURON_IDS` | `AWSNeuronAssignedIndex` | 分配的 Neuron ID |
| `NEURON_ALLOC_TIME` | `AWSNeuronPredicateTime` | 分配时间 |
| `NEURON_RESOURCE_TYPE` | `AWSNeuronResourceType` | 资源类型 |
| `NEURON_ALLOCATED` | `AWSNeuronAllocated` | 分配状态 |

---

## 12. AMD GPU

**文件：** `pkg/device/amd/device.go`

### Resource

| Resource Name | 用途 |
|--------------|------|
| ResourceCountName（可配置） | GPU 数量 |
| ResourceMemoryName（可配置） | GPU 显存 |

### Annotation

| Annotation Key | 常量名 | 用途 |
|---------------|--------|------|
| `hami.io/amd-devices-allocated` | `SupportDevices["AMDGPU"]` | 已分配 |
| `amd.com/gpu-index` | `AMDDeviceSelection` | GPU 索引 |
| `amd.com/use-gpu-uuid` | `AMDUseUUID` | UUID 白名单 |
| `amd.com/nouse-gpu-uuid` | `AMDNoUseUUID` | UUID 黑名单 |
| `amd.com/predicate-node` | `AMDAssignedNode` | 分配的节点 |

### 环境变量

**无**

---

## 总结对比表

### 各设备 InRequestDevices / SupportDevices 注册

| 设备类型 | 设备 Key | InRequestDevices | SupportDevices |
|---------|---------|-----------------|----------------|
| NVIDIA | `NVIDIA` | `hami.io/vgpu-devices-to-allocate` | `hami.io/vgpu-devices-allocated` |
| Hygon DCU | `DCU` | `hami.io/dcu-devices-to-allocate` | `hami.io/dcu-devices-allocated` |
| Cambricon | `MLU` | `hami.io/cambricon-mlu-devices-to-allocate` | `hami.io/cambricon-mlu-devices-allocated` |
| Iluvatar | `{cw}` | `hami.io/{cw}-devices-to-allocate` | `hami.io/{cw}-devices-allocated` |
| Ascend | `{cw}` | `hami.io/{cw}-devices-to-allocate` | `hami.io/{cw}-devices-allocated` |
| Mthreads | `Mthreads` | `hami.io/mthreads-vgpu-devices-to-allocate` | `hami.io/mthreads-vgpu-devices-allocated` |
| Metax GPU | `Metax-GPU` | `hami.io/metax-gpu-devices-to-allocate` | `hami.io/metax-gpu-devices-allocated` |
| Metax SGPU | `Metax-SGPU` | `hami.io/metax-sgpu-devices-to-allocate` | `hami.io/metax-sgpu-devices-allocated` |
| Kunlun XPU | `XPU` | `hami.io/xpu-devices-to-allocate` | `hami.io/xpu-devices-allocated` |
| Enflame GCU | `GCU` | `hami.io/enflame-gcu-devices-to-allocate` | `hami.io/enflame-gcu-devices-allocated` |
| Enflame vGCU | `Enflame` | — | `hami.io/enflame-vgpu-devices-allocated` |
| Kunlun 物理 | `kunlun` | — | `hami.io/kunlun-allocated` |
| AWS Neuron | `AWSNeuron` | — | `hami.io/aws-neuron-devices-allocated` |
| AMD | `AMDGPU` | — | `hami.io/amd-devices-allocated` |

### 各设备环境变量注入情况

| 设备类型 | 注入环境变量数 | 注入方式 |
|---------|-------------|---------|
| **NVIDIA** | **15** | Device Plugin Allocate() + Webhook MutateAdmission |
| **AWS Neuron** | **4** | Webhook |
| **Iluvatar** | **1** | Webhook MutateAdmission |
| **Kunlun 物理** | **1** | Webhook |
| Hygon DCU | 0 | — |
| Cambricon MLU | 0 | — |
| Ascend | 0 | — |
| Mthreads | 0 | — |
| Metax | 0 | — |
| Enflame | 0 | — |
| AMD | 0 | — |

### UUID 过滤 Annotation 命名模式

| 设备类型 | 白名单 | 黑名单 |
|---------|--------|--------|
| NVIDIA | `nvidia.com/use-gpuuuid` | `nvidia.com/nouse-gpuuuid` |
| Hygon | `hygon.com/use-gpuuuid` | `hygon.com/nouse-gpuuuid` |
| Cambricon | `cambricon.com/use-gpuuuid` | `cambricon.com/nouse-gpuuuid` |
| Iluvatar | `hami.io/use-{cw}-uuid` | `hami.io/no-use-{cw}-uuid` |
| Ascend | `hami.io/use-{cw}-uuid` | `hami.io/no-use-{cw}-uuid` |
| Mthreads | `mthreads.ai/use-gpuuuid` | `mthreads.ai/nouse-gpuuuid` |
| Metax SGPU | `metax-tech.com/use-gpuuuid` | `metax-tech.com/nouse-gpuuuid` |
| Kunlun | `baidu.com/use-gpuuuid` | `baidu.com/nouse-gpuuuid` |
| Kunlun XPU | `hami.io/use-xpu-uuid` | `hami.io/no-use-xpu-uuid` |
| Enflame | `enflame.com/use-gpuuuid` | `enflame.com/nouse-gpuuuid` |
| AWS Neuron | `aws.amazon.com/use-neuron-uuid` | `aws.amazon.com/nouse-neuron-uuid` |
| AMD | `amd.com/use-gpu-uuid` | `amd.com/nouse-gpu-uuid` |
