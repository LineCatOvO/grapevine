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

## 10. 组件本地动态构建可行性分析

### 10.1 问题定义

Grapevine 的技术灵活性取决于能否快速适配上游组件的新版本。当前方案依赖预编译二进制分发（从镜像源下载），存在以下限制：

- 新版本发布后需要等待社区或 Grapevine 团队提供预编译包
- 无法针对特定设备/内核进行优化编译
- 无法自由应用自定义补丁
- 对上游发布节奏缺乏主动权

如果 Grapevine 能在用户设备上从源码动态构建最新版本，将彻底解决这些问题。

### 10.2 各组件构建复杂度分级

根据构建依赖、构建时间、交叉编译难度三个维度，将 Grapevine 依赖的组件分为四个等级：

#### Level 0：可在 Android 设备本地构建（可行）

| 组件 | 构建系统 | 依赖 | 典型构建时间 (4核 ARM64) | 产物大小 | 本地构建难度 |
|------|---------|------|:-:|:-:|:-:|
| Box64 | CMake + Make | gcc/cmake, 无外部依赖 | 5-15 分钟 | ~5 MB | 🟢 极低 |
| Box86 | CMake + Make | gcc/cmake, 无外部依赖 | 5-15 分钟 | ~4 MB | 🟢 极低 |

**Box64/Box86 是唯一真正可以在 Android 设备本地构建的组件**。它们：
- 只需 cmake + gcc，无外部库依赖
- 代码量小（~50K 行 C），编译快
- 支持按设备 SoC 选择优化目标（SD845/RK3588/Generic 等）
- 已有 Termux 包构建脚本可直接使用

#### Level 1：需要交叉编译环境（中等难度）

| 组件 | 构建系统 | 依赖 | 典型构建时间 (x86_64 主机) | 产物大小 | 本地构建难度 |
|------|---------|------|:-:|:-:|:-:|
| DXVK | Meson + Ninja | MinGW-w64, glslang | 5-10 分钟 | ~15 MB | 🟡 中 |
| VKD3D-Proton | Meson + Ninja | MinGW-w64, glslang, SPIRV-Headers | 10-20 分钟 | ~10 MB | 🟡 中 |

**特点**：
- DXVK 和 VKD3D-Proton 编译为 Windows DLL（使用 MinGW 交叉编译），不依赖目标平台
- 构建依赖相对简单（MinGW-w64 + Meson + glslang）
- 但在 Android 上运行 MinGW 交叉编译工具链需要完整的 glibc 环境
- 理论上可在 PRoot rootfs 中构建，但性能损耗严重

#### Level 2：需要复杂交叉编译环境（高难度）

| 组件 | 构建系统 | 依赖 | 典型构建时间 (x86_64 主机) | 产物大小 | 本地构建难度 |
|------|---------|------|:-:|:-:|:-:|
| Wine | Autoconf + Make | 100+ 依赖库 | 30-60 分钟 | ~100 MB | 🔴 高 |
| PRoot | Make | gcc, libtalloc | 2-5 分钟 | ~1 MB | 🟡 中 |

**Wine 构建的挑战**：
- 依赖 100+ 个开发库（freetype, libpng, libjpeg, gstreamer, v4l2, cups, etc.）
- 需要先构建原生 wine-tools，再交叉编译目标架构
- GameNative 的 Android 构建流程需要：Android NDK r27d + LLVM MinGW + termuxfs
- 构建过程需要 ~40 分钟（GitHub Actions aarch64 runner，含缓存）
- 在 Android 设备上构建需要完整的编译工具链和所有 -dev 包

#### Level 3：需要容器化构建环境（极高难度）

| 组件 | 构建系统 | 依赖 | 典型构建时间 (x86_64 主机) | 产物大小 | 本地构建难度 |
|------|---------|------|:-:|:-:|:-:|
| Proton | Docker/Podman + Make | Proton SDK 容器, Wine, DXVK, VKD3D, FAudio, gstreamer, etc. | 1-3 小时 | ~300 MB | 🔴🔴 极高 |

**Proton 构建的挑战**：
- **强制要求 Docker/Podman**：Proton 设计为在 Proton SDK 容器内构建，无法脱离容器
- **NixOS 社区的经验**：NixOS 社区在 2025 年 5 月提出将 Proton 打包为从源码构建的请求（nixpkgs#409340），至今未能实现，核心障碍正是容器化构建系统
- **GameNative 的解决方案**：GameNative/proton-wine 项目通过 GitHub Actions 实现了 Proton Wine 的 Android 构建，但这是在 x86_64 云端 runner 上交叉编译完成的
- **构建步骤**：
  1. 下载 termuxfs（~200MB）
  2. 安装 Android NDK r27d（~1.5GB）
  3. 安装 LLVM MinGW 工具链（~500MB）
  4. 构建 wine-tools（step 0）
  5. 构建 sysvshm 库（aarch64）
  6. 配置并编译 Proton Wine（~23 分钟在云端）
  7. 打包为 WCP 格式（~14 分钟）
- **总构建时间**：~40 分钟（GitHub Actions aarch64 runner，含缓存命中）
- **Android 设备上完全不可行**：Docker/Podman 无法在非 root Android 上运行

### 10.3 Android 设备本地构建的硬件限制

| 限制 | 典型旗舰手机 | 构建需求 | 差距 |
|------|:-:|:-:|:-:|
| CPU | 8 核 ARM64 (3GHz) | 4+ 核 x86_64 (3GHz+) | 指令集不同，PRoot 翻译损耗 30-50% |
| RAM | 8-16 GB | 16+ GB (Wine/Proton) | 不足，链接阶段 OOM 风险高 |
| 存储 | 128-512 GB UFS | 10+ GB 临时空间 | 足够 |
| 散热 | 被动散热 | 持续高负载 30+ 分钟 | 严重降频，构建时间翻倍 |
| 电池 | 4000-5000 mAh | 持续高负载 30+ 分钟 | 消耗 30-50% 电量 |

**结论**：即使在旗舰手机上，Wine/Proton 的本地构建也因 RAM 不足、散热降频、Docker 不可用等原因**不可行**。Box64/Box86 是唯一可在设备本地构建的组件。

### 10.4 三种组件更新策略对比

#### 策略 A：纯预编译分发（当前方案）

```
上游发布新版本 → Grapevine 团队构建 → 上传到镜像源 → 用户下载更新
```

| 优点 | 缺点 |
|------|------|
| 用户体验好（下载即用） | 新版本延迟取决于团队构建速度 |
| 无需用户设备编译 | 无法针对特定设备优化 |
| 构建质量可控 | 用户无法自定义补丁 |

#### 策略 B：CI/CD 云端自动构建

```
上游发布新版本 → GitHub Actions 自动触发构建 → 构建产物发布到 Release → 用户下载更新
```

| 优点 | 缺点 |
|------|------|
| 新版本延迟极短（自动触发） | GitHub Actions 免费额度有限 |
| 构建环境标准化 | 需要为每个组件维护 workflow |
| 可同时构建多架构 | 依赖 GitHub 基础设施 |
| 可自动测试构建产物 | 大型构建（Proton）消耗大量 CI 时间 |

#### 策略 C：混合策略（推荐）

```
组件分类:
  Level 0 (Box64/Box86) → 支持本地构建 + 预编译包
  Level 1-2 (DXVK/Wine) → CI/CD 云端自动构建 + 预编译包
  Level 3 (Proton) → CI/CD 云端自动构建 + 预编译包
  自定义补丁 → 用户提交 → CI/CD 构建 → 下载
```

### 10.5 推荐方案：CI/CD 云端自动构建 + 可选本地构建

#### 10.5.1 核心架构

```
┌─────────────────────────────────────────────────────────┐
│                    Grapevine 组件仓库                      │
│  (GitHub Repository + GitHub Actions + GitHub Releases)  │
│                                                          │
│  ┌──────────┐  触发  ┌──────────────┐  产物  ┌────────┐ │
│  │ 上游      │──────→│ GitHub Actions│──────→│ Release│ │
│  │ 新版本    │       │ 自动构建      │       │ 页面   │ │
│  └──────────┘       └──────────────┘       └────┬───┘ │
│       ↑                   ↑                      │      │
│       │              ┌────┴────┐                │      │
│  定时检查/           │ 手动触发 │                │      │
│  webhook             │ (自定义  │                │      │
│                      │  补丁)  │                │      │
│                      └─────────┘                │      │
└──────────────────────────────────────────────────┼──────┘
                                                   │
                                                   ↓
┌──────────────────────────────────────────────────────────┐
│                    Grapevine APK                          │
│                                                          │
│  ┌──────────────────┐  下载  ┌──────────────────────┐   │
│  │ 组件管理器 GUI    │←──────│ 镜像源 / GitHub Release│   │
│  │                  │       └──────────────────────┘   │
│  │ • 查看可用版本    │                                    │
│  │ • 一键更新组件    │       ┌──────────────────────┐   │
│  │ • 切换 Wine/Proton│←─────│ 本地构建 (Box64 only) │   │
│  │ • 应用自定义补丁  │       └──────────────────────┘   │
│  └──────────────────┘                                    │
└──────────────────────────────────────────────────────────┘
```

#### 10.5.2 CI/CD 构建流水线设计

**Wine 构建流水线**（参考 GameNative/proton-wine）：

```yaml
# .github/workflows/build-wine.yml
name: Build Wine
on:
  workflow_dispatch:
    inputs:
      wine_version:
        description: 'Wine version (e.g. 9.4)'
        required: true
      wine_source:
        description: 'Source (kron4ek/proton/valve)'
        default: 'kron4ek'
  schedule:
    - cron: '0 0 * * 1'  # 每周一检查新版本

jobs:
  build:
    strategy:
      matrix:
        arch: [x86_64, aarch64]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up build environment
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential git wget curl \
            flex bison gettext autoconf automake libtool pkg-config \
            mingw-w64 gcc-multilib g++-multilib
      - name: Download termuxfs
        run: |
          wget https://github.com/GameNative/termux-on-gha/releases/download/build-20260218/termuxfs-${{ matrix.arch }}.tar.gz
          tar xf termuxfs-${{ matrix.arch }}.tar.gz
      - name: Cache Android NDK
        uses: actions/cache@v4
        with:
          key: android-ndk-r27d
          path: ~/android-ndk-r27d
      - name: Build Wine
        run: ./build-scripts/build-step-${{ matrix.arch }}.sh
      - name: Package and Release
        uses: softprops/action-gh-release@v2
        with:
          tag: wine-${{ inputs.wine_version }}
          files: |
            wine-*.wcp
            wine-*.wcp.xz
```

**Proton 构建流水线**（参考 GameNative/proton-wine PR#15）：

```yaml
# .github/workflows/build-proton.yml
name: Build Proton Wine
on:
  workflow_dispatch:
    inputs:
      proton_branch:
        description: 'Proton branch (e.g. proton_11.0)'
        default: 'proton_11.0'
  schedule:
    - cron: '0 0 * * 1'  # 每周一检查新版本

jobs:
  build:
    strategy:
      matrix:
        arch: [x86_64, aarch64]
        sdk: [28, 35]  # Android SDK 28 和 35
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: GameNative/proton-wine
          ref: ${{ inputs.proton_branch }}
          submodules: recursive
      - name: Build Proton Wine
        run: |
          # 参考 GameNative/proton-wine 的构建流程
          # 总构建时间: ~40 分钟 (aarch64, 含缓存)
      - name: Package and Release
        uses: softprops/action-gh-release@v2
        with:
          tag: proton-${{ inputs.proton_branch }}-sdk${{ matrix.sdk }}
          files: |
            proton-*.wcp
            proton-wine-*.wcp.xz
```

**Box64 本地构建支持**（在 Grapevine APK 内）：

```bash
# grapevine-core/lib/component/build-box64.sh
build_box64_local() {
    local version="$1"
    local target="$2"  # SD845/RK3588/Generic
    local build_dir="${GRAPEVINE_HOME}/build/box64-${version}"

    if ! command -v cmake &>/dev/null || ! command -v gcc &>/dev/null; then
        print_error "构建 Box64 需要 cmake 和 gcc，请先安装"
        print_hint "运行: grapevine component install-build-deps"
        return 1
    fi

    git clone --depth 1 --branch "v${version}" \
        https://github.com/ptitSeb/box64.git "${build_dir}/src"

    mkdir -p "${build_dir}/build" && cd "${build_dir}/build"
    cmake ../src -D${target}=1 -DCMAKE_BUILD_TYPE=RelWithDebInfo \
        -DCMAKE_INSTALL_PREFIX="${GRAPEVINE_HOME}/components/box64/${version}"
    make -j"$(nproc)"
    make install

    grapevine component register box64 "${version}" "${GRAPEVINE_HOME}/components/box64/${version}"
}
```

#### 10.5.3 组件版本自动跟踪

```yaml
# .github/workflows/track-upstream.yml
name: Track Upstream Releases
on:
  schedule:
    - cron: '0 6 * * *'  # 每天 06:00 UTC 检查
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    outputs:
      wine_new: ${{ steps.wine.outputs.new }}
      box64_new: ${{ steps.box64.outputs.new }}
      proton_new: ${{ steps.proton.outputs.new }}
    steps:
      - name: Check Wine releases
        id: wine
        run: |
          latest=$(curl -s https://api.github.com/repos/Kron4ek/Wine-Builds/releases/latest | jq -r .tag_name)
          current=$(cat registry/wine.version)
          if [ "$latest" != "$current" ]; then
            echo "new=true" >> $GITHUB_OUTPUT
            echo "version=$latest" >> $GITHUB_OUTPUT
          fi
      - name: Check Box64 releases
        id: box64
        run: |
          latest=$(curl -s https://api.github.com/repos/ptitSeb/box64/releases/latest | jq -r .tag_name)
          current=$(cat registry/box64.version)
          if [ "$latest" != "$current" ]; then
            echo "new=true" >> $GITHUB_OUTPUT
            echo "version=$latest" >> $GITHUB_OUTPUT
          fi
      - name: Check Proton releases
        id: proton
        run: |
          latest=$(curl -s https://api.github.com/repos/ValveSoftware/Proton/releases/latest | jq -r .tag_name)
          current=$(cat registry/proton.version)
          if [ "$latest" != "$current" ]; then
            echo "new=true" >> $GITHUB_OUTPUT
            echo "version=$latest" >> $GITHUB_OUTPUT
          fi

  trigger-builds:
    needs: check
    if: needs.check.outputs.wine_new == 'true' || needs.check.outputs.box64_new == 'true' || needs.check.outputs.proton_new == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Wine build
        if: needs.check.outputs.wine_new == 'true'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.actions.createWorkflowDispatch({
              owner: context.repo.owner,
              repo: context.repo.repo,
              workflow_id: 'build-wine.yml',
              ref: 'main',
              inputs: { wine_version: '${{ needs.check.outputs.wine_version }}' }
            })
      - name: Trigger Box64 build
        if: needs.check.outputs.box64_new == 'true'
        run: echo "Trigger Box64 build workflow"
      - name: Trigger Proton build
        if: needs.check.outputs.proton_new == 'true'
        run: echo "Trigger Proton build workflow"
```

### 10.6 技术灵活性评估

| 灵活性维度 | 纯预编译分发 | CI/CD + 可选本地构建 | 纯本地构建 |
|----------|:-:|:-:|:-:|
| 新版本跟进速度 | ⚠️ 天-周级 | ✅ 小时级 | ❌ 不可行 (Wine/Proton) |
| 自定义补丁 | ❌ 不支持 | ✅ 提交 PR → CI 构建 | ⚠️ 仅 Box64 |
| 设备特定优化 | ❌ 通用构建 | ⚠️ 按架构构建 | ✅ Box64 按 SoC 优化 |
| 离线可用性 | ✅ 下载后离线 | ✅ 下载后离线 | ✅ 完全离线 |
| 对上游依赖 | ⚠️ 依赖社区预编译包 | ✅ 直接从源码构建 | ❌ 不可行 (Wine/Proton) |
| 维护成本 | 🟡 中 | 🟡 中 | 🟢 低 (仅 Box64) |
| 用户体验 | ✅ 下载即用 | ✅ 下载即用 + 高级选项 | ❌ 构建耗时且易失败 |

### 10.7 结论

**Proton/Wine 的本地动态构建在 Android 设备上不可行**，核心障碍：

1. **Proton 强制要求 Docker/Podman**，Android 非 root 环境无法运行
2. **Wine 构建需要 100+ 依赖库**，Android 设备 RAM 不足以支撑链接阶段
3. **构建时间过长**（Proton 1-3 小时，Wine 30-60 分钟），手机散热无法承受

**但 CI/CD 云端自动构建可以实现同等灵活性**：

1. 上游新版本发布 → 自动触发构建 → 小时级可用
2. 用户提交自定义补丁 → CI 构建 → 下载安装
3. 多架构并行构建 → x86_64 + aarch64 同时产出
4. 构建缓存策略 → 增量构建加速

**Box64/Box86 是唯一支持本地构建的组件**，可在 Grapevine APK 内提供"从源码构建"选项，按设备 SoC 优化性能。

**最终推荐**：采用混合策略（策略 C），以 CI/CD 云端自动构建为主，Box64 本地构建为辅，实现最大技术灵活性。

---

## 11. 官方构建产物可访问性与直接下载更新可行性

### 11.1 各组件官方构建产物清单

| 组件 | 官方构建来源 | 格式 | Android ARM64 可用 | 公开访问 | 更新频率 |
|------|------------|------|:-:|:-:|:-:|
| **Wine** | [Kron4ek/Wine-Builds](https://github.com/Kron4ek/Wine-Builds/releases) | tar.xz | ⚠️ 仅 x86_64/amd64 | ✅ GitHub Releases | 每 2 周 |
| **Wine (Android)** | [GameNative/proton-wine](https://github.com/GameNative/proton-wine/releases) | .wcp / .wcp.xz | ✅ arm64ec + x86_64 | ✅ GitHub Releases | 跟随 Proton 版本 |
| **Proton** | [ValveSoftware/Proton](https://github.com/ValveSoftware/Proton/releases) | 整体 tar | ❌ 仅 x86_64 | ✅ GitHub Releases | 跟随 Steam 更新 |
| **Proton (Android)** | [GameNative/proton-wine](https://github.com/GameNative/proton-wine/releases) | .wcp / .wcp.xz | ✅ arm64ec + x86_64 | ✅ GitHub Releases | 社区驱动 |
| **Box64** | [ptitSeb/box64/releases](https://github.com/ptitSeb/box64/releases) | 源码 tar.gz | ⚠️ 需自行编译 | ✅ GitHub Releases | ~每 2 月 |
| **Box64 (预编译)** | [ryanfortner/box64-debs](https://ryanfortner.github.io/box64-debs/) | .deb | ✅ box64-android | ✅ APT 仓库 + GitHub | 每日自动构建 |
| **Box64 (每日构建)** | [fabricatorsltd/box64](https://github.com/fabricatorsltd/box64/releases) | .deb | ✅ arm64 | ✅ GitHub Releases | 每日自动构建 |
| **Box86** | [ptitSeb/box86/releases](https://github.com/ptitSeb/box86/releases) | 源码 tar.gz | ⚠️ 需自行编译 | ✅ GitHub Releases | ~每 2 月 |
| **Box86 (预编译)** | [ryanfortner/box86-debs](https://github.com/ryanfortner/box86-debs) | .deb | ✅ box86-android | ✅ APT 仓库 + GitHub | 每日自动构建 |
| **DXVK** | [doitsujin/dxvk/releases](https://github.com/doitsujin/dxvk/releases) | tar.gz (x32+x64 DLL) | ✅ 平台无关 DLL | ✅ GitHub Releases | ~每月 |
| **DXVK (WCP)** | [Winlator-WCP-Collections](https://github.com/Nick088Official/Winlator-WCP-Collections) | .wcp | ✅ Winlator 格式 | ✅ GitHub Releases | 跟随上游 |
| **DXVK (Nightly WCP)** | [Xnick417x/Winlator-Bionic-Nightly-wcp](https://github.com/Xnick417x/Winlator-Bionic-Nightly-wcp) | .wcp | ✅ Winlator 格式 | ✅ content.json API | 每日自动构建 |
| **VKD3D-Proton** | [HansKristian-Work/vkd3d-proton/releases](https://github.com/HansKristian-Work/vkd3d-proton/releases) | tar.gz (x32+x64 DLL) | ✅ 平台无关 DLL | ✅ GitHub Releases | ~每 2 月 |
| **VKD3D-Proton (WCP)** | [Winlator-WCP-Collections](https://github.com/Nick088Official/Winlator-WCP-Collections) | .wcp | ✅ Winlator 格式 | ✅ GitHub Releases | 跟随上游 |
| **Mesa (Turnip/VirGL/Zink)** | [Xnick417x/Winlator-Bionic-Nightly-wcp](https://github.com/Xnick417x/Winlator-Bionic-Nightly-wcp) | .wcp | ✅ arm64 | ✅ content.json API | 每日自动构建 |
| **FEX-Emu** | [Winlator-WCP-Collections](https://github.com/Nick088Official/Winlator-WCP-Collections) | .wcp | ✅ Winlator 格式 | ✅ GitHub Releases | 跟随上游 |

### 11.2 关键发现

#### 11.2.1 大部分组件有官方公开构建，可直接下载

**结论：是的，几乎所有组件都有官方或社区维护的公开构建产物，可以直接下载到项目内更新。**

但需要注意以下分类：

**可直接使用的构建（Android ARM64 原生）**：

| 组件 | 来源 | 格式 | 直接可用 |
|------|------|------|:-:|
| Wine/Proton (Android) | GameNative/proton-wine | .wcp | ✅ |
| Box64 | ryanfortner/box64-debs | .deb | ✅ (需解压) |
| Box86 | ryanfortner/box86-debs | .deb | ✅ (需解压) |
| DXVK | doitsujin/dxvk | tar.gz | ✅ (DLL 平台无关) |
| VKD3D-Proton | HansKristian-Work/vkd3d-proton | tar.gz | ✅ (DLL 平台无关) |
| Mesa 驱动 | Xnick417x Nightly | .wcp | ✅ |

**不可直接使用的构建（需转换或重新打包）**：

| 组件 | 来源 | 问题 | 解决方案 |
|------|------|------|----------|
| Wine (Kron4ek) | Kron4ek/Wine-Builds | 仅 x86_64，非 Android 构建 | 使用 GameNative 版本 |
| Proton (Valve) | ValveSoftware/Proton | 仅 x86_64，含 Steam 依赖 | 使用 GameNative 版本 |
| Box64 (ptitSeb) | ptitSeb/box64 | 仅源码 | 使用 ryanfortner 预编译版 |

#### 11.2.2 WCP 格式是 Android 生态的事实标准

Winlator CMOD 社区已建立了成熟的 WCP (Winlator Content Package) 分发生态：

- **WCP 格式**：XZ/Zstd 压缩包 + `profile.json` 清单文件
- **content.json API**：Winlator 通过远程 JSON 索引发现和下载可用组件
- **Winlator-WCP-Toolkit**：自动化工具，可将上游构建转换为 WCP 格式
- **Winlator-WCP-Collections**：预转换的 WCP 组件集合
- **Xnick417x Nightly**：每日自动构建 DXVK/Mesa/Box64 等组件的 WCP 包

**Grapevine 可以直接复用这个生态**，无需从零构建组件分发系统。

#### 11.2.3 GameNative 是 Wine/Proton Android 构建的权威来源

GameNative/proton-wine 是目前唯一提供 Android ARM64 构建的 Wine/Proton 仓库：

- 支持 Proton 11.0 ARM64EC + x86_64
- 同时输出 Proton 类型 (.wcp) 和 Wine 类型 (.wcp.xz)
- 支持 Android SDK 28 和 SDK 35
- 通过 GitHub Actions 自动构建和发布
- 已被 Winlator CMOD、Ludashi 等项目采用

### 11.3 Grapevine 实现直接下载更新的技术方案

#### 11.3.1 方案概述

Grapevine 可以实现与 Winlator CMOD 类似的组件在线更新功能，核心流程：

```
用户点击"检查更新"
  → Grapevine 下载 content.json 索引
  → 解析可用组件列表和版本
  → 与本地已安装版本对比
  → 显示可更新组件列表
  → 用户选择更新
  → 下载 .wcp / tar.xz 组件包
  → 校验 SHA256
  → 解压到组件目录
  → 更新 registry.yml
  → 完成
```

#### 11.3.2 组件索引设计

Grapevine 应维护自己的组件索引（兼容 WCP 生态）：

```json
{
  "version": 2,
  "contents": [
    {
      "name": "Wine 11.5 (Staging)",
      "type": "wine",
      "version": "11.5",
      "source": "kron4ek",
      "arch": ["x86_64", "arm64ec"],
      "downloadUrl": "https://github.com/GameNative/proton-wine/releases/download/...",
      "sha256": "abc123...",
      "size": 104857600,
      "license": "LGPL-2.1+"
    },
    {
      "name": "Proton 11.0 Experimental",
      "type": "proton",
      "version": "11.0-1",
      "source": "gamenative",
      "arch": ["x86_64", "arm64ec"],
      "downloadUrl": "https://github.com/GameNative/proton-wine/releases/download/...",
      "sha256": "def456...",
      "size": 314572800,
      "license": "BSD-3-Clause / LGPL-2.1+"
    },
    {
      "name": "Box64 0.4.2",
      "type": "box64",
      "version": "0.4.2",
      "source": "ryanfortner",
      "arch": ["arm64"],
      "downloadUrl": "https://github.com/ryanfortner/box64-debs/raw/master/debian/box64-android_...",
      "sha256": "ghi789...",
      "size": 5242880,
      "license": "MIT"
    },
    {
      "name": "DXVK 2.4.1",
      "type": "dxvk",
      "version": "2.4.1",
      "source": "doitsujin",
      "arch": ["any"],
      "downloadUrl": "https://github.com/doitsujin/dxvk/releases/download/v2.4.1/dxvk-2.4.1.tar.gz",
      "sha256": "jkl012...",
      "size": 15728640,
      "license": "zlib"
    },
    {
      "name": "VKD3D-Proton 2.14.1",
      "type": "vkd3d",
      "version": "2.14.1",
      "source": "hansKristian",
      "arch": ["any"],
      "downloadUrl": "https://github.com/HansKristian-Work/vkd3d-proton/releases/download/v2.14.1/vkd3d-proton-2.14.1.tar.gz",
      "sha256": "mno345...",
      "size": 10485760,
      "license": "LGPL-2.1+ / MIT"
    },
    {
      "name": "Mesa Turnip 25.1.0",
      "type": "mesa",
      "variant": "turnip",
      "version": "25.1.0",
      "source": "xnick417x",
      "arch": ["arm64"],
      "downloadUrl": "https://github.com/Xnick417x/Winlator-Bionic-Nightly-wcp/releases/download/...",
      "sha256": "pqr678...",
      "size": 20971520,
      "license": "MIT / SGI-B-2.0"
    }
  ]
}
```

#### 11.3.3 多源下载策略

Grapevine 应支持从多个来源下载同一组件，以提高可用性：

```yaml
# registry.yml 中的下载源配置
mirrors:
  wine:
    - name: "GameNative (Primary)"
      url: "https://github.com/GameNative/proton-wine/releases"
    - name: "Kron4ek (Fallback)"
      url: "https://github.com/Kron4ek/Wine-Builds/releases"
      note: "仅 x86_64，需 Box64 翻译"
  box64:
    - name: "ryanfortner (Primary)"
      url: "https://ryanfortner.github.io/box64-debs/"
    - name: "fabricatorsltd (Nightly)"
      url: "https://github.com/fabricatorsltd/box64/releases"
  dxvk:
    - name: "doitsujin (Official)"
      url: "https://github.com/doitsujin/dxvk/releases"
    - name: "Xnick417x (WCP Nightly)"
      url: "https://github.com/Xnick417x/Winlator-Bionic-Nightly-wcp/releases"
```

#### 11.3.4 格式适配层

不同来源的组件使用不同的打包格式，Grapevine 需要一个格式适配层：

| 来源格式 | 适配操作 | 复杂度 |
|---------|---------|:-:|
| .wcp (Winlator) | 解压 XZ/Zstd → 读取 profile.json → 提取 bin/lib/share | 🟢 低 |
| .wcp.xz (Winlator Wine) | 解压 XZ → 同上 | 🟢 低 |
| tar.xz (Kron4ek Wine) | 解压 → 直接使用 bin/lib/share 结构 | 🟢 低 |
| tar.gz (DXVK/VKD3D) | 解压 → 复制 x32/x64 DLL 到 Wine prefix | 🟢 低 |
| .deb (Box64) | 解压 ar → 解压 data.tar → 提取 /usr/bin/box64 | 🟡 中 |

#### 11.3.5 实现可行性评估

| 功能 | 技术可行性 | 实现难度 | 依赖 |
|------|:-:|:-:|------|
| 从 GitHub Releases 下载组件 | ✅ | 🟢 低 | HTTP 客户端 |
| 解析 content.json 索引 | ✅ | 🟢 低 | JSON 解析 |
| 校验 SHA256 | ✅ | 🟢 低 | 标准库 |
| 解压 .wcp / tar.xz / tar.gz | ✅ | 🟢 低 | libarchive / zlib / xz |
| 解压 .deb | ✅ | 🟡 中 | ar 解压 + tar 解压 |
| 版本对比和更新提示 | ✅ | 🟢 低 | 语义版本解析 |
| 多源下载和故障转移 | ✅ | 🟡 中 | 下载队列管理 |
| 下载进度显示 | ✅ | 🟢 低 | Android DownloadManager |
| 断点续传 | ✅ | 🟡 中 | HTTP Range 请求 |
| 增量更新（仅下载差异） | ⚠️ | 🔴 高 | bsdiff/delta 方案 |

### 11.4 Winlator 已有的成熟实现可参考

Winlator CMOD 已经实现了完整的组件在线更新功能：

1. **Settings → Downloadable Content URL**：用户可自定义组件源 URL
2. **content.json API**：远程索引发现可用组件
3. **Driver Manager URL**：图形驱动（Turnip/VirGL/Zink）独立源
4. **一键安装 .wcp 组件**：下载 → 校验 → 解压 → 注册
5. **组件版本管理**：安装多个版本，按容器选择使用

**Grapevine 可以直接复用 WCP 生态的组件**，无需自己构建所有组件。只需要：

1. 实现 WCP 格式的解析和安装
2. 维护自己的 content.json 索引（或直接兼容 Winlator 的索引格式）
3. 在 APK 内提供组件管理器 GUI

### 11.5 结论

**所有核心组件都有官方或社区维护的公开构建产物，Grapevine 完全可以实现直接下载官方构建到项目内更新。**

关键要点：

1. **Wine/Proton Android 构建**：GameNative/proton-wine 提供公开的 .wcp 格式构建，可直接下载使用
2. **Box64/Box86 预编译**：ryanfortner 每日自动构建 Android 版本，.deb 格式可直接下载
3. **DXVK/VKD3D-Proton**：官方提供平台无关的 DLL 包，可直接下载
4. **Mesa 驱动**：Xnick417x Nightly 每日自动构建 WCP 格式
5. **WCP 生态**：Winlator CMOD 社区已建立成熟的组件分发生态，Grapevine 可直接复用

**Grapevine 不需要自己构建任何组件**，只需实现下载→校验→解压→注册的流程，即可实现组件在线更新。这与 Winlator CMOD 的实现方式完全一致，技术成熟度已验证。

**实现优先级**：

1. Phase 1：支持从 GameNative 下载 Wine/Proton .wcp 包
2. Phase 2：支持从 ryanfortner 下载 Box64 .deb 包
3. Phase 3：支持从官方下载 DXVK/VKD3D tar.gz 包
4. Phase 4：兼容 Winlator content.json 索引格式，复用 WCP 生态

---

## 12. Winlator CMOD 存在下的 Grapevine 差异化价值评估

### 12.1 Winlator CMOD 现状（截至 2026 年 5 月）

Winlator CMOD 是 Winlator 最活跃的社区 fork，由 coffincolors 维护，当前版本 v13.1.1。其功能已经非常完善：

| 功能领域 | Winlator CMOD 已有功能 | 成熟度 |
|---------|----------------------|:-:|
| **容器管理** | 创建/删除/克隆/导入/导出容器 | ✅ 成熟 |
| **组件在线更新** | content.json API + WCP 格式 + 可自定义源 URL | ✅ 成熟 |
| **图形驱动** | Turnip/VirGL/Zink + DXVK/VKD3D/D8VK 版本管理 | ✅ 成熟 |
| **音频** | ALSA-Reflector（自研，抗断连）+ PulseAudio | ✅ 成熟 |
| **输入控制** | SDL2 原生手柄（XInput/DInput）、4 人多人、陀螺仪、震动、Turbo/Macros | ✅ 成熟 |
| **屏幕效果** | FXAA/CRT/ToonShader/VkBasalt（per-shortcut） | ✅ 成熟 |
| **快捷方式管理** | 桌面快捷方式、收藏启动器、Big Picture Mode | ✅ 成熟 |
| **游戏统计** | 游戏时长追踪、启动次数统计 | ✅ 成熟 |
| **Wine/Proton** | Wine + Proton 双支持，Box64 版本切换 | ✅ 成熟 |
| **Bionic 原生库** | Pipetto-crypto 的 Bionic 原生库集成 | ✅ 成熟 |
| **前端集成** | 导出快捷方式到 Android 前端（Daijisho等） | ✅ 成熟 |
| **多实例** | 支持多 APK 实例并行安装 | ✅ 成熟 |

### 12.2 诚实对比：Grapevine 设计文档中的特性 vs Winlator CMOD

| Grapevine 功能 | Winlator CMOD 对应功能 | Grapevine 是否有差异化 |
|---------------|----------------------|:---:|
| F01 容器管理（create/start/stop/delete） | ✅ 已有，且 GUI 操作 | ❌ 无差异 |
| F01 容器克隆 | ✅ 已有（Clone Shortcuts between Containers） | ❌ 无差异 |
| F01 容器导入/导出 | ✅ 已有（Import/Export Containers） | ❌ 无差异 |
| F02 快照系统（增量快照/回滚） | ❌ **无** | ✅ **核心差异** |
| F03 模板系统（6 个预置模板） | ⚠️ 部分有（Container 预设配置，但非模板化） | ✅ **差异化** |
| F04 图形子系统（驱动选择/DXVK 管理） | ✅ 已有，且更成熟（含 VkBasalt per-shortcut） | ❌ 无差异 |
| F05 音频子系统 | ✅ 已有，且更先进（ALSA-Reflector） | ❌ 无差异，CMOD 更优 |
| F06 输入控制 | ✅ 已有，且远超设计（SDL2 原生手柄、4 人、陀螺仪） | ❌ 无差异，CMOD 远超 |
| F07 存储映射 | ✅ 已有（D: 盘映射） | ❌ 无差异 |
| F08 网络支持 | ✅ 已有 | ❌ 无差异 |
| F09 组件管理 | ✅ 已有（WCP 生态 + content.json） | ❌ 无差异，CMOD 生态更成熟 |
| F10 CLI 命令行 | ❌ **无**（纯 GUI 操作） | ✅ **核心差异** |
| F11 TUI 交互界面 | ❌ **无** | ✅ **差异化**（但价值有限） |
| F12 声明式配置（container.yml） | ❌ **无**（配置存储在 Java SharedPreferences） | ✅ **核心差异** |
| Proton 支持 | ✅ 已有（GameNative WCP） | ❌ 无差异 |
| 屏幕效果 | ✅ 已有（FXAA/CRT/ToonShader） | ❌ 无差异，CMOD 更优 |
| 游戏统计 | ✅ 已有 | ❌ 无差异 |

### 12.3 Grapevine 的三个真正差异化特性

经过诚实对比，Grapevine 只有 **3 个真正差异化的核心特性**：

#### 差异化 1：快照系统（F02）— Winlator CMOD 完全没有

Winlator CMOD 的容器没有快照/回滚能力。用户如果安装了一个导致容器损坏的软件，只能删除容器重建。Grapevine 的增量快照系统是**真正的杀手级功能**：

- 安装前创建快照 → 安装失败 → 一键回滚
- 尝试不同配置 → 保留多个快照点 → 自由切换
- 容器升级前快照 → 升级后出问题 → 立即恢复

**这是 Winlator CMOD 开发者自己承认的痛点**。CMOD v10 的 release notes 中明确说 "You'll get best results with new containers"，暗示旧容器升级容易出问题。CMOD 开发者在 2025 年 12 月的更新中也表示希望 "overhaul how Containers are created"，但至今未实现。

#### 差异化 2：声明式配置（F12）— Winlator CMOD 没有

Winlator CMOD 的配置存储在 Java SharedPreferences 中，是二进制格式，不可读、不可版本控制、不可分享。Grapevine 的 container.yml 声明式配置带来：

- **配置可读**：用户可以直接查看和编辑容器配置
- **配置可分享**：将 container.yml 分享给其他用户，一键复现环境
- **配置可版本控制**：将配置纳入 Git 管理，追踪变更历史
- **配置即代码**：通过 CLI 批量创建/修改容器，支持自动化
- **配置可迁移**：导出 container.yml 即可在新设备上重建环境

#### 差异化 3：CLI 命令行接口（F10）— Winlator CMOD 完全没有

Winlator CMOD 是纯 GUI 应用，没有命令行接口。Grapevine 的 CLI 带来：

- **自动化脚本**：批量创建容器、批量更新组件、定时备份快照
- **远程管理**：通过 adb shell 或 SSH 远程管理设备上的容器
- **CI/CD 集成**：在自动化测试中使用 CLI 创建/销毁容器
- **高级用户工作流**：快速操作无需打开 GUI，命令组合实现复杂逻辑
- **JSON 输出**：结构化数据输出，便于与其他工具集成

### 12.4 Grapevine 不具备差异化的领域（不应投入资源）

以下功能 Winlator CMOD 已经做得很好，Grapevine 不应重复造轮子：

| 功能 | 原因 | 建议 |
|------|------|------|
| 输入控制系统 | CMOD 的 SDL2 原生手柄远超 Grapevine 设计 | 直接复用 Winlator 的输入模块 |
| 音频系统 | CMOD 的 ALSA-Reflector 是自研创新 | 直接复用 Winlator 的音频模块 |
| 屏幕效果 | CMOD 的 FXAA/CRT/ToonShader/VkBasalt 已成熟 | 直接复用 |
| 组件在线更新 | CMOD 的 WCP 生态已成熟 | 兼容 WCP 格式，复用其生态 |
| 前端集成 | CMOD 已支持 Daijisho 等前端 | 直接复用 |

### 12.5 开发价值评估

#### 12.5.1 如果 Grapevine 只是"Winlator + 快照 + CLI + YAML 配置"

**价值判断：开发价值有限。**

理由：
1. Winlator CMOD 已经覆盖了 90% 的用户需求，且持续快速迭代
2. 快照、CLI、YAML 配置是高级用户需求，普通用户并不关心
3. Grapevine 需要从零构建整个 APK，而 CMOD 已经有 3 年的积累
4. CMOD 开发者已经意识到容器管理的痛点，可能在未来版本中加入类似功能

#### 12.5.2 如果 Grapevine 重新定位为"容器编排平台"

**价值判断：开发价值显著。**

Grapevine 不应该做"另一个 Winlator"，而应该做 **"Windows 兼容容器的 Docker"**——专注于容器编排、快照、模板、声明式配置，底层运行时复用 Winlator 的成熟模块。

核心定位转变：

```
Winlator CMOD = Windows 应用的运行器（关注"运行"）
Grapevine     = Windows 兼容容器的编排平台（关注"管理"）
```

类比：
- Winlator ≈ 虚拟机（运行一个 Windows 环境）
- Grapevine ≈ Docker（管理多个容器的生命周期、快照、模板、配置）

#### 12.5.3 推荐的产品定位

**Grapevine = Winlator 生态的容器编排层**

Grapevine 不是 Winlator 的替代品，而是 Winlator 的上层编排工具。具体来说：

1. **底层运行时**：直接复用 Winlator 的 Java/C 代码（MIT 许可证，合法合规）
2. **上层编排**：Grapevine 自研快照系统、声明式配置、CLI、模板系统
3. **兼容 WCP 生态**：直接使用 Winlator CMOD 的组件仓库和 WCP 包

这样的定位意味着：
- Grapevine 不需要重新实现图形/音频/输入等底层模块
- Grapevine 的核心开发精力集中在差异化特性上
- Grapevine 与 Winlator CMOD 是互补关系，不是竞争关系
- Grapevine 可以吸引 Winlator 的高级用户群体

### 12.6 修订后的开发策略

基于以上分析，修订 Grapevine 的开发策略：

#### 12.6.1 核心开发重点（自研）

| 优先级 | 功能 | 差异化价值 | 开发量 |
|--------|------|----------|--------|
| P0 | 增量快照系统 | ⭐⭐⭐ 杀手级功能 | 中 |
| P0 | 声明式配置 (container.yml) | ⭐⭐⭐ 核心架构 | 小 |
| P0 | CLI 命令行接口 | ⭐⭐⭐ 高级用户入口 | 中 |
| P1 | 模板系统 | ⭐⭐ 降低上手门槛 | 小 |
| P1 | 容器批量操作 | ⭐⭐ 编排能力 | 小 |
| P2 | 容器健康检查与自修复 | ⭐⭐ 运维能力 | 中 |
| P2 | 配置版本控制集成 | ⭐ Git 友好 | 小 |

#### 12.6.2 直接复用 Winlator 的模块（不自研）

| 模块 | 复用方式 | 维护策略 |
|------|---------|---------|
| 图形子系统 | Fork Winlator 代码 | 跟随上游更新 |
| 音频子系统 | Fork Winlator 代码 | 跟随上游更新 |
| 输入控制 | Fork Winlator 代码 | 跟随上游更新 |
| 组件管理 (WCP) | 兼容 WCP 格式 | 复用 CMOD 生态 |
| PRoot JNI | Fork Winlator 代码 | 跟随上游更新 |
| Xlorie | Fork Winlator 代码 | 跟随上游更新 |
| android_alsa | Fork Winlator 代码 | 跟随上游更新 |
| android_sysvshm | Fork Winlator 代码 | 跟随上游更新 |

#### 12.6.3 不做的功能

| 功能 | 原因 |
|------|------|
| 自研输入控制 | CMOD 的 SDL2 方案远超自研可行性 |
| 自研音频引擎 | CMOD 的 ALSA-Reflector 已验证 |
| 自研屏幕效果 | CMOD 的 VkBasalt 集成已成熟 |
| 自研组件构建系统 | WCP 生态已成熟，直接复用 |
| TUI 交互界面 | 作为独立 APK 已有原生 GUI，TUI 价值有限 |

### 12.7 风险评估

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|----------|
| Winlator CMOD 未来加入快照功能 | 中 | 高——丧失核心差异化 | 加快 MVP 开发，先发优势；快照的技术实现（增量存储、链式管理）有一定门槛 |
| Winlator CMOD 未来加入 CLI | 低 | 中 | CLI 与声明式配置深度绑定，CMOD 的 Java 架构不易加入 |
| Winlator 上游 License 变更 | 低 | 高 | 保持代码解耦，核心逻辑独立于 Winlator 代码 |
| 社区分裂（Grapevine vs CMOD） | 中 | 中 | 定位为互补而非竞争，强调编排能力 |
| 开发资源不足 | 高 | 高 | 聚焦核心差异化，复用 Winlator 底层模块 |

### 12.8 最终结论

**Grapevine 的开发价值取决于定位选择：**

1. **如果定位为"Winlator 替代品"** → ❌ 开发价值低。Winlator CMOD 已经覆盖了 90% 的用户需求，且持续快速迭代。重复造轮子没有意义。

2. **如果定位为"Winlator 生态的容器编排平台"** → ✅ 开发价值显著。快照系统、声明式配置、CLI 是 Winlator 生态中缺失的关键能力，且与 CMOD 形成互补而非竞争。

**推荐定位：Grapevine = Windows 兼容容器的编排平台（Docker for Windows-on-Android）**

核心价值主张：
- **快照即安全**：任何操作前创建快照，一键回滚，永不丢失环境
- **配置即代码**：YAML 声明式配置，可读、可分享、可版本控制
- **命令即自动化**：CLI 接口支持脚本化、批量化、远程化管理

---

*文档结束*
