# Grapevine 技术设计文档

---

## 1. 文档信息

| 项目 | 内容 |
|------|------|
| 项目名称 | Grapevine — 基于 Termux 的 Windows 兼容容器管理平台 |
| 文档版本 | 1.0.0 |
| 文档状态 | 草案 |
| 创建日期 | 2026-05-06 |
| 最后更新 | 2026-05-06 |
| 文档范围 | 系统架构、核心模块、数据模型、接口规范、安全与性能设计 |
| 目标读者 | 架构师、核心开发者、技术评审人员 |

### 1.1 术语定义

| 术语 | 定义 |
|------|------|
| Container | Grapevine 管理的 Windows 兼容运行环境实例 |
| Prefix | Wine/Proton 的 Windows 模拟文件系统前缀 |
| RootFS | 容器的根文件系统，包含 glibc 运行时与兼容库 |
| Component | 可独立安装/升级的运行时组件（Wine、Box64、DXVK 等） |
| Template | 预定义的容器配置模板，用于快速创建容器 |
| Snapshot | 容器在某一时刻的状态快照，支持增量存储与回滚 |
| ICD | Installable Client Driver，Vulkan 驱动加载机制 |
| PRoot | 用户空间 chroot 实现，无需 root 权限的文件系统隔离 |

### 1.2 参考文档

- Wine Architecture: https://www.winehq.org/docs/wine-devel/
- Box86/Box64 Design: https://github.com/ptitSeb/box64
- PRoot Documentation: https://proot-me.github.io/
- Vulkan ICD Specification: https://vulkan.lunarg.com/doc/view/
- DXVK Design: https://github.com/doitsujin/dxvk

---

## 2. 设计目标与原则

### 2.1 设计目标

1. **无缝兼容**：在 ARM64 Android 设备上透明运行 x86/x64 Windows 应用程序，用户无需感知底层转译过程
2. **安全隔离**：每个容器拥有独立的文件系统、注册表与运行环境，容器间零干扰
3. **开箱即用**：提供预设模板（游戏、办公、开发等），用户一条命令即可创建可用环境
4. **可复现构建**：容器配置与组件版本完全声明式，支持精确复现
5. **性能优先**：启动时间 < 5s（热启动），图形转译开销 < 原生 15%
6. **无 Root 运行**：全程在 Termux 用户空间运行，无需设备 Root

### 2.2 设计原则

| 原则 | 描述 |
|------|------|
| **分层解耦** | 七层架构严格分层，层间仅通过定义良好的接口交互，上层不依赖下层实现细节 |
| **声明式配置** | 所有容器状态由 YAML 配置声明，系统负责收敛至期望状态 |
| **不可变基础设施** | 模板与组件版本不可变，变更通过新建实例而非原地修改实现 |
| **渐进增强** | 核心功能无外部依赖，高级特性（GPU 加速、音频桥接）按需启用 |
| **防御性编程** | 所有外部输入均需校验，所有 Shell 变量均需引号包裹，关键操作需确认 |
| **最小权限** | 仅请求必要的文件系统访问权限，组件安装遵循最小安装原则 |

---

## 3. 系统架构设计

### 3.1 七层架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7: 用户交互层 (User Interface Layer)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │   CLI    │  │   TUI    │  │ 脚本 API │                      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                      │
├───────┼──────────────┼─────────────┼────────────────────────────┤
│  Layer 6: 容器管理层 (Container Management Layer)                │
│  ┌────┴──────────────┴─────────────┴──────────────────────┐    │
│  │  生命周期 │ 快照 │ 模板 │ 导入导出 │ 资源控制 │ 网络管理  │    │
│  └────────────────────────┬───────────────────────────────┘    │
├───────────────────────────┼────────────────────────────────────┤
│  Layer 5: 兼容运行时层 (Compatibility Runtime Layer)            │
│  ┌────────────────────────┴───────────────────────────────┐    │
│  │        Wine / Proton     │     Box86 / Box64            │    │
│  └────────────────────────┬───────────────────────────────┘    │
├───────────────────────────┼────────────────────────────────────┤
│  Layer 4: 图形翻译层 (Graphics Translation Layer)               │
│  ┌────────────────────────┴───────────────────────────────┐    │
│  │  DXVK │ VKD3D │ WineD3D │ D8VK                         │    │
│  └────────────────────────┬───────────────────────────────┘    │
├───────────────────────────┼────────────────────────────────────┤
│  Layer 3: 驱动适配层 (Driver Adaptation Layer)                  │
│  ┌────────────────────────┴───────────────────────────────┐    │
│  │       Turnip        │       VirGL      │     Zink       │    │
│  └────────────────────────┬───────────────────────────────┘    │
├───────────────────────────┼────────────────────────────────────┤
│  Layer 2: 系统集成层 (System Integration Layer)                 │
│  ┌────────────────────────┴───────────────────────────────┐    │
│  │ PRoot │ glibc-runtime │ ALSA │ PulseAudio │ Termux-X11 │    │
│  └────────────────────────┬───────────────────────────────┘    │
├───────────────────────────┼────────────────────────────────────┤
│  Layer 1: Android 平台层 (Android Platform Layer)               │
│  ┌────────────────────────┴───────────────────────────────┐    │
│  │              Linux Kernel  │  GPU (Adreno/Mali/PowerVR) │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 各层详细设计

#### 3.2.1 用户交互层（Layer 7）

**职责**：接收用户输入，解析命令与参数，调用容器管理层接口，格式化输出结果。

**组成**：

| 组件 | 入口 | 描述 |
|------|------|------|
| CLI | `bin/grapevine` | 命令行接口，支持子命令、参数、补全 |
| TUI | `tui/main.sh` | 终端用户界面，基于 dialog/whiptail |
| 脚本 API | `source lib/core/*.sh` | Shell 函数库，供外部脚本直接调用 |

**CLI 命令树**：

```
grapevine
├── container
│   ├── create    [-t template] [-n name] [OPTIONS]
│   ├── start     <name>
│   ├── stop      <name> [-f]
│   ├── restart   <name>
│   ├── delete    <name> [--purge]
│   ├── list      [--format json|table|yaml]
│   ├── inspect   <name>
│   ├── exec      <name> <command> [ARGS...]
│   ├── run       [-t template] <command> [ARGS...]   # 创建+启动+执行+销毁
│   ├── export    <name> [-o path]
│   └── import    <archive> [-n name]
├── snapshot
│   ├── create    <container> [-m message]
│   ├── restore   <container> <snapshot-id>
│   ├── list      <container>
│   └── delete    <container> <snapshot-id>
├── template
│   ├── list
│   ├── show      <template>
│   └── validate  <template-file>
├── component
│   ├── list
│   ├── install   <component>[@version]
│   ├── remove    <component>
│   ├── update    [component]
│   └── info      <component>
├── config
│   ├── get       <key>
│   ├── set       <key> <value>
│   └── show
└── diagnose
    ├── check     [--fix]
    ├── logs      [-n lines] [-f]
    └── gpu-info
```

#### 3.2.2 容器管理层（Layer 6）

**职责**：容器全生命周期管理，是系统的核心编排层。

**模块划分**：

| 模块 | 源文件 | 职责 |
|------|--------|------|
| 生命周期管理 | `lib/core/container.sh` | 创建、启动、停止、删除容器 |
| 快照管理 | `lib/core/snapshot.sh` | 创建、恢复、删除快照 |
| 模板管理 | `lib/core/template.sh` | 解析、验证、实例化模板 |
| 配置管理 | `lib/core/config.sh` | 全局/容器配置读写与合并 |
| 状态管理 | `lib/core/state.sh` | 容器运行时状态持久化与查询 |

**核心编排流程**：

```
container create:
  1. template.sh::resolve_template()     → 解析模板，生成容器配置
  2. config.sh::merge_config()           → 合并模板配置与用户覆盖
  3. config.sh::validate_config()        → 校验配置合法性
  4. container.sh::init_rootfs()         → 初始化 RootFS 目录结构
  5. container.sh::init_prefix()         → 初始化 Wine Prefix
  6. component.sh::resolve_deps()        → 解析组件依赖
  7. component.sh::ensure_installed()    → 确保组件已安装
  8. state.sh::set_state("created")      → 持久化容器状态

container start:
  1. state.sh::get_state()               → 检查当前状态
  2. state.sh::transition("running")     → 状态转换校验
  3. proot.sh::setup_bindings()          → 配置 PRoot bind mount
  4. box64.sh::setup_env()               → 配置 Box64 环境
  5. box86.sh::setup_env()               → 配置 Box86 环境
  6. driver.sh::detect_and_configure()   → 检测并配置 GPU 驱动
  7. dxvk.sh|vkd3d.sh::inject()          → 注入图形翻译层
  8. pulseaudio.sh::setup()              → 配置音频桥接
  9. x11.sh::setup()                     → 配置显示输出
 10. wine.sh|proton.sh::launch()         → 启动 Wine/Proton 进程
 11. state.sh::set_state("running")      → 更新运行时状态
```

#### 3.2.3 兼容运行时层（Layer 5）

**职责**：提供 x86/x64 指令转译与 Windows API 兼容。

| 组件 | 源文件 | 功能 |
|------|--------|------|
| Wine | `lib/runtime/wine.sh` | Windows API 兼容层，提供 Win32/Win64 实现 |
| Proton | `lib/runtime/proton.sh` | Valve 增强版 Wine，集成游戏兼容补丁 |
| Box64 | `lib/runtime/box64.sh` | x86_64 → ARM64 指令转译器 |
| Box86 | `lib/runtime/box86.sh` | x86 → ARM(32/64) 指令转译器 |

**转译链路**：

```
x64 Windows 应用
    │
    ▼
┌─────────┐    x64 → ARM64 指令转译
│  Box64  │──────────────────────────┐
└────┬────┘                          │
     │ 加载 .dll.so                  ▼
┌────┴────┐                    ARM64 指令流
│  Wine   │  Windows API → POSIX 调用
│ (64bit) │──────────────────────────┐
└────┬────┘                          │
     │                               ▼
     │                         Linux 系统调用
     │
x86 Windows 应用
     │
     ▼
┌─────────┐    x86 → ARM 指令转译
│  Box86  │──────────────────────────┐
└────┬────┘                          │
     │ 加载 .dll.so                  ▼
┌────┴────┐                    ARM 指令流
│  Wine   │  Windows API → POSIX 调用
│ (32bit) │──────────────────────────┐
└─────────┘                          │
                                     ▼
                               Linux 系统调用
```

#### 3.2.4 图形翻译层（Layer 4）

**职责**：将 Windows 图形 API (DirectX) 翻译为跨平台图形 API (Vulkan/OpenGL)。

| 组件 | 源文件 | 输入 API | 输出 API |
|------|--------|----------|----------|
| DXVK | `lib/graphics/dxvk.sh` | DirectX 9/10/11 | Vulkan |
| VKD3D | `lib/graphics/vkd3d.sh` | DirectX 12 | Vulkan |
| WineD3D | Wine 内置 | DirectX 1-11 | OpenGL |
| D8VK | `lib/graphics/dxvk.sh` | DirectX 8 | Vulkan |

**图形翻译选择策略**：

```
detect_required_dx_version(container_config)
  │
  ├── dx_version == "dx12" ──→ VKD3D (必须 Vulkan)
  ├── dx_version == "dx11" ──→ DXVK (优先) / WineD3D (降级)
  ├── dx_version == "dx10" ──→ DXVK (优先) / WineD3D (降级)
  ├── dx_version == "dx9"  ──→ DXVK (优先) / WineD3D (降级)
  ├── dx_version == "dx8"  ──→ D8VK (优先) / WineD3D (降级)
  └── dx_version == "auto" ──→ 安装全部可用翻译层
                                运行时按需加载
```

#### 3.2.5 驱动适配层（Layer 3）

**职责**：将 Vulkan/OpenGL 调用映射到 Android GPU 硬件。

| 组件 | 源文件 | API | 目标硬件 |
|------|--------|-----|----------|
| Turnip | `lib/graphics/turnip.sh` | Vulkan → Mesa Vulkan | Adreno GPU |
| VirGL | `lib/graphics/virgl.sh` | OpenGL → VirGL协议 → 主机OpenGL | 通用 GPU |
| Zink | `lib/graphics/zink.sh` | OpenGL → Vulkan | 支持 Vulkan 的 GPU |

**驱动选择算法**：

```
function select_graphics_driver():
    gpu_vendor = detect_gpu_vendor()  # 读取 /sys/class/kgsl-3d0/dev 或 ro.hardware

    if gpu_vendor == "adreno":
        vulkan_support = check_vulkan_icd("turnip")
        if vulkan_support == OK:
            return "turnip"           # Adreno 原生 Vulkan
        else:
            return "virgl"           # 降级到 VirGL

    elif gpu_vendor == "mali":
        vulkan_support = check_vulkan_icd("mali")
        if vulkan_support == OK:
            return "zink"            # Mali Vulkan → Zink(OpenGL→Vulkan)
        else:
            return "virgl"

    elif gpu_vendor == "powervr":
        return "virgl"               # PowerVR 使用 VirGL

    else:
        return "virgl"               # 未知 GPU，默认 VirGL
```

#### 3.2.6 系统集成层（Layer 2）

**职责**：提供 Linux 运行时环境，桥接 Android 系统服务。

| 组件 | 源文件 | 功能 |
|------|--------|------|
| PRoot | `lib/runtime/proot.sh` | 用户空间 chroot，文件系统隔离 |
| glibc-runtime | 系统内置 | 提供 glibc 运行时（Termux 默认 bionic） |
| ALSA | `lib/audio/alsa.sh` | Advanced Linux Sound Architecture 配置 |
| PulseAudio | `lib/audio/pulseaudio.sh` | 音频服务器，提供混音与网络音频 |
| Termux-X11 | `lib/display/x11.sh` | X11 显示服务器，输出到 Android 屏幕 |

#### 3.2.7 Android 平台层（Layer 1）

**职责**：提供底层 Linux 内核与 GPU 硬件能力。

- **Linux Kernel**：Android 内核提供系统调用接口，PRoot 通过 `ptrace` 或 `seccomp` 实现拦截
- **GPU**：通过 `/dev/dri/renderD128` 或 Turnip 用户态驱动暴露 Vulkan 能力

### 3.3 层间接口定义

层间通信遵循以下约定：

| 接口方向 | 接口类型 | 约定 |
|----------|----------|------|
| Layer 7 → Layer 6 | Shell 函数调用 | `lib/core/*.sh` 导出的函数，参数通过位置参数传递 |
| Layer 6 → Layer 5 | Shell 函数调用 | `lib/runtime/*.sh` 导出的函数 |
| Layer 6 → Layer 4 | Shell 函数调用 | `lib/graphics/*.sh` 导出的函数 |
| Layer 6 → Layer 3 | Shell 函数调用 | `lib/graphics/driver.sh` 导出的函数 |
| Layer 6 → Layer 2 | Shell 函数调用 + 环境变量 | `lib/runtime/proot.sh`、`lib/audio/*.sh`、`lib/display/*.sh` |
| Layer 5 → Layer 4 | 环境变量 + DLL 注入 | `WINEDLLOVERRIDES`、`DXVK_STATE_CACHE_PATH` 等 |
| Layer 4 → Layer 3 | 环境变量 + ICD JSON | `VK_ICD_FILENAMES`、`MESA_VK_WSI_PRESENT_MODE` 等 |
| Layer 3 → Layer 2 | 系统调用 + 设备文件 | `/dev/dri/*`、`/dev/snd/*` |
| Layer 2 → Layer 1 | 系统调用 + ioctl | 内核接口 |

**通用接口约定**：

```bash
# 所有层间函数遵循以下签名约定：
#   <module>_<action>  [OPTIONS] <required-args...>
#
# 返回值约定：
#   0 = 成功
#   1 = 一般错误
#   2 = 配置错误
#   3 = 依赖缺失
#   4 = 状态冲突
#   5 = 权限不足
#
# 输出约定：
#   stdout = 结构化数据（JSON/YAML/KEY=VALUE）
#   stderr = 日志与诊断信息
#   返回码 = 操作结果
```

---

## 4. 核心模块详细设计

### 4.1 容器管理引擎

#### 4.1.1 状态机设计

容器具有以下状态，状态转换必须遵循状态机规则：

```
                    ┌──────────┐
                    │          │
          create    │ CREATED  │
         ──────────►│          │
        │           └────┬─────┘
        │                │ start
        │                ▼
        │           ┌──────────┐
        │           │          │
        │           │ RUNNING  │◄──── restart ────┐
        │           └────┬─────┘                   │
        │                │                         │
        │        ┌───────┼───────┐                 │
        │        │       │       │                 │
        │        ▼       ▼       │                 │
        │   ┌────────┐ ┌──────┐ │                 │
        │   │STOPPED │ │PAUSED│ │                 │
        │   │        │ │      │ │                 │
        │   └───┬────┘ └──┬───┘ │                 │
        │       │         │     │                 │
        │  start│    resume│    │stop             │
        │       │         │     │                 │
        │       └────┬────┘     │                 │
        │            │          │                 │
        │            └──────────┴── start ────────┘
        │                       │
        │                  delete│
        │                       ▼
        │                 ┌──────────┐
        └────────────────►│ DELETED  │
                          └──────────┘
```

**状态定义**：

| 状态 | 值 | 描述 | 允许转换 |
|------|-----|------|----------|
| CREATED | `created` | 容器已创建但未启动 | → running, deleted |
| RUNNING | `running` | 容器正在运行 | → stopped, paused |
| STOPPED | `stopped` | 容器已正常停止 | → running, deleted |
| PAUSED | `paused` | 容器已暂停（进程挂起） | → running |
| DELETED | `deleted` | 容器已标记删除 | → (终态) |

**状态转换函数**：

```bash
# state.sh::transition()
# 参数: $1=容器名, $2=目标状态
# 返回: 0=转换成功, 4=非法转换
function state_transition() {
    local name="$1"
    local target_state="$2"
    local current_state
    current_state=$(state_get "${name}")

    local -A allowed_transitions=(
        ["created:running"]=1
        ["created:deleted"]=1
        ["running:stopped"]=1
        ["running:paused"]=1
        ["stopped:running"]=1
        ["stopped:deleted"]=1
        ["paused:running"]=1
    )

    if [[ -z "${allowed_transitions[${current_state}:${target_state}]}" ]]; then
        log_error "非法状态转换: ${current_state} → ${target_state}"
        return 4
    fi

    state_set "${name}" "${target_state}"
    return 0
}
```

#### 4.1.2 生命周期转换

**容器创建流程**：

```
container_create(name, template, overrides)
  │
  ├─ 1. 校验容器名合法性
  │     - 正则: ^[a-z][a-z0-9_-]{1,62}$
  │     - 检查名称唯一性
  │
  ├─ 2. 解析模板
  │     template_resolve(template) → base_config
  │
  ├─ 3. 合并用户覆盖
  │     config_merge(base_config, overrides) → final_config
  │
  ├─ 4. 校验最终配置
  │     config_validate(final_config)
  │
  ├─ 5. 创建目录结构
  │     mkdir -p ~/.grapevine/containers/<name>/{rootfs,prefix,snapshots,logs}
  │
  ├─ 6. 写入容器配置
  │     write container.yml
  │
  ├─ 7. 初始化状态
  │     state_init(name, "created")
  │
  └─ 8. 返回 0
```

**容器启动流程**：

```
container_start(name)
  │
  ├─ 1. 状态校验: state ∈ {created, stopped}
  │
  ├─ 2. 读取容器配置
  │     config_load(name) → config
  │
  ├─ 3. 前置检查
  │     ├─ 检查组件完整性 (component_verify_all)
  │     ├─ 检查磁盘空间 (df -P)
  │     └─ 检查端口占用 (如需网络服务)
  │
  ├─ 4. 构建运行时环境
  │     ├─ proot_setup_bindings(config)
  │     ├─ box64_setup_env(config)
  │     ├─ box86_setup_env(config)
  │     ├─ driver_detect_and_configure(config)
  │     ├─ graphics_inject(config)
  │     ├─ audio_setup(config)
  │     └─ display_setup(config)
  │
  ├─ 5. 启动主进程
  │     wine_launch(config) → pid
  │
  ├─ 6. 记录运行时信息
  │     state_set(name, "running")
  │     state_set_pid(name, pid)
  │     state_set_started_at(name, epoch)
  │
  ├─ 7. 启动监控协程
  │     container_monitor(name) &
  │
  └─ 8. 返回 0
```

**容器停止流程**：

```
container_stop(name, force=false)
  │
  ├─ 1. 状态校验: state ∈ {running, paused}
  │
  ├─ 2. 获取主进程 PID
  │     pid = state_get_pid(name)
  │
  ├─ 3. 优雅停止 (force=false)
  │     ├─ 发送 SIGTERM → wine process tree
  │     ├─ 等待 grace_period (默认 30s)
  │     └─ 超时则升级为 force
  │
  ├─ 4. 强制停止 (force=true)
  │     ├─ 发送 SIGKILL → wine process tree
  │     └─ 等待进程退出 (最多 5s)
  │
  ├─ 5. 清理运行时资源
  │     ├─ proot_cleanup(name)
  │     ├─ audio_cleanup(name)
  │     └─ display_cleanup(name)
  │
  ├─ 6. 更新状态
  │     state_set(name, "stopped")
  │     state_clear_pid(name)
  │
  └─ 7. 返回 0
```

#### 4.1.3 并发控制

**锁机制**：使用文件锁（flock）保证容器操作的原子性。

```bash
# container.sh::container_lock()
# 参数: $1=容器名, $2=超时(秒, 默认30)
# 返回: 0=获取成功, 1=超时
function container_lock() {
    local name="$1"
    local timeout="${2:-30}"
    local lock_file="${GRAPEVINE_HOME}/containers/${name}/.lock"

    exec 200>"${lock_file}"
    flock -w "${timeout}" 200
    return $?
}

function container_unlock() {
    local name="$1"
    local lock_fd=200
    exec 200>&-
}
```

**并发安全保证**：

| 操作 | 锁类型 | 范围 |
|------|--------|------|
| create | 全局锁 | 防止同名容器并发创建 |
| start/stop/restart | 容器锁 | 单容器操作串行化 |
| snapshot create/restore | 容器锁 | 快照与生命周期互斥 |
| delete | 容器锁 + 全局锁 | 防止删除与启动并发 |
| config set | 容器锁 | 配置修改串行化 |

### 4.2 配置管理引擎

#### 4.2.1 YAML 解析

使用轻量级 Shell YAML 解析器（`lib/utils/yaml.sh`），不依赖 Python/Ruby 等外部运行时。

**解析算法**：

```
yaml_parse(input):
  1. 逐行读取输入
  2. 根据缩进级别构建层级树
  3. 识别标量、序列、映射三种节点类型
  4. 输出 KEY=VALUE 格式（支持嵌套键用 '.' 连接）

示例:
  输入 YAML:
    wine:
      version: "9.0"
      arch: win64
    graphics:
      driver: turnip
      dxvk:
        version: "2.3"

  解析输出:
    wine.version="9.0"
    wine.arch="win64"
    graphics.driver="turnip"
    graphics.dxvk.version="2.3"
```

**解析器限制**（出于 Shell 实现的性能考虑）：

- 不支持 YAML 锚点与别名（&/*）
- 不支持多文档流（---）
- 不支持行内映射的嵌套超过 4 层
- 字符串值不包含未转义的冒号+空格

#### 4.2.2 配置合并策略

配置按以下优先级从低到高合并（后者覆盖前者）：

```
优先级（低 → 高）:
  1. 内置默认值        (lib/core/config.sh::DEFAULTS)
  2. 模板配置          (templates/*.yml)
  3. 全局用户配置      (~/.grapevine/config.yml → containers.defaults)
  4. 容器配置文件      (container.yml)
  5. 命令行覆盖参数    (--set key=value)
```

**合并算法**：

```bash
# config.sh::config_merge()
# 策略: 深度合并（递归合并映射，覆盖标量与序列）
function config_merge() {
    local base="$1"
    local override="$2"
    local result

    for key in $(all_keys "${base}" "${override}"); do
        local base_val="${base[${key}]}"
        local over_val="${override[${key}]}"

        if is_map "${base_val}" && is_map "${over_val}"; then
            result[${key}]=$(config_merge "${base_val}" "${over_val}")
        elif [[ -n "${over_val}" ]]; then
            result[${key}]="${over_val}"
        else
            result[${key}]="${base_val}"
        fi
    done

    echo "${result}"
}
```

**合并示例**：

```yaml
# 模板默认 (gaming-dx11.yml)
wine:
  version: "9.0"
  arch: win64
graphics:
  driver: auto
  dxvk:
    version: "2.3"

# 用户覆盖 (--set wine.version=9.4 --set graphics.driver=turnip)
wine:
  version: "9.4"      # 覆盖
  arch: win64          # 保留
graphics:
  driver: turnip       # 覆盖
  dxvk:
    version: "2.3"     # 保留
```

#### 4.2.3 配置验证规则

```bash
# config.sh::config_validate()
# 校验规则定义
VALIDATION_RULES=(
    "wine.version:required|string|match:^[0-9]+\\.[0-9]+(\\.[0-9]+)?$"
    "wine.arch:required|enum:win32,win64"
    "graphics.driver:optional|enum:auto,turnip,virgl,zink"
    "graphics.dxvk.version:optional|string|match:^[0-9]+\\.[0-9]+$"
    "graphics.vkd3d.version:optional|string|match:^[0-9]+\\.[0-9]+$"
    "audio.backend:optional|enum:pulseaudio,alsa,none"
    "display.backend:optional|enum:x11,none"
    "container.name:required|string|match:^[a-z][a-z0-9_-]{1,62}$"
    "container.disk_limit:optional|string|match:^[0-9]+[KMGT]?$"
    "container.memory_limit:optional|string|match:^[0-9]+[KMGT]?$"
    "proot.rootfs:required|path_exists"
    "proot.binds:optional|list_of:path_exists"
)
```

### 4.3 快照引擎

#### 4.3.1 增量快照算法

采用**基于文件的增量快照**策略，使用文件级差异检测：

```
snapshot_create(container, message):
  │
  ├─ 1. 获取容器当前状态
  │     current_state = state_get(container)
  │     要求: state ∈ {stopped, created}
  │
  ├─ 2. 生成快照 ID
  │     snapshot_id = "snap_${timestamp}_${short_hash}"
  │     格式: snap_20260506143022_a3f2
  │
  ├─ 3. 扫描文件元数据
  │     find prefix/ -type f -printf '%P\t%s\t%T@\t%M\n' > manifest.new
  │
  ├─ 4. 与上次快照 manifest 对比
  │     if 上次 manifest 存在:
  │       diff manifest.old manifest.new → {added, modified, removed}
  │     else:
  │       所有文件视为 added（全量快照）
  │
  ├─ 5. 创建增量存档
  │     for file in added + modified:
  │       tar -rf snapshot.tar <file>     # 追加到增量存档
  │     for file in removed:
  │       echo "<file>" >> removed.list   # 记录删除列表
  │
  ├─ 6. 压缩存档
  │     zstd -19 snapshot.tar -o snapshot.tar.zst
  │
  ├─ 7. 写入快照元数据
  │     snapshot_meta = {
  │       id: snapshot_id,
  │       parent: last_snapshot_id,
  │       timestamp: epoch,
  │       message: message,
  │       file_count: N,
  │       size_bytes: M,
  │       type: incremental | full
  │     }
  │     write snapshots/<snapshot_id>/meta.yml
  │
  └─ 8. 保存当前 manifest
        mv manifest.new manifest.old
```

**增量检测算法**：

```
文件差异检测:
  对每个文件计算: fingerprint = mtime_ns + size + mode
  若 fingerprint 变化 → 文件已修改
  若文件不存在于新 manifest → 文件已删除
  若文件不存在于旧 manifest → 文件已新增

优化:
  - 文件 > 64MB 时，额外计算 CRC32 块校验，避免误判
  - 忽略路径匹配 *.log, *.tmp, prefix/.update-timestamp 的文件
  - 符号链接仅比较目标路径
```

#### 4.3.2 存储格式

```
~/.grapevine/containers/<name>/snapshots/
├── snap_20260506143022_a3f2/         # 快照目录
│   ├── meta.yml                      # 快照元数据
│   ├── data.tar.zst                  # 增量数据（zstd 压缩）
│   ├── removed.list                  # 删除文件列表
│   └── manifest.txt                  # 文件清单（当前状态）
├── snap_20260506120000_b7c1/
│   ├── meta.yml
│   ├── data.tar.zst
│   ├── removed.list
│   └── manifest.txt
└── chain.yml                         # 快照链索引
```

**chain.yml 格式**：

```yaml
chain:
  - id: snap_20260506120000_b7c1
    parent: null
    type: full
  - id: snap_20260506143022_a3f2
    parent: snap_20260506120000_b7c1
    type: incremental
```

#### 4.3.3 回滚机制

```
snapshot_restore(container, snapshot_id):
  │
  ├─ 1. 校验容器状态: state == stopped
  │
  ├─ 2. 创建安全快照（回滚前自动保存当前状态）
  │     snapshot_create(container, "pre-rollback-auto")
  │
  ├─ 3. 构建恢复链
  │     chain = build_restore_chain(snapshot_id)
  │     # 从最近的 full snapshot 开始，按序应用增量
  │
  ├─ 4. 清空当前 prefix
  │     # 保留 snapshots/ 目录不动
  │     find prefix/ -delete
  │
  ├─ 5. 按序恢复
  │     for snap in chain:
  │       tar -xf snap/data.tar.zst -C prefix/
  │       for file in snap/removed.list:
  │         rm -f "prefix/${file}"
  │
  ├─ 6. 恢复 state.json
  │     # 从快照中恢复容器运行时状态
  │
  └─ 7. 更新 chain.yml
        # 将后续快照标记为 orphaned
```

### 4.4 模板引擎

#### 4.4.1 模板解析

模板文件为 YAML 格式，包含以下顶级键：

```yaml
# templates/gaming-dx11.yml
apiVersion: v1
kind: ContainerTemplate
metadata:
  name: gaming-dx11
  description: "DirectX 11 游戏容器模板"
  tags: [gaming, dx11, vulkan]
  author: grapevine

spec:
  wine:
    version: "9.0"
    arch: win64
    staging: true

  box64:
    version: "0.2.8"

  box86:
    version: "0.8.6"

  graphics:
    driver: auto
    dxvk:
      version: "2.3"
      enabled: true
    vkd3d:
      version: "2.12"
      enabled: false
    d8vk:
      version: "1.0"
      enabled: false

  audio:
    backend: pulseaudio

  display:
    backend: x11
    resolution: "1280x720"

  proot:
    binds:
      - /system/lib64
      - /vendor/lib64

  container:
    disk_limit: "8G"
    memory_limit: "4G"

  post_create:
    - command: "winetricks dxvk"
      description: "Install DXVK"
    - command: "winetricks vcrun2022"
      description: "Install Visual C++ Runtime"
```

**解析流程**：

```
template_resolve(template_name_or_path):
  │
  ├─ 1. 定位模板文件
  │     if 是文件路径 → 直接使用
  │     else → 在 templates/ 目录查找 <name>.yml
  │
  ├─ 2. 解析 YAML
  │     yaml_parse(template_content) → raw_config
  │
  ├─ 3. 校验模板格式
  │     - 必须包含 apiVersion, kind, metadata, spec
  │     - kind 必须为 ContainerTemplate
  │     - apiVersion 必须为支持的版本
  │
  ├─ 4. 处理模板继承
  │     if spec.extends 存在:
  │       parent = template_resolve(spec.extends)
  │       raw_config = config_merge(parent, raw_config)
  │
  └─ 5. 返回解析后配置
```

#### 4.4.2 变量替换

模板支持 `{{ variable }}` 格式的变量占位符：

```yaml
spec:
  wine:
    version: "{{ wine_version | default('9.0') }}"
  display:
    resolution: "{{ resolution | default('1280x720') }}"
```

**替换算法**：

```bash
function template_substitute() {
    local content="$1"
    local -A vars="$2"

    # 匹配 {{ var | default('val') }} 模式
    local regex='\{\{\s*(\w+)(\s*\|\s*default\(\s*'\''([^'\'']*)'\''\s*\))?\s*\}\}'

    while [[ "${content}" =~ ${regex} ]]; do
        local var_name="${BASH_REMATCH[1]}"
        local default_val="${BASH_REMATCH[3]}"
        local replacement

        if [[ -n "${vars[${var_name}]}" ]]; then
            replacement="${vars[${var_name}]}"
        elif [[ -n "${default_val}" ]]; then
            replacement="${default_val}"
        else
            log_error "模板变量 '${var_name}' 未提供值且无默认值"
            return 2
        fi

        content="${content//${BASH_REMATCH[0]}/${replacement}}"
    done

    echo "${content}"
}
```

#### 4.4.3 模板继承

模板可通过 `spec.extends` 字段继承另一个模板：

```yaml
# templates/gaming-dx12.yml
apiVersion: v1
kind: ContainerTemplate
metadata:
  name: gaming-dx12
  description: "DirectX 12 游戏容器模板"
spec:
  extends: gaming-dx11          # 继承 gaming-dx11 模板
  graphics:
    vkd3d:
      version: "2.12"
      enabled: true             # 覆盖父模板的 false
    dxvk:
      enabled: false            # DX12 不需要 DXVK
  post_create:
    - command: "winetricks vkd3d"
      description: "Install VKD3D"
```

**继承规则**：

- 单继承，不支持多继承
- 继承深度限制为 3 层，防止循环继承
- 子模板的 `spec` 与父模板深度合并
- `post_create` 序列采用追加策略（先执行父模板命令，再执行子模板命令）
- 循环继承检测：解析过程中维护已访问模板列表，遇到重复则报错

### 4.5 PRoot 隔离层

#### 4.5.1 文件系统映射规则

PRoot 为每个容器构建虚拟根文件系统，映射规则如下：

```
PRoot 文件系统布局:
  /                    → <container>/rootfs/              (容器根)
  /usr                 → <component>/glibc-rootfs/usr/    (glibc 运行时)
  /lib                 → <component>/glibc-rootfs/lib/    (系统库)
  /lib64               → <component>/glibc-rootfs/lib64/  (64位库)
  /opt/wine            → <component>/wine-<ver>/          (Wine 安装)
  /opt/box64           → <component>/box64-<ver>/         (Box64 安装)
  /opt/box86           → <component>/box86-<ver>/         (Box86 安装)
  /home/user           → <container>/prefix/              (Wine Prefix)
  /tmp                 → <container>/rootfs/tmp/           (临时目录)
  /dev                 → /dev                              (设备文件, 只读)
  /proc                → /proc                             (进程信息, 只读)
  /sys                 → /sys                              (系统信息, 只读)
  /sdcard              → /sdcard                           (Android 存储, 可选)
```

#### 4.5.2 Bind Mount 策略

```bash
# proot.sh::proot_build_cmd()
function proot_build_cmd() {
    local name="$1"
    local config="$2"

    local cmd="proot"
    cmd+=" --rootfs=${GRAPEVINE_HOME}/containers/${name}/rootfs"

    # 核心 bind mount
    cmd+=" --bind=${GRAPEVINE_HOME}/components/glibc-rootfs/usr:/usr"
    cmd+=" --bind=${GRAPEVINE_HOME}/components/glibc-rootfs/lib:/lib"
    cmd+=" --bind=${GRAPEVINE_HOME}/components/glibc-rootfs/lib64:/lib64"

    # 组件 bind mount
    local wine_path=$(component_get_path "wine" "${config[wine.version]}")
    cmd+=" --bind=${wine_path}:/opt/wine"

    local box64_path=$(component_get_path "box64" "${config[box64.version]}")
    cmd+=" --bind=${box64_path}:/opt/box64"

    local box86_path=$(component_get_path "box86" "${config[box86.version]}")
    cmd+=" --bind=${box86_path}:/opt/box86"

    # Wine Prefix
    cmd+=" --bind=${GRAPEVINE_HOME}/containers/${name}/prefix:/home/user"

    # 用户自定义 bind mount
    for bind in "${config[proot.binds[@]]}"; do
        cmd+=" --bind=${bind}"
    done

    # 可选: Android 存储
    if [[ "${config[container.share_sdcard]}" == "true" ]]; then
        cmd+=" --bind=/sdcard:/sdcard"
    fi

    # PRoot 选项
    cmd+=" --link2symlink"          # 将硬链接转为符号链接
    cmd+=" --kill-on-exit"          # 退出时杀死子进程
    cmd+=" --cwd=/home/user"        # 工作目录

    echo "${cmd}"
}
```

#### 4.5.3 Seccomp 过滤

PRoot 运行时可配置 seccomp 过滤，限制容器内可用的系统调用：

```bash
# 默认 seccomp 规则
SECCOMP_ALLOW=(
    read write open openat close
    fstat fstatat lstat stat
    mmap mprotect munmap brk
    ioctl access faccessat
    pipe pipe2 select poll
    fork vfork clone clone3
    execve execveat exit exit_group
    waitid wait4
    getpid getppid getuid getgid geteuid getegid
    sigaction sigprocmask sigreturn
    mmap2 fcntl fcntl64
    dup dup2 dup3
    getdents getdents64
    clock_gettime clock_getres
    nanosleep clock_nanosleep
    socket connect bind listen accept
    recvmsg sendmsg recvfrom sendto
    shutdown setsockopt getsockopt
    getsockname getpeername
    futex
    madvise
    arch_prctl
    set_thread_area
    modify_ldt
)

SECCOMP_DENY=(
    mount umount2
    pivot_root
    chroot
    kexec_load
    acct
    reboot
    swapon swapoff
    mknod
)
```

### 4.6 运行时管理器

#### 4.6.1 Wine/Proton 启动流程

```
wine_launch(config):
  │
  ├─ 1. 选择运行时 (Wine 或 Proton)
  │     if config.wine.runtime == "proton":
  │       runtime_path = component_get_path("proton", config.wine.version)
  │     else:
  │       runtime_path = component_get_path("wine", config.wine.version)
  │
  ├─ 2. 组装环境变量
  │     env = build_runtime_env(config)
  │
  ├─ 3. 构建 PRoot 命令
  │     proot_cmd = proot_build_cmd(name, config)
  │
  ├─ 4. 构建 Box64/Box86 命令
  │     if config.wine.arch == "win64":
  │       box_cmd = "box64"
  │     else:
  │       box_cmd = "box86"
  │
  ├─ 5. 组装完整命令链
  │     full_cmd = "${proot_cmd} ${box_cmd} ${runtime_path}/bin/wine64"
  │     if config.container.command:
  │       full_cmd += " ${config.container.command}"
  │     else:
  │       full_cmd += " wineboot"
  │
  ├─ 6. 启动进程
  │     eval "${full_cmd}" &
  │     pid = $!
  │
  ├─ 7. 记录 PID
  │     state_set_pid(name, pid)
  │     echo "${pid}" > "${GRAPEVINE_HOME}/containers/${name}/.pid"
  │
  └─ 8. 启动进程监控
        container_monitor(name) &
```

#### 4.6.2 环境变量组装

```bash
# wine.sh::build_runtime_env()
function build_runtime_env() {
    local config="$1"
    local -A env

    # === Wine 核心变量 ===
    env[WINEPREFIX]="/home/user"                          # Prefix 路径（PRoot 内）
    env[WINEARCH]="${config[wine.arch]}"                  # win32 或 win64
    env[WINEDEBUG]="${config[wine.debug_level]:-fixme-all}" # 调试级别
    env[WINEESYNC]="${config[wine.esync]:-1}"             # esync 启用
    env[WINEFSYNC]="${config[wine.fsync]:-0}"             # fsync (需内核支持)

    # === Box64/Box86 变量 ===
    env[BOX64_LOG]="${config[box64.log_level]:-0}"
    env[BOX64_DYNAREC]="${config[box64.dynarec]:-1}"
    env[BOX64_MMAP32]="${config[box64.mmap32]:-1}"
    env[BOX86_LOG]="${config[box86.log_level]:-0}"
    env[BOX86_DYNAREC]="${config[box86.dynarec]:-1}"

    # === 图形变量 ===
    env[WINEDLLOVERRIDES]=""

    if [[ "${config[graphics.dxvk.enabled]}" == "true" ]]; then
        env[DXVK_STATE_CACHE_PATH]="/home/user/.dxvk-cache"
        env[DXVK_CONFIG_FILE]="/home/user/dxvk.conf"
        env[WINEDLLOVERRIDES]+="d3d9,d3d10,d3d10_1,d3d11,dxgi=n;"
    fi

    if [[ "${config[graphics.vkd3d.enabled]}" == "true" ]]; then
        env[WINEDLLOVERRIDES]+="d3d12=n;"
    fi

    if [[ "${config[graphics.d8vk.enabled]}" == "true" ]]; then
        env[WINEDLLOVERRIDES]+="d3d8=n;"
    fi

    # === Vulkan 变量 ===
    local driver="${config[graphics.driver]}"
    if [[ "${driver}" == "turnip" ]]; then
        env[VK_ICD_FILENAMES]="/opt/vulkan/icd.d/freedreno_icd.aarch64.json"
        env[MESA_VK_WSI_PRESENT_MODE]="fifo"
        env[TU_DEBUG]="${config[graphics.turnip.debug]:-noconform}"
    elif [[ "${driver}" == "zink" ]]; then
        env[VK_ICD_FILENAMES]="/opt/vulkan/icd.d/lavapipe_icd.aarch64.json"
        env[MESA_GL_VERSION_OVERRIDE]="4.3"
    elif [[ "${driver}" == "virgl" ]]; then
        env[VK_ICD_FILENAMES]=""     # VirGL 不使用 Vulkan ICD
        env[GALLIUM_DRIVER]="virpipe"
    fi

    # === 音频变量 ===
    if [[ "${config[audio.backend]}" == "pulseaudio" ]]; then
        env[PULSE_SERVER]="unix:/run/pulse/pulse-socket"
    elif [[ "${config[audio.backend]}" == "alsa" ]]; then
        env[ALSA_CONFIG_PATH]="/etc/alsa/alsa.conf"
    fi

    # === 显示变量 ===
    env[DISPLAY]="${config[display.display]:-:0}"
    env[XCURSOR_SIZE]="${config[display.cursor_size]:-24}"

    # === 路径变量 ===
    env[PATH]="/opt/wine/bin:/opt/box64:/opt/box86:/usr/bin:/bin"
    env[LD_LIBRARY_PATH]="/opt/wine/lib:/opt/box64/lib:/opt/box86/lib:/usr/lib:/lib"
    env[XDG_DATA_HOME]="/home/user/.local/share"

    # 输出为 export 格式
    for key in "${!env[@]}"; do
        echo "export ${key}=\"${env[${key}]}\""
    done
}
```

#### 4.6.3 进程监控

```bash
# container.sh::container_monitor()
# 后台监控协程，检测主进程退出并执行清理
function container_monitor() {
    local name="$1"
    local pid_file="${GRAPEVINE_HOME}/containers/${name}/.pid"
    local pid
    pid=$(cat "${pid_file}")

    # 等待主进程退出
    while kill -0 "${pid}" 2>/dev/null; do
        sleep 2
    done

    # 主进程已退出，获取退出码
    wait "${pid}" 2>/dev/null
    local exit_code=$?

    # 记录退出信息
    log_info "容器 ${name} 主进程退出, exit_code=${exit_code}"
    state_set_exit_code "${name}" "${exit_code}"

    # 清理运行时资源
    proot_cleanup "${name}"
    audio_cleanup "${name}"
    display_cleanup "${name}"

    # 更新状态
    state_set "${name}" "stopped"
    state_clear_pid "${name}"

    # 如果异常退出，记录诊断信息
    if [[ ${exit_code} -ne 0 ]]; then
        diagnose_collect "${name}" >> "${GRAPEVINE_HOME}/containers/${name}/logs/crash-$(date +%Y%m%d%H%M%S).log"
    fi
}
```

### 4.7 图形管线

#### 4.7.1 驱动检测算法

```bash
# driver.sh::detect_gpu()
function detect_gpu() {
    local gpu_info

    # 方法1: 读取 Android 系统属性
    local hardware
    hardware=$(getprop ro.hardware 2>/dev/null || echo "")

    # 方法2: 读取 /sys 文件系统
    local gpu_name=""
    if [[ -f /sys/class/kgsl-3d0/gpu_model ]]; then
        gpu_name=$(cat /sys/class/kgsl-3d0/gpu_model)
        gpu_info="adreno:${gpu_name}"
    elif [[ -f /sys/class/misc/mali/device/gpuinfo ]]; then
        gpu_name=$(cat /sys/class/misc/mali/device/gpuinfo)
        gpu_info="mali:${gpu_name}"
    elif [[ -d /sys/class/devfreq/gpu ]]; then
        gpu_info="unknown:generic-gpu"
    fi

    # 方法3: 从 /proc/device-tree 推断
    if [[ -z "${gpu_info}" ]]; then
        local compatible
        compatible=$(cat /proc/device-tree/compatible 2>/dev/null | tr '\0' '\n')
        if echo "${compatible}" | grep -qi "adreno"; then
            gpu_info="adreno:unknown"
        elif echo "${compatible}" | grep -qi "mali"; then
            gpu_info="mali:unknown"
        else
            gpu_info="unknown:unknown"
        fi
    fi

    echo "${gpu_info}"
}

# driver.sh::check_vulkan_support()
function check_vulkan_support() {
    local icd_path="$1"

    if [[ ! -f "${icd_path}" ]]; then
        return 1
    fi

    # 尝试使用 vulkaninfo 检测
    if command -v vulkaninfo &>/dev/null; then
        VK_ICD_FILENAMES="${icd_path}" vulkaninfo --summary &>/dev/null
        return $?
    fi

    # 降级: 检查 ICD JSON 文件是否有效
    if grep -q '"library_path"' "${icd_path}"; then
        local lib_path
        lib_path=$(grep '"library_path"' "${icd_path}" | sed 's/.*: *"//;s/".*//')
        [[ -f "${lib_path}" ]] && return 0
    fi

    return 1
}
```

#### 4.7.2 DXVK/VKD3D 注入流程

```
graphics_inject(config):
  │
  ├─ 1. 确定注入目标
  │     prefix = container_path/prefix/
  │
  ├─ 2. DXVK 注入 (如启用)
  │     ├─ 定位 DXVK 组件路径
  │     │   dxvk_path = component_get_path("dxvk", version)
  │     │
  │     ├─ 复制 DLL 到 Wine Prefix
  │     │   for dll in d3d9 d3d10 d3d10_1 d3d11 dxgi; do
  │     │     cp "${dxvk_path}/x64/${dll}.dll" "${prefix}/drive_c/windows/system32/"
  │     │     cp "${dxvk_path}/x32/${dll}.dll" "${prefix}/drive_c/windows/syswow64/"
  │     │   done
  │     │
  │     └─ 设置 DLL 覆盖
  │         WINEDLLOVERRIDES += "d3d9,d3d10,d3d10_1,d3d11,dxgi=n"
  │
  ├─ 3. VKD3D 注入 (如启用)
  │     ├─ 定位 VKD3D 组件路径
  │     │   vkd3d_path = component_get_path("vkd3d", version)
  │     │
  │     ├─ 复制 DLL
  │     │   for dll in d3d12; do
  │     │     cp "${vkd3d_path}/x64/${dll}.dll" "${prefix}/drive_c/windows/system32/"
  │     │     cp "${vkd3d_path}/x32/${dll}.dll" "${prefix}/drive_c/windows/syswow64/"
  │     │   done
  │     │
  │     └─ 设置 DLL 覆盖
  │         WINEDLLOVERRIDES += "d3d12=n"
  │
  ├─ 4. D8VK 注入 (如启用)
  │     └─ 同 DXVK 流程，处理 d3d8.dll
  │
  └─ 5. 写入 DXVK 配置
        write "${prefix}/dxvk.conf"
```

#### 4.7.3 Vulkan ICD 配置

```bash
# driver.sh::configure_vulkan_icd()
function configure_vulkan_icd() {
    local driver="$1"
    local container_name="$2"
    local icd_dir="${GRAPEVINE_HOME}/containers/${container_name}/rootfs/etc/vulkan/icd.d"

    mkdir -p "${icd_dir}"

    case "${driver}" in
        turnip)
            local turnip_lib
            turnip_lib=$(component_get_path "turnip" "latest")/libvulkan_freedreno.so
            cat > "${icd_dir}/freedreno_icd.json" <<EOF
{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "${turnip_lib}",
        "api_version": "1.3.0"
    }
}
EOF
            ;;
        zink)
            local zink_lib
            zink_lib=$(component_get_path "mesa" "latest")/libvulkan_lvp.so
            cat > "${icd_dir}/zink_icd.json" <<EOF
{
    "file_format_version": "1.0.0",
    "ICD": {
        "library_path": "${zink_lib}",
        "api_version": "1.3.0"
    }
}
EOF
            ;;
        virgl)
            # VirGL 不使用 Vulkan ICD，使用 Gallium3D 驱动
            ;;
    esac
}
```

### 4.8 音频桥接

#### 4.8.1 PulseAudio 配置流程

```
audio_setup(config):
  │
  ├─ 1. 检查后端选择
  │     backend = config[audio.backend]
  │
  ├─ 2. PulseAudio 配置 (backend == "pulseaudio")
  │     ├─ 检查 PulseAudio 是否已运行
  │     │   if pulseaudio --check; then
  │     │     使用现有实例
  │     │   else
  │     │     启动新实例
  │     │   fi
  │     │
  │     ├─ 生成 daemon.conf
  │     │   daemon.conf:
  │     │     exit-idle-time = -1        # 不自动退出
  │     │     flat-volumes = no          # 独立音量控制
  │     │     resample-method = speex-fixed-1  # 低 CPU 重采样
  │     │     default-sample-rate = 48000
  │     │     alternate-sample-rate = 44100
  │     │     default-fragments = 2
  │     │     default-fragment-size-msec = 5   # 低延迟
  │     │
  │     ├─ 生成 default.pa
  │     │   加载 module-native-protocol-unix
  │     │   加载 module-alsa-sink (输出到 ALSA)
  │     │   加载 module-alsa-source (如需输入)
  │     │   加载 module-rescue-streams
  │     │
  │     └─ 启动 PulseAudio
  │         pulseaudio --daemonize=no \
  │           --fail=false \
  │           --use-pid-file=false \
  │           --log-target=stderr \
  │           --log-level=notice \
  │           --exit-idle-time=-1
  │
  └─ 3. ALSA 配置 (backend == "alsa")
        ├─ 生成 .asoundrc
        │   defaults.pcm.card 0
        │   defaults.ctl.card 0
        │   pcm.!default { type plug slave.pcm "default" }
        └─ 设置 WINE_NO_AUDIO=0
```

#### 4.8.2 延迟优化

```bash
# pulseaudio.sh::optimize_latency()
function optimize_latency() {
    local config="$1"
    local latency_target="${config[audio.latency_ms]:-50}"

    # 根据延迟目标调整参数
    if [[ ${latency_target} -le 20 ]]; then
        # 超低延迟模式 (可能增加 CPU 使用)
        echo "default-fragments = 2"
        echo "default-fragment-size-msec = 2"
        echo "resample-method = trivial"         # 最快重采样
    elif [[ ${latency_target} -le 50 ]]; then
        # 低延迟模式 (推荐)
        echo "default-fragments = 2"
        echo "default-fragment-size-msec = 5"
        echo "resample-method = speex-fixed-1"
    elif [[ ${latency_target} -le 100 ]]; then
        # 标准延迟模式
        echo "default-fragments = 4"
        echo "default-fragment-size-msec = 10"
        echo "resample-method = speex-fixed-1"
    else
        # 高延迟模式 (最低 CPU)
        echo "default-fragments = 4"
        echo "default-fragment-size-msec = 25"
        echo "resample-method = speex-fixed-3"
    fi
}
```

### 4.9 组件仓库

#### 4.9.1 版本解析

```bash
# manager.sh::component_resolve_version()
# 支持语义化版本范围解析
function component_resolve_version() {
    local component="$1"
    local version_spec="$2"

    # 获取可用版本列表
    local -a available_versions
    available_versions=$(component_list_versions "${component}")

    case "${version_spec}" in
        latest)
            echo "${available_versions[-1]}"
            ;;
        stable)
            # 最新稳定版（不含 -rc, -beta, -alpha 后缀）
            for v in $(echo "${available_versions}" | tac); do
                if [[ ! "${v}" =~ -(rc|beta|alpha) ]]; then
                    echo "${v}"
                    return 0
                fi
            done
            ;;
        *)
            # 精确版本匹配
            if echo "${available_versions}" | grep -qx "${version_spec}"; then
                echo "${version_spec}"
            else
                log_error "组件 ${component} 版本 ${version_spec} 不存在"
                return 3
            fi
            ;;
    esac
}
```

#### 4.9.2 依赖关系

组件间存在依赖关系，定义在组件注册表中：

```yaml
# 组件依赖关系图
dependencies:
  wine:
    requires: [glibc-runtime, box64]
    optional: [box86]           # win32 兼容需要
    conflicts: []

  proton:
    requires: [glibc-runtime, box64]
    optional: [box86]
    conflicts: [wine]           # Proton 与 Wine 互斥

  dxvk:
    requires: [wine, vulkan-icd]    # 需要 Wine + Vulkan 驱动
    optional: []
    conflicts: []

  vkd3d:
    requires: [wine, vulkan-icd]
    optional: []
    conflicts: []

  turnip:
    requires: [mesa-vulkan]
    optional: []
    conflicts: [virgl]          # Turnip 与 VirGL 不应同时使用

  virgl:
    requires: [virglrenderer]
    optional: []
    conflicts: [turnip]

  zink:
    requires: [mesa-vulkan, vulkan-icd]
    optional: []
    conflicts: []

  glibc-runtime:
    requires: []
    optional: []
    conflicts: []
```

**依赖解析算法**（拓扑排序）：

```
component_resolve_deps(component):
  1. 从注册表读取 component 的直接依赖
  2. 递归解析每个依赖的子依赖
  3. 检测循环依赖 (维护 visited 集合)
  4. 拓扑排序，输出安装顺序
  5. 检查冲突关系
  6. 返回有序安装列表

示例:
  安装 dxvk:
    解析: dxvk → wine + vulkan-icd
           wine → glibc-runtime + box64
           box64 → (无)
           glibc-runtime → (无)
           vulkan-icd → turnip (如 GPU 为 Adreno)
           turnip → mesa-vulkan
           mesa-vulkan → (无)

    安装顺序: [glibc-runtime, box64, mesa-vulkan, turnip, wine, dxvk]
```

#### 4.9.3 下载镜像

```bash
# repository.sh::component_download()
function component_download() {
    local component="$1"
    local version="$2"
    local output_dir="$3"

    # 镜像源列表（按优先级）
    local -a mirrors=(
        "${GRAPEVINE_MIRROR:-https://mirror.grapevine.dev}"
        "https://github.com/grapevine/components/releases"
        "https://ghproxy.com/https://github.com/grapevine/components/releases"
    )

    local filename="${component}-${version}-aarch64.tar.zst"
    local success=0

    for mirror in "${mirrors[@]}"; do
        local url="${mirror}/${component}/${version}/${filename}"
        log_info "尝试下载: ${url}"

        if curl --fail --location --progress-bar \
             --connect-timeout 15 \
             --max-time 600 \
             --retry 3 \
             --retry-delay 5 \
             -o "${output_dir}/${filename}" \
             "${url}"; then
            success=1
            break
        fi
    done

    if [[ ${success} -eq 0 ]]; then
        log_error "所有镜像源下载失败: ${component}@${version}"
        return 1
    fi

    echo "${output_dir}/${filename}"
}
```

#### 4.9.4 校验机制

```bash
# repository.sh::component_verify()
function component_verify() {
    local archive_path="$1"
    local component="$2"
    local version="$3"

    # 获取校验和文件
    local checksum_url="${GRAPEVINE_MIRROR:-https://mirror.grapevine.dev}/${component}/${version}/SHA256SUMS"
    local expected_hash
    expected_hash=$(curl --fail --silent "${checksum_url}" | grep "$(basename "${archive_path}")" | awk '{print $1}')

    if [[ -z "${expected_hash}" ]]; then
        log_error "无法获取组件 ${component}@${version} 的校验和"
        return 1
    fi

    # 计算本地文件校验和
    local actual_hash
    actual_hash=$(sha256sum "${archive_path}" | awk '{print $1}')

    if [[ "${actual_hash}" != "${expected_hash}" ]]; then
        log_error "校验和不匹配: expected=${expected_hash}, actual=${actual_hash}"
        return 1
    fi

    # 可选: GPG 签名验证
    if [[ "${GRAPEVINE_VERIFY_SIGNATURE}" == "true" ]]; then
        local sig_url="${GRAPEVINE_MIRROR}/${component}/${version}/SHA256SUMS.sig"
        curl --fail --silent -o "${archive_path}.sig" "${sig_url}"
        gpg --verify "${archive_path}.sig" <<< "${expected_hash}" 2>/dev/null
        if [[ $? -ne 0 ]]; then
            log_error "GPG 签名验证失败"
            return 1
        fi
    fi

    log_info "组件 ${component}@${version} 校验通过"
    return 0
}
```

---

## 5. 数据模型设计

### 5.1 容器数据模型

#### container.yml

```yaml
apiVersion: v1
kind: Container
metadata:
  name: my-game                  # [必填] 容器名, 正则: ^[a-z][a-z0-9_-]{1,62}$
  created_at: "2026-05-06T14:30:00Z"  # [系统生成] ISO 8601 时间戳
  updated_at: "2026-05-06T14:30:00Z"  # [系统生成] 最后更新时间
  template: gaming-dx11          # [可选] 来源模板名
  labels:                        # [可选] 用户自定义标签
    purpose: gaming

spec:
  wine:
    runtime: wine                # [必填] wine | proton
    version: "9.0"               # [必填] 语义化版本
    arch: win64                  # [必填] win32 | win64
    staging: true                # [可选] 是否使用 wine-staging, 默认 false
    debug_level: "fixme-all"     # [可选] Wine 调试级别
    esync: true                  # [可选] 启用 esync, 默认 true
    fsync: false                 # [可选] 启用 fsync, 默认 false

  box64:
    version: "0.2.8"             # [必填] Box64 版本
    dynarec: true                # [可选] 启用动态重编译器, 默认 true
    log_level: 0                 # [可选] 日志级别 0-3, 默认 0

  box86:
    version: "0.8.6"             # [可选] Box86 版本, win32 应用需要
    dynarec: true
    log_level: 0

  graphics:
    driver: auto                 # [必填] auto | turnip | virgl | zink
    dxvk:
      version: "2.3"             # [条件必填] DXVK 版本
      enabled: true              # [可选] 默认 true
    vkd3d:
      version: "2.12"
      enabled: false
    d8vk:
      version: "1.0"
      enabled: false
    turnip:
      debug: "noconform"         # [可选] Turnip 调试标志

  audio:
    backend: pulseaudio          # [必填] pulseaudio | alsa | none
    latency_ms: 50               # [可选] 目标延迟(毫秒), 默认 50

  display:
    backend: x11                 # [必填] x11 | none
    resolution: "1280x720"       # [可选] 分辨率, 格式: WxH
    cursor_size: 24              # [可选] 光标大小, 默认 24

  proot:
    binds:                       # [可选] 额外 bind mount 列表
      - /system/lib64
      - /vendor/lib64
    share_sdcard: false          # [可选] 挂载 /sdcard, 默认 false

  container:
    disk_limit: "8G"             # [可选] 磁盘限额
    memory_limit: "4G"           # [可选] 内存限额
    command: ""                  # [可选] 启动命令, 为空则执行 wineboot
    env:                         # [可选] 额外环境变量
      MY_VAR: "my_value"

  post_create:                   # [可选] 创建后执行命令列表
    - command: "winetricks dxvk"
      description: "Install DXVK"
```

#### state.json

```json
{
    "name": "my-game",
    "state": "stopped",
    "pid": null,
    "exit_code": 0,
    "created_at": "2026-05-06T14:30:00Z",
    "started_at": "2026-05-06T14:35:22Z",
    "stopped_at": "2026-05-06T16:10:45Z",
    "snapshot_count": 2,
    "disk_usage_bytes": 3145728000,
    "last_error": null
}
```

**字段约束**：

| 字段 | 类型 | 约束 |
|------|------|------|
| name | string | 与 container.yml metadata.name 一致 |
| state | enum | created, running, stopped, paused, deleted |
| pid | integer\|null | 仅 running 状态有值 |
| exit_code | integer\|null | 仅 stopped 状态有值，0=正常退出 |
| created_at | string | ISO 8601 格式 |
| started_at | string\|null | ISO 8601 格式 |
| stopped_at | string\|null | ISO 8601 格式 |
| snapshot_count | integer | ≥ 0 |
| disk_usage_bytes | integer | ≥ 0 |
| last_error | string\|null | 最近一次错误信息 |

### 5.2 快照数据模型

#### meta.yml

```yaml
apiVersion: v1
kind: Snapshot
metadata:
  id: snap_20260506143022_a3f2    # [系统生成] 快照 ID
  container: my-game               # [必填] 所属容器名
  created_at: "2026-05-06T14:30:22Z"
  message: "安装游戏前"            # [可选] 快照描述
  type: incremental                # full | incremental
  parent: snap_20260506120000_b7c1 # [可选] 父快照 ID, full 类型为 null

spec:
  file_count: 12453                # 快照包含文件数
  added_count: 234                 # 新增文件数
  modified_count: 56               # 修改文件数
  removed_count: 12                # 删除文件数
  data_size_bytes: 52428800        # 增量数据大小
  total_size_bytes: 1073741824     # 完整恢复所需总大小
  compression: zstd                # 压缩算法
  compression_level: 19            # 压缩级别
  checksum_sha256: "abc123..."     # data.tar.zst 的 SHA256
```

### 5.3 模板数据模型

```yaml
apiVersion: v1                     # [必填] 模板 API 版本, 当前仅 v1
kind: ContainerTemplate            # [必填] 固定值
metadata:
  name: gaming-dx11                # [必填] 模板名, 正则: ^[a-z][a-z0-9-]{1,63}$
  description: "DirectX 11 游戏容器模板"  # [必填] 描述
  tags: [gaming, dx11, vulkan]     # [可选] 标签列表
  author: grapevine                # [可选] 作者
  version: "1.0.0"                 # [必填] 模板版本

spec:
  extends: null                    # [可选] 父模板名, 用于继承
  wine:                            # 同容器 spec.wine
  box64:                           # 同容器 spec.box64
  box86:                           # 同容器 spec.box86
  graphics:                        # 同容器 spec.graphics
  audio:                           # 同容器 spec.audio
  display:                         # 同容器 spec.display
  proot:                           # 同容器 spec.proot
  container:                       # 同容器 spec.container
  post_create:                     # 同容器 spec.post_create
```

### 5.4 组件注册表数据模型

#### ~/.grapevine/components/registry.yml

```yaml
apiVersion: v1
kind: ComponentRegistry
components:
  wine:
    type: runtime
    latest: "9.0"
    versions:
      - version: "9.0"
        stable: true
        release_date: "2024-01-01"
        sha256: "abc123..."
        size_bytes: 104857600
        url: "wine/9.0/wine-9.0-aarch64.tar.zst"
        requires: [glibc-runtime, box64]
        optional: [box86]
        conflicts: [proton]
      - version: "8.0"
        stable: true
        release_date: "2023-01-01"
        sha256: "def456..."
        size_bytes: 98304000
        url: "wine/8.0/wine-8.0-aarch64.tar.zst"
        requires: [glibc-runtime, box64]
        optional: [box86]
        conflicts: [proton]

  box64:
    type: runtime
    latest: "0.2.8"
    versions:
      - version: "0.2.8"
        stable: true
        release_date: "2024-03-01"
        sha256: "789abc..."
        size_bytes: 5242880
        url: "box64/0.2.8/box64-0.2.8-aarch64.tar.zst"
        requires: []
        optional: []
        conflicts: []

  dxvk:
    type: graphics
    latest: "2.3"
    versions:
      - version: "2.3"
        stable: true
        release_date: "2024-02-01"
        sha256: "123def..."
        size_bytes: 15728640
        url: "dxvk/2.3/dxvk-2.3-aarch64.tar.zst"
        requires: [wine, vulkan-icd]
        optional: []
        conflicts: []

  glibc-runtime:
    type: system
    latest: "2.38"
    versions:
      - version: "2.38"
        stable: true
        release_date: "2024-01-01"
        sha256: "456789..."
        size_bytes: 209715200
        url: "glibc-runtime/2.38/glibc-runtime-2.38-aarch64.tar.zst"
        requires: []
        optional: []
        conflicts: []
```

### 5.5 全局配置数据模型

#### ~/.grapevine/config.yml

```yaml
apiVersion: v1
kind: GlobalConfig

# 存储配置
storage:
  base_dir: "~/.grapevine"              # 基础目录
  temp_dir: "${base_dir}/tmp"           # 临时目录
  cache_dir: "${base_dir}/cache"        # 缓存目录
  compression: zstd                      # 压缩算法: zstd | gzip | xz
  compression_level: 3                   # 压缩级别 (zstd: 1-19)

# 网络配置
network:
  mirror: "https://mirror.grapevine.dev"  # 主镜像源
  fallback_mirrors:                        # 备用镜像源
    - "https://github.com/grapevine/components/releases"
  proxy: ""                                # HTTP 代理
  connect_timeout: 15                      # 连接超时(秒)
  max_retries: 3                           # 最大重试次数

# 安全配置
security:
  verify_signature: false                  # GPG 签名验证
  verify_checksum: true                    # SHA256 校验和验证
  allowed_bind_paths:                      # 允许 bind mount 的路径白名单
    - /system
    - /vendor
    - /sdcard
    - /data

# 容器默认值
containers:
  defaults:
    wine:
      runtime: wine
      version: "9.0"
      arch: win64
    graphics:
      driver: auto
    audio:
      backend: pulseaudio
    display:
      backend: x11

# 日志配置
logging:
  level: info                              # debug | info | warn | error
  max_file_size: "10M"                     # 单个日志文件最大大小
  max_files: 5                             # 日志文件轮转数量
  log_dir: "${base_dir}/logs"              # 日志目录

# 组件配置
components:
  auto_update: false                       # 自动更新组件
  keep_old_versions: 2                     # 保留旧版本数量
```

---

## 6. 接口设计

### 6.1 CLI 接口规范

#### 6.1.1 通用规范

| 项目 | 规范 |
|------|------|
| 命令格式 | `grapevine <resource> <action> [arguments] [options]` |
| 参数风格 | GNU 风格长选项 (`--option`)，短选项 (`-o`) |
| 布尔选项 | `--force`, `--no-force` |
| 值选项 | `--name value` 或 `--name=value` |
| 多值选项 | `--set key1=val1 --set key2=val2` |
| 输出格式 | 默认人类可读，`--format json|yaml|table` 切换 |
| 分页 | 长输出自动分页 (`less -R`) |
| 进度 | 耗时操作显示进度条 |
| 彩色 | 自动检测 TTY，`--color=always|never|auto` |

#### 6.1.2 返回码定义

| 返回码 | 含义 |
|--------|------|
| 0 | 成功 |
| 1 | 一般错误 |
| 2 | 配置错误（配置文件语法错误、校验失败） |
| 3 | 依赖缺失（组件未安装、版本不兼容） |
| 4 | 状态冲突（非法状态转换、容器已存在） |
| 5 | 权限不足（文件不可写、设备不可访问） |
| 6 | 资源不足（磁盘空间不足、内存不足） |
| 7 | 网络错误（下载失败、镜像不可达） |
| 8 | 操作超时（进程未响应、锁超时） |
| 9 | 校验失败（SHA256 不匹配、签名验证失败） |
| 10 | 用户取消（Ctrl+C、交互式拒绝） |

#### 6.1.3 命令详细规范

**grapevine container create**

```
名称:   grapevine container create
用法:   grapevine container create -t TEMPLATE -n NAME [OPTIONS]
参数:
  -t, --template TEMPLATE       模板名称或路径 [必填]
  -n, --name NAME               容器名称 [必填]
      --set KEY=VALUE           覆盖配置值（可重复）
      --from-archive PATH       从归档文件导入创建
      --no-post-create          跳过 post_create 命令
      --dry-run                 仅校验配置，不实际创建
输出:   创建成功输出容器名称
返回码: 0=成功, 2=配置错误, 4=名称冲突, 6=资源不足

示例:
  grapevine container create -t gaming-dx11 -n my-game
  grapevine container create -t gaming-dx11 -n my-game --set wine.version=9.4
```

**grapevine container start**

```
名称:   grapevine container start
用法:   grapevine container start NAME [OPTIONS]
参数:
  NAME                          容器名称 [必填]
      --command CMD             覆盖启动命令
      --attach                  启动后附加到进程输出
      --env KEY=VALUE           额外环境变量（可重复）
输出:   启动成功输出 PID
返回码: 0=成功, 3=依赖缺失, 4=状态冲突, 5=权限不足

示例:
  grapevine container start my-game
  grapevine container start my-game --command "wine steam.exe"
```

**grapevine container list**

```
名称:   grapevine container list
用法:   grapevine container list [OPTIONS]
参数:
      --format FORMAT           输出格式: table|json|yaml [默认: table]
      --filter STATE            按状态过滤: running|stopped|created|all
      --label KEY=VALUE         按标签过滤
输出:   容器列表（表格/JSON/YAML）
返回码: 0=成功

示例:
  grapevine container list
  grapevine container list --format json --filter running
```

**grapevine snapshot create**

```
名称:   grapevine snapshot create
用法:   grapevine snapshot create CONTAINER [OPTIONS]
参数:
  CONTAINER                     容器名称 [必填]
  -m, --message MESSAGE         快照描述
      --full                    强制全量快照
输出:   快照 ID
返回码: 0=成功, 4=状态冲突（容器运行中）

示例:
  grapevine snapshot create my-game -m "安装游戏前"
```

**grapevine component install**

```
名称:   grapevine component install
用法:   grapevine component install COMPONENT[@VERSION] [OPTIONS]
参数:
  COMPONENT[@VERSION]           组件名及可选版本 [必填]
      --no-deps                 跳过依赖安装
      --force                   强制重新安装
输出:   安装进度条 + 结果摘要
返回码: 0=成功, 3=依赖缺失, 7=网络错误, 9=校验失败

示例:
  grapevine component install wine@9.0
  grapevine component install dxvk
```

### 6.2 模块间接口

#### 6.2.1 Shell 函数签名约定

所有模块间接口遵循以下约定：

```bash
# 命名规范: <module>_<action>
# 参数传递: 位置参数
# 返回值: exit code (0=成功, 非零=失败)
# 数据输出: stdout (结构化文本)
# 日志输出: stderr

# === lib/core/container.sh ===
container_create()        # 用法: container_create NAME TEMPLATE [OVERRIDES...]
container_start()         # 用法: container_start NAME [OPTIONS...]
container_stop()          # 用法: container_stop NAME [FORCE]
container_delete()        # 用法: container_delete NAME [PURGE]
container_list()          # 用法: container_list [FORMAT] [FILTER]
container_inspect()       # 用法: container_inspect NAME
container_exec()          # 用法: container_exec NAME COMMAND [ARGS...]

# === lib/core/snapshot.sh ===
snapshot_create()         # 用法: snapshot_create CONTAINER [MESSAGE] [FULL]
snapshot_restore()        # 用法: snapshot_restore CONTAINER SNAPSHOT_ID
snapshot_list()           # 用法: snapshot_list CONTAINER
snapshot_delete()         # 用法: snapshot_delete CONTAINER SNAPSHOT_ID

# === lib/core/template.sh ===
template_resolve()        # 用法: template_resolve TEMPLATE_NAME_OR_PATH
template_validate()       # 用法: template_validate TEMPLATE_PATH
template_list()           # 用法: template_list
template_substitute()     # 用法: template_substitute CONTENT VARS...

# === lib/core/config.sh ===
config_load()             # 用法: config_load NAME
config_merge()            # 用法: config_merge BASE OVERRIDE
config_validate()         # 用法: config_validate CONFIG
config_get()              # 用法: config_get KEY
config_set()              # 用法: config_set KEY VALUE

# === lib/core/state.sh ===
state_get()               # 用法: state_get NAME → 输出状态字符串
state_set()               # 用法: state_set NAME STATE
state_transition()        # 用法: state_transition NAME TARGET_STATE
state_get_pid()           # 用法: state_get_pid NAME → 输出 PID
state_set_pid()           # 用法: state_set_pid NAME PID
state_clear_pid()         # 用法: state_clear_pid NAME

# === lib/runtime/wine.sh ===
wine_launch()             # 用法: wine_launch CONFIG
wine_build_env()          # 用法: wine_build_env CONFIG → 输出 export 语句
wine_init_prefix()        # 用法: wine_init_prefix PREFIX_PATH ARCH

# === lib/runtime/proton.sh ===
proton_launch()           # 用法: proton_launch CONFIG
proton_build_env()        # 用法: proton_build_env CONFIG

# === lib/runtime/box64.sh ===
box64_setup_env()         # 用法: box64_setup_env CONFIG
box64_get_path()          # 用法: box64_get_path VERSION → 输出路径

# === lib/runtime/box86.sh ===
box86_setup_env()         # 用法: box86_setup_env CONFIG
box86_get_path()          # 用法: box86_get_path VERSION → 输出路径

# === lib/runtime/proot.sh ===
proot_build_cmd()         # 用法: proot_build_cmd NAME CONFIG → 输出完整命令
proot_setup_bindings()    # 用法: proot_setup_bindings NAME CONFIG
proot_cleanup()           # 用法: proot_cleanup NAME

# === lib/graphics/driver.sh ===
driver_detect()           # 用法: driver_detect → 输出 GPU 信息
driver_select()           # 用法: driver_select CONFIG → 输出驱动名
driver_configure_vulkan() # 用法: driver_configure_vulkan DRIVER CONTAINER

# === lib/graphics/dxvk.sh ===
dxvk_inject()             # 用法: dxvk_inject PREFIX_PATH VERSION
dxvk_remove()             # 用法: dxvk_remove PREFIX_PATH

# === lib/graphics/vkd3d.sh ===
vkd3d_inject()            # 用法: vkd3d_inject PREFIX_PATH VERSION
vkd3d_remove()            # 用法: vkd3d_remove PREFIX_PATH

# === lib/audio/pulseaudio.sh ===
pulseaudio_setup()        # 用法: pulseaudio_setup CONFIG
pulseaudio_cleanup()      # 用法: pulseaudio_cleanup NAME
pulseaudio_optimize()     # 用法: pulseaudio_optimize LATENCY_MS

# === lib/audio/alsa.sh ===
alsa_setup()              # 用法: alsa_setup CONFIG
alsa_cleanup()            # 用法: alsa_cleanup NAME

# === lib/display/x11.sh ===
x11_setup()               # 用法: x11_setup CONFIG
x11_cleanup()             # 用法: x11_cleanup NAME

# === lib/component/manager.sh ===
component_install()       # 用法: component_install COMPONENT [VERSION]
component_remove()        # 用法: component_remove COMPONENT
component_update()        # 用法: component_update [COMPONENT]
component_resolve_version() # 用法: component_resolve_version COMPONENT SPEC
component_resolve_deps()  # 用法: component_resolve_deps COMPONENT → 输出有序列表
component_get_path()      # 用法: component_get_path COMPONENT VERSION → 输出路径
component_verify()        # 用法: component_verify ARCHIVE COMPONENT VERSION

# === lib/component/repository.sh ===
repository_fetch_index()  # 用法: repository_fetch_index → 更新本地缓存
repository_download()     # 用法: repository_download COMPONENT VERSION OUTPUT_DIR
repository_get_versions() # 用法: repository_get_versions COMPONENT → 输出版本列表

# === lib/utils/yaml.sh ===
yaml_parse()              # 用法: yaml_parse INPUT → 输出 KEY=VALUE
yaml_get()                # 用法: yaml_get KEY < YAML_FILE → 输出值
yaml_set()                # 用法: yaml_set KEY VALUE YAML_FILE

# === lib/utils/log.sh ===
log_debug()               # 用法: log_debug MESSAGE
log_info()                # 用法: log_info MESSAGE
log_warn()                # 用法: log_warn MESSAGE
log_error()               # 用法: log_error MESSAGE

# === lib/utils/io.sh ===
io_confirm()              # 用法: io_confirm MESSAGE [DEFAULT] → 0=确认, 1=取消
io_progress()             # 用法: io_progress CURRENT TOTAL MESSAGE
io_spinner()              # 用法: io_spinner PID MESSAGE

# === lib/utils/gpu.sh ===
gpu_detect()              # 用法: gpu_detect → 输出 GPU 信息
gpu_check_vulkan()        # 用法: gpu_check_vulkan ICD_PATH → 0=支持
```

#### 6.2.2 调用约定

```bash
# 1. 模块加载
#    所有模块通过 source 加载，使用防重复加载守卫:
if [[ -z "${_GRAPEVINE_CORE_CONTAINER_LOADED:-}" ]]; then
    _GRAPEVINE_CORE_CONTAINER_LOADED=1
    # ... 模块定义 ...
fi

# 2. 全局变量命名
#    所有全局变量使用 GRAPEVINE_ 前缀:
GRAPEVINE_HOME="${HOME}/.grapevine"
GRAPEVINE_VERSION="1.0.0"
GRAPEVINE_LOG_LEVEL="info"

# 3. 错误传播
#    调用链中任何函数失败，立即返回非零退出码
#    使用 set -e 或显式检查:
result=$(some_function) || return $?

# 4. 输出隔离
#    函数的"返回数据"通过 stdout 输出
#    日志信息通过 stderr 输出（使用 log_* 函数）
#    调用方使用 $(...) 捕获 stdout
```

### 6.3 配置文件接口

#### 6.3.1 container.yml Schema

```yaml
# JSON Schema 风格定义（用于文档与校验）
type: object
required: [apiVersion, kind, metadata, spec]
properties:
  apiVersion:
    type: string
    enum: [v1]
    description: "API 版本"
  kind:
    type: string
    enum: [Container]
    description: "资源类型"
  metadata:
    type: object
    required: [name]
    properties:
      name:
        type: string
        pattern: "^[a-z][a-z0-9_-]{1,62}$"
        description: "容器名称"
      created_at:
        type: string
        format: date-time
        description: "创建时间"
      updated_at:
        type: string
        format: date-time
        description: "更新时间"
      template:
        type: string
        description: "来源模板"
      labels:
        type: object
        additionalProperties:
          type: string
  spec:
    type: object
    required: [wine]
    properties:
      wine:
        type: object
        required: [runtime, version, arch]
        properties:
          runtime:
            type: string
            enum: [wine, proton]
          version:
            type: string
            pattern: "^[0-9]+\\.[0-9]+(\\.[0-9]+)?$"
          arch:
            type: string
            enum: [win32, win64]
          staging:
            type: boolean
            default: false
          debug_level:
            type: string
            default: "fixme-all"
          esync:
            type: boolean
            default: true
          fsync:
            type: boolean
            default: false
      # ... 其他 spec 字段同上 ...
```

---

## 7. 进程模型与并发设计

### 7.1 进程树结构

```
Termux Shell (bash)
  │
  └── grapevine (主进程)
        │
        ├── proot (文件系统隔离)
        │     │
        │     ├── box64 (x64 指令转译)
        │     │     │
        │     │     ├── wine64 (Windows 兼容层)
        │     │     │     ├── wineserver (Wine 服务进程)
        │     │     │     ├── application.exe (目标应用)
        │     │     │     └── services.exe (Windows 服务)
        │     │     │
        │     │     └── (其他 x64 进程)
        │     │
        │     └── box86 (x86 指令转译, 按需)
        │           │
        │           └── wine (32位 Windows 兼容)
        │                 └── (32位应用进程)
        │
        ├── pulseaudio (音频服务, 按需)
        │
        ├── termux-x11 (显示服务, 按需)
        │
        └── container_monitor (监控协程)
```

### 7.2 信号处理

```bash
# grapevine 主进程信号处理
function setup_signal_handlers() {
    trap 'handle_sigterm' SIGTERM
    trap 'handle_sigint' SIGINT
    trap 'handle_sighup' SIGHUP
    trap 'handle_sigchld' SIGCHLD
}

function handle_sigterm() {
    log_info "收到 SIGTERM，优雅停止容器..."
    container_stop "${CURRENT_CONTAINER}" false
    exit 0
}

function handle_sigint() {
    log_info "收到 SIGINT (Ctrl+C)..."
    if io_confirm "确定要停止容器吗?" "n"; then
        container_stop "${CURRENT_CONTAINER}" false
    fi
}

function handle_sighup() {
    log_info "收到 SIGHUP，重新加载配置..."
    config_reload "${CURRENT_CONTAINER}"
}

function handle_sigchld() {
    # 子进程退出通知
    # 检查关键子进程状态
    local pid
    for pid in $(jobs -p); do
        if ! kill -0 "${pid}" 2>/dev/null; then
            wait "${pid}"
            local exit_code=$?
            log_warn "子进程 ${pid} 退出, code=${exit_code}"
        fi
    done
}
```

### 7.3 资源清理

```bash
# container.sh::container_cleanup()
# 确保在任何退出路径上清理运行时资源
function container_cleanup() {
    local name="$1"

    # 注册清理函数（使用 EXIT trap 确保执行）
    trap 'cleanup_on_exit "${name}"' EXIT
}

function cleanup_on_exit() {
    local name="$1"

    # 1. 杀死进程树
    local pid
    pid=$(state_get_pid "${name}" 2>/dev/null)
    if [[ -n "${pid}" ]]; then
        # 杀死整个进程组
        kill -- -"${pid}" 2>/dev/null || kill "${pid}" 2>/dev/null
    fi

    # 2. 卸载 PRoot 绑定
    proot_cleanup "${name}" 2>/dev/null

    # 3. 停止音频服务
    audio_cleanup "${name}" 2>/dev/null

    # 4. 停止显示服务
    display_cleanup "${name}" 2>/dev/null

    # 5. 释放文件锁
    container_unlock "${name}" 2>/dev/null

    # 6. 清理临时文件
    rm -f "${GRAPEVINE_HOME}/containers/${name}/.pid"
    rm -f "${GRAPEVINE_HOME}/containers/${name}/.lock"

    # 7. 更新状态
    state_set "${name}" "stopped" 2>/dev/null

    log_info "容器 ${name} 资源清理完成"
}
```

---

## 8. 存储设计

### 8.1 目录布局

```
~/.grapevine/
├── config.yml                          # 全局配置
├── components/                         # 组件存储
│   ├── registry.yml                    # 组件注册表
│   ├── cache/                          # 下载缓存
│   │   └── wine-9.0-aarch64.tar.zst
│   ├── wine-9.0/                       # Wine 9.0 安装
│   │   ├── bin/
│   │   ├── lib/
│   │   └── share/
│   ├── box64-0.2.8/                    # Box64 安装
│   │   ├── bin/
│   │   └── lib/
│   ├── box86-0.8.6/
│   ├── dxvk-2.3/
│   ├── vkd3d-2.12/
│   ├── glibc-runtime-2.38/
│   │   ├── usr/
│   │   ├── lib/
│   │   └── lib64/
│   ├── turnip-latest/
│   └── virgl-latest/
├── containers/                         # 容器存储
│   ├── my-game/                        # 容器实例
│   │   ├── container.yml               # 容器配置
│   │   ├── state.json                  # 运行时状态
│   │   ├── .pid                        # PID 文件
│   │   ├── .lock                       # 文件锁
│   │   ├── rootfs/                     # 根文件系统
│   │   │   ├── etc/
│   │   │   ├── tmp/
│   │   │   └── opt/
│   │   ├── prefix/                     # Wine Prefix
│   │   │   ├── drive_c/
│   │   │   │   ├── windows/
│   │   │   │   ├── Program Files/
│   │   │   │   └── users/
│   │   │   ├── drive_d/
│   │   │   ├── dosdevices/
│   │   │   ├── system.reg
│   │   │   ├── user.reg
│   │   │   └── dxvk.conf
│   │   ├── snapshots/                  # 快照存储
│   │   │   ├── chain.yml
│   │   │   ├── snap_20260506120000_b7c1/
│   │   │   └── snap_20260506143022_a3f2/
│   │   └── logs/                       # 日志
│   │       ├── container.log
│   │       ├── wine.log
│   │       └── crash-20260506161045.log
│   └── office-app/
├── templates/                          # 模板存储（符号链接到安装目录）
│   ├── gaming-dx11.yml
│   ├── gaming-dx9.yml
│   ├── gaming-dx12.yml
│   ├── office.yml
│   ├── development.yml
│   └── minimal.yml
├── cache/                              # 全局缓存
│   ├── downloads/                      # 下载缓存
│   └── gpu-info.cache                  # GPU 信息缓存
├── logs/                               # 全局日志
│   └── grapevine.log
└── tmp/                                # 临时文件
```

### 8.2 文件格式

| 文件类型 | 格式 | 说明 |
|----------|------|------|
| 配置文件 | YAML | 人类可读，支持注释 |
| 状态文件 | JSON | 机器可读，原子写入 |
| 快照数据 | tar.zst | tar 归档 + zstd 压缩 |
| 组件包 | tar.zst | tar 归档 + zstd 压缩 |
| 日志文件 | 纯文本 | 带时间戳的结构化日志 |
| 校验和 | SHA256SUMS | GNU 核心工具格式 |
| 锁文件 | flock | 文件描述符锁 |

**原子写入策略**：

```bash
# io.sh::io_atomic_write()
# 确保文件写入的原子性（避免半写状态）
function io_atomic_write() {
    local target="$1"
    local content="$2"
    local tmp_file

    tmp_file=$(mktemp "${target}.XXXXXX")
    echo "${content}" > "${tmp_file}"
    mv "${tmp_file}" "${target}"    # mv 在同一文件系统上是原子操作
}
```

### 8.3 压缩策略

| 场景 | 算法 | 级别 | 压缩比 | 速度 |
|------|------|------|--------|------|
| 快照存储 | zstd | 19 | 高 | 慢（可接受，离线操作） |
| 组件包分发 | zstd | 3 | 中 | 快 |
| 日志轮转 | gzip | 6 | 中 | 中 |
| 临时传输 | zstd | 1 | 低 | 极快 |

### 8.4 清理策略

```bash
# 自动清理规则
CLEANUP_RULES=(
    # 规则格式: "目标:条件:动作"
    "cache/downloads:age>7d:delete"           # 下载缓存超过 7 天删除
    "logs/*.log:size>10M:rotate"              # 日志文件超过 10M 轮转
    "logs/*.log:count>5:delete_oldest"        # 日志文件超过 5 个删除最旧
    "containers/*/snapshots:count>10:warn"    # 快照超过 10 个发出警告
    "tmp/*:age>1d:delete"                     # 临时文件超过 1 天删除
    "components/cache:age>30d:delete"         # 组件缓存超过 30 天删除
)

# grapevine config set storage.auto_cleanup true  启用自动清理
# 每次容器停止时执行清理检查
```

---

## 9. 错误处理设计

### 9.1 错误码体系

```bash
# 错误码定义: E<类别><编号>
# 类别: 1=系统, 2=容器, 3=配置, 4=组件, 5=运行时, 6=图形, 7=音频, 8=网络, 9=存储

declare -A ERROR_CODES=(
    # 系统错误 (1xxx)
    [E1001]="PRoot 启动失败"
    [E1002]="PRoot 绑定挂载失败"
    [E1003]="文件系统初始化失败"
    [E1004]="权限不足"
    [E1005]="Shell 环境不兼容"

    # 容器错误 (2xxx)
    [E2001]="容器不存在"
    [E2002]="容器已存在"
    [E2003]="非法状态转换"
    [E2004]="容器锁获取超时"
    [E2005]="容器进程未响应"
    [E2006]="容器进程异常退出"

    # 配置错误 (3xxx)
    [E3001]="YAML 语法错误"
    [E3002]="必填字段缺失"
    [E3003]="字段值校验失败"
    [E3004]="配置合并冲突"
    [E3005]="模板未找到"
    [E3006]="模板继承循环"

    # 组件错误 (4xxx)
    [E4001]="组件未安装"
    [E4002]="组件版本不兼容"
    [E4003]="组件依赖缺失"
    [E4004]="组件依赖冲突"
    [E4005]="组件校验失败"
    [E4006]="组件安装失败"

    # 运行时错误 (5xxx)
    [E5001]="Wine 启动失败"
    [E5002]="Wine Prefix 初始化失败"
    [E5003]="Box64 初始化失败"
    [E5004]="Box86 初始化失败"
    [E5005]="wineserver 通信失败"

    # 图形错误 (6xxx)
    [E6001]="GPU 检测失败"
    [E6002]="Vulkan 驱动不可用"
    [E6003]="DXVK 注入失败"
    [E6004]="VKD3D 注入失败"
    [E6005]="ICD 配置失败"

    # 音频错误 (7xxx)
    [E7001]="PulseAudio 启动失败"
    [E7002]="ALSA 设备不可用"
    [E7003]="音频配置失败"

    # 网络错误 (8xxx)
    [E8001]="镜像源不可达"
    [E8002]="下载失败"
    [E8003]="下载超时"
    [E8004]="DNS 解析失败"

    # 存储错误 (9xxx)
    [E9001]="磁盘空间不足"
    [E9002]="文件写入失败"
    [E9003]="快照创建失败"
    [E9004]="快照恢复失败"
    [E9005]="归档解压失败"
)
```

### 9.2 错误传播链

```
底层错误 → 中间层包装 → 顶层用户提示

示例:
  底层: curl 返回 exit code 28 (超时)
    ↓
  repository.sh: 包装为 E8003 "下载超时: https://mirror.grapevine.dev/dxvk/2.3/..."
    ↓
  manager.sh: 包装为 E4006 "组件 dxvk@2.3 安装失败: E8003"
    ↓
  container.sh: 包装为 "容器 my-game 启动失败: 组件 dxvk@2.3 未安装 (E4006)"
    ↓
  CLI: 输出用户友好提示 + 建议操作
```

**错误包装函数**：

```bash
# log.sh::error_wrap()
function error_wrap() {
    local error_code="$1"
    local context="$2"
    local detail="${3:-}"
    local message="${ERROR_CODES[${error_code}]:-未知错误}"

    local full_message="[${error_code}] ${message}"
    [[ -n "${context}" ]] && full_message+=" | 上下文: ${context}"
    [[ -n "${detail}" ]] && full_message+=" | 详情: ${detail}"

    log_error "${full_message}"
    echo "${full_message}" >&2
    return 1
}
```

### 9.3 恢复策略

| 错误类别 | 恢复策略 |
|----------|----------|
| 网络错误 | 自动重试 3 次（指数退避），切换备用镜像源 |
| 组件校验失败 | 删除缓存文件，重新下载 |
| Wine 启动失败 | 检查 Prefix 完整性，尝试修复或重建 |
| GPU 检测失败 | 降级到软件渲染（VirGL/WineD3D） |
| PRoot 启动失败 | 检查 glibc-runtime 完整性，重新安装 |
| 快照恢复失败 | 回滚到恢复前自动创建的安全快照 |
| 磁盘空间不足 | 提示清理缓存/旧快照，自动清理临时文件 |
| 进程无响应 | 先 SIGTERM，30s 后 SIGKILL，收集崩溃日志 |

### 9.4 日志规范

```bash
# 日志格式
# [TIMESTAMP] [LEVEL] [MODULE] [CONTEXT] MESSAGE

# 示例:
# [2026-05-06T14:30:22+08:00] [INFO] [container] [my-game] 容器启动成功, pid=12345
# [2026-05-06T14:30:23+08:00] [WARN] [driver] [my-game] Vulkan ICD 不可用, 降级到 VirGL
# [2026-05-06T14:30:24+08:00] [ERROR] [wine] [my-game] [E5001] Wine 启动失败 | 详情: wineserver 无法绑定 socket

# 日志级别使用规范:
# DEBUG: 函数调用追踪、变量值、详细执行步骤
# INFO:  操作开始/完成、状态变更、组件安装
# WARN:  降级操作、非致命错误、即将过期
# ERROR: 操作失败、错误码、需要用户干预
```

---

## 10. 安全设计

### 10.1 文件系统隔离

| 隔离措施 | 实现方式 |
|----------|----------|
| 根文件系统隔离 | PRoot chroot，容器仅能访问 rootfs 内文件 |
| 组件只读挂载 | Wine/Box64 等组件通过 `--bind` 只读挂载 |
| 敏感路径保护 | `/proc`, `/sys` 仅只读挂载 |
| Android 数据保护 | `/sdcard` 默认不挂载，需显式启用 |
| 容器间隔离 | 每个容器独立 rootfs，无共享路径 |

```bash
# proot.sh::build_bind_options()
# 安全的 bind mount 配置
function build_bind_options() {
    local config="$1"
    local -a binds

    # 只读系统目录
    binds+=("--bind=/dev:/dev:ro")
    binds+=("--bind=/proc:/proc:ro")
    binds+=("--bind=/sys:/sys:ro")

    # 只读组件目录
    binds+=("--bind=${COMPONENT_PATH}/wine:/opt/wine:ro")
    binds+=("--bind=${COMPONENT_PATH}/box64:/opt/box64:ro")
    binds+=("--bind=${COMPONENT_PATH}/box86:/opt/box86:ro")

    # 读写容器数据
    binds+=("--bind=${CONTAINER_PATH}/prefix:/home/user:rw")

    # 可选: Android 存储（需用户确认）
    if [[ "${config[container.share_sdcard]}" == "true" ]]; then
        binds+=("--bind=/sdcard:/sdcard:rw")
    fi

    echo "${binds[@]}"
}
```

### 10.2 权限控制

```bash
# 权限模型:
# 1. Grapevine 数据目录: 0700 (仅用户可访问)
# 2. 容器目录: 0700
# 3. 配置文件: 0600 (仅用户可读写)
# 4. 可执行文件: 0700
# 5. 组件文件: 0500 (只读，防篡改)

function ensure_permissions() {
    chmod 700 "${GRAPEVINE_HOME}"
    chmod 700 "${GRAPEVINE_HOME}/containers"
    chmod 600 "${GRAPEVINE_HOME}/config.yml"
    chmod 700 "${GRAPEVINE_HOME}/containers"/*/
    chmod 600 "${GRAPEVINE_HOME}/containers"/*/container.yml
    chmod 600 "${GRAPEVINE_HOME}/containers"/*/state.json
}
```

### 10.3 输入校验

```bash
# 所有用户输入必须经过校验

# 容器名校验
function validate_container_name() {
    local name="$1"
    if [[ ! "${name}" =~ ^[a-z][a-z0-9_-]{1,62}$ ]]; then
        error_wrap "E3003" "容器名" "名称 '${name}' 不符合规范: 必须以小写字母开头，仅包含小写字母、数字、下划线和连字符，长度 2-63"
        return 1
    fi
    return 0
}

# 路径校验（防止路径遍历）
function validate_path() {
    local path="$1"
    local base_dir="$2"

    # 解析为绝对路径
    local resolved
    resolved=$(readlink -f "${path}")

    # 检查是否在允许的基目录下
    if [[ "${resolved}" != "${base_dir}"* ]]; then
        error_wrap "E3003" "路径" "路径 '${path}' 超出允许范围"
        return 1
    fi

    # 检查路径遍历
    if [[ "${path}" == *".. "* || "${path}" == *"../"* ]]; then
        error_wrap "E3003" "路径" "路径包含非法遍历: '${path}'"
        return 1
    fi

    return 0
}

# 命令注入防护
function sanitize_command() {
    local cmd="$1"
    # 禁止 shell 元字符（容器内执行命令时）
    local dangerous_chars=';|`$(&)<>'
    if [[ "${cmd}" == *[\"${dangerous_chars}]* ]]; then
        error_wrap "E3003" "命令" "命令包含非法字符"
        return 1
    fi
    return 0
}

# YAML 值校验（防止 YAML 注入）
function validate_yaml_value() {
    local key="$1"
    local value="$2"

    # 值不能包含 YAML 特殊序列
    if [[ "${value}" == *"---"* || "${value}" == *"... "* ]]; then
        error_wrap "E3001" "YAML" "值包含非法 YAML 序列"
        return 1
    fi

    return 0
}
```

### 10.4 组件签名验证

```bash
# 组件签名验证流程
function component_verify_signature() {
    local archive_path="$1"
    local component="$2"
    local version="$3"

    # 1. 下载签名文件
    local sig_url="${MIRROR}/${component}/${version}/SHA256SUMS.sig"
    local sig_path="${archive_path}.sig"
    curl --fail --silent -o "${sig_path}" "${sig_url}" || return 1

    # 2. 下载公钥（首次）
    if [[ ! -f "${GRAPEVINE_HOME}/.gpg/grapevine.pub" ]]; then
        mkdir -p "${GRAPEVINE_HOME}/.gpg"
        curl --fail --silent -o "${GRAPEVINE_HOME}/.gpg/grapevine.pub" \
            "${MIRROR}/keys/grapevine.pub" || return 1
        gpg --dearmor -o "${GRAPEVINE_HOME}/.gpg/grapevine.gpg" \
            "${GRAPEVINE_HOME}/.gpg/grapevine.pub" 2>/dev/null
    fi

    # 3. 验证签名
    local checksum_file="${archive_path%/*}/SHA256SUMS"
    gpg --no-default-keyring \
        --keyring "${GRAPEVINE_HOME}/.gpg/grapevine.gpg" \
        --verify "${sig_path}" "${checksum_file}" 2>/dev/null

    return $?
}
```

---

## 11. 性能设计

### 11.1 启动时间优化

**目标**：冷启动 < 8s，热启动 < 5s

```
启动时间分解（目标）:
  ┌──────────────────────────────────────────────────┐
  │ 配置加载与校验           │  200ms │  ████         │
  │ 组件完整性检查           │  300ms │  ██████       │
  │ PRoot 环境构建           │  500ms │  ██████████   │
  │ GPU 驱动检测与配置       │  800ms │  ████████████ │
  │ 图形翻译层注入           │  400ms │  ████████     │
  │ 音频服务启动             │  600ms │  ████████████ │
  │ X11 显示服务启动         │  700ms │  ██████████   │
  │ Wine 进程启动            │ 1500ms │  █████████████│
  │ ─────────────────────────┼────────┼──────────────│
  │ 总计（冷启动）           │ 5000ms │               │
  └──────────────────────────────────────────────────┘
```

**优化策略**：

| 策略 | 描述 | 预期收益 |
|------|------|----------|
| 配置缓存 | 解析后的配置序列化到 `.cache` 文件，下次直接加载 | -150ms |
| GPU 信息缓存 | GPU 检测结果缓存到 `gpu-info.cache`，避免重复检测 | -500ms |
| 组件预检缓存 | 组件完整性校验结果缓存，仅在版本变更时重新校验 | -200ms |
| 并行初始化 | 音频、显示、图形服务并行启动 | -800ms |
| Wine Prefix 预热 | 首次创建时执行 `wineboot --init`，后续启动跳过 | -1000ms |
| 热启动检测 | 检测 Wine Prefix 已初始化，跳过初始化步骤 | -1500ms |

**并行初始化实现**：

```bash
# container.sh::parallel_setup()
function parallel_setup() {
    local name="$1"
    local config="$2"

    local -a pids=()

    # 并行启动独立服务
    (audio_setup "${config}") &
    pids+=($!)

    (display_setup "${config}") &
    pids+=($!)

    (driver_detect_and_configure "${config}") &
    pids+=($!)

    # 等待所有并行任务完成
    for pid in "${pids[@]}"; do
        wait "${pid}" || return $?
    done

    # 串行执行依赖前序步骤的操作
    graphics_inject "${config}"
}
```

### 11.2 内存使用

**目标**：Grapevine 管理进程内存 < 50MB

| 组件 | 预期内存 | 优化措施 |
|------|----------|----------|
| Grapevine 主进程 | ~10MB | Shell 脚本，无常驻内存 |
| PRoot | ~5MB | 限制 ptrace 缓冲区 |
| Box64 | ~20-100MB | 取决于转译代码量 |
| Box86 | ~15-80MB | 取决于转译代码量 |
| Wine/Proton | ~100-500MB | 取决于应用 |
| PulseAudio | ~10MB | 使用轻量配置 |
| Termux-X11 | ~20MB | 限制帧缓冲区 |

### 11.3 I/O 优化

| 优化措施 | 描述 |
|----------|------|
| 组件符号链接 | 多容器共享同一组件安装，使用符号链接而非复制 |
| 增量快照 | 仅存储变更文件，减少 I/O 与存储占用 |
| zstd 压缩 | 快照与组件使用 zstd，解压速度比 gzip 快 5-10 倍 |
| 异步日志 | 日志写入使用缓冲区，批量刷盘 |
| 临时文件 tmpfs | 容器运行时临时文件存储在 tmpfs（如可用） |

### 11.4 缓存策略

```
缓存层级:
  ┌─────────────────────────────────────────┐
  │ L1: Shell 变量缓存 (进程内)              │  命中延迟: ~0ms
  │   - 当前容器配置                         │
  │   - GPU 检测结果                         │
  │   - 组件路径映射                         │
  ├─────────────────────────────────────────┤
  │ L2: 文件缓存 (磁盘)                      │  命中延迟: ~5ms
  │   - ~/.grapevine/cache/gpu-info.cache   │
  │   - ~/.grapevine/cache/config.cache     │
  │   - ~/.grapevine/cache/component.cache  │
  ├─────────────────────────────────────────┤
  │ L3: 组件下载缓存 (磁盘)                  │  命中延迟: ~50ms
  │   - ~/.grapevine/components/cache/      │
  │   - 已下载的组件包                       │
  └─────────────────────────────────────────┘

缓存失效策略:
  - L1: 容器操作完成后自动失效
  - L2: 配置文件修改时失效 (mtime 检测)
  - L3: 组件安装成功后删除缓存包, 或 7 天后自动清理
```

---

## 12. 部署设计

### 12.1 安装流程

```
install 脚本流程:
  │
  ├─ 1. 环境检查
  │     ├─ 检查是否在 Termux 中运行
  │     ├─ 检查架构 (aarch64)
  │     ├─ 检查 Android 版本 (≥ 10)
  │     ├─ 检查磁盘空间 (≥ 2GB 可用)
  │     └─ 检查必要命令 (curl, tar, zstd)
  │
  ├─ 2. 安装依赖
  │     pkg install proot termux-x11-nightly \
  │                 pulseaudio zstd curl
  │
  ├─ 3. 部署 Grapevine
  │     ├─ 创建目录结构
  │     │   mkdir -p ~/.grapevine/{containers,components,templates,cache,logs,tmp}
  │     │
  │     ├─ 复制文件
  │     │   cp -r bin/ lib/ templates/ tui/ ~/.grapevine/
  │     │
  │     ├─ 设置权限
  │     │   chmod +x ~/.grapevine/bin/grapevine
  │     │
  │     └─ 创建符号链接
  │         ln -sf ~/.grapevine/bin/grapevine $PREFIX/bin/grapevine
  │
  ├─ 4. 安装基础组件
  │     grapevine component install glibc-runtime
  │     grapevine component install box64
  │
  ├─ 5. 生成默认配置
  │     write ~/.grapevine/config.yml
  │
  └─ 6. 验证安装
        grapevine diagnose check
```

### 12.2 升级流程

```
upgrade 流程:
  │
  ├─ 1. 版本检查
  │     current = grapevine --version
  │     latest = repository_get_latest_version()
  │
  ├─ 2. 下载新版本
  │     download grapevine-${latest}.tar.zst
  │     verify checksum
  │
  ├─ 3. 备份当前版本
  │     cp -a ~/.grapevine ~/.grapevine.bak.${current}
  │
  ├─ 4. 执行升级
  │     ├─ 停止所有运行中容器
  │     ├─ 替换 bin/, lib/, tui/ 文件
  │     ├─ 运行迁移脚本 (如有)
  │     │   migrate_${current}_to_${latest}.sh
  │     └─ 更新版本号
  │
  ├─ 5. 验证升级
  │     grapevine diagnose check
  │
  └─ 6. 清理
        rm -rf ~/.grapevine.bak.${current}  # 用户确认后
```

### 12.3 回滚流程

```
rollback 流程:
  │
  ├─ 1. 检查备份
  │     ls ~/.grapevine.bak.*
  │
  ├─ 2. 停止所有容器
  │     grapevine container list --filter running
  │     # 逐个停止
  │
  ├─ 3. 交换目录
  │     mv ~/.grapevine ~/.grapevine.failed
  │     mv ~/.grapevine.bak.${target_version} ~/.grapevine
  │
  ├─ 4. 验证
  │     grapevine diagnose check
  │
  └─ 5. 清理
        rm -rf ~/.grapevine.failed
```

### 12.4 多版本共存

```
多版本目录布局:
  ~/.grapevine/                    # 当前版本 (符号链接)
  ~/.grapevine-1.0.0/              # v1.0.0 安装
  ~/.grapevine-1.1.0/              # v1.1.0 安装

切换版本:
  ln -sfn ~/.grapevine-1.0.0 ~/.grapevine

注意:
  - 容器数据与组件存储在 ~/.grapevine/ 下
  - 切换版本时容器数据自动跟随符号链接
  - 配置文件格式向前兼容，旧版本可读新配置（忽略未知字段）
```

---

## 13. 测试策略

### 13.1 单元测试

**框架**：ShellCheck（静态分析）+ shunit2（Shell 单元测试）

**测试范围**：

| 模块 | 测试文件 | 测试重点 |
|------|----------|----------|
| yaml.sh | test_yaml.sh | YAML 解析正确性、边界条件、错误处理 |
| config.sh | test_config.sh | 配置合并、校验、覆盖优先级 |
| state.sh | test_state.sh | 状态转换合法性、并发安全 |
| template.sh | test_template.sh | 模板解析、变量替换、继承 |
| snapshot.sh | test_snapshot.sh | 增量检测、恢复链构建 |
| driver.sh | test_driver.sh | GPU 检测、驱动选择算法 |
| io.sh | test_io.sh | 原子写入、路径校验 |

**测试示例**：

```bash
#!/bin/bash
# test/test_state.sh

source lib/core/state.sh

test_state_transition_valid() {
    state_init "test-container" "created"
    assertEquals 0 $(state_transition "test-container" "running")
    assertEquals "running" $(state_get "test-container")
}

test_state_transition_invalid() {
    state_init "test-container" "created"
    assertNotEquals 0 $(state_transition "test-container" "paused")
    assertEquals "created" $(state_get "test-container")
}

test_state_transition_stopped_to_deleted() {
    state_init "test-container" "stopped"
    assertEquals 0 $(state_transition "test-container" "deleted")
}

. shunit2
```

### 13.2 集成测试

**测试场景**：

| 场景 | 描述 | 验证点 |
|------|------|--------|
| 完整创建流程 | 从模板创建容器 | 目录结构、配置文件、状态正确 |
| 完整启动流程 | 启动容器并验证 Wine 进程 | 进程存在、环境变量正确 |
| 快照创建与恢复 | 创建快照 → 修改 → 恢复 | 文件内容一致 |
| 组件安装与卸载 | 安装组件 → 验证 → 卸载 | 文件存在/消失、注册表更新 |
| 模板继承 | 使用继承模板创建容器 | 配置合并正确 |
| 配置覆盖 | 命令行覆盖模板配置 | 覆盖优先级正确 |
| 错误恢复 | 模拟各种错误场景 | 错误码正确、状态一致 |
| 并发操作 | 同时启动/停止多个容器 | 无死锁、状态一致 |

### 13.3 端到端测试

**测试矩阵**：

| 设备 | GPU | Android | 测试内容 |
|------|-----|---------|----------|
| Pixel 7 | Mali-G710 | 13 | DX11 游戏、办公应用 |
| Samsung S23 | Adreno 740 | 13 | DX12 游戏、DX9 游戏 |
| OnePlus 11 | Adreno 740 | 13 | 开发工具、音频应用 |
| Xiaomi 13 | Adreno 740 | 13 | 全模板测试 |
| 模拟器 (qemu-aarch64) | VirGL | 14 | 无 GPU 降级测试 |

**端到端测试脚本**：

```bash
#!/bin/bash
# test/e2e/test_basic_workflow.sh

set -euo pipefail

echo "=== E2E: 基本工作流测试 ==="

# 1. 创建容器
echo "1. 创建容器..."
grapevine container create -t minimal -n e2e-test
grapevine container list | grep -q "e2e-test"

# 2. 启动容器
echo "2. 启动容器..."
grapevine container start e2e-test
sleep 5
grapevine container list --filter running | grep -q "e2e-test"

# 3. 执行命令
echo "3. 执行命令..."
grapevine container exec e2e-test "wine --version"

# 4. 创建快照
echo "4. 创建快照..."
snap_id=$(grapevine snapshot create e2e-test -m "e2e-test-snapshot")
grapevine snapshot list e2e-test | grep -q "${snap_id}"

# 5. 停止容器
echo "5. 停止容器..."
grapevine container stop e2e-test
grapevine container list --filter stopped | grep -q "e2e-test"

# 6. 删除容器
echo "6. 删除容器..."
grapevine container delete e2e-test --purge

echo "=== E2E: 测试通过 ==="
```

### 13.4 性能基准

| 基准项 | 目标 | 测量方法 |
|--------|------|----------|
| 冷启动时间 | < 8s | `time grapevine container start <name>` |
| 热启动时间 | < 5s | 第二次启动 |
| 容器创建时间 | < 10s | `time grapevine container create -t minimal -n bench` |
| 快照创建时间 | < 30s (1GB Prefix) | `time grapevine snapshot create <name>` |
| 快照恢复时间 | < 45s (1GB Prefix) | `time grapevine snapshot restore <name> <snap>` |
| 组件安装时间 | < 60s (100MB, 本地) | `time grapevine component install wine@9.0` |
| 内存占用 | < 50MB (管理进程) | `ps -o rss -p $(pgrep -f grapevine)` |
| DX11 帧率 | > 30fps (简单场景) | 运行 unigine-heaven benchmark |
| 音频延迟 | < 50ms | PulseAudio 延迟测量 |

---

## 14. 技术风险与缓解措施

| 编号 | 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|------|----------|
| R01 | Box64/Box86 指令转译不完整，部分 x86/x64 指令无法转译 | 应用崩溃或功能异常 | 中 | 1. 跟踪 Box64/Box86 上游更新，及时集成修复<br>2. 维护已知不兼容应用列表<br>3. 提供 Box64/Box86 调试模式辅助问题定位 |
| R02 | Wine/Proton 与特定 Windows 应用兼容性不足 | 目标应用无法运行 | 高 | 1. 提供多种 Wine 版本切换<br>2. 集成 Proton 的游戏兼容补丁<br>3. 支持 winetricks 安装常用运行时<br>4. 社区驱动的兼容性数据库 |
| R03 | Android GPU 驱动碎片化，Turnip/VirGL 兼容性不一致 | 图形渲染异常或性能低下 | 高 | 1. 实现自动 GPU 检测与驱动选择<br>2. 提供多级降级策略（Turnip→Zink→VirGL→WineD3D）<br>3. GPU 兼容性数据库与用户报告机制 |
| R04 | PRoot 性能开销（系统调用拦截导致 I/O 性能下降） | 容器内文件操作缓慢 | 中 | 1. 优化 bind mount 策略，减少不必要的拦截<br>2. 对热路径使用直接路径映射<br>3. 评估 seccomp-bpf 替代方案 |
| R05 | Termux 环境限制（bionic libc、无 systemd、文件系统限制） | 部分功能无法实现 | 中 | 1. 使用 glibc-runtime 提供完整 Linux 运行时<br>2. 自管理服务进程（不依赖 systemd）<br>3. PRoot 绕过文件系统限制 |
| R06 | 快照链过长导致恢复时间线性增长 | 用户体验下降 | 低 | 1. 每 N 个增量快照自动合并为全量快照<br>2. 提供手动合并命令<br>3. 快照数量警告与自动清理策略 |
| R07 | 组件下载依赖外部网络，中国地区 GitHub 访问不稳定 | 安装/升级失败 | 高 | 1. 多镜像源支持（含国内镜像）<br>2. 离线安装模式（从本地文件安装）<br>3. 下载断点续传与重试机制 |
| R08 | Shell 脚本实现的 YAML 解析器性能与兼容性限制 | 复杂配置解析失败 | 低 | 1. 限制 YAML 特性使用范围<br>2. 提供配置校验工具<br>3. 长期考虑使用 C/Rust 实现关键路径 |
| R09 | 并发容器操作导致状态不一致 | 数据损坏 | 低 | 1. 文件锁（flock）保证原子性<br>2. 状态机强制合法转换<br>3. 操作前自动快照保护 |
| R10 | Android 系统更新导致 PRoot/驱动行为变化 | 原有容器无法正常启动 | 中 | 1. 版本化组件管理，支持回退<br>2. 诊断工具自动检测环境变化<br>3. 社区快速响应机制 |

---

*文档结束*
