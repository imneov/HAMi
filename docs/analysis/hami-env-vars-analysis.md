# HAMi 环境变量设置机制完整分析

## 概述

HAMi 通过 **调度器(Scheduler) → Pod Annotations → Device Plugin → 容器环境变量** 的完整流水线，
实现 GPU 虚拟化的资源限制。核心思路是：调度器做决策，写入 annotation；device-plugin 读取 annotation，
转化为容器环境变量和 volume mount；容器内的 `libvgpu.so` 拦截 CUDA 调用，根据环境变量执行限制。

---

## 第一阶段：调度器 Filter（决策 + 写 Annotation）

**文件：** `pkg/scheduler/scheduler.go` — `Filter()` 函数（第644行）

### 流程

```
Pod 请求资源 → Filter() → calcScore() → 选最优节点 → PatchAnnotations() → 写入 Pod Annotation
```

### 关键代码

```go
// scheduler.go:644-716
func (s *Scheduler) Filter(args extenderv1.ExtenderArgs) (*extenderv1.ExtenderFilterResult, error) {
    // 1. 解析 Pod 的资源需求（每个容器需要多少 GPU、显存、算力）
    resourceReqs := device.Resourcereqs(args.Pod)

    // 2. 获取所有节点的设备使用情况
    nodeUsage, failedNodes, _ := s.getNodesUsage(args.NodeNames, args.Pod)

    // 3. 对每个节点打分，找出最优节点和设备分配方案
    nodeScores, _ := s.calcScore(nodeUsage, resourceReqs, args.Pod, failedNodes)

    // 4. 选最高分节点
    sort.Sort(nodeScores)
    m := (*nodeScores).NodeList[len((*nodeScores).NodeList)-1]

    // 5. **关键：将分配结果写入 Pod Annotation**
    annotations := make(map[string]string)
    annotations[util.AssignedNodeAnnotations] = m.NodeID          // "hami.io/vgpu-node"
    annotations[util.AssignedTimeAnnotations] = strconv.FormatInt(time.Now().Unix(), 10)

    for _, val := range device.GetDevices() {
        val.PatchAnnotations(args.Pod, &annotations, m.Devices)   // 各设备类型写自己的 annotation
    }
    util.PatchPodAnnotations(args.Pod, annotations)
}
```

### NVIDIA PatchAnnotations 实现

**文件：** `pkg/device/nvidia/device.go:504-514`

```go
func (dev *NvidiaGPUDevices) PatchAnnotations(pod *corev1.Pod, annoinput *map[string]string, pd device.PodDevices) {
    devlist, ok := pd[NvidiaGPUDevice]
    if ok && len(devlist) > 0 {
        deviceStr := device.EncodePodSingleDevice(devlist)
        // 写入两个 annotation key：
        (*annoinput)[device.InRequestDevices[NvidiaGPUDevice]] = deviceStr  // "hami.io/vgpu-devices-to-allocate"
        (*annoinput)[device.SupportDevices[NvidiaGPUDevice]] = deviceStr    // "hami.io/vgpu-devices-allocated"
    }
}
```

### Annotation 数据格式

设备分配信息序列化为字符串，格式为：

```
GPU-UUID_<idx>,<usedmem>,<usedcores>,<type>:GPU-UUID_<idx>,<usedmem>,<usedcores>,<type>;容器2设备...
```

- `:` 分隔同一容器内的多个设备
- `;` 分隔不同容器

---

## 第二阶段：Device Plugin Allocate（读 Annotation + 设环境变量）

**文件：** `pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go` — `Allocate()` 函数（第475行）

### 流程

```
Kubelet 调用 Allocate() → GetNextDeviceRequest() 读 annotation
    → 设置环境变量 → 挂载 volume → EraseNextDeviceTypeFromAnnotation()
```

### 核心代码（第486-601行）

```go
func (plugin *NvidiaDevicePlugin) Allocate(ctx context.Context, reqs *kubeletdevicepluginv1beta1.AllocateRequest) (*kubeletdevicepluginv1beta1.AllocateResponse, error) {
    current, _ := util.GetPendingPod(ctx, nodename)

    for idx, req := range reqs.ContainerRequests {
        // 1. 从 Pod Annotation 读取调度器的分配决策
        currentCtr, devreq, _ := GetNextDeviceRequest(nvidia.NvidiaGPUDevice, *current)

        // 2. 获取基础 allocate response（包含 NVIDIA_VISIBLE_DEVICES）
        response, _ := plugin.getAllocateResponse(plugin.GetContainerDeviceStrArray(devreq))

        // 3. 标记该容器已处理
        EraseNextDeviceTypeFromAnnotation(nvidia.NvidiaGPUDevice, *current)

        // 4. **核心：设置 HAMi 特有的环境变量**
        if plugin.operatingMode != "mig" {
            // ... 见下方详细列表
        }
    }
}
```

---

## 完整环境变量列表

### 一、Device Plugin 在 Allocate() 中设置的环境变量

| 环境变量 | 设置位置 | 值示例 | 用途 |
|---------|---------|--------|------|
| `NVIDIA_VISIBLE_DEVICES` | `server.go:752` | `GPU-xxxx-yyyy,GPU-aaaa-bbbb` | 控制容器可见的 GPU 设备（NVIDIA 标准变量） |
| `CUDA_DEVICE_MEMORY_LIMIT_0` | `server.go:535-536` | `4096m` | 第 0 号设备的显存上限（MB） |
| `CUDA_DEVICE_MEMORY_LIMIT_1` | `server.go:535-536` | `2048m` | 第 1 号设备的显存上限（MB），依此类推 |
| `CUDA_DEVICE_SM_LIMIT` | `server.go:538` | `50` | SM（流处理器）使用上限百分比（0-100） |
| `CUDA_DEVICE_MEMORY_SHARED_CACHE` | `server.go:539` | `/usr/local/vgpu/vgpu/<uuid>.cache` | vGPU 共享显存缓存文件路径 |
| `CUDA_OVERSUBSCRIBE` | `server.go:540-542` | `true` | 当 DeviceMemoryScaling > 1 时启用显存超分 |
| `LIBCUDA_LOG_LEVEL` | `server.go:543-545` | `0`/`1`/`3`/`4` | libvgpu 日志级别（0=error, 1=warn, 3=info, 4=debug） |
| `GPU_CORE_UTILIZATION_POLICY` | `server.go:546-548` | `disable` | GPU 算力限制策略开关（disable/force/default） |

### 二、NVIDIA 标准环境变量（由 getAllocateResponse 设置）

| 环境变量 | 设置位置 | 值示例 | 用途 |
|---------|---------|--------|------|
| `NVIDIA_VISIBLE_DEVICES` | `server.go:752` | `0,1` 或 UUID | 容器可见 GPU 列表 |
| `NVIDIA_GDRCOPY` | `server.go:639` | `enabled` | GPUDirect RDMA Copy |
| `NVIDIA_GDS` | `server.go:642` | `enabled` | GPUDirect Storage |
| `NVIDIA_MOFED` | `server.go:645` | `enabled` | Mellanox OFED |

### 三、通过 Pod Spec 注入的环境变量

**文件：** `pkg/device/nvidia/device.go:351-357`

| 环境变量 | 设置方式 | 用途 |
|---------|---------|------|
| `GPU_CORE_UTILIZATION_POLICY` | 通过修改 container env（非 device plugin response） | 当节点配置了非 default 的 GPUCorePolicy 时设置 |
| `CUDA_DISABLE_CONTROL` | 用户在 Pod spec 中手动设置 | 若为 `true`，device plugin 不挂载 ld.so.preload（禁用拦截） |

---

## 环境变量设置的详细代码

```go
// server.go:533-598 — 非 MIG 模式下的环境变量和挂载设置
if plugin.operatingMode != "mig" {

    // ① 每个设备的显存限制
    for i, dev := range devreq {
        limitKey := fmt.Sprintf("CUDA_DEVICE_MEMORY_LIMIT_%v", i)
        response.Envs[limitKey] = fmt.Sprintf("%vm", dev.Usedmem)  // 如 "4096m"
    }

    // ② SM 算力限制（百分比）
    response.Envs["CUDA_DEVICE_SM_LIMIT"] = fmt.Sprint(devreq[0].Usedcores)  // 如 "50"

    // ③ 共享缓存文件（用于 vGPU 显存管理）
    response.Envs["CUDA_DEVICE_MEMORY_SHARED_CACHE"] = fmt.Sprintf("%s/vgpu/%v.cache", hostHookPath, uuid.New().String())

    // ④ 显存超分开关
    if *plugin.schedulerConfig.DeviceMemoryScaling > 1 {
        response.Envs["CUDA_OVERSUBSCRIBE"] = "true"
    }

    // ⑤ 日志级别
    if *plugin.schedulerConfig.LogLevel != "" {
        response.Envs["LIBCUDA_LOG_LEVEL"] = string(*plugin.schedulerConfig.LogLevel)
    }

    // ⑥ 算力限制策略开关
    if plugin.schedulerConfig.DisableCoreLimit {
        response.Envs[util.CoreLimitSwitch] = "disable"   // "GPU_CORE_UTILIZATION_POLICY"
    }

    // ⑦ 挂载 libvgpu.so（CUDA 拦截库）
    response.Mounts = append(response.Mounts,
        &Mount{ContainerPath: fmt.Sprintf("%s/vgpu/libvgpu.so", hostHookPath), HostPath: GetLibPath(), ReadOnly: true},
        &Mount{ContainerPath: fmt.Sprintf("%s/vgpu", hostHookPath), HostPath: cacheFileHostDirectory, ReadOnly: false},
        &Mount{ContainerPath: "/tmp/vgpulock", HostPath: "/tmp/vgpulock", ReadOnly: false},
    )

    // ⑧ 挂载 ld.so.preload（除非 CUDA_DISABLE_CONTROL=true）
    if !found {  // found = 用户设置了 CUDA_DISABLE_CONTROL=true
        response.Mounts = append(response.Mounts,
            &Mount{ContainerPath: "/etc/ld.so.preload", HostPath: hostHookPath + "/vgpu/ld.so.preload", ReadOnly: true},
        )
    }
}
```

---

## 数据流图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         用户提交 Pod                                         │
│  resources:                                                                  │
│    limits:                                                                   │
│      nvidia.com/gpu: 2          # 请求 2 个 vGPU                             │
│      nvidia.com/gpumem: 4096    # 每个 4096MB 显存                            │
│      nvidia.com/gpucores: 50    # 每个 50% 算力                               │
└──────────────────┬───────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              HAMi Scheduler — Filter()                                       │
│                                                                              │
│  1. Resourcereqs(pod) → 解析资源需求                                          │
│  2. getNodesUsage() → 获取各节点 GPU 使用情况                                   │
│  3. calcScore() → 打分选节点                                                  │
│  4. PatchAnnotations() → 写入 Pod Annotation:                                │
│       hami.io/vgpu-node: "node-1"                                            │
│       hami.io/vgpu-devices-to-allocate: "GPU-xxx,4096,50,NVIDIA:..."         │
│       hami.io/vgpu-devices-allocated:   "GPU-xxx,4096,50,NVIDIA:..."         │
└──────────────────┬───────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              Kubelet 调用 Device Plugin — Allocate()                          │
│                                                                              │
│  1. GetPendingPod() → 获取待分配的 Pod                                        │
│  2. GetNextDeviceRequest() → 从 annotation 读设备分配                          │
│  3. getAllocateResponse() → 设置 NVIDIA_VISIBLE_DEVICES                       │
│  4. 设置环境变量:                                                              │
│       CUDA_DEVICE_MEMORY_LIMIT_0 = "4096m"                                   │
│       CUDA_DEVICE_MEMORY_LIMIT_1 = "4096m"                                   │
│       CUDA_DEVICE_SM_LIMIT = "50"                                            │
│       CUDA_DEVICE_MEMORY_SHARED_CACHE = "/path/to/<uuid>.cache"              │
│       CUDA_OVERSUBSCRIBE = "true"  (if scaling > 1)                          │
│       LIBCUDA_LOG_LEVEL = "0"      (if configured)                           │
│       GPU_CORE_UTILIZATION_POLICY = "disable" (if DisableCoreLimit)          │
│  5. 挂载 volume:                                                              │
│       /etc/ld.so.preload → libvgpu.so 拦截入口                                │
│       libvgpu.so → CUDA API 拦截库                                            │
│       vgpu cache dir → 显存管理缓存                                            │
│  6. EraseNextDeviceTypeFromAnnotation() → 标记容器已处理                        │
└──────────────────┬───────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│              容器运行时                                                        │
│                                                                              │
│  libvgpu.so 通过 ld.so.preload 机制在容器启动时被加载                            │
│  读取环境变量，拦截 CUDA API 调用：                                              │
│    - CUDA_DEVICE_MEMORY_LIMIT_x → 限制 cudaMalloc 的显存分配                   │
│    - CUDA_DEVICE_SM_LIMIT → 限制 SM 使用百分比                                 │
│    - CUDA_OVERSUBSCRIBE → 允许显存超分配（swap to host memory）                 │
│    - GPU_CORE_UTILIZATION_POLICY → 算力限制执行策略                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键文件索引

| 文件 | 作用 |
|------|------|
| `pkg/scheduler/scheduler.go:644` | `Filter()` — 调度入口，选节点 + 写 annotation |
| `pkg/device/nvidia/device.go:504` | `PatchAnnotations()` — NVIDIA 设备写 annotation |
| `pkg/device/devices.go:334` | `EncodePodDevices()` / `DecodePodDevices()` — 序列化/反序列化 |
| `pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go:475` | `Allocate()` — 读 annotation + 设环境变量 |
| `pkg/device-plugin/nvidiadevice/nvinternal/plugin/util.go:52` | `GetNextDeviceRequest()` — 解析下一个容器的设备分配 |
| `pkg/device-plugin/nvidiadevice/nvinternal/plugin/util.go:72` | `EraseNextDeviceTypeFromAnnotation()` — 标记容器已分配 |
| `pkg/util/types.go:19` | 常量定义（annotation key、环境变量名） |
