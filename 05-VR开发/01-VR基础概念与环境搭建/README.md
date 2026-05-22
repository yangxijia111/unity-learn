# 01-VR基础概念与环境搭建

## 目录

- [1. VR 基本概念](#1-vr-基本概念)
- [2. 主流 VR 设备介绍](#2-主流-vr-设备介绍)
- [3. Unity VR 开发环境搭建](#3-unity-vr-开发环境搭建)
- [4. 项目设置](#4-项目设置)
- [5. 第一个 VR 场景搭建](#5-第一个-vr-场景搭建)
- [6. 常见问题排查](#6-常见问题排查)
- [7. 参考资源](#7-参考资源)

---

## 1. VR 基本概念

### 什么是 VR（Virtual Reality）

虚拟现实（VR）是一种利用计算机技术生成的三维模拟环境，用户可以通过专用设备（头显、手柄等）沉浸其中并与之交互。与传统屏幕显示不同，VR 追求的是让用户感觉自己"身处"虚拟世界。

### 沉浸感三要素

沉浸感是 VR 体验的核心，主要由以下三个维度构成：

| 要素 | 说明 | 关键技术 |
|------|------|----------|
| **视觉沉浸** | 双眼立体渲染、高分辨率、大视场角（FOV）、低延迟追踪 | 立体渲染、6DoF 追踪、高刷新率（90Hz+） |
| **听觉沉浸** | 空间音频，声音随头部转动而变化 | HRTF（头相关传递函数）、3D 空间音频 |
| **交互沉浸** | 用户能自然地与虚拟世界交互 | 手柄输入、手部追踪、触觉反馈 |

### 关键术语

- **6DoF（六自由度）**：头部和手部在 X/Y/Z 三个轴上的平移 + 旋转，共 6 个自由度
- **3DoF（三自由度）**：仅追踪旋转（Pitch/Yaw/Roll），无位置追踪
- **FOV（Field of View）**：视场角，越大越沉浸（主流设备约 100°-120°）
- **刷新率**：屏幕每秒刷新次数，90Hz 是舒适体验的最低标准
- **延迟（Motion-to-Photon）**：从用户移动到画面更新的时间，需 < 20ms 以避免晕动症
- **Inside-out Tracking**：通过头显上的摄像头进行追踪，无需外部基站
- **Outside-in Tracking**：通过外部传感器/基站追踪（如 HTC Vive Lighthouse）

---

## 2. 主流 VR 设备介绍

### Meta Quest 系列

- **Quest 3 / Quest 3S**：当前主流一体机，支持 Inside-out 追踪、手部追踪、混合现实（MR）
- 特点：无线一体机、价格亲民、生态成熟、支持 PC VR（Link/Air Link）
- 开发优势：用户基数大，Meta 官方 SDK 支持完善

### HTC Vive 系列

- **Vive XR Elite / Vive Pro 2**：企业级和高端消费级设备
- 特点：Lighthouse 外部基站追踪精度高，适合高精度场景
- 适用：企业培训、专业模拟

### Apple Vision Pro

- **定位**：空间计算设备，强调 MR（混合现实）
- 特点：超高分辨率 micro-OLED、眼动追踪、手势交互
- 开发：使用 RealityKit/SwiftUI，与 Unity 生态不同

### Pico 系列

- **Pico 4 / Pico 4 Ultra**：字节跳动旗下的一体机
- 特点：性价比高，国内生态支持好
- 开发：基于 OpenXR，兼容性良好

### Valve Index

- **定位**：高端 PC VR 设备
- 特点：144Hz 刷新率、Knuckles 手柄（手指级追踪）、Lighthouse 追踪
- 适用：硬核 VR 游戏玩家

### 设备选择建议（Unity 开发）

| 场景 | 推荐设备 |
|------|----------|
| 入门开发 / 最大用户群 | Meta Quest 3 |
| 国内市场优先 | Pico 4 |
| 高精度企业应用 | HTC Vive Pro 2 |
| 高端游戏 | Valve Index |

---

## 3. Unity VR 开发环境搭建

### 3.1 安装 Unity + Android Build Support

> 以 **Unity 2022 LTS** 或 **Unity 6 (6000.x)** 为例

1. 打开 **Unity Hub** → 安装编辑器 → 选择 **Unity 2022.3 LTS**（或更新的 LTS 版本）
2. 在安装选项中勾选：
   - ✅ **Android Build Support**（Quest/Pico 开发必须）
   - ✅ Android SDK & NDK Tools（Unity 会自动下载）
   - ✅ OpenJDK（Unity 会自动下载）
3. 如果已经安装了 Unity 但没勾选 Android 支持，可以在 Unity Hub → Installs → 对应版本 → ⚙ → Add Module 中补装

### 3.2 安装 XR Plugin Management

1. 打开 Unity 项目（或新建 3D 项目）
2. 菜单 **Edit → Project Settings → XR Plug-in Management**
3. 点击 **Install XR Plug-in Management** 按钮（或通过 Package Manager 安装）
4. 安装后会在 Project Settings 中看到 XR Plug-in Management 面板

### 3.3 配置 OpenXR Plugin

**OpenXR** 是跨平台 XR 标准，推荐作为首选后端。

1. **Window → Package Manager** → 搜索 **OpenXR Plugin** → Install
2. **Project Settings → XR Plug-in Management → OpenXR**：
   - ✅ 启用 OpenXR
   - 在 **Android** 标签页也启用（Quest/Pico 需要）
3. 添加 **Interaction Profiles**（交互配置文件）：
   - 点击 **+** 添加目标设备的手柄配置
   - 常用：`Meta Quest Touch Pro Controller`、`Pico Controller`、`HTC Vive Controller`
4. 如果出现黄色警告，点击 **Fix** 自动修复

### 3.4 安装 XR Interaction Toolkit（推荐）

1. **Package Manager** → 搜索 **XR Interaction Toolkit** → Install
2. 导入示例场景（可选但推荐）：
   - 在 Package Manager 中找到 XRI → Samples → 导入 **Starter Assets** 和 **XR Device Simulator**
3. **XR Device Simulator** 允许你在没有 VR 设备的情况下用鼠标键盘测试

### 3.5 配置 Meta XR SDK（Quest 开发专用，可选）

如果需要使用 Meta 特有功能（如 Passthrough、手部追踪高级功能）：

1. 在 Package Manager 中添加 Meta XR SDK（通过 git URL 或 Asset Store）
2. 核心包：
   - **Meta XR Core SDK**：基础功能
   - **Meta XR Platform SDK**：社交、成就等平台功能
   - **Meta XR Interaction SDK**：Meta 自有的交互系统

> 💡 如果只用 OpenXR + XRI，不需要安装 Meta SDK 也能开发 Quest 应用。

---

## 4. 项目设置

### Player Settings

**File → Build Settings → Player Settings**：

| 设置项 | 推荐值 | 说明 |
|--------|--------|------|
| Color Space | **Linear** | VR 渲染推荐 Linear 色彩空间 |
| Graphics APIs | OpenGLES3 / Vulkan | Quest 使用 OpenGLES3 或 Vulkan |
| Minimum API Level | **Android 10 (API 29)** | Quest 2+ 最低要求 |
| Scripting Backend | **IL2CPP** | Quest 发布必须 |
| Target Architecture | **ARM64** | Quest 必须 64 位 |
| Internet Access | 按需 | 需要联网时才开启 |

### XR Plug-in Management 配置

```
Project Settings
└── XR Plug-in Management
    ├── PC (Windows/Mac/Linux)
    │   └── ✅ OpenXR
    │       └── Interaction Profiles: 添加你的设备
    └── Android
        └── ✅ OpenXR
            └── Interaction Profiles: 添加 Quest/Pico 控制器
```

### Quality Settings

- 关闭或降低 **Anti Aliasing**（MSAA）以节省性能（VR 对帧率要求高）
- 推荐 **4x MSAA** 作为平衡点
- 关闭不需要的后处理效果

---

## 5. 第一个 VR 场景搭建

### 步骤 1：创建场景

1. **File → New Scene** → 选择 **Basic (Built-in)** 或 **URP** 模板
2. 保存场景为 `FirstVRScene`

### 步骤 2：添加 XR Origin

如果使用 **XR Interaction Toolkit**：

1. 删除场景中的 Main Camera
2. **Hierarchy 右键 → XR → XR Origin (XR Interaction Setup)**
3. 这会自动创建：
   - XR Origin（根物体）
   - Camera Offset
   - Main Camera（头显相机）
   - Left Controller（左手柄）
   - Right Controller（右手柄）

### 步骤 3：添加地面和测试物体

1. **Hierarchy → 3D Object → Plane**（作为地面）
2. 添加几个 **Cube** 和 **Sphere** 作为测试交互物体
3. 给物体添加 **Rigidbody** 和 **Box Collider**

### 步骤 4：添加交互组件

给 Cube 添加：
- **XR Grab Interactable** 组件（可抓取）

### 步骤 5：运行测试

**方式 A：有 VR 设备**
1. 连接 VR 头显
2. 点击 Play 按钮
3. 应该能在 VR 中看到场景并抓取物体

**方式 B：无 VR 设备（使用 Simulator）**
1. 确保已导入 XR Device Simulator
2. 在场景中添加 **XR Device Simulator** 预制体
3. 使用 WASD 移动，鼠标旋转视角
4. 按住鼠标左键/右键模拟左右手柄交互

### 基础场景结构

```
Hierarchy:
├── XR Origin (XR Interaction Setup)
│   ├── Camera Offset
│   │   ├── Main Camera
│   │   ├── Left Controller (XR Controller)
│   │   └── Right Controller (XR Controller)
│   └── Locomotion System (如已配置)
├── Plane (地面)
├── Cube (添加了 XR Grab Interactable + Rigidbody)
└── Directional Light
```

---

## 6. 常见问题排查

### 黑屏 / 无画面

| 问题 | 解决方案 |
|------|----------|
| Play 后黑屏 | 检查 XR Plug-in Management 是否启用了对应平台 |
| Quest Link 黑屏 | 确认 Oculus PC App 中启用了 Unknown Sources |
| 画面只有 2D | 确认场景中有 XR Origin 且 Camera 是其子物体 |
| URP 黑屏 | 确认 URP Asset 中启用了 XR 设置 |

### 追踪相关

| 问题 | 解决方案 |
|------|----------|
| 头部追踪不工作 | 检查 OpenXR 插件是否启用，Interaction Profile 是否添加 |
| 手柄不出现 | 确认添加了正确的 Interaction Profile（如 Meta Quest Touch） |
| 手柄位置偏移 | 检查 Camera Offset 的位置是否为 (0, 0, 0) |

### 构建相关

| 问题 | 解决方案 |
|------|----------|
| Quest 无法安装 APK | 确认 Target Architecture = ARM64, Scripting Backend = IL2CPP |
| 构建时间过长 | 首次 IL2CPP 构建较慢，后续有缓存会快很多 |
| Minimum API Level 错误 | Quest 3 需要 Android 10 (API 29) 或更高 |

### 性能相关

| 问题 | 解决方案 |
|------|----------|
| 画面卡顿 | 打开 **Oculus Debug Tool** 或 **Meta Quest Developer Hub** 查看帧率 |
| 帧率低于 72fps | 降低 MSAA、减少 Draw Call、使用 LOD |
| 热门帧率目标 | Quest: 72/90/120fps, PC VR: 90fps |

### 调试技巧

1. **Unity Console**：查看编译错误和警告
2. **Meta Quest Developer Hub (MQDH)**：实时查看 Quest 日志、性能数据
3. **Oculus Debug Tool**：查看渲染统计、启用性能叠加层
4. **adb logcat**：Android 底层日志

---

## 7. 参考资源

### 官方文档

- 📘 [Unity XR Documentation](https://docs.unity3d.com/Manual/XR.html)
- 📘 [OpenXR Plugin](https://docs.unity3d.com/Packages/com.unity.xr.openxr@latest)
- 📘 [XR Interaction Toolkit](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- 📘 [Meta Quest Developer Hub](https://developer.meta.com/horizon/documentation/)
- 📘 [Meta Quest Developer Guide](https://developer.meta.com/horizon/documentation/unity/)

### 教程 & 社区

- 🎥 [Unity Learn - VR Development Pathway](https://learn.unity.com/pathway/vr-development)
- 🎥 [Valem Tutorials (YouTube)](https://www.youtube.com/@ValemTutorials)
- 🎥 [Justin P. Barnett (YouTube)](https://www.youtube.com/@JustinPBarnett)
- 💬 [Unity VR/AR Forum](https://forum.unity.com/forums/ar-vr-xr.74/)
- 💬 [Meta Developer Forums](https://communityforums.atmeta.com/)

### 开源项目

- 🔗 [XR Interaction Toolkit Samples](https://github.com/Unity-Technologies/XR-Interaction-Toolkit-Examples)
- 🔗 [Meta XR SDK Samples](https://github.com/oculus-samples/Unity-Movement)

---

> 📝 **学习建议**：先确保基础环境能跑通（看到 VR 画面 + 手柄出现），再进入后续的交互系统学习。环境搭建是 VR 开发最容易卡住的环节，耐心排查。
