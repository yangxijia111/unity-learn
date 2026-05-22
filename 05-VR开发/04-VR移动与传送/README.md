# 04-VR移动与传送

## 目录

- [1. VR 移动方式概述](#1-vr-移动方式概述)
- [2. 晕动症成因与缓解](#2-晕动症成因与缓解)
- [3. 传送系统](#3-传送系统)
- [4. 平滑移动系统](#4-平滑移动系统)
- [5. 攀爬系统基础](#5-攀爬系统基础)
- [6. 旋转方式](#6-旋转方式)
- [7. 完整实战示例](#7-完整实战示例)
- [8. 常见问题排查](#8-常见问题排查)
- [9. 参考资源](#9-参考资源)

---

## 1. VR 移动方式概述

VR 中的移动是最核心也最棘手的问题之一——它直接关系到用户的舒适度。不同的移动方式适合不同的游戏类型和用户偏好。

### 三种主要移动方式

| 方式 | 原理 | 舒适度 | 沉浸感 | 适用场景 |
|------|------|--------|--------|----------|
| **传送（Teleport）** | 瞬间跳转到目标位置 | ⭐⭐⭐⭐⭐ | ⭐⭐ | 探索、解谜、休闲 |
| **平滑移动（Continuous）** | 像传统游戏一样持续移动 | ⭐⭐ | ⭐⭐⭐⭐⭐ | FPS、竞速、动作 |
| **攀爬（Climbing）** | 手抓住表面，用手拉动身体 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 平台跳跃、探索 |

### 设计建议

> **最佳实践：提供多种移动方式让用户选择。** 很多 VR 游戏会在设置里同时提供传送和平滑移动。

---

## 2. 晕动症成因与缓解

### 为什么会晕？

晕动症（Motion Sickness）的根本原因是 **视觉感知与前庭系统（内耳平衡器官）的信息不一致**：

```
眼睛：我看到身体在移动
内耳：我没感觉到任何加速度
大脑：??? 中毒了吧 → 引发恶心反应
```

### 常见触发因素

| 触发因素 | 说明 |
|----------|------|
| **强制移动** | 用户没有主动移动，但画面在动 |
| **加速/减速** | 非匀速运动更容易引发不适 |
| **视野遮挡不足** | 周边视野的运动变化是主要诱因 |
| **低帧率** | 低于 90Hz 会加重不适 |
| **非自然旋转** | 不是由用户头部控制的旋转 |

### 缓解策略

```csharp
using UnityEngine;

/// <summary>
/// 晕动症缓解系统
/// 提供隧道视野（Tunneling）等效果来减少不适
/// </summary>
public class MotionSicknessMitigation : MonoBehaviour
{
    [Header("隧道视野效果")]
    [Tooltip("隧道视野遮罩（带透明度的球形遮罩）")]
    public GameObject tunnelVignette;
    
    [Tooltip("移动时遮罩强度（0-1）")]
    [Range(0f, 1f)]
    public float movingIntensity = 0.5f;
    
    [Tooltip("静止时遮罩强度")]
    [Range(0f, 1f)]
    public float idleIntensity = 0f;
    
    [Tooltip("平滑速度")]
    public float smoothSpeed = 5f;
    
    private float currentIntensity;
    private Material vignetteMaterial;
    private bool isMoving = false;

    private void Start()
    {
        if (tunnelVignette != null)
        {
            // 获取遮罩材质
            Renderer renderer = tunnelVignette.GetComponent<Renderer>();
            if (renderer != null)
                vignetteMaterial = renderer.material;
        }
        
        currentIntensity = idleIntensity;
    }

    /// <summary>
    /// 通知系统用户正在移动
    /// </summary>
    public void SetMoving(bool moving)
    {
        isMoving = moving;
    }

    private void Update()
    {
        float targetIntensity = isMoving ? movingIntensity : idleIntensity;
        currentIntensity = Mathf.Lerp(currentIntensity, targetIntensity, smoothSpeed * Time.deltaTime);
        
        // 应用到材质
        if (vignetteMaterial != null)
        {
            vignetteMaterial.SetFloat("_ApertureSize", 1f - currentIntensity);
        }
    }
}
```

### 缓解方案速查

- **隧道视野（Tunneling/Vignette）**：移动时缩小视野，减少周边运动感知
- **遮罩网格（Comfort Cage）**：在视野中显示稳定的参考网格
- **渐进式加速**：避免突然的加速/减速
- **稳定参考点**：在视野中固定一个不会移动的 UI 元素
- **限制平移速度**：不要让用户飞太快

---

## 3. 传送系统

传送是 VR 中最舒适的移动方式，也是 XR Interaction Toolkit 内置支持的方式。

### XRI Teleportation Provider 架构

```
XR Origin (XR Interaction Setup)
├── Locomotion System
│   ├── Teleportation Provider（传送系统管理器）
│   ├── Snap Turn Provider（快转）
│   └── Continuous Turn Provider（连续旋转）
├── Left Controller
│   └── Ray Interactor（射线交互器，用于选择传送点）
└── Right Controller
    └── Ray Interactor
```

### 传送区域（Teleportation Area）

让任意平面可以作为传送目标：

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 传送区域配置
/// 在地面上添加 TeleportationArea 组件即可让整个区域可传送
/// </summary>
public class TeleportAreaSetup : MonoBehaviour
{
    [Tooltip("传送区域的地面")]
    public Collider groundCollider;
    
    private void SetupTeleportArea()
    {
        // 确保有 Collider
        if (groundCollider == null)
            groundCollider = GetComponent<Collider>();
        
        if (groundCollider == null)
        {
            Debug.LogError("传送区域需要一个 Collider 组件！");
            return;
        }
        
        // 添加 Teleportation Area
        TeleportationArea area = GetComponent<TeleportationArea>();
        if (area == null)
            area = gameObject.AddComponent<TeleportationArea>();
        
        // 配置传送区域
        // Teleportation Anchor 用于特定传送点（带朝向）
        // Teleportation Area 用于任意位置传送
        
        Debug.Log($"[传送] 已配置传送区域：{gameObject.name}");
    }
}
```

### 传送锚点（Teleportation Anchor）

传送锚点不仅指定位置，还指定传送后的朝向：

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 传送锚点管理器
/// 管理多个传送点，支持条件解锁
/// </summary>
public class TeleportAnchorManager : MonoBehaviour
{
    [System.Serializable]
    public class TeleportAnchorEntry
    {
        public string anchorName;
        public TeleportationAnchor anchor;
        public bool isUnlocked = true;
        public GameObject[] requiredObjects; // 到达此锚点前需要交互的物体
    }
    
    [Header("传送锚点列表")]
    public TeleportAnchorEntry[] anchors;
    
    /// <summary>
    /// 解锁指定锚点
    /// </summary>
    public void UnlockAnchor(string name)
    {
        foreach (var entry in anchors)
        {
            if (entry.anchorName == name)
            {
                entry.isUnlocked = true;
                entry.anchor.enabled = true;
                
                // 可以在这里改变锚点的视觉效果
                // 比如从灰色变为高亮
                SetAnchorVisual(entry.anchor, true);
                
                Debug.Log($"[传送] 锚点已解锁：{name}");
                return;
            }
        }
    }
    
    private void SetAnchorVisual(TeleportationAnchor anchor, bool active)
    {
        // 获取锚点的 Renderer 组件，改变颜色
        Renderer renderer = anchor.GetComponentInChildren<Renderer>();
        if (renderer != null)
        {
            renderer.material.color = active ? Color.green : Color.gray;
        }
    }
}
```

### 传送弧线预览

XR Interaction Toolkit 的 Ray Interactor 默认会显示传送弧线。自定义弧线效果：

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 自定义传送弧线效果
/// 使用 LineRenderer 显示从手到传送点的曲线
/// </summary>
[RequireComponent(typeof(LineRenderer))]
public class TeleportArcVisual : MonoBehaviour
{
    [Header("弧线设置")]
    [Tooltip("弧线分段数")]
    public int arcSegments = 30;
    
    [Tooltip("弧线重力（向下弯曲程度）")]
    public float arcGravity = 9.8f;
    
    [Tooltip("最大弧线距离")]
    public float maxDistance = 20f;
    
    [Tooltip("弧线速度")]
    public float arcSpeed = 10f;
    
    [Header("视觉效果")]
    [Tooltip("可传送时的颜色")]
    public Color validColor = Color.green;
    
    [Tooltip("不可传送时的颜色")]
    public Color invalidColor = Color.red;
    
    [Tooltip("弧线宽度")]
    public float arcWidth = 0.02f;
    
    private LineRenderer lineRenderer;
    private bool isValidTarget = false;

    private void Awake()
    {
        lineRenderer = GetComponent<LineRenderer>();
        lineRenderer.positionCount = 0;
        lineRenderer.startWidth = arcWidth;
        lineRenderer.endWidth = arcWidth;
        lineRenderer.useWorldSpace = true;
    }

    /// <summary>
    /// 更新弧线显示
    /// </summary>
    public void UpdateArc(Vector3 startPos, Vector3 forward, bool valid)
    {
        isValidTarget = valid;
        lineRenderer.positionCount = arcSegments;
        lineRenderer.startColor = valid ? validColor : invalidColor;
        lineRenderer.endColor = valid ? validColor : invalidColor;
        
        Vector3 velocity = forward * arcSpeed;
        Vector3 position = startPos;
        float timeStep = 0.1f;
        
        for (int i = 0; i < arcSegments; i++)
        {
            lineRenderer.SetPosition(i, position);
            
            // 抛物线运动
            position += velocity * timeStep;
            velocity.y -= arcGravity * timeStep;
        }
        
        // 如果弧线末端碰到地面，截断到碰撞点
        // （简化处理，实际可以用射线检测）
    }

    /// <summary>
    /// 隐藏弧线
    /// </summary>
    public void HideArc()
    {
        lineRenderer.positionCount = 0;
    }
}
```

---

## 4. 平滑移动系统

平滑移动提供更沉浸的体验，但舒适度较低。XR Toolkit 提供了 **Continuous Move Provider**。

### XR Continuous Move Provider 配置

```
XR Origin
├── Locomotion System
│   ├── Continuous Move Provider
│   │   ├── Left Hand XR Controller（绑定摇杆输入）
│   │   └── Move Speed = 2.0 m/s
│   └── Continuous Turn Provider
│       └── Turn Speed = 60°/s
```

### 自定义平滑移动控制器

```csharp
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 带有晕动症缓解的平滑移动控制器
/// 支持隧道视野、渐进加速、移动音效
/// </summary>
public class SmoothLocomotion : MonoBehaviour
{
    [Header("移动设置")]
    [Tooltip("基础移动速度")]
    public float moveSpeed = 2.0f;
    
    [Tooltip("冲刺速度倍率")]
    public float sprintMultiplier = 1.5f;
    
    [Tooltip("渐进加速时间")]
    public float accelerationTime = 0.3f;
    
    [Tooltip("渐进减速时间")]
    public float decelerationTime = 0.2f;
    
    [Header("输入")]
    [Tooltip("移动输入（摇杆）")]
    public InputActionProperty moveAction;
    
    [Tooltip("冲刺输入（按钮）")]
    public InputActionProperty sprintAction;
    
    [Header("舒适度")]
    [Tooltip("隧道视野效果")]
    public MotionSicknessMitigation tunnelEffect;
    
    [Tooltip("启用隧道视野")]
    public bool useTunnelEffect = true;
    
    [Header("音效")]
    [Tooltip("脚步声列表")]
    public AudioClip[] footstepSounds;
    
    [Tooltip("脚步声间隔（秒）")]
    public float footstepInterval = 0.5f;
    
    private CharacterController characterController;
    private AudioSource audioSource;
    private Vector3 currentVelocity;
    private float currentSpeed;
    private float footstepTimer;

    private void Awake()
    {
        characterController = GetComponentInParent<CharacterController>();
        if (characterController == null)
            characterController = gameObject.AddComponent<CharacterController>();
        
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null)
            audioSource = gameObject.AddComponent<AudioSource>();
        
        audioSource.spatialBlend = 1.0f;
    }

    private void OnEnable()
    {
        moveAction.action?.Enable();
        sprintAction.action?.Enable();
    }

    private void OnDisable()
    {
        moveAction.action?.Disable();
        sprintAction.action?.Disable();
    }

    private void Update()
    {
        // 读取摇杆输入
        Vector2 input = moveAction.action?.ReadValue<Vector2>() ?? Vector2.zero;
        bool isSprinting = sprintAction.action?.IsPressed() ?? false;
        
        bool isMoving = input.magnitude > 0.1f;
        
        // 计算目标速度
        float targetSpeed = 0f;
        if (isMoving)
        {
            targetSpeed = moveSpeed * (isSprinting ? sprintMultiplier : 1f);
        }
        
        // 渐进加速/减速
        float smoothTime = isMoving ? accelerationTime : decelerationTime;
        currentSpeed = Mathf.SmoothDamp(currentSpeed, targetSpeed, ref currentVelocity.x, smoothTime);
        
        // 计算移动方向（基于头部朝向）
        Transform head = Camera.main.transform;
        Vector3 forward = Vector3.ProjectOnPlane(head.forward, Vector3.up).normalized;
        Vector3 right = Vector3.ProjectOnPlane(head.right, Vector3.up).normalized;
        
        Vector3 moveDirection = (forward * input.y + right * input.x).normalized * currentSpeed;
        
        // 应用移动
        if (characterController != null)
        {
            // 加上重力
            moveDirection.y = Physics.gravity.y * Time.deltaTime;
            characterController.Move(moveDirection * Time.deltaTime);
        }
        
        // 通知隧道视野效果
        if (useTunnelEffect && tunnelEffect != null)
        {
            tunnelEffect.SetMoving(isMoving);
        }
        
        // 脚步声
        if (isMoving && currentSpeed > 0.5f)
        {
            footstepTimer -= Time.deltaTime;
            if (footstepTimer <= 0f)
            {
                PlayFootstep();
                footstepTimer = footstepInterval / (currentSpeed / moveSpeed);
            }
        }
    }

    private void PlayFootstep()
    {
        if (footstepSounds == null || footstepSounds.Length == 0) return;
        
        AudioClip clip = footstepSounds[Random.Range(0, footstepSounds.Length)];
        audioSource.PlayOneShot(clip);
    }
}
```

---

## 5. 攀爬系统基础

攀爬是 VR 中最具沉浸感的移动方式——用手抓住表面，然后用手"拉"自己移动。

### 攀爬原理

1. 用户用手（或手柄）接触可攀爬表面
2. 按下抓取键 → 锁定手的位置
3. 移动手 → 反向移动 XR Origin（用户视角）
4. 松开手 → 解除锁定，自由落体或保持位置

### 攀爬控制器实现

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// VR 攀爬系统
/// 允许玩家抓住可攀爬表面并用手移动自己
/// </summary>
public class VRClimbing : MonoBehaviour
{
    [Header("攀爬设置")]
    [Tooltip("攀爬速度倍率")]
    public float climbSpeed = 1.0f;
    
    [Tooltip("是否启用重力")]
    public bool useGravity = true;
    
    [Tooltip("重力加速度")]
    public float gravity = 9.8f;
    
    [Tooltip("松手后的跳跃力（沿最后移动方向）")]
    public float jumpForce = 0f;
    
    [Header("引用")]
    [Tooltip("XR Origin")]
    public Transform xrOrigin;
    
    [Tooltip("左控制器的 XR Grab Interactable 代理")]
    public ClimbHand leftHand;
    
    [Tooltip("右控制器的 XR Grab Interactable 代理")]
    public ClimbHand rightHand;

    private Vector3 velocity;
    private bool isClimbing = false;
    private Vector3 lastHandPosition;
    private Vector3 lastMoveDirection;

    /// <summary>
    /// 手部代理类——用于跟踪攀爬手的状态
    /// </summary>
    [System.Serializable]
    public class ClimbHand
    {
        public XRBaseInteractor interactor;
        [HideInInspector] public bool isGrabbing;
        [HideInInspector] public Vector3 previousPosition;
        [HideInInspector] public Vector3 currentPosition;
    }

    private void Update()
    {
        bool leftActive = leftHand.isGrabbing;
        bool rightActive = rightHand.isGrabbing;
        
        if (leftActive || rightActive)
        {
            // 正在攀爬
            isClimbing = true;
            Vector3 moveDelta = Vector3.zero;
            
            if (leftActive && rightActive)
            {
                // 双手攀爬：取两只手的平均移动
                Vector3 leftMove = leftHand.currentPosition - leftHand.previousPosition;
                Vector3 rightMove = rightHand.currentPosition - rightHand.previousPosition;
                moveDelta = (leftMove + rightMove) * 0.5f;
            }
            else if (leftActive)
            {
                moveDelta = leftHand.currentPosition - leftHand.previousPosition;
            }
            else
            {
                moveDelta = rightHand.currentPosition - rightHand.previousPosition;
            }
            
            // 反向移动 XR Origin（手往上拉 = 人往上移）
            if (xrOrigin != null)
            {
                xrOrigin.position -= moveDelta * climbSpeed;
            }
            
            lastMoveDirection = -moveDelta.normalized;
            velocity = Vector3.zero; // 攀爬中重置速度
        }
        else if (isClimbing)
        {
            // 刚松手
            isClimbing = false;
            
            if (jumpForce > 0)
            {
                // 松手跳投：沿最后移动方向给予力
                velocity = lastMoveDirection * jumpForce;
            }
        }
        
        // 应用重力
        if (!isClimbing && useGravity)
        {
            velocity.y -= gravity * Time.deltaTime;
            if (xrOrigin != null)
            {
                xrOrigin.position += velocity * Time.deltaTime;
            }
        }
        
        // 更新手的位置记录
        leftHand.previousPosition = leftHand.currentPosition;
        rightHand.previousPosition = rightHand.currentPosition;
    }
}

/// <summary>
/// 可攀爬表面标记组件
/// 添加到可攀爬的墙壁、梯子等物体上
/// </summary>
[RequireComponent(typeof(Collider))]
public class ClimbableSurface : MonoBehaviour
{
    [Tooltip("攀爬摩擦力（影响攀爬手感）")]
    [Range(0.5f, 2.0f)]
    public float gripFactor = 1.0f;
    
    [Tooltip("允许在此表面上移动")]
    public bool allowMovement = true;
    
    [Tooltip("表面类型（用于音效/视觉区分）")]
    public SurfaceType surfaceType = SurfaceType.Wall;
    
    public enum SurfaceType
    {
        Wall,       // 墙壁
        Ladder,     // 梯子
        Rope,       // 绳索
        Rock,       // 岩石
    }
    
    private void Awake()
    {
        Collider col = GetComponent<Collider>();
        // 攀爬表面需要是触发器，这样手可以"穿入"
        // 但同时需要物理碰撞来检测接触
        // 实际做法：使用两个 Collider，一个物理的，一个触发器
    }
}
```

---

## 6. 旋转方式

VR 中的旋转有两种主要方式：

### Snap Turn（快转）

瞬间旋转一个固定角度（如 30° 或 45°），舒适度高：

```
XR Origin
├── Locomotion System
│   └── Snap Turn Provider
│       ├── Input Action = 右摇杆 X 轴
│       ├── Turn Amount = 45°
│       └── Debounce Time = 0.5s（防止连续触发）
```

### Continuous Turn（连续旋转）

像传统游戏一样连续旋转，更自然但舒适度较低：

```
XR Origin
└── Locomotion System
    └── Continuous Turn Provider
        ├── Input Action = 右摇杆 X 轴
        └── Turn Speed = 60°/s
```

### 混合旋转方案

```csharp
using UnityEngine;
using UnityEngine.InputSystem;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 自适应旋转控制器
/// 根据用户偏好或摇杆输入速度自动选择 Snap/Continuous
/// </summary>
public class AdaptiveTurnProvider : MonoBehaviour
{
    public enum TurnMode { Snap, Continuous, Auto }
    
    [Header("旋转设置")]
    public TurnMode turnMode = TurnMode.Snap;
    
    [Header("Snap Turn 设置")]
    public float snapAngle = 45f;
    public float snapDebounceTime = 0.3f;
    
    [Header("Continuous Turn 设置")]
    public float turnSpeed = 60f;
    
    [Header("Auto 模式设置")]
    [Tooltip("摇杆偏移量阈值：超过则用连续旋转")]
    [Range(0.1f, 0.9f)]
    public float continuousThreshold = 0.7f;
    
    [Header("输入")]
    public InputActionProperty turnAction;
    
    private SnapTurnProvider snapProvider;
    private ContinuousTurnProvider continuousProvider;
    private float lastSnapTime;
    private float currentTurnAngle;

    private void Awake()
    {
        snapProvider = GetComponent<SnapTurnProvider>();
        continuousProvider = GetComponent<ContinuousTurnProvider>();
        
        UpdateProviderState();
    }

    private void OnEnable()
    {
        turnAction.action?.Enable();
    }

    private void OnDisable()
    {
        turnAction.action?.Disable();
    }

    private void Update()
    {
        float input = turnAction.action?.ReadValue<Vector2>().x ?? 0f;
        
        switch (turnMode)
        {
            case TurnMode.Snap:
                HandleSnapTurn(input);
                break;
            case TurnMode.Continuous:
                HandleContinuousTurn(input);
                break;
            case TurnMode.Auto:
                HandleAutoTurn(input);
                break;
        }
    }

    private void HandleSnapTurn(float input)
    {
        if (Mathf.Abs(input) < 0.5f) return;
        if (Time.time - lastSnapTime < snapDebounceTime) return;
        
        float angle = Mathf.Sign(input) * snapAngle;
        transform.Rotate(0, angle, 0);
        lastSnapTime = Time.time;
        
        Debug.Log($"[旋转] Snap Turn: {angle}°");
    }

    private void HandleContinuousTurn(float input)
    {
        if (Mathf.Abs(input) < 0.1f) return;
        
        float angle = input * turnSpeed * Time.deltaTime;
        transform.Rotate(0, angle, 0);
    }

    private void HandleAutoTurn(float input)
    {
        if (Mathf.Abs(input) >= continuousThreshold)
        {
            // 大幅度输入 → 连续旋转
            float angle = input * turnSpeed * Time.deltaTime;
            transform.Rotate(0, angle, 0);
        }
        else if (Mathf.Abs(input) >= 0.5f)
        {
            // 中等幅度 → Snap Turn
            HandleSnapTurn(input);
        }
    }

    private void UpdateProviderState()
    {
        if (snapProvider != null)
            snapProvider.enabled = (turnMode == TurnMode.Snap);
        
        if (continuousProvider != null)
            continuousProvider.enabled = (turnMode == TurnMode.Continuous);
    }
}
```

---

## 7. 完整实战示例

### 搭建完整移动系统

#### 步骤 1：创建 Locomotion System

在 XR Origin 下创建空物体 `Locomotion System`，添加以下组件：
- `Locomotion System`（管理器）
- `Teleportation Provider`（传送）
- `Snap Turn Provider` 或 `Continuous Turn Provider`（旋转）

#### 步骤 2：配置传送

1. 地面添加 `Teleportation Area` 组件
2. 特定位置添加 `Teleportation Anchor` 组件
3. 控制器添加 `XR Ray Interactor`（选择 Teleport 交互层）

#### 步骤 3：配置平滑移动（可选）

1. 添加 `CharacterController` 到 XR Origin
2. 添加 `SmoothLocomotion` 脚本
3. 绑定摇杆输入

#### 步骤 4：添加攀爬（可选）

1. 可攀爬物体添加 `ClimbableSurface` 组件
2. 创建 `VRClimbing` 管理器
3. 在抓取事件中更新手部位置

### 移动方式切换菜单

```csharp
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// VR 设置面板——移动方式切换
/// 挂载在 VR UI 面板上
/// </summary>
public class LocomotionSettingsPanel : MonoBehaviour
{
    [Header("传送系统")]
    public TeleportationProvider teleportProvider;
    
    [Header("平滑移动")]
    public SmoothLocomotion smoothLocomotion;
    
    [Header("旋转")]
    public AdaptiveTurnProvider turnProvider;
    
    [Header("UI 元素")]
    public Toggle teleportToggle;
    public Toggle smoothMoveToggle;
    public Slider moveSpeedSlider;
    public Toggle snapTurnToggle;
    public Slider turnAmountSlider;

    private void Start()
    {
        // 初始化 UI 状态
        if (teleportToggle != null)
            teleportToggle.onValueChanged.AddListener(OnTeleportToggle);
        
        if (smoothMoveToggle != null)
            smoothMoveToggle.onValueChanged.AddListener(OnSmoothMoveToggle);
        
        if (moveSpeedSlider != null)
            moveSpeedSlider.onValueChanged.AddListener(OnMoveSpeedChanged);
        
        if (snapTurnToggle != null)
            snapTurnToggle.onValueChanged.AddListener(OnSnapTurnToggle);
    }

    private void OnTeleportToggle(bool enabled)
    {
        if (teleportProvider != null)
            teleportProvider.enabled = enabled;
    }

    private void OnSmoothMoveToggle(bool enabled)
    {
        if (smoothLocomotion != null)
            smoothLocomotion.enabled = enabled;
    }

    private void OnMoveSpeedChanged(float value)
    {
        if (smoothLocomotion != null)
            smoothLocomotion.moveSpeed = value;
    }

    private void OnSnapTurnToggle(bool enabled)
    {
        if (turnProvider != null)
            turnProvider.turnMode = enabled 
                ? AdaptiveTurnProvider.TurnMode.Snap 
                : AdaptiveTurnProvider.TurnMode.Continuous;
    }
}
```

---

## 8. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 传送不触发 | 射线没碰到 Teleportation Area | 检查层级（Layer）是否匹配 Interactor 的 Raycast Mask |
| 传送后面朝方向不对 | 没有使用 Teleportation Anchor | 使用 Anchor 指定朝向 |
| 平滑移动穿墙 | 没用 CharacterController | 使用 CharacterController.Move() 代替直接修改 position |
| 攀爬松手后浮空 | 没有应用重力 | 在非攀爬状态下持续应用重力 |
| 快转卡住不触发 | Debounce 时间太长 | 降低 debounce 时间到 0.2-0.3 秒 |
| 连续旋转太快 | Turn Speed 太高 | 降低到 45-90°/s |

---

## 9. 参考资源

- [Unity XR Interaction Toolkit - Locomotion](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- [Meta Locomotion Best Practices](https://developer.oculus.com/resources/locomotion-design/)
- [Valve SteamVR Plugin - Teleport](https://github.com/ValveSoftware/steamvr_unity_plugin)
- [Climbing in VR - GDC Talk](https://www.gdcvault.com/)
