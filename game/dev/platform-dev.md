# 各操作系统平台：代表游戏 & 开发套件

> 最后更新：2026-04-19（增补：开发语言）

---

## 🪟 Windows

### 代表游戏
| 游戏 | 类型 | 引擎 |
|---|---|---|
| 《艾尔登法环》 | 动作 RPG | 自研引擎 |
| 《CS2》 | FPS | Source 2 |
| 《赛博朋克 2077》 | 开放世界 RPG | REDengine 4 |
| 《我的世界》| 沙盒 | 自研（Java / Bedrock） |
| 《星露谷物语》 | 模拟经营 | XNA / MonoGame |
| 《坦克大战（复刻）》 | 2D 射击 | Godot ← 本项目 |

### 开发平台与套件
| 工具 / 套件 | 用途 | 语言 |
|---|---|---|
| **Godot 4** | 2D / 3D 游戏引擎 | GDScript / C# |
| **Unity** | 跨平台游戏引擎 | C# |
| **Unreal Engine 5** | 3A 级游戏引擎 | C++ / Blueprints |
| **Visual Studio 2022** | IDE，C++ / C# 开发 | C++ / C# |
| **DirectX 12** | 微软官方图形 API | C++ |
| **Vulkan SDK** | 跨平台高性能图形 API | C++ |
| **Steam SDK / Steamworks** | 发行、成就、联机 | C++ / C# |
| **FMOD / Wwise** | 游戏音频中间件 | — |

### 🗣️ Windows 游戏开发主流语言

| 语言 | 用途 | 典型场景 |
|---|---|---|
| **C++** | 底层引擎、高性能逻辑 | UE5 游戏逻辑、DirectX 渲染 |
| **C#** | 游戏逻辑脚本 | Unity 脚本、Godot 脚本 |
| **GDScript** | Godot 专用脚本 | 类 Python，上手快 |
| **HLSL** | 着色器语言 | DirectX 图形渲染 |
| **Lua** | 轻量嵌入式脚本 | 游戏内 MOD / 配置脚本 |
| **Python** | 工具链 / 自动化 | 构建脚本、资产处理管线 |

---

## 🐧 Linux

### 代表游戏
| 游戏 | 类型 | 备注 |
|---|---|---|
| 《我的世界》 | 沙盒 | 官方原生支持 |
| 《CS2》 | FPS | Steam / Proton 原生 |
| 《Hollow Knight》 | 平台动作 | Unity 原生 Linux 版 |
| 《Celeste》 | 平台跳跃 | FNA 原生 Linux |
| 《Factorio》 | 工厂模拟 | 原生 Linux，极佳优化 |
| 《Among Us》 | 社交推理 | Unity 跨平台 |

### 开发平台与套件
| 工具 / 套件 | 用途 | 说明 |
|---|---|---|
| **Godot 4** | 2D / 3D 游戏引擎 | 原生支持 Linux 导出 |
| **Unity（Linux Editor）** | 跨平台引擎 | 支持 Linux 编辑器与导出 |
| **SDL2** | 窗口、输入、音频底层库 | C / C++，轻量跨平台 |
| **OpenGL / Vulkan** | 图形 API | Linux 主流图形接口 |
| **GCC / Clang** | 编译器 | Linux 原生 C++ 工具链 |
| **CMake** | 构建系统 | C++ 项目跨平台构建 |
| **Steam for Linux** | 发行平台 | 含 Proton 兼容层 |
| **Proton / Wine** | Windows 游戏兼容运行 | 非原生游戏可借此运行 |

### 🗣️ Linux 游戏开发主流语言

| 语言 | 用途 | 典型场景 |
|---|---|---|
| **C++** | 底层引擎、高性能逻辑 | SDL2 游戏、Vulkan 渲染 |
| **C** | 底层系统调用 | 底层平台适配、嵌入式 |
| **C#** | 游戏脚本 | Unity / Godot on Linux |
| **GDScript** | Godot 专用脚本 | 快速开发 2D / 3D |
| **GLSL** | 着色器语言 | OpenGL / Vulkan 渲染 |
| **Python** | 工具链 / 自动化 | 构建脚本、资产处理 |
| **Rust** | 安全高性能系统语言 | 新兴游戏引擎（Bevy 等） |

---

## 🍎 iOS（iPhone / iPad）

### 代表游戏
| 游戏 | 类型 | 引擎 |
|---|---|---|
| 《原神》 | 开放世界 RPG | 自研引擎（miHoYo） |
| 《王者荣耀》 | MOBA | 自研引擎（腾讯） |
| 《纪念碑谷》 | 解谜 | Unity |
| 《Alto's Odyssey》 | 跑酷 | 自研（Swift） |
| 《Clash of Clans》 | 策略塔防 | 自研（Supercell） |
| 《地下城堡》 | RPG | Unity |

### 开发平台与套件
| 工具 / 套件 | 用途 | 说明 |
|---|---|---|
| **Xcode** | 苹果官方 IDE | 必须使用 macOS |
| **Swift / Objective-C** | iOS 原生开发语言 | Swift 为现代主流 |
| **SpriteKit** | 苹果官方 2D 游戏框架 | 内置于 iOS SDK |
| **SceneKit** | 苹果官方 3D 游戏框架 | 内置于 iOS SDK |
| **Metal** | 苹果官方图形 API | 替代 OpenGL，性能更强 |
| **Unity（iOS 导出）** | 跨平台引擎 | 需 macOS + Xcode 打包 |
| **Godot 4（iOS 导出）** | 跨平台引擎 | 需 macOS + Xcode 签名 |
| **App Store Connect** | 发布与分发平台 | 苹果官方上架工具 |
| **TestFlight** | 内测分发工具 | iOS 官方 Beta 测试 |

### 🗣️ iOS 游戏开发主流语言

| 语言 | 用途 | 典型场景 |
|---|---|---|
| **Swift** | iOS 原生首选语言 | SpriteKit / SceneKit 游戏 |
| **Objective-C** | 旧版原生语言 | 老项目维护 |
| **C#** | 跨平台游戏脚本 | Unity iOS 导出 |
| **C++** | 高性能核心逻辑 | 跨平台引擎底层 |
| **GDScript** | Godot 脚本 | Godot iOS 导出项目 |
| **Metal Shading Language** | 着色器语言 | Metal 图形渲染 |
| **Lua** | 轻量脚本 | 热更新、游戏内配置 |

---

## 📊 三平台横向对比

| 维度 | Windows | Linux | iOS |
|---|---|---|---|
| 市场占有率（游戏） | ⭐⭐⭐⭐⭐ 最大 | ⭐⭐ 小众但增长 | ⭐⭐⭐⭐ 手游第一 |
| 开发门槛 | 低 | 中 | 高（需 Mac + 苹果账号） |
| 图形 API | DirectX / Vulkan | OpenGL / Vulkan | Metal |
| 推荐引擎 | Godot / Unity / UE5 | Godot / Unity | Unity / Godot |
| 发行平台 | Steam / Epic | Steam / itch.io | App Store |
| 费用 | 无强制抽成 | 无强制抽成 | App Store 抽成 30% |
| 首选语言 | C++ / C# / GDScript | C++ / C / GDScript | Swift / C# |
| 着色器语言 | HLSL | GLSL | Metal Shading Language |

---

## 相关笔记

- [[engin]] — 游戏引擎对比（Godot / Unity / Unreal）
- [[platforms]] — 各游戏主机平台代表游戏
