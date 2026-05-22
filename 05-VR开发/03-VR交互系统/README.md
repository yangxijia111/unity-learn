# 03-VR交互系统

## 目录

- [1. VR 交互设计原则](#1-vr-交互设计原则)
- [2. 手部追踪 vs 手柄控制](#2-手部追踪-vs-手柄控制)
- [3. 抓取系统详解](#3-抓取系统详解)
- [4. 物理交互](#4-物理交互)
- [5. 手势识别基础](#5-手势识别基础)
- [6. 完整实战示例](#6-完整实战示例)
- [7. 参考资源](#7-参考资源)

---

## 1. VR 交互设计原则

VR 交互与传统 2D 屏幕交互有本质区别。好的 VR 交互应该让用户感觉"自然"，而不是在学一套复杂的操作逻辑。

### 三大核心原则

| 原则 | 说明 | 反面例子 |
|------|------|----------|
| **直觉性** | 用户看到一个物体，不需要教程就知道怎么操作——能抓的就伸手抓，能按的就伸手按 | 需要按特定组合键才能拿起杯子 |
| **反馈感** | 每个操作都应该有即时的视觉、听觉或触觉反馈，让用户确认"我做了什么" | 抓起物体没有振动、没有任何提示 |
| **舒适度** | 避免引发晕动症，不要强制移动用户视角，减少加速/减速 | 强制推镜头、快速平移 |

### 设计要点

- **手的位置要可见**：无论是虚拟手模型还是手柄模型，用户必须随时看到自己的手在哪里
- **交互距离**：大部分交互应该在手臂能触及的范围内（约 0.5m-2m）
- **避免 UI 悬浮**：VR 中的文字 UI 应该附着在物体上，而不是漂浮在空中
- **给用户选择权**：提供多种移动方式、多种交互方式，让用户选自己舒服的

---

## 2. 手部追踪 vs 手柄控制

### 手柄控制（Controller-based）

**优点：**
- 精度高，按键反馈明确
- 几乎所有 VR 设备都支持
- 开发相对简单，已有成熟框架（XR Interaction Toolkit）

**缺点：**
- 手指级交互受限（无法做精细手势）
- 增加了物理设备的认知负担

**典型场景：** 射击游戏、运动游戏、需要精确操作的场景

### 手部追踪（Hand Tracking）

**优点：**
- 最自然的交互方式，不需要额外硬件
- 支持精细手势（捏、指、握拳等）
- 适合社交 VR、教育类应用

**缺点：**
- 精度不如手柄（尤其是快速运动时）
- 遮挡问题（一只手挡住了另一只手的摄像头视野）
- 缺少物理反馈（按钮感、振动）

**典型场景：** 社交应用、虚拟培训、手势菜单

### XR Toolkit 中的输入适配

Unity XR Interaction Toolkit 通过 **XR Controller** 组件抽象了输入设备差异：

```
XR Controller (Action-based)
├── Left Hand / Right Hand
│   ├── Position Action → 控制手的位置
│   ├── Rotation Action → 控制手的旋转
│   ├── Select Action → 抓取/选择
│   ├── Activate Action → 激活（如开枪）
│   └── UI Press Action → UI 点击
└── 可以同时绑定手柄和手部追踪作为输入源
```

---

## 3. 抓取系统详解

抓取是 VR 中最核心的交互之一。XR Interaction Toolkit 提供了 **XR Grab Interactable** 组件来处理抓取逻辑。

### 抓取方式分类

#### 即时抓取 vs 跟随抓取

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **即时抓取（Instant）** | 物体瞬间跳到手上 | UI 元素、小道具 |
| **跟随抓取（Velocity Tracking）** | 物体跟随手部运动，有物理延迟感 | 需要投掷的物体、物理交互 |

#### 单手抓取 vs 双手抓取

- **单手抓取**：一只手抓住物体，物体跟随手移动和旋转
- **双手抓取**：两只手同时抓住物体，支持缩放和旋转（如拉弓弦、端枪）

#### 抓取点配置

通过 **Attach Transform** 来定义物体被抓取时的位置和朝向：

```
物体 (XR Grab Interactable)
├── Attach Transform（空 GameObject）
│   └── 位置 = 抓取点（比如杯子的把手位置）
│       旋转 = 抓取时手的朝向
```

### 投掷力学

XR Grab Interactable 内置了投掷速度计算。关键参数：

- **Throw Smoothing**：平滑手部释放时的速度数据
- **Throw Velocity Scale**：投掷速度缩放（默认 1.0，可以调大增强投掷力度感）
- **Throw Angular Velocity Scale**：旋转速度缩放

### 核心代码示例

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 自定义抓取行为——让物体被抓取时播放动画和音效
/// </summary>
[RequireComponent(typeof(XRGrabInteractable))]
public class CustomGrabHandler : MonoBehaviour
{
    [Header("抓取设置")]
    [Tooltip("抓取时播放的音效")]
    public AudioClip grabSound;
    
    [Tooltip("抓取时的粒子效果")]
    public ParticleSystem grabEffect;
    
    [Tooltip("是否允许投掷")]
    public bool allowThrow = true;
    
    [Tooltip("投掷力倍率")]
    [Range(0.5f, 3.0f)]
    public float throwForceMultiplier = 1.0f;

    private XRGrabInteractable grabInteractable;
    private AudioSource audioSource;
    private Rigidbody rb;

    private void Awake()
    {
        grabInteractable = GetComponent<XRGrabInteractable>();
        rb = GetComponent<Rigidbody>();
        audioSource = GetComponent<AudioSource>();
        
        if (audioSource == null)
            audioSource = gameObject.AddComponent<AudioSource>();
        
        // spatialBlend = 1.0 表示 3D 空间音效
        audioSource.spatialBlend = 1.0f;
    }

    private void OnEnable()
    {
        // 订阅 XR Interaction Toolkit 的事件
        grabInteractable.selectEntered.AddListener(OnGrab);
        grabInteractable.selectExited.AddListener(OnRelease);
        grabInteractable.activated.AddListener(OnActivate);
    }

    private void OnDisable()
    {
        grabInteractable.selectEntered.RemoveListener(OnGrab);
        grabInteractable.selectExited.RemoveListener(OnRelease);
        grabInteractable.activated.RemoveListener(OnActivate);
    }

    /// <summary>
    /// 抓取时触发
    /// </summary>
    private void OnGrab(SelectEnterEventArgs args)
    {
        // 播放抓取音效
        if (grabSound != null)
            audioSource.PlayOneShot(grabSound);

        // 播放粒子效果
        if (grabEffect != null)
            grabEffect.Play();

        // 设置投掷力度
        if (!allowThrow)
            grabInteractable.throwOnDetach = false;
        else
        {
            grabInteractable.throwOnDetach = true;
            grabInteractable.throwSmoothingDuration = 0.15f; // 投掷平滑时间
            grabInteractable.throwVelocityScale = throwForceMultiplier;
        }

        Debug.Log($"[抓取] 抓住了 {gameObject.name}");
    }

    /// <summary>
    /// 释放时触发
    /// </summary>
    private void OnRelease(SelectExitEventArgs args)
    {
        Debug.Log($"[抓取] 放下了 {gameObject.name}");
    }

    /// <summary>
    /// 激活时触发（如按下手柄扳机键）
    /// </summary>
    private void OnActivate(ActivateEventArgs args)
    {
        Debug.Log($"[激活] 对 {gameObject.name} 执行了激活操作");
        // 可以在这里实现"使用"逻辑，比如按下按钮、发射武器等
    }
}
```

---

## 4. 物理交互

VR 中的物理交互让虚拟世界更有真实感。XR Toolkit 提供了 Hinge Joint、Socket 等支持，但很多物理交互需要自己实现。

### 可推拉的门/抽屉

使用 **Configurable Joint** 实现门的推拉效果：

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 可推拉的门/抽屉
/// 使用 Configurable Joint 限制移动范围
/// </summary>
[RequireComponent(typeof(XRGrabInteractable))]
public class SlidableObject : MonoBehaviour
{
    [Header("滑动设置")]
    [Tooltip("滑动方向（本地坐标系）")]
    public Vector3 slideDirection = Vector3.forward;
    
    [Tooltip("最小位置")]
    public float minPosition = 0f;
    
    [Tooltip("最大位置")]
    public float maxPosition = 0.5f;
    
    [Tooltip("弹簧回弹力度（0 = 不回弹）")]
    public float springForce = 0f;
    
    [Tooltip("阻尼")]
    public float damping = 10f;

    private XRGrabInteractable grabInteractable;
    private Vector3 startPosition;
    private float currentSlide;

    private void Awake()
    {
        grabInteractable = GetComponent<XRGrabInteractable>();
        startPosition = transform.localPosition;
        
        SetupJoint();
    }

    private void SetupJoint()
    {
        // 如果没有刚体，添加一个
        Rigidbody rb = GetComponent<Rigidbody>();
        if (rb == null)
            rb = gameObject.AddComponent<Rigidbody>();
        
        rb.useGravity = false; // 不受重力影响
        rb.isKinematic = false;
    }

    private void FixedUpdate()
    {
        // 把物体限制在滑动轴上
        Vector3 offset = transform.localPosition - startPosition;
        float projection = Vector3.Dot(offset, slideDirection.normalized);
        
        // 超出范围则拉回
        if (projection < minPosition || projection > maxPosition)
        {
            float clamped = Mathf.Clamp(projection, minPosition, maxPosition);
            Vector3 targetPos = startPosition + slideDirection.normalized * clamped;
            
            if (springForce > 0)
            {
                // 弹簧回弹
                Vector3 springDir = (targetPos - transform.localPosition) * springForce;
                GetComponent<Rigidbody>().AddForce(springDir - GetComponent<Rigidbody>().linearVelocity * damping);
            }
            else
            {
                transform.localPosition = targetPos;
            }
        }
        
        currentSlide = Mathf.Clamp(projection, minPosition, maxPosition);
    }

    /// <summary>
    /// 获取当前滑动进度（0-1）
    /// </summary>
    public float GetSlideProgress()
    {
        return Mathf.InverseLerp(minPosition, maxPosition, currentSlide);
    }
}
```

### 可旋转的旋钮

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 可旋转的旋钮（如音量旋钮、门把手）
/// 限制在单一旋转轴上
/// </summary>
[RequireComponent(typeof(XRGrabInteractable))]
public class RotatableKnob : MonoBehaviour
{
    [Header("旋转设置")]
    [Tooltip("旋转轴")]
    public Vector3 rotationAxis = Vector3.up;
    
    [Tooltip("最小角度")]
    public float minAngle = 0f;
    
    [Tooltip("最大角度")]
    public float maxAngle = 270f;
    
    [Tooltip("是否使用离散档位")]
    public bool useSteps = false;
    
    [Tooltip("档位数量")]
    public int stepCount = 10;

    private XRGrabInteractable grabInteractable;
    private float currentAngle;
    private Quaternion initialRotation;

    /// <summary>
    /// 当前旋钮值（0-1）
    /// </summary>
    public float Value => Mathf.InverseLerp(minAngle, maxAngle, currentAngle);

    private void Awake()
    {
        grabInteractable = GetComponent<XRGrabInteractable>();
        initialRotation = transform.localRotation;
        currentAngle = minAngle;
    }

    private void Update()
    {
        // 将旋转限制在指定轴上
        Vector3 euler = transform.localRotation.eulerAngles;
        
        // Unity 的 euler angles 有万向锁问题，简单处理：
        // 取旋转轴对应的分量
        float angle = 0f;
        if (rotationAxis == Vector3.up) angle = euler.y;
        else if (rotationAxis == Vector3.right) angle = euler.x;
        else if (rotationAxis == Vector3.forward) angle = euler.z;
        
        // 处理角度环绕（360°→0°）
        if (angle > 180f) angle -= 360f;
        
        // 限制范围
        angle = Mathf.Clamp(angle, minAngle, maxAngle);
        
        // 如果启用档位，对齐到最近的档位
        if (useSteps && stepCount > 0)
        {
            float stepSize = (maxAngle - minAngle) / stepCount;
            angle = Mathf.Round(angle / stepSize) * stepSize;
        }
        
        currentAngle = angle;
        
        // 重新应用受限的旋转
        Vector3 clampedEuler = Vector3.zero;
        if (rotationAxis == Vector3.up) clampedEuler.y = angle;
        else if (rotationAxis == Vector3.right) clampedEuler.x = angle;
        else if (rotationAxis == Vector3.forward) clampedEuler.z = angle;
        
        transform.localRotation = initialRotation * Quaternion.Euler(clampedEuler);
    }
}
```

### 可按压的按钮

```csharp
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 可按压的 VR 按钮
/// 支持物理下沉效果和事件回调
/// </summary>
public class VRPushButton : MonoBehaviour
{
    [Header("按钮设置")]
    [Tooltip("按钮按下的深度")]
    public float pressDepth = 0.02f;
    
    [Tooltip("按钮复位速度")]
    public float returnSpeed = 5f;
    
    [Tooltip("触发阈值（0-1）")]
    [Range(0.5f, 1.0f)]
    public float triggerThreshold = 0.8f;
    
    [Tooltip("是否需要手离开才能再次触发")]
    public bool requireRelease = true;
    
    [Header("事件")]
    public UnityEvent onPressed;
    public UnityEvent onReleased;
    
    [Header("反馈")]
    [Tooltip("按下音效")]
    public AudioClip pressSound;
    
    [Tooltip("释放音效")]
    public AudioClip releaseSound;

    private Vector3 restPosition;
    private Vector3 pressedPosition;
    private bool isPressed = false;
    private bool canTrigger = true;
    private AudioSource audioSource;
    private float pressProgress;

    private void Awake()
    {
        restPosition = transform.localPosition;
        pressedPosition = restPosition - transform.up * pressDepth;
        
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null)
            audioSource = gameObject.AddComponent<AudioSource>();
        
        audioSource.spatialBlend = 1.0f;
        audioSource.playOnAwake = false;
    }

    private void OnTriggerStay(Collider other)
    {
        // 检测是否有手或控制器在按钮上方
        // 简单实现：用碰撞体的 Y 位置来判断按压深度
        Vector3 localPos = transform.parent.InverseTransformPoint(other.transform.position);
        float distance = Vector3.Distance(localPos, restPosition);
        
        // 根据距离计算按压进度
        pressProgress = Mathf.Clamp01(distance / (pressDepth * 2));
        
        if (pressProgress >= triggerThreshold && canTrigger && !isPressed)
        {
            Press();
        }
    }

    private void OnTriggerExit(Collider other)
    {
        if (requireRelease)
            canTrigger = true;
        
        if (isPressed)
            Release();
    }

    private void Update()
    {
        // 平滑复位
        if (!isPressed)
        {
            transform.localPosition = Vector3.Lerp(
                transform.localPosition, 
                restPosition, 
                returnSpeed * Time.deltaTime
            );
        }
        else
        {
            transform.localPosition = pressedPosition;
        }
    }

    private void Press()
    {
        isPressed = true;
        canTrigger = !requireRelease;
        
        if (pressSound != null)
            audioSource.PlayOneShot(pressSound);
        
        onPressed?.Invoke();
        Debug.Log($"[按钮] {gameObject.name} 被按下");
    }

    private void Release()
    {
        isPressed = false;
        
        if (releaseSound != null)
            audioSource.PlayOneShot(releaseSound);
        
        onReleased?.Invoke();
        Debug.Log($"[按钮] {gameObject.name} 被释放");
    }
}
```

---

## 5. 手势识别基础

手势识别在手部追踪模式下尤为重要。常见手势包括：

### 常见手势

| 手势 | 用途 | 检测方式 |
|------|------|----------|
| **捏合（Pinch）** | 选择、拾取小物体 | 拇指尖与食指尖距离 < 阈值 |
| **握拳（Fist）** | 抓取大物体 | 所有手指弯曲度 > 阈值 |
| **指向（Point）** | 菜单选择、激光指示 | 仅食指伸直，其他手指弯曲 |
| **张开（Open）** | 释放、展示 | 手指全部伸展 |
| **OK 手势** | 确认操作 | 拇指与食指形成圆圈 |

### 手势检测实现

```csharp
using UnityEngine;
using UnityEngine.Events;
using UnityEngine.XR.Hands;
using System.Collections.Generic;

/// <summary>
/// 基于 XR Hands 的手势检测器
/// 需要 Unity XR Hands 包（com.unity.xr.hands）
/// </summary>
public class GestureDetector : MonoBehaviour
{
    public enum HandType { Left, Right }
    
    [Header("检测设置")]
    public HandType hand = HandType.Right;
    
    [Tooltip("捏合距离阈值（米）")]
    public float pinchThreshold = 0.02f;
    
    [Tooltip("手指弯曲阈值")]
    public float curlThreshold = 0.7f;
    
    [Header("手势事件")]
    public UnityEvent onPinch;
    public UnityEvent onPinchEnd;
    public UnityEvent onFist;
    public UnityEvent onFistEnd;
    public UnityEvent onPoint;
    public UnityEvent onPointEnd;
    
    private XRHandSubsystem handSubsystem;
    private bool wasPinching = false;
    private bool wasFist = false;
    private bool wasPointing = false;

    private void OnEnable()
    {
        // 获取 XR Hand Subsystem
        var subsystems = new List<XRHandSubsystem>();
        SubsystemManager.GetSubsystems(subsystems);
        
        if (subsystems.Count > 0)
        {
            handSubsystem = subsystems[0];
            handSubsystem.updatedHands += OnHandsUpdated;
        }
    }

    private void OnDisable()
    {
        if (handSubsystem != null)
            handSubsystem.updatedHands -= OnHandsUpdated;
    }

    private void OnHandsUpdated(XRHandSubsystem subsystem,
        XRHandSubsystem.UpdateSuccessFlags updateFlags,
        XRHandSubsystem.UpdateType updateType)
    {
        // 获取对应的手
        XRHand handData = (hand == HandType.Left) 
            ? subsystem.leftHand 
            : subsystem.rightHand;

        if (!handData.isTracked)
            return;

        // 检测各种手势
        DetectPinch(handData);
        DetectFist(handData);
        DetectPoint(handData);
    }

    /// <summary>
    /// 检测捏合手势：拇指尖和食指尖的距离
    /// </summary>
    private void DetectPinch(XRHand handData)
    {
        if (handData.GetJoint(XRHandJointID.ThumbTip).TryGetPose(out Pose thumbTip) &&
            handData.GetJoint(XRHandJointID.IndexTip).TryGetPose(out Pose indexTip))
        {
            float distance = Vector3.Distance(thumbTip.position, indexTip.position);
            bool isPinching = distance < pinchThreshold;

            if (isPinching && !wasPinching)
            {
                onPinch?.Invoke();
                Debug.Log($"[{hand}手] 捏合开始");
            }
            else if (!isPinching && wasPinching)
            {
                onPinchEnd?.Invoke();
                Debug.Log($"[{hand}手] 捏合结束");
            }

            wasPinching = isPinching;
        }
    }

    /// <summary>
    /// 检测握拳：所有手指尖靠近手掌
    /// </summary>
    private void DetectFist(XRHand handData)
    {
        // 简化实现：检查食指、中指、无名指、小指的指尖到手掌的距离
        var fingerTips = new XRHandJointID[]
        {
            XRHandJointID.IndexTip,
            XRHandJointID.MiddleTip,
            XRHandJointID.RingTip,
            XRHandJointID.LittleTip
        };

        if (!handData.GetJoint(XRHandJointID.Palm).TryGetPose(out Pose palmPose))
            return;

        float totalCurl = 0f;
        int validFingers = 0;

        foreach (var tipId in fingerTips)
        {
            if (handData.GetJoint(tipId).TryGetPose(out Pose tipPose))
            {
                float distance = Vector3.Distance(tipPose.position, palmPose.position);
                // 距离越小说明手指越弯曲
                // 假设完全伸展距离约 0.1m，完全弯曲约 0.03m
                float curl = Mathf.InverseLerp(0.1f, 0.03f, distance);
                totalCurl += curl;
                validFingers++;
            }
        }

        if (validFingers == 0) return;

        float avgCurl = totalCurl / validFingers;
        bool isFist = avgCurl > curlThreshold;

        if (isFist && !wasFist)
        {
            onFist?.Invoke();
            Debug.Log($"[{hand}手] 握拳");
        }
        else if (!isFist && wasFist)
        {
            onFistEnd?.Invoke();
            Debug.Log($"[{hand}手] 松开");
        }

        wasFist = isFist;
    }

    /// <summary>
    /// 检测指向：仅食指伸直
    /// </summary>
    private void DetectPoint(XRHand handData)
    {
        // 简化：检查食指是否伸直，其他手指是否弯曲
        if (!handData.GetJoint(XRHandJointID.Palm).TryGetPose(out Pose palmPose))
            return;

        bool indexExtended = false;
        bool othersCurl = true;

        // 食指
        if (handData.GetJoint(XRHandJointID.IndexTip).TryGetPose(out Pose indexTip))
        {
            float dist = Vector3.Distance(indexTip.position, palmPose.position);
            indexExtended = dist > 0.08f;
        }

        // 中指
        if (handData.GetJoint(XRHandJointID.MiddleTip).TryGetPose(out Pose middleTip))
        {
            float dist = Vector3.Distance(middleTip.position, palmPose.position);
            if (dist > 0.06f) othersCurl = false;
        }

        bool isPointing = indexExtended && othersCurl;

        if (isPointing && !wasPointing)
        {
            onPoint?.Invoke();
            Debug.Log($"[{hand}手] 指向");
        }
        else if (!isPointing && wasPointing)
        {
            onPointEnd?.Invoke();
            Debug.Log($"[{hand}手] 取消指向");
        }

        wasPointing = isPointing;
    }
}
```

---

## 6. 完整实战示例

### 场景搭建步骤

1. **创建 VR 场景**
   - 添加 XR Origin (XR Interaction Setup)
   - 添加手柄/手部模型
   - 放置可交互物体

2. **配置抓取物体**
   - 添加 Rigidbody 组件
   - 添加 Collider
   - 添加 XR Grab Interactable 组件
   - 创建 Attach Transform 定义抓取点

3. **配置物理交互**
   - 门：Hinge Joint 或 Configurable Joint + SlidableObject
   - 按钮：VRPushButton + 触发器 Collider
   - 旋钮：RotatableKnob + XR Grab Interactable

4. **添加反馈**
   - 音效（AudioSource + 3D Spatial Blend）
   - 粒子效果
   - 手柄振动（XR Haptic Impulse）

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 物体被抓取后穿模 | Collider 太小或物体移动太快 | 增大 Collider，或设置 Rigidbody 为 Continuous Dynamic |
| 双手抓取物体抖动 | 两只手的旋转冲突 | 使用 Two-Handed Grab 配置，明确主手/副手 |
| 按钮不触发 | 触发器 Collider 太小 | 确保触发器覆盖整个按钮区域 |
| 旋钮旋转异常 | 万向锁或轴向配置错误 | 使用 Configurable Joint 限制单轴旋转 |

---

## 7. 参考资源

- [Unity XR Interaction Toolkit 文档](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- [XR Hands 包文档](https://docs.unity3d.com/Packages/com.unity.xr.hands@latest)
- [Meta Interaction SDK](https://developer.oculus.com/documentation/unity/unity-ovrinput/)
- [Unity VR Best Practices](https://developer.oculus.com/resources/bp-302-pp/)
