# 02-XR Interaction Toolkit 入门

## 目录

- [1. XR Interaction Toolkit 简介](#1-xr-interaction-toolkit-简介)
- [2. 核心组件详解](#2-核心组件详解)
- [3. Input Actions 配置](#3-input-actions-配置)
- [4. 手柄动作映射表](#4-手柄动作映射表)
- [5. 基础交互代码示例](#5-基础交互代码示例)
- [6. 参考资源](#6-参考资源)

---

## 1. XR Interaction Toolkit 简介

### 是什么

**XR Interaction Toolkit (XRI)** 是 Unity 官方提供的跨平台 XR 交互框架。它基于 Unity 的 **Input System** 构建，提供了一套高层次的交互抽象，让你不用关心底层设备差异就能实现常见的 VR/AR 交互。

### 为什么用它

| 优势 | 说明 |
|------|------|
| **跨平台** | 基于 OpenXR，一套代码跑 Quest、Pico、Vive、Index 等 |
| **组件化** | 通过挂组件实现交互，减少手写代码 |
| **官方支持** | Unity 官方维护，持续更新 |
| **Input System 集成** | 使用新的 Input System 做动作映射，灵活且可配置 |
| **可扩展** | 所有核心类都可继承重写 |

### 版本选择

- **XRI 2.5+**（推荐）：交互架构更清晰，基于 **Interactor → Interactable** 模型
- **XRI 3.0+**：Unity 6 中默认包含，架构进一步优化
- 本笔记以 **XRI 2.5+** 为基础，3.0 向后兼容

---

## 2. 核心组件详解

### 2.1 XR Origin（交互根节点）

**XR Origin** 是整个 VR 交互的根物体，替代了旧版的 `XR Rig`。

```
XR Origin
├── Camera Offset（相机偏移节点）
│   ├── Main Camera（头显相机）
│   ├── Left Controller（左手 Controller）
│   └── Right Controller（右手 Controller）
└── Locomotion System（移动系统，可选）
```

**关键属性：**
- `Camera Y Offset`：相机离地面的高度偏移（默认 1.2m，根据玩家坐高/站高调整）
- `Tracking Origin Mode`：追踪原点模式，通常设为 `Floor`（站立体验）

### 2.2 XR Controller（Action-based）

每个手柄对应一个 **XR Controller (Action-based)** 组件，挂在 Controller 物体上。

**核心属性：**
- `Select Action`：选择/抓取动作（扳机键）
- `Activate Action`：激活/使用动作（ grip 键或其他）
- `UI Press Action`：UI 点击动作
- `Rotate Anchor Action`：旋转锚点
- `Translate Anchor Action`：移动锚点
- `Model Prefab`：手柄 3D 模型（可选）
- `Model Animator`：手柄动画控制器（可选）

### 2.3 XR Interactor（交互者）

**Interactor** 是"发起交互的一方"，即你的手柄或射线。有两种主要类型：

#### XR Ray Interactor（射线交互）

从手柄发出一条射线，用于远距离交互。

```
适用场景：
- 远距离抓取物体
- UI 按钮点击
- 传送目标选择
- 菜单操作
```

**关键属性：**
- `Max Raycast Distance`：射线最大距离（默认 30m）
- `Line Type`：射线类型（Straight/StraightLine、Projectile/Curved）
- `Reference Frame`：射线参考坐标系
- `Enable Interaction with UI`：是否与 UI 交互
- `Render Line`：是否渲染射线（需配合 XR Interactor Line Visual）

#### XR Direct Interactor（直接交互）

近距离直接触碰交互。

```
适用场景：
- 手动抓取近处物体
- 按压按钮
- 物理接触
```

**关键属性：**
- 需要一个 **Collider**（通常是 Sphere Collider）作为触发区域
- `Use Interaction Layers`：过滤哪些 Layer 的物体可以被交互

### 2.4 XR Interactable（可交互物体）

**Interactable** 是"被交互的一方"，挂在可被抓取、可被点击等的物体上。

#### XR Grab Interactable（可抓取物体）

最常用的组件，让物体可以被手柄抓取。

```
关键属性：
├── Movement Type
│   ├── Velocity Tracking（物理跟随，最真实）
│   ├── Kinematic（无物理碰撞，最稳定）
│   └── Instantaneous（瞬间移动到手柄位置）
├── Throw On Release（松手时投掷）
├── Throw Smoothing Duration（投掷平滑时间）
├── Throw Velocity Scale（投掷力度缩放）
├── Throw Angular Velocity Scale（旋转投掷缩放）
├── Attach Transform（抓取点偏移）
└── Track Position / Track Rotation（是否追踪位置/旋转）
```

**Movement Type 选择建议：**

| 类型 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Velocity Tracking | 物理真实 | 可能穿墙 | 需要物理效果的物体 |
| Kinematic | 稳定，不穿墙 | 无物理响应 | 武器、工具 |
| Instantaneous | 最简单 | 不真实 | UI、简单原型 |

### 2.5 XR Socket Interactor（插槽/吸附）

用于创建"插槽"效果——物体放入特定位置时自动吸附。

```
使用场景：
- 武器挂载点
- 物品收纳格
- 钥匙插入锁孔
- 拼图吸附
```

**关键属性：**
- `Show Interactable Hover Meshes`：悬停时显示物体的放置预览
- `Socket Scale`：吸附时物体的缩放
- `Recycle Delay Time`：回收延迟

---

## 3. Input Actions 配置

### 3.1 创建 Input Actions Asset

XRI 使用 Unity 的 **Input System** 进行动作映射。

1. **Project 窗口右键 → Create → Input Actions → Input Actions Asset**
2. 命名为 `XRI Default Input Actions`
3. 双击打开编辑器

### 3.2 配置 Action Map

创建以下 Action Map 和 Actions：

```
XRI Default Input Actions
├── XRI Right Hand（右手动作）
│   ├── Position（Vector2 → Vector3，手柄位置）
│   ├── Rotation（Quaternion，手柄旋转）
│   ├── Select（Button，扳机键）
│   ├── Select Value（Single，扳机力度 0-1）
│   ├── Activate（Button，Grip 键）
│   ├── Activate Value（Single，Grip 力度 0-1）
│   ├── UI Press（Button）
│   ├── Haptic Device（Pass Through）
│   └── Rotate Anchor（Vector2，摇杆旋转）
│   └── Translate Anchor（Vector2，摇杆移动）
├── XRI Left Hand（左手动作，结构同上）
├── XRI UI（UI 交互）
│   ├── Navigate（Vector2）
│   ├── Submit（Button）
│   └── Cancel（Button）
└── XRI Head（头部追踪）
    └── Position / Rotation
```

### 3.3 绑定具体按键

以 `XRI Right Hand → Select` 为例：

1. 展开 Action → 点击 **+** 添加 Binding
2. 选择 **XR Controller** → **RightHand** → **Trigger Button**
3. 同样的方法绑定其他按键

### 3.4 在场景中引用

1. 在场景中创建空物体 `Input Action Manager`，挂载 **Input Action Manager** 组件
2. 将创建的 Input Actions Asset 拖入 `Action Assets` 列表
3. 在各个 XR Controller 上引用对应的 Actions

---

## 4. 手柄动作映射表

### Meta Quest Touch Controller

| 按键 | 对应 Action | Input System Path |
|------|-------------|-------------------|
| 扳机（Trigger） | Select / Select Value | `<XRController>{RightHand}/trigger` |
| 握把（Grip） | Activate / Activate Value | `<XRController>{RightHand}/grip` |
| A 按钮 | 自定义交互 | `<XRController>{RightHand}/primaryButton` |
| B 按钮 | 自定义交互 | `<XRController>{RightHand}/secondaryButton` |
| 摇杆 | 移动/旋转 | `<XRController>{RightHand}/primary2DAxis` |
| 摇杆按下 | 自定义 | `<XRController>{RightHand}/primary2DAxisClick` |
| 菜单按钮 | 暂停/菜单 | `<XRController>{LeftHand}/menu` |

### Pico Controller

| 按键 | 对应 Action | Input System Path |
|------|-------------|-------------------|
| 扳机（Trigger） | Select | `<XRController>{RightHand}/trigger` |
| 握把（Grip） | Activate | `<XRController>{RightHand}/grip` |
| X/A 按钮 | 自定义 | `<XRController>{*Hand}/primaryButton` |
| Y/B 按钮 | 自定义 | `<XRController>{*Hand}/secondaryButton` |
| 摇杆 | 移动/旋转 | `<XRController>{*Hand}/primary2DAxis` |

> 💡 **提示**：Quest 和 Pico 在 OpenXR 下的按键映射基本一致，区别主要在 Interaction Profile 的选择上。

---

## 5. 基础交互代码示例

### 5.1 抓取物体 - 基础回调

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 可抓取物体的行为脚本
/// 监听抓取/释放事件，执行相应逻辑
/// </summary>
public class GrabableObject : MonoBehaviour
{
    // XR Grab Interactable 组件引用
    private XRGrabInteractable grabInteractable;

    [Header("抓取设置")]
    [Tooltip("抓取时是否禁用物理")]
    public bool disablePhysicsOnGrab = true;

    [Tooltip("抓取时播放的音效")]
    public AudioClip grabSound;

    private AudioSource audioSource;

    private void Awake()
    {
        // 获取同物体上的 XRGrabInteractable 组件
        grabInteractable = GetComponent<XRGrabInteractable>();
        audioSource = GetComponent<AudioSource>();
    }

    private void OnEnable()
    {
        // 注册抓取事件
        // selectEntered = 当玩家按下扳机抓取物体时触发
        grabInteractable.selectEntered.AddListener(OnGrab);

        // selectReleased = 当玩家松开扳机释放物体时触发
        grabInteractable.selectExited.AddListener(OnRelease);
    }

    private void OnDisable()
    {
        // 取消注册，避免内存泄漏
        grabInteractable.selectEntered.RemoveListener(OnGrab);
        grabInteractable.selectExited.RemoveListener(OnRelease);
    }

    /// <summary>
    /// 抓取时的回调
    /// </summary>
    /// <param name="args">抓取事件参数，包含抓取者（Interactor）信息</param>
    private void OnGrab(SelectEnterEventArgs args)
    {
        Debug.Log($"物体 {gameObject.name} 被抓取了！");

        // 播放抓取音效
        if (grabSound != null && audioSource != null)
        {
            audioSource.PlayOneShot(grabSound);
        }

        // 可以在这里做更多逻辑，比如触发剧情、记录状态等
    }

    /// <summary>
    /// 释放时的回调
    /// </summary>
    /// <param name="args">释放事件参数</param>
    private void OnRelease(SelectExitEventArgs args)
    {
        Debug.Log($"物体 {gameObject.name} 被释放了！");
    }
}
```

### 5.2 使用物体 - 激活交互

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 可"使用"的物体，比如手电筒、枪、工具等
/// 当玩家按下 Grip 键（或配置的 Activate 按键）时触发
/// </summary>
public class UsableObject : MonoBehaviour
{
    private XRGrabInteractable grabInteractable;

    [Header("使用设置")]
    [Tooltip("使用时的效果（如子弹发射点、激光起点等）")]
    public Transform effectPoint;

    [Tooltip("使用时的粒子特效")]
    public GameObject useEffectPrefab;

    [Tooltip("冷却时间（秒）")]
    public float cooldown = 0.5f;

    private float lastUseTime;
    private bool isGrabbed; // 是否正被抓在手里

    private void Awake()
    {
        grabInteractable = GetComponent<XRGrabInteractable>();
    }

    private void OnEnable()
    {
        // 先监听抓取状态
        grabInteractable.selectEntered.AddListener(OnGrab);
        grabInteractable.selectExited.AddListener(OnRelease);

        // activated = 当 Interactor 发出 Activate 动作时触发（通常是 Grip 键）
        grabInteractable.activated.AddListener(OnUse);
    }

    private void OnDisable()
    {
        grabInteractable.selectEntered.RemoveListener(OnGrab);
        grabInteractable.selectExited.RemoveListener(OnRelease);
        grabInteractable.activated.RemoveListener(OnUse);
    }

    private void OnGrab(SelectEnterEventArgs args)
    {
        isGrabbed = true;
    }

    private void OnRelease(SelectExitEventArgs args)
    {
        isGrabbed = false;
    }

    /// <summary>
    /// 按下使用键时触发
    /// </summary>
    /// <param name="args">激活事件参数</param>
    private void OnUse(ActivateEventArgs args)
    {
        // 冷却检查
        if (Time.time - lastUseTime < cooldown)
        {
            Debug.Log("还在冷却中...");
            return;
        }

        lastUseTime = Time.time;

        Debug.Log($"使用了 {gameObject.name}！");

        // 播放使用特效
        if (useEffectPrefab != null)
        {
            Transform spawnPoint = effectPoint != null ? effectPoint : transform;
            Instantiate(useEffectPrefab, spawnPoint.position, spawnPoint.rotation);
        }

        // 在这里添加具体的使用逻辑
        // 比如：发射子弹、开灯、播放动画等
    }
}
```

### 5.3 高亮悬停效果

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 当射线悬停在物体上时显示高亮效果
/// 提供视觉反馈，让玩家知道"这个东西可以交互"
/// </summary>
public class HighlightOnHover : MonoBehaviour
{
    private XRBaseInteractable interactable;

    [Header("高亮设置")]
    [Tooltip("高亮颜色")]
    public Color highlightColor = Color.yellow;

    [Tooltip("高亮时的发光强度")]
    [Range(0f, 2f)]
    public float emissionIntensity = 1.0f;

    // 原始材质属性（用于还原）
    private Renderer[] renderers;
    private Color[] originalColors;
    private float[] originalEmissionIntensities;

    private void Awake()
    {
        interactable = GetComponent<XRBaseInteractable>();

        // 缓存所有 Renderer 及其原始颜色
        renderers = GetComponentsInChildren<Renderer>();
        originalColors = new Color[renderers.Length];
        originalEmissionIntensities = new float[renderers.Length];

        for (int i = 0; i < renderers.Length; i++)
        {
            if (renderers[i].material.HasProperty("_Color"))
            {
                originalColors[i] = renderers[i].material.color;
            }
            if (renderers[i].material.HasProperty("_EmissionColor"))
            {
                originalEmissionIntensities[i] = renderers[i].material.GetColor("_EmissionColor").maxColorComponent;
            }
        }
    }

    private void OnEnable()
    {
        // hoverEntered = 射线/手部碰到物体时触发
        interactable.hoverEntered.AddListener(OnHoverEnter);

        // hoverExited = 射线/手部离开物体时触发
        interactable.hoverExited.AddListener(OnHoverExit);
    }

    private void OnDisable()
    {
        interactable.hoverEntered.RemoveListener(OnHoverEnter);
        interactable.hoverExited.RemoveListener(OnHoverExit);
    }

    /// <summary>
    /// 悬停进入时，切换到高亮颜色
    /// </summary>
    private void OnHoverEnter(HoverEnterEventArgs args)
    {
        SetHighlight(true);
    }

    /// <summary>
    /// 悬停退出时，还原原始颜色
    /// </summary>
    private void OnHoverExit(HoverExitEventArgs args)
    {
        SetHighlight(false);
    }

    /// <summary>
    /// 设置高亮状态
    /// </summary>
    /// <param name="highlight">true=高亮, false=还原</param>
    private void SetHighlight(bool highlight)
    {
        for (int i = 0; i < renderers.Length; i++)
        {
            if (renderers[i] == null) continue;

            Material mat = renderers[i].material;

            if (highlight)
            {
                // 设置高亮颜色
                if (mat.HasProperty("_Color"))
                {
                    mat.color = highlightColor;
                }

                // 设置自发光（需要材质支持 Emission）
                if (mat.HasProperty("_EmissionColor"))
                {
                    mat.EnableKeyword("_EMISSION");
                    mat.SetColor("_EmissionColor", highlightColor * emissionIntensity);
                }
            }
            else
            {
                // 还原原始颜色
                if (mat.HasProperty("_Color"))
                {
                    mat.color = originalColors[i];
                }

                if (mat.HasProperty("_EmissionColor"))
                {
                    mat.SetColor("_EmissionColor", originalColors[i] * originalEmissionIntensities[i]);
                }
            }
        }
    }
}
```

---

## 6. 参考资源

### 官方文档

- 📘 [XR Interaction Toolkit Documentation](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- 📘 [Input System Documentation](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest)
- 📘 [XR Interaction Toolkit API Reference](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest/api/)

### 教程

- 🎥 [Valem Tutorials - XRI 入门系列](https://www.youtube.com/@ValemTutorials)
- 🎥 [Justin P. Barnett - VR 开发教程](https://www.youtube.com/@JustinPBarnett)
- 🎥 [Unity Learn - VR Development](https://learn.unity.com/pathway/vr-development)

### 示例项目

- 🔗 [XRI Starter Assets（Package Manager 中导入）](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- 🔗 [XRI Examples on GitHub](https://github.com/Unity-Technologies/XR-Interaction-Toolkit-Examples)

---

> 📝 **学习建议**：先通过 Starter Assets 示例场景理解每个组件的作用，再自己搭建场景练习。重点掌握 Interactor ↔ Interactable 的交互模型，这是 XRI 的核心设计思想。
