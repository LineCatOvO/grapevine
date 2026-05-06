# Grapevine

Manage and run Wine/Proton containers on Android with ease.

Grapevine 是一个基于 [Termux](https://github.com/termux/termux-app) 的 Windows 兼容容器管理平台，通过 [Box86](https://github.com/ptitSeb/box86)/[Box64](https://github.com/ptitSeb/box64) 指令转译与 [Wine](https://www.winehq.org/)/[Proton](https://github.com/ValveSoftware/Proton) 兼容层，在 ARM64 Android 设备上运行 Windows x86/x64 应用程序。

与现有方案（Winlator、Mobox）不同，Grapevine 以**容器化**为核心设计理念，提供完整的容器生命周期管理、声明式配置、快照回滚和模板系统，使每个 Windows 运行环境可复现、可迁移、可隔离。

---

## 目录

- [项目定位与对比](#项目定位与对比)
- [系统架构](#系统架构)
- [容器管理系统](#容器管理系统)
- [安装](#安装)
- [快速开始](#快速开始)
- [使用指南](#使用指南)
- [配置参考](#配置参考)
- [图形子系统](#图形子系统)
- [音频子系统](#音频子系统)
- [输入控制](#输入控制)
- [存储与文件映射](#存储与文件映射)
- [网络支持](#网络支持)
- [组件管理](#组件管理)
- [技术实现细节](#技术实现细节)
- [项目结构](#项目结构)
- [常见问题](#常见问题)
- [致谢与第三方组件](#致谢与第三方组件)
- [许可证](#许可证)

---

## 项目定位与对比

### 为什么选择 Grapevine

| 特性 | Winlator | Mobox | Grapevine |
|---|---|---|---|
| 运行平台 | Android 原生 APK | Termux | Termux |
| 需要 Root | 否 | 否 | 否（可选，用于 OOM 调优） |
| 容器管理 | 基础（创建/删除/配置） | 仅 Wine prefix 切换 | 完整生命周期管理 |
| 容器隔离 | PRoot 隔离 | 无隔离 | PRoot + namespace 隔离 |
| 快照/回滚 | 不支持 | 不支持 | 支持 |
| 容器模板 | 不支持 | 不支持 | 内置 + 自定义模板 |
| 容器导入/导出 | 不支持 | 不支持 | 支持（压缩归档） |
| 容器克隆 | 不支持 | 不支持 | 支持 |
| Wine 版本管理 | 固定版本 | 菜单切换 | 每容器独立版本 |
| Proton 支持 | 否 | 否 | 支持 |
| 声明式配置 | 否 | 否 | YAML 配置文件 |
| 图形驱动 | Turnip/VirGL/Zink | Turnip/VirGL/Zink | Turnip/VirGL/Zink + 自动选择 |
| DXVK/VKD3D | 支持 | 支持 | 支持 + 版本切换 |
| 音频 | PulseAudio/ALSA | ALSA | PulseAudio/ALSA |
| 输入控制 | 内置编辑器 | Input Bridge | Input Bridge + 自定义映射 |
| CLI/TUI | 无（仅 GUI） | 菜单式 TUI | CLI + 交互式 TUI |
| 脚本化/自动化 | 不支持 | 有限 | 完整 CLI 支持 |
| 组件热更新 | 需更新 APK | 脚本安装 | 组件版本管理器 |

### 核心差异

**Winlator** 是一个完整的 Android 原生应用，通过 Java/Kotlin UI 和 C/C++ JNI 层实现，所有组件打包在 APK 中。优点是用户体验流畅，缺点是更新组件需要发布新版 APK，容器管理能力有限。

**Mobox** 基于 Termux 运行，使用 Shell 脚本构建菜单式界面。优点是轻量灵活，缺点是缺乏真正的容器隔离和管理——本质上只是 Wine prefix 的切换器。

**Grapevine** 吸收两者优点，基于 Termux 的开放生态，同时引入专业级容器管理：

- **容器即环境**：每个容器拥有独立的文件系统视图、Wine prefix、注册表、驱动配置和资源限制
- **声明式配置**：通过 YAML 文件描述容器期望状态，支持版本控制和共享
- **可复现性**：模板 + 配置 = 任何人可在任意设备上复现相同的运行环境
- **安全性**：容器间文件系统隔离，防止应用间互相干扰

---

## 系统架构

Grapevine 采用分层架构设计，将 Windows API 调用逐步转换为 ARM64 Android 设备可执行的指令：

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层                             │
│   CLI (grapevine)  │  TUI (grapevine tui)  │  脚本 API  │
├─────────────────────────────────────────────────────────┤
│                   容器管理层                              │
│  生命周期 │ 快照 │ 模板 │ 导入导出 │ 资源控制 │ 网络管理  │
├─────────────────────────────────────────────────────────┤
│                   兼容运行时层                            │
│     Wine / Proton    │    Box86 / Box64                 │
├─────────────────────────────────────────────────────────┤
│                   图形翻译层                              │
│   DXVK  │  VKD3D  │  WineD3D  │  D8VK                  │
├─────────────────────────────────────────────────────────┤
│                   驱动适配层                              │
│   Turnip (Adreno)  │  VirGL  │  Zink (Mesa)            │
├─────────────────────────────────────────────────────────┤
│                   系统集成层                              │
│  PRoot │ glibc-runtime │ ALSA/PulseAudio │ Termux-X11   │
├─────────────────────────────────────────────────────────┤
│                   Android 平台层                         │
│        Linux Kernel  │  GPU (Adreno/Mali/PowerVR)       │
└─────────────────────────────────────────────────────────┘
```

### 各层职责

**用户交互层**：提供 CLI 命令行、交互式 TUI 界面和脚本 API 三种交互方式。CLI 适合自动化和高级用户，TUI 适合日常使用，脚本 API 供第三方工具集成。

**容器管理层**：Grapevine 的核心层。管理容器的完整生命周期，包括创建、启动、停止、删除、克隆、快照、回滚、导入导出。每个容器通过 PRoot 实现文件系统隔离，通过配置文件定义运行时参数。

**兼容运行时层**：Wine/Proton 提供 Windows API 兼容，Box86/Box64 提供 x86/x64 到 ARM64 的指令转译。Wine 负责将 Windows 系统调用翻译为 POSIX 调用，Box86/Box64 负责将 x86 指令动态翻译为 ARM 指令。

**图形翻译层**：将 DirectX 图形调用转换为 Vulkan 调用。DXVK 处理 DirectX 9/10/11，VKD3D 处理 DirectX 12，WineD3D 提供 OpenGL 回退路径，D8VK 处理 DirectX 8。

**驱动适配层**：将 Vulkan 调用映射到 Android GPU 硬件。Turnip 为 Adreno GPU 提供原生 Vulkan 驱动，VirGL 通过 OpenGL ES 模拟 Vulkan，Zink 通过 Mesa 将 Vulkan 调用转译为 OpenGL。

**系统集成层**：PRoot 提供用户空间虚拟化（无需 root），glibc-runtime 提供标准 Linux 系统库，ALSA/PulseAudio 处理音频输出，Termux-X11 提供图形显示服务。

**Android 平台层**：底层 Linux 内核和 GPU 硬件。

---

## 容器管理系统

容器管理是 Grapevine 的核心功能，也是与 Winlator/Mobox 的根本区别。

### 容器模型

每个 Grapevine 容器包含以下要素：

```
~/.grapevine/containers/<container-name>/
├── container.yml          # 容器声明式配置
├── rootfs/                # PRoot 隔离的文件系统视图
│   ├── usr/               # 系统库和程序
│   ├── etc/               # 配置文件
│   ├── tmp/               # 临时文件
│   └── ...
├── prefix/                # Wine prefix (虚拟 Windows 环境)
│   ├── drive_c/           # C: 盘
│   ├── dosdevices/        # 设备映射
│   ├── system.reg         # 系统注册表
│   ├── user.reg           # 用户注册表
│   └── ...
├── snapshots/             # 快照存储
│   ├── snap_20260506_001/
│   └── ...
├── logs/                  # 容器日志
│   ├── wine.log
│   ├── box64.log
│   └── grapevine.log
└── state.json             # 运行时状态
```

### 容器生命周期

```
  create ──→ stopped ──→ running ──→ stopped ──→ deleted
               │            │           │
               ├─ clone     ├─ pause    ├─ snapshot
               ├─ export    └─ resume   ├─ rollback
               ├─ template              └─ export
               └─ delete
```

**状态说明**：

- `creating`：正在初始化容器文件系统和 Wine prefix
- `stopped`：容器已创建但未运行，可修改配置
- `running`：容器正在执行，Wine 进程活跃
- `paused`：容器进程被挂起（信号暂停）
- `error`：容器因错误退出，需要检查日志

### 容器配置文件

每个容器通过 `container.yml` 进行声明式配置：

```yaml
name: my-game-container
description: "Container for running Skyrim SE"

runtime:
  wine:
    version: "wine-9.22-staging-tkg"
    arch: "win64"
  box64:
    version: "0.3.4"
    dynarec:
      strong_mem: 2
      maxsel: 40
      hotpage: 16
  box86:
    version: "0.3.8"
    dynarec:
      strong_mem: 2
      maxsel: 40

graphics:
  driver: "turnip"          # turnip | virgl | zink | auto
  dxvk:
    version: "2.4"
    enabled: true
  vkd3d:
    version: "2.13"
    enabled: true
  resolution: "1280x720"
  fullscreen: true

audio:
  backend: "pulseaudio"     # pulseaudio | alsa
  sample_rate: 48000

storage:
  shared_dirs:
    - host: "/sdcard/Games"
      guest: "D:"
      readonly: false
    - host: "/sdcard/Documents"
      guest: "E:"
      readonly: true

resources:
  cpu_affinity: "0-3"       # 绑定到前 4 个核心
  memory_limit: "4G"        # 内存限制（需 root）
  priority: "high"          # low | normal | high

network:
  mode: "host"              # host | isolated

input:
  profile: "gamepad"        # gamepad | keyboard | touch | custom
  custom_mapping: null

compatibility:
  env_vars:
    WINEDLLOVERRIDES: "d3d11,dxgi=n"
    MESA_GL_VERSION_OVERRIDE: "4.5"
  dll_overrides:
    d3d11: "native"
    dxgi: "native"
  registry_patches: []
```

### 快照系统

快照保存容器在某一时刻的完整状态，包括 Wine prefix、注册表和配置文件：

```bash
# 创建快照
grapevine snapshot create my-container --tag "before-mod-install"

# 列出快照
grapevine snapshot list my-container

# 回滚到快照
grapevine snapshot rollback my-container --tag "before-mod-install"

# 删除快照
grapevine snapshot delete my-container --tag "before-mod-install"
```

快照采用增量存储策略，仅保存与前一快照的差异，节省存储空间。

### 模板系统

模板是预配置的容器蓝图，用于快速创建特定用途的容器：

**内置模板**：

| 模板名 | 用途 | Wine 版本 | 图形驱动 | 预装组件 |
|---|---|---|---|---|
| `gaming-dx11` | DirectX 11 游戏 | wine-staging-tkg | Turnip + DXVK | DXVK, vcredist, d3dcompiler |
| `gaming-dx9` | DirectX 9 老游戏 | wine-staging-tkg | Turnip + DXVK | D8VK, d3dx9, d3dcompiler |
| `gaming-dx12` | DirectX 12 游戏 | proton-experimental | Turnip + VKD3D | VKD3D, vcredist |
| `office` | 办公软件 | wine-stable | VirGL | corefonts, msxml |
| `development` | 开发工具 | wine-devel | VirGL | .NET Framework, cmd |
| `minimal` | 最小环境 | wine-stable | VirGL | 无 |

```bash
# 从模板创建容器
grapevine create my-game --template gaming-dx11

# 导出当前容器为模板
grapevine template export my-container --name my-custom-template

# 从自定义模板创建
grapevine create another-game --template my-custom-template

# 列出可用模板
grapevine template list
```

### 容器导入/导出

```bash
# 导出容器为压缩归档
grapevine export my-container -o my-container.tar.zst

# 导入容器
grapevine import my-container.tar.zst --name restored-container

# 仅导出 Wine prefix（不含系统文件）
grapevine export my-container --prefix-only -o prefix.tar.zst
```

---

## 安装

### 前置要求

- Android 10 或更高版本
- ARM64 设备（aarch64）
- 至少 4GB 可用存储空间
- 稳定的网络连接（用于下载组件）

### 安装步骤

1. **安装 Termux**

   从 [F-Droid](https://f-droid.org/en/packages/com.termux/) 安装 Termux（不要使用 Google Play 版本，已过时）。

2. **安装 Termux-X11**

   ```bash
   # 在 Termux 中执行
   pkg install x11-repo
   pkg install termux-x11-nightly
   ```

3. **安装 Grapevine**

   ```bash
   curl -s -o ~/grapevine-install https://raw.githubusercontent.com/grapevine-project/grapevine/main/install
   chmod +x ~/grapevine-install
   ~/grapevine-install
   ```

4. **初始化**

   ```bash
   grapevine init
   ```

   初始化过程将：
   - 创建 `~/.grapevine/` 目录结构
   - 下载 glibc-runtime
   - 安装 Box86/Box64
   - 下载默认 Wine 版本
   - 配置 Termux-X11 连接

5. **安装 Input Bridge（可选，用于触摸控制）**

   从项目 Release 页面下载 Input Bridge APK 并安装。

### 卸载

```bash
grapevine uninstall
```

或通过菜单：`grapevine` → `Settings` → `Uninstall Grapevine`

---

## 快速开始

```bash
# 初始化（首次使用）
grapevine init

# 从模板创建一个游戏容器
grapevine create my-first-game --template gaming-dx11

# 启动 Termux-X11 显示服务
grapevine display start

# 启动容器
grapevine start my-first-game

# 在容器中运行 Windows 程序
grapevine run my-first-game /path/to/game.exe

# 停止容器
grapevine stop my-first-game

# 查看所有容器状态
grapevine list
```

---

## 使用指南

### CLI 命令参考

#### 容器管理

```bash
grapevine create <name> [--template <template>] [--wine <version>] [--driver <driver>]
grapevine list [--all] [--format table|json]
grapevine start <name>
grapevine stop <name> [--force]
grapevine restart <name>
grapevine delete <name> [--force]
grapevine clone <src> <dst>
grapevine info <name>
```

#### 容器内执行

```bash
grapevine run <name> <executable> [args...]
grapevine shell <name>                  # 进入容器 Wine 命令行
grapevine wineboot <name> [--restart|--shutdown]
grapevine winecfg <name>                # 打开 Wine 配置
grapevine regedit <name>                # 打开注册表编辑器
grapevine taskmgr <name>                # 打开任务管理器
grapevine explorer <name>               # 打开文件资源管理器
grapevine control <name>                # 打开控制面板
```

#### 快照管理

```bash
grapevine snapshot create <name> [--tag <tag>]
grapevine snapshot list <name>
grapevine snapshot rollback <name> --tag <tag>
grapevine snapshot delete <name> --tag <tag>
```

#### 模板管理

```bash
grapevine template list
grapevine template export <container> --name <template-name>
grapevine template delete <template-name>
grapevine template inspect <template-name>
```

#### 导入导出

```bash
grapevine export <name> [-o <path>] [--prefix-only]
grapevine import <path> --name <name>
```

#### 组件管理

```bash
grapevine component list
grapevine component install <component> [--version <ver>]
grapevine component uninstall <component>
grapevine component update [<component>]
```

#### 显示服务

```bash
grapevine display start [--resolution <WxH>]
grapevine display stop
grapevine display status
```

#### 配置管理

```bash
grapevine config get <name> <key>
grapevine config set <name> <key> <value>
grapevine config edit <name>             # 使用编辑器打开 container.yml
grapevine config validate <name>
grapevine config show <name>             # 显示合并后的完整配置
```

#### 系统设置

```bash
grapevine init                           # 初始化 Grapevine
grapevine doctor                         # 诊断环境问题
grapevine settings                       # 打开设置 TUI
grapevine uninstall                      # 卸载 Grapevine
```

### 交互式 TUI

直接运行 `grapevine` 或 `grapevine tui` 进入交互式终端界面：

```
┌─ Grapevine ──────────────────────────────────────────────┐
│                                                          │
│  Containers                    Components                │
│  ┌─────────────────────────┐   ┌──────────────────────┐ │
│  │ ● my-game      running  │   │ Wine 9.22-staging    │ │
│  │ ○ office-env   stopped  │   │ Box64 0.3.4          │ │
│  │ ○ dev-tools    stopped  │   │ Box86 0.3.8          │ │
│  │ ○ retro-games  stopped  │   │ DXVK 2.4             │ │
│  └─────────────────────────┘   │ VKD3D 2.13           │ │
│                                 │ Turnip latest         │ │
│  [Start] [Stop] [Delete]       │ VirGL latest          │ │
│  [Clone] [Export] [Snapshot]   │ Zink latest           │ │
│                                 └──────────────────────┘ │
│                                                          │
│  [Settings]  [Templates]  [Display]  [Help]             │
└──────────────────────────────────────────────────────────┘
```

---

## 配置参考

### 全局配置

全局配置文件位于 `~/.grapevine/config.yml`：

```yaml
default_wine: "wine-9.22-staging-tkg"
default_driver: "auto"           # auto | turnip | virgl | zink
default_resolution: "1280x720"
auto_detect_gpu: true
language: "zh_CN"
log_level: "info"                # debug | info | warn | error
component_mirror: "github"       # github | ghproxy | custom
custom_mirror_url: null
```

### 容器配置（container.yml）

#### runtime 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `wine.version` | string | 全局 default_wine | Wine/Proton 版本 |
| `wine.arch` | string | `win64` | Wine 架构：`win32`/`win64` |
| `box64.version` | string | 最新版 | Box64 版本 |
| `box64.dynarec.*` | map | - | Box64 动态编译变量 |
| `box86.version` | string | 最新版 | Box86 版本 |
| `box86.dynarec.*` | map | - | Box86 动态编译变量 |

#### graphics 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `driver` | string | `auto` | 图形驱动选择 |
| `dxvk.version` | string | 最新版 | DXVK 版本 |
| `dxvk.enabled` | bool | `true` | 是否启用 DXVK |
| `vkd3d.version` | string | 最新版 | VKD3D 版本 |
| `vkd3d.enabled` | bool | `false` | 是否启用 VKD3D |
| `resolution` | string | `1280x720` | 虚拟屏幕分辨率 |
| `fullscreen` | bool | `true` | 全屏模式 |

**driver 选项说明**：

- `auto`：自动检测 GPU 类型并选择最优驱动
  - Adreno 6xx / 725-740 → Turnip
  - 其他 GPU → VirGL
- `turnip`：Adreno GPU 原生 Vulkan 驱动，性能最佳
- `virgl`：通过 OpenGL ES 模拟 Vulkan，兼容性最广
- `zink`：Mesa Vulkan → OpenGL 转译层，适合部分 Mali GPU

#### audio 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `backend` | string | `pulseaudio` | 音频后端 |
| `sample_rate` | int | `48000` | 采样率 |

#### storage 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `shared_dirs` | list | - | 主机-容器共享目录列表 |
| `shared_dirs[].host` | string | - | 主机路径 |
| `shared_dirs[].guest` | string | - | 容器内盘符（如 `D:`） |
| `shared_dirs[].readonly` | bool | `false` | 是否只读挂载 |

#### resources 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `cpu_affinity` | string | - | CPU 核心绑定（需 root） |
| `memory_limit` | string | - | 内存限制（需 root） |
| `priority` | string | `normal` | 进程优先级 |

#### network 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `mode` | string | `host` | 网络模式：`host`/`isolated` |

#### compatibility 节

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `env_vars` | map | - | 环境变量覆盖 |
| `dll_overrides` | map | - | DLL 加载顺序覆盖 |
| `registry_patches` | list | - | 注册表补丁列表 |

---

## 图形子系统

### 渲染管线

```
Windows 应用
    │
    ▼ (DirectX 调用)
DXVK / VKD3D / D8VK / WineD3D
    │
    ▼ (Vulkan 调用)
Turnip / VirGL / Zink
    │
    ▼ (GPU 指令)
Android GPU (Adreno / Mali / PowerVR)
```

### 驱动选择指南

| GPU | 推荐驱动 | DirectX 支持 | 备注 |
|---|---|---|---|
| Adreno 630-680 | Turnip | DX9-DX12 | 性能最佳 |
| Adreno 725-740 | Turnip | DX9-DX12 | 需启用 a7xx 闪烁修复 |
| Adreno 610-620 | VirGL | DX9 | 性能有限 |
| Mali-G7x | Zink | DX9-DX11 | 部分设备可用 |
| Mali-G5x/G6x | VirGL | DX9 | 兼容模式 |
| PowerVR | VirGL | DX9 | 兼容模式 |

### DXVK 配置

DXVK 将 DirectX 9/10/11 调用转换为 Vulkan：

```bash
# 在容器配置中启用 DXVK
grapevine config set my-container graphics.dxvk.enabled true
grapevine config set my-container graphics.dxvk.version "2.4"

# DXVK HUD 显示
grapevine run my-container game.exe --env DXVK_HUD=fps,devinfo
```

### VKD3D 配置

VKD3D 将 DirectX 12 调用转换为 Vulkan（实验性）：

```bash
grapevine config set my-container graphics.vkd3d.enabled true
grapevine config set my-container graphics.dxvk.enabled false
```

---

## 音频子系统

### 架构

```
Windows 应用
    │
    ▼ (Windows 音频 API)
Wine 音频服务
    │
    ▼ (ALSA/PulseAudio 协议)
ALSA 插件 / PulseAudio
    │
    ▼ (Android AudioTrack)
Android 音频框架
```

### 配置

```bash
# 选择音频后端
grapevine config set my-container audio.backend pulseaudio

# 调整采样率
grapevine config set my-container audio.sample_rate 44100
```

PulseAudio 通常提供更低的延迟和更好的兼容性，推荐使用。如果遇到音频问题，可尝试切换到 ALSA。

---

## 输入控制

### 输入方案

| 方案 | 说明 | 适用场景 |
|---|---|---|
| Input Bridge | 第三方触摸映射应用 | 触屏游戏 |
| 外接键鼠 | USB/蓝牙键盘鼠标 | 策略游戏、办公 |
| 手柄 | USB/蓝牙游戏手柄 | 动作游戏 |
| 自定义映射 | 容器级按键映射文件 | 特殊需求 |

### 输入配置

```yaml
input:
  profile: "gamepad"
  custom_mapping:
    - key: "KEY_VOLUME_UP"
      action: "VK_F5"
    - key: "KEY_BACK"
      action: "VK_ESCAPE"
```

---

## 存储与文件映射

### 默认映射

每个容器自动创建以下映射：

| 容器内路径 | 主机路径 | 说明 |
|---|---|---|
| `C:` | `<container>/prefix/drive_c` | Wine prefix 系统盘 |
| `Z:` | `<container>/rootfs` | PRoot 文件系统根 |

### 自定义共享目录

```yaml
storage:
  shared_dirs:
    - host: "/sdcard/Games/Skyrim"
      guest: "D:"
      readonly: false
    - host: "/sdcard/Music"
      guest: "E:"
      readonly: true
```

### 容器间文件共享

通过创建共享存储卷实现容器间文件共享：

```bash
# 创建共享卷
grapevine volume create game-data

# 将卷挂载到容器
grapevine config set game-a storage.volumes.game-data.mount "D:"
grapevine config set game-b storage.volumes.game-data.mount "D:"
```

---

## 网络支持

### 网络模式

- **host**（默认）：容器共享主机网络栈，Windows 应用可直接访问网络
- **isolated**：容器网络隔离，无法访问外部网络（适合不需要网络的离线应用）

```bash
grapevine config set my-container network.mode isolated
```

---

## 组件管理

Grapevine 将所有运行时组件进行版本化管理，支持独立安装、更新和卸载：

### 可用组件

| 组件 | 说明 | 版本示例 |
|---|---|---|
| `wine-stable` | Wine 稳定版 | 9.0 |
| `wine-staging` | Wine 暂存版 | 9.22 |
| `wine-staging-tkg` | Wine Staging TKG 定制版 | 9.22 |
| `wine-devel` | Wine 开发版 | 10.0-rc1 |
| `proton-experimental` | Proton 实验版 | 9.0 |
| `box64` | x64→ARM64 指令转译 | 0.3.4 |
| `box86` | x86→ARM32 指令转译 | 0.3.8 |
| `dxvk` | DX9/10/11→Vulkan | 2.4 |
| `dxvk-async` | DXVK 异步着色器编译版 | 2.4 |
| `dxvk-gplasync` | DXVK GPL Async 版 | 2.4 |
| `vkd3d` | DX12→Vulkan | 2.13 |
| `d8vk` | DX8→Vulkan | 1.0 |
| `turnip` | Adreno Vulkan 驱动 | latest |
| `virgl` | VirGL OpenGL 模拟 | latest |
| `zink` | Mesa Vulkan→OpenGL | latest |
| `mesa` | Mesa 图形库 | 24.2 |
| `mono` | Wine .NET 替代 | 9.4.0 |
| `gecko` | Wine HTML 引擎 | 2.47.4 |

### 组件操作

```bash
# 列出已安装和可用组件
grapevine component list

# 安装组件
grapevine component install wine-staging-tkg --version 9.22

# 更新组件
grapevine component update dxvk

# 卸载组件（不影响正在使用的容器）
grapevine component uninstall wine-devel
```

---

## 技术实现细节

### PRoot 隔离

Grapevine 使用 PRoot 实现用户空间文件系统隔离，无需 root 权限：

- 每个容器拥有独立的 `rootfs` 目录树
- PRoot 通过 `ptrace` 系统调用拦截文件路径访问
- 将容器内路径重映射到主机上的容器目录
- 支持绑定挂载（bind mount）共享目录

### glibc 运行时

Termux 使用 Bionic libc（Android 原生 C 库），而 Wine 和许多 Linux 程序依赖 glibc。Grapevine 通过 [glibc-packages](https://github.com/termux-pacman/glibc-packages) 项目提供 glibc 运行时环境：

- 在 PRoot 隔离的 rootfs 中安装 glibc 和相关库
- 通过 `LD_LIBRARY_PATH` 和 `LD_PRELOAD` 确保正确的库加载顺序
- 处理 Bionic/glibc 符号冲突

### Box86/Box64 动态翻译

Box86/Box64 使用动态二进制翻译（Dynarec）技术：

1. **代码块识别**：扫描 x86 指令流，识别基本块
2. **即时编译**：将 x86 基本块翻译为 ARM64 指令
3. **缓存复用**：翻译结果缓存在内存中，避免重复翻译
4. **库函数包装**：对常用系统库函数直接转发到 ARM 原生实现

关键环境变量（可在容器配置中设置）：

| 变量 | 说明 | 推荐值 |
|---|---|---|
| `BOX64_DYNAREC_STRONGMEM` | 内存模型严格程度 | 2 |
| `BOX64_DYNAREC_BIGBLOCK` | 大块翻译优化 | 2 |
| `BOX64_DYNAREC_SAFEFLAGS` | 标志位安全处理 | 0 |
| `BOX64_DYNAREC_FASTNAN` | NaN 快速处理 | 1 |
| `BOX64_DYNAREC_FASTROUND` | 舍入快速处理 | 1 |
| `BOX64_DYNAREC_CALLRET` | 调用返回优化 | 1 |

### Wine Prefix 管理

Wine prefix 是一个目录，包含完整的虚拟 Windows 环境：

- `drive_c/`：模拟 C: 盘，包含 Windows 目录结构
- `system.reg` / `user.reg`：注册表文件
- `dosdevices/`：设备映射（盘符 → 主机路径）
- `update-timestamp`：标记 prefix 版本

Grapevine 的增强：
- 每个容器独立 prefix，互不干扰
- 快照系统可保存和恢复 prefix 状态
- 模板系统可克隆 prefix 作为新容器起点
- 注册表补丁支持声明式修改

### GPU 自动检测

Grapevine 在初始化时自动检测设备 GPU：

```bash
# 查看检测结果
grapevine doctor
```

检测逻辑：
1. 读取 `/sys/class/kgsl/kgsl-3d0/gpu_model`（Adreno）
2. 读取 `/sys/class/misc/mali/device/gpuinfo`（Mali）
3. 回退到 `adb shell getprop ro.hardware` 判断
4. 根据检测结果自动选择最优图形驱动

---

## 项目结构

```
grapevine/
├── bin/
│   └── grapevine                 # 主入口脚本
├── lib/
│   ├── core/
│   │   ├── container.sh          # 容器生命周期管理
│   │   ├── snapshot.sh           # 快照管理
│   │   ├── template.sh           # 模板管理
│   │   ├── config.sh             # 配置解析与合并
│   │   └── state.sh              # 运行时状态管理
│   ├── runtime/
│   │   ├── wine.sh               # Wine 启动与管理
│   │   ├── proton.sh             # Proton 启动与管理
│   │   ├── box64.sh              # Box64 配置与启动
│   │   ├── box86.sh              # Box86 配置与启动
│   │   └── proot.sh              # PRoot 隔离设置
│   ├── graphics/
│   │   ├── driver.sh             # 驱动检测与选择
│   │   ├── dxvk.sh               # DXVK 安装与配置
│   │   ├── vkd3d.sh              # VKD3D 安装与配置
│   │   ├── turnip.sh             # Turnip 驱动管理
│   │   ├── virgl.sh              # VirGL 驱动管理
│   │   └── zink.sh               # Zink 驱动管理
│   ├── audio/
│   │   ├── pulseaudio.sh         # PulseAudio 配置
│   │   └── alsa.sh               # ALSA 配置
│   ├── display/
│   │   └── x11.sh                # Termux-X11 管理
│   ├── component/
│   │   ├── manager.sh            # 组件版本管理器
│   │   └── repository.sh         # 组件仓库与下载
│   └── utils/
│       ├── yaml.sh               # YAML 解析工具
│       ├── log.sh                # 日志工具
│       ├── io.sh                 # 输入输出工具
│       └── gpu.sh                # GPU 检测工具
├── templates/
│   ├── gaming-dx11.yml           # 游戏 DX11 模板
│   ├── gaming-dx9.yml            # 游戏 DX9 模板
│   ├── gaming-dx12.yml           # 游戏 DX12 模板
│   ├── office.yml                # 办公模板
│   ├── development.yml           # 开发模板
│   └── minimal.yml               # 最小模板
├── tui/
│   ├── main.sh                   # TUI 主界面
│   ├── containers.sh             # 容器管理面板
│   ├── components.sh             # 组件管理面板
│   └── settings.sh               # 设置面板
├── install                       # 安装脚本
├── LICENSE                       # AGPL-3.0 许可证
└── README.md                     # 本文件
```

---

## 常见问题

### Termux 在后台被杀

Android 12+ 的幻影进程限制可能导致 Termux 被杀。解决方案：

```bash
# 通过 ADB（需要电脑）
adb shell "/system/bin/device_config put activity_manager max_phantom_processes 2147483647"

# 或在 Grapevine 设置中启用 OOM 调整器（需要 root）
grapevine config set --global oom_adjuster.enabled true
```

### 图形渲染异常

- **Adreno 7xx 闪烁**：启用 a7xx 闪烁修复
  ```bash
  grapevine config set my-container compatibility.env_vars.TU_DEBUG noconform
  ```
- **黑屏/无渲染**：尝试切换图形驱动
  ```bash
  grapevine config set my-container graphics.driver virgl
  ```
- **性能差**：降低分辨率，确认使用 Turnip 驱动

### Wine prefix 创建卡住

部分设备在安装 PhysX 时 prefix 创建会冻结。解决方案：

```bash
grapevine config set my-container compatibility.env_vars.WINE_NO_PHYSX 1
```

### SD8450 设备 DRI3 问题

```bash
grapevine config set my-container compatibility.env_vars.DRI3_DISABLE 1
```

### 诊断工具

```bash
# 运行完整诊断
grapevine doctor

# 查看容器日志
grapevine info my-container --logs

# 启用调试日志
grapevine config set --global log_level debug
```

---

## 致谢与第三方组件

Grapevine 的实现依赖以下开源项目：

| 项目 | 许可证 | 用途 |
|---|---|---|
| [Termux](https://github.com/termux/termux-app) | GPLv3 | Android 终端模拟器 |
| [Termux-X11](https://github.com/termux/termux-x11) | GPLv3 | X11 显示服务 |
| [Wine](https://www.winehq.org/) | LGPL-2.1 | Windows API 兼容层 |
| [Proton](https://github.com/ValveSoftware/Proton) | BSD-3 / LGPL | Valve 增强版 Wine |
| [Box64](https://github.com/ptitSeb/box64) | MIT | x64→ARM64 指令转译 |
| [Box86](https://github.com/ptitSeb/box86) | MIT | x86→ARM32 指令转译 |
| [PRoot](https://github.com/proot-me/proot) | GPLv2 | 用户空间虚拟化 |
| [glibc-packages](https://github.com/termux-pacman/glibc-packages) | GPLv3 | Termux glibc 运行时 |
| [DXVK](https://github.com/doitsujin/dxvk) | Zlib | DX9/10/11→Vulkan |
| [VKD3D-Proton](https://github.com/HansKristian-Work/vkd3d-proton) | LGPL-2.1 | DX12→Vulkan |
| [D8VK](https://github.com/AlpyneDreams/d8vk) | Zlib | DX8→Vulkan |
| [Mesa](https://docs.mesa3d.org/) | MIT | 开源图形库 |
| [Turnip](https://gitlab.freedesktop.org/mesa/mesa) | MIT | Adreno Vulkan 驱动 |
| [VirGL](https://gitlab.freedesktop.org/virgl/virglrenderer) | MIT | OpenGL 模拟 Vulkan |
| [Mesa-Zink](https://docs.mesa3d.org/) | MIT | Vulkan→OpenGL 转译 |

特别感谢 Winlator、Mobox、Box64Droid 项目对 Android 上 Windows 兼容性领域的探索和贡献。

---

## 许可证

本项目基于 [GNU Affero General Public License v3.0](LICENSE) 发布。
