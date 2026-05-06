# Grapevine 分发方案决策文档

---

## 1. 文档信息

| 项目 | 内容 |
|------|------|
| 项目名称 | Grapevine — Windows 兼容容器管理平台 |
| 文档版本 | 1.0.0 |
| 文档状态 | 草案 |
| 创建日期 | 2026-05-06 |
| 最后更新 | 2026-05-06 |
| 决策结论 | 采用独立 Android APK 方案 |

---

## 2. 决策背景

Grapevine 需要在两种分发方案之间做出选择：

- **方案 A**：作为 Termux 软件包，依赖 Termux 运行环境
- **方案 B**：作为独立 Android APK，内嵌运行时环境

竞品 Winlator、GameHub、Vectras VM 均采用独立 APK 方案，提供原生 Android GUI 交互界面。Grapevine 的核心卖点之一是易用的交互界面，与上述竞品直接竞争。

---

## 3. 方案对比

### 3.1 用户体验对比

| 维度 | Termux 包 | 独立 APK |
|------|:-:|:-:|
| 安装步骤 | 安装 Termux → 配置权限 → 安装 Grapevine → 安装组件 (4+步) | 安装 APK (1步) |
| 交互界面 | TUI (终端界面) | 原生 Android GUI |
| 目标用户适配 | 仅技术用户 | 全部用户 |
| 环境一致性 | ❌ 依赖用户 Termux 版本 | ✅ Bootstrap 完全可控 |
| 外部依赖 | Termux + Termux-X11 | 无 (内嵌 Xlorie) |
| 后台保活 | Termux 通知栏 | 前台 Service (更可控) |

### 3.2 技术挑战对比

两种方案面对的技术挑战本质相同，区别在于谁承担适配成本：

| 技术挑战 | Termux 包 | 独立 APK | 承担方 |
|---------|:-:|:-:|:-:|
| glibc/Bionic 兼容 | 同样需要解决 | 同样需要解决 | Termux: 用户自行; APK: 开发者一次解决 |
| PRoot 文件系统隔离 | pkg install | 内嵌 (复用 Winlator JNI) | APK 方案有成熟实现可复用 |
| 音频桥接 | 安装 PulseAudio + 配置 | 自研 ALSA 桥接 (复用 Winlator) | 工作量相当 |
| X11 显示 | 依赖外部 Termux-X11 | 内嵌 Xlorie | APK 方案消除外部依赖 |
| W^X / SELinux | Termux 已处理 | 需自行处理 | APK 开发者一次性解决 |
| 组件热更新 | Shell 脚本管理 | installable_components 机制 | 两种方案均支持 |

### 3.3 开发成本对比

| 开发项 | Termux 包 | APK (基于 Winlator 修改) |
|--------|:-:|:-:|
| Shell 核心逻辑 | 相同 | 完全复用 |
| GUI 开发 | 无 (仅 TUI) | 需开发 (核心卖点) |
| ALSA/sysvshm 模块 | 不需要 | 复用 Winlator 已有实现 |
| glibc rootfs 构建 | 依赖 termux-pacman | 自建 (Winlator-glibc 已有参考) |
| PRoot 集成 | pkg install | 复用 Winlator JNI |
| CI/CD 构建 | 简单 (tar.gz) | 需 Android 构建流水线 |

---

## 4. 决策结论

### 选择方案 B：独立 Android APK

核心理由：

1. **易用性是核心卖点**：与 Winlator/GameHub/Vectras VM 竞争，没有原生 GUI 就没有竞争力
2. **技术挑战等价**：glibc、音频、显示等问题在 Termux 方案中同样存在，APK 方案将复杂度内化，用户体验更好
3. **环境一致性天然解决**：Bootstrap 机制确保所有用户运行完全相同的环境
4. **Winlator 已铺路**：大量底层模块可直接复用，Grapevine 的增量价值（容器管理、快照、模板、声明式配置）均为 Shell 脚本逻辑，可直接复用
5. **组件热更新不丧失**：Winlator 的 installable_components 机制已证明 APK 内可独立更新组件
6. **安装体验碾压**：一个 APK vs 多步骤安装，无可比性

---

## 5. 推荐架构

### 5.1 项目结构

```
grapevine/
├── app/                          # Android 主应用
│   ├── src/main/java/            # Java/Kotlin: GUI + 容器管理
│   │   ├── ui/                   # Android 原生界面
│   │   │   ├── ContainerListActivity
│   │   │   ├── ContainerDetailActivity
│   │   │   ├── ComponentManagerActivity
│   │   │   ├── TemplateBrowserActivity
│   │   │   └── SnapshotManagerActivity
│   │   ├── service/              # 前台服务 + 进程管理
│   │   └── runtime/              # 运行时桥接层
│   └── src/main/cpp/             # JNI: 原生组件
│       ├── proot/                # PRoot (复用 Winlator)
│       ├── xlorie/               # X11 显示服务 (内嵌)
│       └── grapevine_jni/        # Grapevine JNI 桥接
├── android_alsa/                 # ALSA 音频桥接 (复用 Winlator)
├── android_sysvshm/              # System V 共享内存 (复用 Winlator)
├── grapevine-core/               # Grapevine 核心逻辑 (Shell 脚本)
│   ├── bin/grapevine             # CLI 入口
│   ├── lib/core/                 # 核心模块
│   ├── lib/runtime/              # 运行时模块
│   ├── lib/graphics/             # 图形模块
│   ├── lib/audio/                # 音频模块
│   ├── lib/component/            # 组件管理
│   ├── lib/utils/                # 工具函数
│   └── templates/                # 内置模板
├── bootstrap/                    # 首次启动引导
│   ├── rootfs/                   # glibc rootfs (Ubuntu-based)
│   └── scripts/                  # 初始化脚本
├── installable_components/       # 可安装组件 (热更新)
│   └── index.txt                 # 组件索引
├── input_controls/               # 输入控制预设
├── gradle/                       # Gradle 构建配置
├── build.gradle
└── settings.gradle
```

### 5.2 Shell 脚本保留策略

Grapevine 的 Shell 脚本核心逻辑完全保留，运行在 APK 内嵌的 rootfs 环境中。Java GUI 层通过进程调用 Shell 脚本：

```
用户操作 (GUI)
  → Java 层调用 Runtime.exec("grapevine <command> --format json")
  → Shell 脚本在 rootfs 环境中执行
  → 结果通过 stdout (JSON) 返回 Java 层
  → GUI 解析并渲染
```

### 5.3 组件热更新机制

```
installable_components/
├── index.txt              # 可用组件索引 (远程更新)
├── wine/                  # Wine 组件
│   ├── 9.0/
│   └── 9.4/
├── box64/
├── dxvk/
└── ...
```

用户在 GUI 中点击"更新组件" → Java 层调用 `grapevine component update` → Shell 脚本从镜像源下载新版本 → 无需更新 APK。

### 5.4 开发路径

1. Fork Winlator-glibc 作为基础框架
2. 叠加 Grapevine 的 Shell 脚本核心逻辑到 rootfs 中
3. 开发 Android GUI 层（容器管理、快照、模板、组件管理界面）
4. 适配 Shell 脚本路径（从 `$PREFIX`/`$HOME` 改为 APK 私有目录）
5. 保留 CLI 入口（高级用户可通过 adb shell 调用）

---

## 6. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| Winlator 上游 License 变更 | 需要重新评估复用策略 | 持续跟踪上游 License，保持解耦 |
| glibc rootfs 构建复杂 | 首次构建成本高 | 参考 Winlator-glibc 构建脚本，复用 OBB 生成器 |
| APK 体积过大 | 用户体验下降 | OBB 分包 + 组件按需下载 |
| Android 版本碎片化 | 兼容性风险 | 定义最低 API 29 (Android 10)，参考 Winlator 兼容性矩阵 |
| Shell 脚本路径适配 | 工作量中等 | 封装路径变量，统一通过 `GRAPEVINE_HOME` 管理 |

---

## 7. Proton 兼容性与灵活性分析

### 7.1 Proton 技术现状

Proton 是 Valve 与 CodeWeavers 联合开发的 Windows 兼容层，基于 Wine 构建，为 Steam 客户端提供 Windows 游戏在 Linux 上的运行能力。截至 2026 年 4 月，Proton 11.0 Beta1 已正式支持 ARM64 架构。

**关键里程碑**：

| 时间 | 事件 |
|------|------|
| 2018 | Proton 首次发布，仅支持 x86_64 |
| 2024 | Box64 v0.3.0 宣布支持 Proton |
| 2025.11 | Valve 公开赞助 FEX-Emu 开发，Steam Frame 发布 |
| 2026.04 | Proton 11.0 Beta1 发布，新增 ARM64 构建配置，集成 FEX 2604 |

### 7.2 Proton 在 Android/ARM64 上的运行架构

Proton 11 ARM64 采用双层翻译架构：

```
Windows x86/x86_64 应用程序
        ↓
Proton (Wine 11 + Valve 补丁)     ← Windows API → Linux API 翻译
        ↓
FEX 2604                           ← x86/x86_64 → ARM64 指令翻译
        ↓
ARM64 Linux 内核
```

这与 Grapevine 当前的 Box86/Box64 + Wine 架构形成对比：

```
Windows x86/x86_64 应用程序
        ↓
Wine (标准版或 Proton 补丁版)      ← Windows API → Linux API 翻译
        ↓
Box86 / Box64                      ← x86/x86_64 → ARM64 指令翻译
        ↓
PRoot (用户空间 chroot)
        ↓
Android Bionic 内核
```

### 7.3 Proton vs Wine：Grapevine 的选择

| 维度 | Wine (标准版) | Proton |
|------|:-:|:-:|
| 开发者 | Wine 社区 | Valve + CodeWeavers + 社区 |
| 游戏优化 | 通用兼容性 | 针对 Steam 游戏深度优化 |
| DXVK/VKD3D 集成 | 需手动安装 | 内置，版本匹配 |
| 游戏特定补丁 | 无 | 大量 per-game 补丁 (Proton-specific) |
| Steam 集成 | 无 | 深度集成 (Steam overlay, controller) |
| ARM64 原生支持 | Wine 11+ 实验性 | Proton 11+ 官方支持 |
| FEX 集成 | 需手动配置 | Proton 11 ARM64 内置 FEX 2604 |
| 更新频率 | 每 2 周 (开发版) | 跟随 Steam 更新节奏 |
| License | LGPL v2.1+ | BSD-3-Clause (Proton 核心) + LGPL (Wine 部分) |
| 非 Steam 场景 | ✅ 通用 | ⚠️ 主要面向 Steam 游戏 |
| Android 适配 | 社区构建 (Kron4ek 等) | GameNative 社区 Android 构建 |

### 7.4 Proton 在 Grapevine 中的兼容性评估

#### 7.4.1 优势

1. **游戏兼容性显著提升**：Proton 包含大量 per-game 补丁（Anti-cheat 绕过、特定游戏崩溃修复、性能优化），这些是标准 Wine 不具备的。Grapevine 的目标用户（游戏玩家）将直接受益。

2. **DXVK/VKD3D 版本一致性**：Proton 内置的 DXVK 和 VKD3D-Proton 版本经过 Valve 测试验证，避免了用户自行搭配版本导致的兼容性问题。

3. **NTSync 支持**：Proton 11 基于 Wine 11，引入 NTSync 内核驱动支持，减少 Windows 同步原语的开销，提升游戏帧率稳定性。

4. **Valve 持续投入**：Valve 正在大力投资 ARM64 生态（Steam Frame、FEX 赞助），Proton 的 ARM64 支持将持续改善。

5. **GameNative 社区已验证 Android 可行性**：[GameNative/proton-wine](https://github.com/GameNative/proton-wine) 已合并 Android 支持的 PR（#15），使用 Android NDK r27d + LLVM MinGW 构建 Proton Wine，支持 ARM64EC 架构和 16KB 页面大小。

#### 7.4.2 挑战

1. **Proton 与 Steam 的耦合**：Proton 的部分功能（Steam overlay、Steam controller、Steam shader pre-caching）依赖 Steam 客户端。在 Grapevine 的非 Steam 环境中，这些功能不可用。但核心兼容层（Wine + DXVK + VKD3D）独立于 Steam 运行。

2. **FEX vs Box64 的选择**：Proton 11 ARM64 使用 FEX 作为指令翻译层，而 Grapevine 当前设计使用 Box86/Box64。两者互为竞争关系：
   - **FEX**：Valve 官方支持，与 Proton 深度集成，但在 Android 上需要 root 或特定内核支持
   - **Box64**：Android 生态更成熟（Winlator/Mobox 已验证），无需 root，社区活跃

3. **构建复杂度**：Proton 的构建依赖 Docker/Podman + Proton SDK 容器，构建流程远比标准 Wine 复杂。GameNative 的 Android 构建流程使用 GitHub Actions，需要 termuxfs + Android NDK 交叉编译。

4. **体积**：Proton 完整构建包含 Wine + DXVK + VKD3D + FEX + Steam helper 等组件，体积显著大于标准 Wine。

5. **更新节奏依赖 Valve**：Proton 的更新节奏由 Valve 控制，社区无法自由决定何时更新 Wine 基线版本。

#### 7.4.3 灵活性评估

Grapevine 的组件管理架构天然支持 Wine/Proton 的灵活切换：

```yaml
# registry.yml 中的组件配置
runtime:
  wine:
    - name: "wine-stable"
      version: "9.0"
      source: "kron4ek"
    - name: "wine-staging"
      version: "9.4"
      source: "kron4ek"
    - name: "proton-experimental"
      version: "11.0-beta1"
      source: "valve"
      notes: "需配合 FEX 或 Box64 使用"
  translator:
    - name: "box64"
      version: "0.3.0"
    - name: "fex"
      version: "2604"
      notes: "需配合 Proton ARM64 使用"
```

**建议策略**：

- **默认配置**：Box64 + Wine Staging（成熟稳定，Android 生态验证充分）
- **高级选项**：Box64 + Proton Experimental（游戏兼容性更好，需用户主动选择）
- **实验性选项**：FEX + Proton ARM64（Valve 官方方案，但 Android 支持尚不成熟）

### 7.5 Proton 引入路径

| 阶段 | 目标 | 实现方式 |
|------|------|----------|
| Phase 1 (MVP) | Wine + Box64 基础运行 | 复用 Winlator 现有方案 |
| Phase 2 | Proton 作为可选组件 | 通过组件管理器下载 Proton 构建包，替换默认 Wine |
| Phase 3 | FEX + Proton ARM64 原生 | 跟随 Valve 官方进展，待 FEX Android 支持成熟后集成 |

---

## 8. 开源库 License 侵权风险分析

### 8.1 Grapevine 依赖的开源库 License 清单

| 组件 | 仓库 | License | Copyleft 强度 | 侵权风险 |
|------|------|---------|:-:|:-:|
| **Winlator** | brunodev85/winlator | MIT | 无 | 🟢 低 |
| **Box86** | ptitSeb/box86 | MIT | 无 | 🟢 低 |
| **Box64** | ptitSeb/box64 | MIT | 无 | 🟢 低 |
| **Wine** | wine-mirror/wine | LGPL-2.1+ | 弱 | 🟡 中 |
| **Proton (核心)** | ValveSoftware/Proton | BSD-3-Clause | 无 | 🟢 低 |
| **Proton (Wine 部分)** | ValveSoftware/wine | LGPL-2.1+ | 弱 | 🟡 中 |
| **PRoot** | termux/proot | GPL-2.0+ | 强 | 🔴 高 |
| **Mesa (Turnip/Zink/VirGL)** | mesa3d/mesa | MIT / SGI-B-2.0 / Khronos | 无 | 🟢 低 |
| **DXVK** | doitsujin/dxvk | zlib | 无 | 🟢 低 |
| **VKD3D-Proton** | HansKristian-Work/vkd3d-proton | LGPL-2.1+ / MIT | 弱 | 🟡 中 |
| **FEX-Emu** | FEX-Emu/FEX | MIT | 无 | 🟢 低 |
| **Xlorie** | termux/x11-modules | GPL-3.0 | 强 | 🔴 高 |
| **Termux 终端 UI** | termux/terminal-emulator | GPL-3.0 | 强 | 🔴 高 |
| **Termux 终端 View** | termux/terminal-view | GPL-3.0 | 强 | 🔴 高 |
| **Vectras VM** | xoureldeen/Vectras-VM-Android | GPL-2.0 | 强 | 🔴 高 |
| **Ubuntu RootFS** | Canonical | 多种 (含 GPL) | 混合 | 🟡 中 |
| **android_alsa** | Winlator 内置 | MIT (跟随 Winlator) | 无 | 🟢 低 |
| **android_sysvshm** | Winlator 内置 | MIT (跟随 Winlator) | 无 | 🟢 低 |
| **glibc** | GNU Project | LGPL-2.1+ / GPL-2.0+ | 弱/强 | 🟡 中 |

### 8.2 高风险组件详细分析

#### 8.2.1 PRoot (GPL-2.0+) — 🔴 高风险

**风险描述**：PRoot 采用 GPL-2.0+ 许可证，这是强 Copyleft 协议。如果 Grapevine 将 PRoot 编译为 JNI 库并链接到 APK 中，整个 APK 可能需要以 GPL-2.0+ 开源。

**Winlator 的做法**：Winlator 将 PRoot 作为独立进程通过 `Runtime.exec()` 调用，而非通过 JNI 链接。这种"臂距"（arm's length）调用方式在 GPL 社区中通常被认为不构成衍生作品，因此 Winlator 的 MIT 许可证与 PRoot 的 GPL 许可证不冲突。

**Grapevine 的合规策略**：
- ✅ 与 Winlator 相同，通过进程调用方式使用 PRoot，不进行 JNI 链接
- ✅ 在 APK 的 about/credits 页面中保留 PRoot 的版权声明和 GPL 许可证文本
- ✅ 在项目仓库中提供 PRoot 源码的获取说明
- ❌ 不要将 PRoot 代码直接编译进 APK 的 native 库中

#### 8.2.2 Xlorie (GPL-3.0) — 🔴 高风险

**风险描述**：Xlorie（Termux-X11 的 X11 服务器模块）采用 GPL-3.0 许可证。如果 Grapevine 内嵌 Xlorie，需要满足 GPL-3.0 的要求。

**Grapevine 的合规策略**：
- ✅ 将 Xlorie 作为独立二进制文件放入 rootfs，通过进程方式调用
- ✅ 保留版权声明和 GPL-3.0 许可证文本
- ✅ 提供 Xlorie 源码获取方式（指向上游仓库 + 构建说明）
- ❌ 不要将 Xlorie 代码静态链接到 Grapevine 的 JNI 库中

#### 8.2.3 Wine / Proton-Wine (LGPL-2.1+) — 🟡 中风险

**风险描述**：Wine 采用 LGPL-2.1+ 许可证。LGPL 允许以共享库方式链接而不要求整个应用开源，但有条件：
1. 必须允许用户替换 LGPL 库的版本（即提供 .so 文件而非静态链接）
2. 必须保留版权声明
3. 如果修改了 Wine 本身，修改部分必须以 LGPL 开源

**Grapevine 的合规策略**：
- ✅ Wine/Proton 作为独立二进制放入 rootfs，以共享库方式提供
- ✅ 用户可通过组件管理器替换 Wine 版本（天然满足"允许替换"要求）
- ✅ 保留版权声明和 LGPL 许可证文本
- ⚠️ 如果 Grapevine 对 Wine 打了自定义补丁，补丁部分必须以 LGPL 开源

#### 8.2.4 glibc (LGPL-2.1+ / GPL-2.0+) — 🟡 中风险

**风险描述**：glibc 核心库采用 LGPL-2.1+，但部分辅助工具和测试套件采用 GPL-2.0+。Grapevine 只需使用 glibc 的运行时库（LGPL），不涉及 GPL 部分。

**Grapevine 的合规策略**：
- ✅ glibc 运行时库以共享库方式放入 rootfs
- ✅ 保留版权声明和 LGPL 许可证文本
- ✅ 不修改 glibc 源码，直接使用发行版预编译包

### 8.3 GameHub 的前车之鉴

GameHub（GameSir 开发的 Windows 模拟器）因 License 合规问题遭到社区强烈批评：

1. **未标注 Winlator 来源**：GameHub 基于 Winlator（MIT 许可证）开发，但未在应用内提供 Winlator 的版权声明。MIT 许可证要求"在所有副本中保留版权声明和许可声明"。

2. **疑似使用 Wine 代码未遵守 LGPL**：社区指控 GameHub 使用了 Wine 代码但未提供源码或获取途径，违反 LGPL 的要求。

3. **闭源发布**：GameHub 是闭源应用，但其使用的多个组件（Wine、Box64 等）要求至少提供源码获取途径。

4. **社区反制**：社区创建了 GameHub Lite（去追踪器版本）和 GameNative（完全开源替代），对 GameHub 的声誉造成了显著损害。

**教训**：即使使用宽松许可证的代码（如 MIT），也必须遵守署名要求。LGPL 组件必须提供源码获取途径和库替换能力。忽视 License 合规会导致社区信任危机。

### 8.4 Grapevine 的 License 合规框架

#### 8.4.1 项目自身 License 选择

**建议：GPL-3.0 或 AGPL-3.0**

理由：
1. Grapevine 依赖多个 GPL 组件（PRoot、Xlorie），如果项目自身采用更宽松的许可证，会导致 License 不兼容问题
2. GPL-3.0 与项目使用的所有组件许可证兼容
3. 强 Copyleft 可以防止闭源商业 fork（如 GameHub 对 Winlator 的做法）
4. 与社区价值观一致，有利于吸引贡献者

**备选方案：MPL-2.0**

如果希望允许部分闭源（如商业插件），MPL-2.0 是一个折中选择：
- 文件级 Copyleft：修改过的文件必须开源，新增文件可以闭源
- 与 GPL/LGPL 兼容
- 允许商业使用

#### 8.4.2 合规检查清单

| 检查项 | 要求 | 实现方式 |
|--------|------|----------|
| 版权声明 | 所有组件保留原始版权声明 | APK 内 about/credits 页面展示 |
| 许可证文本 | 所有组件保留原始许可证全文 | APK 内 assets/licenses/ 目录存放 |
| 源码提供 | GPL/LGPL 组件提供源码获取途径 | GitHub 仓库 + 构建说明文档 |
| 库替换 | LGPL 组件允许用户替换版本 | 组件管理器支持 Wine/glibc 版本切换 |
| 修改声明 | 修改过的组件标注修改内容 | 修改文件头部添加修改说明注释 |
| 补丁开源 | 对 GPL/LGPL 组件的补丁必须开源 | 补丁文件存放在 GitHub 仓库的 patches/ 目录 |

#### 8.4.3 各组件合规要求汇总

| 组件 | License | 必须开源 | 必须署名 | 允许替换 | Grapevine 操作 |
|------|---------|:-:|:-:|:-:|------|
| Winlator | MIT | ❌ | ✅ | N/A | 保留版权声明 |
| Box86/64 | MIT | ❌ | ✅ | N/A | 保留版权声明 |
| Wine | LGPL-2.1+ | 修改部分 ✅ | ✅ | ✅ | 组件管理器支持替换 |
| Proton 核心 | BSD-3-Clause | ❌ | ✅ | N/A | 保留版权声明 |
| PRoot | GPL-2.0+ | ✅ | ✅ | N/A | 进程调用 + 源码提供 |
| Mesa | MIT/SGI-B | ❌ | ✅ | N/A | 保留版权声明 |
| DXVK | zlib | ❌ | ✅ | N/A | 保留版权声明 |
| VKD3D-Proton | LGPL-2.1+ | 修改部分 ✅ | ✅ | ✅ | 组件管理器支持替换 |
| FEX | MIT | ❌ | ✅ | N/A | 保留版权声明 |
| Xlorie | GPL-3.0 | ✅ | ✅ | N/A | 进程调用 + 源码提供 |
| glibc | LGPL-2.1+ | 修改部分 ✅ | ✅ | ✅ | 不修改，直接使用发行版包 |

### 8.5 Fork Winlator 的 License 风险

Winlator 采用 MIT 许可证，这是最宽松的开源许可证之一。Fork Winlator 的合规要求非常简单：

1. ✅ 保留原始版权声明：`Copyright (c) 2023 BrunoSX`
2. ✅ 保留 MIT 许可证文本
3. ✅ 不得使用 BrunoSX 或 Winlator 名称进行推广

**但需注意**：Winlator 内部集成的组件（PRoot、Wine、Mesa 等）各有自己的许可证，Fork 后仍需逐一遵守。

### 8.6 不可引用的项目

| 项目 | License | 不可引用原因 |
|------|---------|------------|
| Vectras VM | GPL-2.0 | 如果引用其代码，Grapevine 必须以 GPL-2.0 开源；且 Vectras VM 内嵌 Termux 组件的方式存在与独立 Termux 冲突的 bug |
| GameHub | 闭源 | 闭源商业软件，不可引用其代码 |

---

## 9. AGPL-3.0 合规使用指南

### 9.1 AGPL-3.0 核心义务

AGPL-3.0 在 GPL-3.0 基础上增加了**第 13 条（网络交互条款）**，是当前最强的 Copyleft 协议。其核心义务如下：

| 义务 | 条款 | 说明 |
|------|------|------|
| 源码分发 | 第 6 条 | 分发二进制时必须提供完整源码 |
| 网络源码提供 | 第 13 条 | 用户通过网络与修改版程序交互时，必须向其提供源码 |
| 修改声明 | 第 5(a) 条 | 修改过的文件必须标注修改说明 |
| 版权声明保留 | 第 5(a) 条 | 保留所有原始版权声明和许可证文本 |
| 许可证附加 | 第 5(a) 条 | 分发的所有副本必须附带 AGPL-3.0 许可证文本 |
| 反附加限制 | 第 7 条 | 不得对用户施加超出 AGPL 的额外限制 |
| 专利授权 | 第 11 条 | 贡献者自动授予用户专利使用权 |

**对 Grapevine 的影响**：

- Grapevine 作为本地 Android 应用分发，标准源码分发义务适用
- 如果未来增加云端功能（云容器存储、远程管理等），第 13 条要求向网络用户提供服务端源码
- AGPL 的传染性意味着：任何与 Grapevine AGPL 代码合并形成衍生作品的代码，也必须以 AGPL 开源

### 9.2 AGPL-3.0 与各组件 License 的兼容性

#### 9.2.1 兼容性判定规则

AGPL-3.0 的兼容性判定遵循以下规则：

1. **宽松型许可证 → AGPL**：✅ 兼容。MIT/BSD/zlib 代码可被 AGPL 吸收，合并后的整体以 AGPL 分发
2. **LGPL → AGPL**：✅ 兼容。LGPL 库可作为共享库链接，合并后的整体以 AGPL 分发，但 LGPL 部分仍保留 LGPL
3. **GPL-2.0+ → AGPL**：✅ 兼容。GPL-2.0+ 中的 "+" 表示 "or any later version"，用户可选择以 GPL-3.0 条款使用，而 GPL-3.0 与 AGPL-3.0 兼容
4. **GPL-3.0 → AGPL**：✅ 兼容。GPL-3.0 代码可合并入 AGPL-3.0 作品，整体以 AGPL 分发
5. **GPL-2.0-only → AGPL**：❌ 不兼容。GPL-2.0-only 不允许升级到 GPL-3.0/AGPL-3.0
6. **Apache-2.0 → AGPL**：❌ 不兼容。Apache-2.0 的专利条款与 GPL-2.0 冲突，但与 GPL-3.0/AGPL-3.0 兼容（GPL-3.0 已修复此冲突）

**修正**：Apache-2.0 与 AGPL-3.0 实际上是兼容的，因为 AGPL-3.0 基于 GPL-3.0，而 GPL-3.0 明确与 Apache-2.0 兼容。

#### 9.2.2 Grapevine 各组件与 AGPL-3.0 的兼容性

| 组件 | License | 与 AGPL 兼容 | 原因 |
|------|---------|:-:|------|
| Winlator | MIT | ✅ | 宽松型，可被 AGPL 吸收 |
| Box86/64 | MIT | ✅ | 宽松型，可被 AGPL 吸收 |
| Wine | LGPL-2.1+ | ✅ | 弱 Copyleft，允许作为共享库链接 |
| Proton 核心 | BSD-3-Clause | ✅ | 宽松型，可被 AGPL 吸收 |
| PRoot | GPL-2.0+ | ✅ | "or later" 允许升级到 GPL-3.0/AGPL-3.0 |
| Mesa | MIT / SGI-B-2.0 | ✅ | 宽松型，可被 AGPL 吸收 |
| DXVK | zlib | ✅ | 宽松型，可被 AGPL 吸收 |
| VKD3D-Proton | LGPL-2.1+ / MIT | ✅ | 弱 Copyleft + 宽松型 |
| FEX-Emu | MIT | ✅ | 宽松型，可被 AGPL 吸收 |
| Xlorie | GPL-3.0 | ✅ | GPL-3.0 与 AGPL-3.0 兼容 |
| Termux 终端 UI | GPL-3.0 | ✅ | GPL-3.0 与 AGPL-3.0 兼容 |
| Termux 终端 View | GPL-3.0 | ✅ | GPL-3.0 与 AGPL-3.0 兼容 |
| glibc | LGPL-2.1+ | ✅ | 弱 Copyleft，允许作为共享库链接 |
| android_alsa | MIT | ✅ | 宽松型，可被 AGPL 吸收 |
| android_sysvshm | MIT | ✅ | 宽松型，可被 AGPL 吸收 |
| Ubuntu RootFS | 多种 | ⚠️ | RootFS 中的各软件包有各自 License，需逐一检查 |

**结论**：Grapevine 使用的所有核心组件均与 AGPL-3.0 兼容。不存在 License 不可逾越的冲突。

### 9.3 关键概念：衍生作品 vs 聚合

AGPL-3.0 的传染性仅作用于**衍生作品**（derivative work），不作用于**聚合**（aggregate）。这是合规架构设计的核心依据。

#### 9.3.1 判定标准

```
衍生作品（传染）                    聚合（不传染）
┌─────────────────────┐          ┌─────────────────────┐
│  程序 A (AGPL)       │          │  介质 (APK/ISO)      │
│  ┌─────────────────┐│          │  ┌─────────────────┐│
│  │ 程序 B (GPL)     ││          │  │ 程序 A (AGPL)   ││
│  │ (链接/嵌入)      ││          │  │ (独立进程)      ││
│  └─────────────────┘│          │  ├─────────────────┤│
│  A 和 B 形成单一作品  │          │  │ 程序 B (GPL)    ││
│  → 整体必须 AGPL     │          │  │ (独立进程)      ││
└─────────────────────┘          │  └─────────────────┘│
                                  │  A 和 B 是独立程序  ││
                                  │  → 各自保留原 License││
                                  └─────────────────────┘
```

**判定要素**：

| 链接方式 | 是否衍生作品 | AGPL 传染 |
|---------|:-:|:-:|
| 静态链接 | ✅ 是 | ✅ 传染 |
| JNI 调用 | ✅ 是 | ✅ 传染 |
| 动态链接 (.so) | ⚠️ 有争议 | ⚠️ 可能传染 |
| 管道/IPC 通信 | ❌ 否 | ❌ 不传染 |
| Runtime.exec() 进程调用 | ❌ 否 | ❌ 不传染 |
| 独立二进制在 rootfs 中共存 | ❌ 否 | ❌ 不传染 |

**AGPL 第 5 条的说明**：

> "The output from running a covered work is covered by this License only if the output, given its content, constitutes a covered work."

这意味着：Grapevine (AGPL) 调用 PRoot (GPL) 作为独立进程，PRoot 的输出不属于 Grapevine 的衍生作品。

### 9.4 各组件的正确使用方式

#### 9.4.1 进程隔离方式使用的组件（不触发 AGPL 传染）

这些组件作为独立进程运行，与 Grapevine 主程序通过 IPC/管道/命令行通信，不构成衍生作品：

| 组件 | License | 使用方式 | 合规要求 |
|------|---------|---------|----------|
| PRoot | GPL-2.0+ | `Runtime.exec("proot ...")` 进程调用 | ① 保留版权声明 ② 提供 PRoot 源码获取途径 ③ 不修改 PRoot 源码或修改后以 GPL 开源 |
| Xlorie | GPL-3.0 | 作为 rootfs 中的独立二进制运行 | ① 保留版权声明 ② 提供 Xlorie 源码获取途径 ③ 不修改或修改后以 GPL-3.0 开源 |
| Wine | LGPL-2.1+ | 作为 rootfs 中的独立二进制运行 | ① 保留版权声明 ② 提供 Wine 源码获取途径 ③ 允许用户替换 Wine 版本 |
| Box86/64 | MIT | 作为 rootfs 中的独立二进制运行 | ① 保留版权声明 |
| DXVK | zlib | 作为 rootfs 中的独立 DLL 运行 | ① 保留版权声明 |
| Mesa 驱动 | MIT | 作为 rootfs 中的独立共享库运行 | ① 保留版权声明 |
| FEX | MIT | 作为 rootfs 中的独立二进制运行（Phase 3） | ① 保留版权声明 |
| Proton | BSD+LGPL | 作为 rootfs 中的独立二进制运行（Phase 2） | ① 保留版权声明 ② Proton Wine 部分遵守 LGPL |
| glibc | LGPL-2.1+ | 作为 rootfs 中的共享库运行 | ① 保留版权声明 ② 不修改 glibc 源码 |

**进程隔离的技术实现**：

```java
// Java 层：通过进程调用 PRoot，不触发 AGPL 传染
Process process = Runtime.getRuntime().exec(
    "proot --rootfs=/data/.../rootfs " +
    "--bind=/data/.../wine " +
    "/bin/wine wineboot"
);
// 通过 stdin/stdout 通信，不共享内存，不使用 JNI
```

```bash
# Shell 层：grapevine-core 调用 PRoot
proot --rootfs="${GRAPEVINE_HOME}/rootfs" \
      --bind="${GRAPEVINE_HOME}/wine" \
      --bind="${GRAPEVINE_HOME}/box64" \
      /bin/wine wineboot
```

#### 9.4.2 链接方式使用的组件（触发 AGPL 传染）

这些组件通过 JNI 或动态链接与 Grapevine 主程序合并，构成衍生作品，整体以 AGPL 开源：

| 组件 | License | 使用方式 | 合规要求 |
|------|---------|---------|----------|
| Winlator 代码 (MIT) | MIT | Fork 并修改 Java/C 代码 | ① 保留 BrunoSX 版权声明 ② 修改后的代码以 AGPL 开源 |
| android_alsa (MIT) | MIT | 作为 JNI 库编译进 APK | ① 保留版权声明 ② 以 AGPL 开源 |
| android_sysvshm (MIT) | MIT | 作为 JNI 库编译进 APK | ① 保留版权声明 ② 以 AGPL 开源 |
| Grapevine 自研代码 | AGPL-3.0 | Java/Kotlin GUI + Shell 脚本 | 以 AGPL-3.0 开源 |

**注意**：由于 Winlator、android_alsa、android_sysvshm 均为 MIT 许可证，Grapevine 可以自由地将其代码合并到 AGPL 项目中。MIT 代码在 AGPL 项目中仍保留 MIT 许可证，但整体作品以 AGPL 分发。

#### 9.4.3 Termux 终端 UI 组件的特殊处理

Termux 的 `terminal-emulator` 和 `terminal-view` 采用 GPL-3.0 许可证。如果 Grapevine 希望内嵌这些组件，有两种合规方式：

**方式一：进程隔离（推荐）**

不内嵌 Termux 终端 UI，改用自研终端模拟器或完全使用原生 Android GUI。这样完全避免 GPL-3.0 传染问题。

**方式二：AGPL 合并**

由于 GPL-3.0 与 AGPL-3.0 兼容，可以将 Termux 终端 UI 代码合并到 Grapevine 中，整体以 AGPL-3.0 分发。但需要：
1. 保留 Termux 原始版权声明
2. 提供 Termux 终端 UI 源码获取途径
3. 标注对 Termux 代码的修改内容

**方式三：独立 Activity 调用**

将 Termux 终端 UI 编译为独立 APK 或独立模块，通过 Android Intent 启动。这种方式类似于进程隔离，不构成衍生作品。

### 9.5 项目目录结构与 License 标注

```
grapevine/                           # AGPL-3.0 (Grapevine 自研代码)
├── LICENSE                          # AGPL-3.0 全文
├── NOTICE                           # 第三方组件版权声明汇总
├── app/
│   ├── src/main/java/               # AGPL-3.0 (Grapevine 自研)
│   ├── src/main/cpp/
│   │   ├── proot/                   # GPL-2.0+ (进程调用，不链接)
│   │   │   └── LICENSE              # PRoot GPL-2.0+ 许可证
│   │   ├── xlorie/                  # GPL-3.0 (进程调用，不链接)
│   │   │   └── LICENSE              # Xlorie GPL-3.0 许可证
│   │   └── grapevine_jni/           # AGPL-3.0 (Grapevine 自研)
│   └── assets/
│       └── licenses/                # 所有第三方许可证文本
│           ├── LICENSE.winlator     # MIT
│           ├── LICENSE.box86        # MIT
│           ├── LICENSE.box64        # MIT
│           ├── LICENSE.wine         # LGPL-2.1+
│           ├── LICENSE.proton       # BSD-3-Clause + LGPL
│           ├── LICENSE.proot        # GPL-2.0+
│           ├── LICENSE.mesa         # MIT / SGI-B-2.0
│           ├── LICENSE.dxvk         # zlib
│           ├── LICENSE.vkd3d-proton # LGPL-2.1+ / MIT
│           ├── LICENSE.fex          # MIT
│           ├── LICENSE.xlorie       # GPL-3.0
│           ├── LICENSE.glibc        # LGPL-2.1+
│           └── LICENSE.ubuntu       # 多种
├── android_alsa/                    # MIT (Winlator 原始代码)
│   └── LICENSE                      # MIT 许可证 + BrunoSX 版权声明
├── android_sysvshm/                 # MIT (Winlator 原始代码)
│   └── LICENSE                      # MIT 许可证 + BrunoSX 版权声明
├── grapevine-core/                  # AGPL-3.0 (Grapevine 自研)
├── bootstrap/
│   └── rootfs/                      # 各组件保留各自 License
│       ├── usr/bin/wine             # LGPL-2.1+
│       ├── usr/bin/proot            # GPL-2.0+
│       ├── usr/lib/libGL.so         # MIT (Mesa)
│       └── ...
├── patches/                         # 对第三方组件的补丁
│   ├── wine/                        # Wine 补丁 (LGPL-2.1+)
│   ├── proot/                       # PRoot 补丁 (GPL-2.0+)
│   └── xlorie/                      # Xlorie 补丁 (GPL-3.0)
└── THIRD_PARTY_NOTICES.md           # 第三方组件声明文档
```

### 9.6 AGPL-3.0 合规操作清单

#### 9.6.1 源码分发义务

| 场景 | 义务 | 实现方式 |
|------|------|----------|
| APK 分发 | 提供完整源码 | GitHub 公开仓库，APK 内 about 页面提供链接 |
| 修改 AGPL 代码 | 标注修改 | 修改文件头部添加 `Modified by Grapevine - <date> - <description>` |
| 修改 GPL/LGPL 代码 | 补丁以原 License 开源 | `patches/` 目录存放补丁文件，以原组件 License 发布 |
| 网络交互（未来） | 向网络用户提供源码 | 如增加云端功能，服务端代码必须公开 |

#### 9.6.2 版权声明义务

每个源文件头部应包含如下声明：

**Grapevine 自研文件**：
```
Copyright (C) 2026 Grapevine Contributors
SPDX-License-Identifier: AGPL-3.0-or-later
```

**基于 Winlator 修改的文件**：
```
Copyright (c) 2023 BrunoSX (original Winlator code)
Copyright (C) 2026 Grapevine Contributors (modifications)
SPDX-License-Identifier: AGPL-3.0-or-later
Original source: https://github.com/brunodev85/winlator
```

**未修改的第三方文件**：保留原始版权声明和 License 头部，不做任何修改。

#### 9.6.3 用户权利保障

| 用户权利 | AGPL 要求 | Grapevine 实现 |
|---------|----------|---------------|
| 获取源码 | 分发二进制时必须提供源码 | GitHub 仓库 + APK 内链接 |
| 修改和再分发 | 允许用户修改和再分发 | AGPL 天然允许 |
| 替换 LGPL 组件 | 用户必须能替换 LGPL 库版本 | 组件管理器支持 Wine/glibc 版本切换 |
| 网络使用获取源码 | 网络交互用户也有权获取源码 | APK 内 about 页面提供源码链接 |
| 查看完整许可证 | 必须提供 AGPL 全文 | APK 内 assets/licenses/ 存放 |

#### 9.6.4 NOTICE 文件模板

```
Grapevine - Windows 兼容容器管理平台
Copyright (C) 2026 Grapevine Contributors

本程序是自由软件：您可以根据自由软件基金会发布的 GNU Affero 通用公共许可证
（版本 3 或更高版本）的条款重新分发和/或修改它。

本程序分发的目的是希望它有用，但没有任何保证；甚至没有对适销性或特定用途适
用性的暗示保证。有关更多详细信息，请参阅 GNU Affero 通用公共许可证。

您应该已经随本程序收到了 GNU Affero 通用公共许可证的副本。如果没有，请参阅
<https://www.gnu.org/licenses/>。

---

本软件包含以下第三方开源组件：

1. Winlator - Copyright (c) 2023 BrunoSX - MIT License
2. Box86 - Copyright (C) ptitSeb - MIT License
3. Box64 - Copyright (C) ptitSeb - MIT License
4. Wine - Copyright (C) Wine Project authors - LGPL-2.1+
5. Proton - Copyright (C) Valve Corporation - BSD-3-Clause / LGPL-2.1+
6. PRoot - Copyright (C) STMicroelectronics - GPL-2.0+
7. Mesa 3D - Copyright (C) Mesa project contributors - MIT / SGI-B-2.0
8. DXVK - Copyright (C) Philip Rebohle - zlib License
9. VKD3D-Proton - Copyright (C) Hans-Kristian Arntzen - LGPL-2.1+ / MIT
10. FEX-Emu - Copyright (C) FEX-Emu contributors - MIT License
11. Xlorie - Copyright (C) Termux contributors - GPL-3.0
12. glibc - Copyright (C) Free Software Foundation - LGPL-2.1+
13. Ubuntu RootFS - Copyright (C) Canonical Ltd. - Various licenses

各组件的完整许可证文本可在应用的"关于"页面或 assets/licenses/ 目录中找到。
各组件的源代码可从其上游仓库获取，具体链接见 THIRD_PARTY_NOTICES.md。
```

### 9.7 常见合规陷阱与规避

| 陷阱 | 后果 | 规避方式 |
|------|------|----------|
| 将 PRoot 代码编译进 JNI 库 | 整个 APK 必须以 GPL-2.0+ 开源 | 通过 `Runtime.exec()` 进程调用 |
| 将 Xlorie 静态链接进 APK | 整个 APK 必须以 GPL-3.0 开源 | 作为 rootfs 中的独立二进制 |
| 修改 Wine 但不开源补丁 | 违反 LGPL，授权终止 | 补丁放入 `patches/` 目录以 LGPL 发布 |
| 删除 MIT/BSD 组件的版权声明 | 违反许可证，授权终止 | NOTICE 文件 + assets/licenses/ 完整保留 |
| 使用 GPL-2.0-only 代码 | 与 AGPL 不兼容，无法合并 | 不使用任何 GPL-2.0-only 组件 |
| 未来增加闭源云服务 | 违反 AGPL 第 13 条 | 云服务端代码也必须以 AGPL 开源 |
| 不提供源码获取途径 | 违反 AGPL 第 6 条 | GitHub 公开仓库 + APK 内链接 |

---

*文档结束*
