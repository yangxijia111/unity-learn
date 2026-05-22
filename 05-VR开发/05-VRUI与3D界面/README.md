# 05-VR UI 与 3D 界面

> 在 VR 世界空间中构建直观的用户界面

---

## 目录

- [VR UI 设计理念](#vr-ui-设计理念)
- [World Space Canvas 配置](#world-space-canvas-配置)
- [3D UI 组件](#3d-ui-组件)
- [手柄激光指针交互 UI](#手柄激光指针交互-ui)
- [UI 动画与过渡效果](#ui-动画与过渡效果)
- [最佳实践](#最佳实践)
- [完整代码示例](#完整代码示例)

---

## VR UI 设计理念

### 世界空间 UI（World Space）

UI 直接存在于 3D 场景中，作为可交互的物体存在：

```
优点：
✅ 符合 VR 沉浸感
✅ 有深度感和空间感
✅ 可以与物理世界交互
✅ 玩家自然理解

缺点：
❌ 位置固定，可能被遮挡
❌ 远距离可读性差
❌ 占用 3D 空间
```

### 叠加 UI（Overlay）

UI 叠加在玩家视野上，类似传统屏幕 UI：

```
优点：
✅ 始终可见
✅ 可读性好
✅ 不被场景遮挡

缺点：
❌ 打破沉浸感
❌ 与 3D 世界割裂
❌ 可能引起不适（注视焦点冲突）
```

### 选择建议

| UI 类型 | 适用场景 |
|---------|----------|
| 世界空间 | 菜单面板、控制台、信息牌、HUD 浮动元素 |
| 叠加 UI | 暂停菜单、加载画面、重要提示 |
| 跟随 UI | 小地图、快捷栏、玩家状态（挂在手柄/手腕上） |

---

## World Space Canvas 配置

### 基础设置步骤

1. 创建 Canvas → Render Mode 选 `World Space`
2. 设置 Canvas 的 RectTransform 尺寸
3. 添加 `BoxCollider` 用于射线检测
4. 配置 EventSystem（XR 场景需要）

### 关键参数

```
Canvas Scaler:
  - Reference Resolution: 1920 x 1080（或根据需求调整）
  - Dynamic Pixels Per Unit: 10

Canvas（Rect Transform）:
  - Width: 根据内容，常用 0.5 ~ 2 米
  - Height: 根据内容，常用 0.3 ~ 1.5 米
  - Scale: 0.001 ~ 0.002（让像素转换为米）

Collider:
  - BoxCollider 大小与 Canvas 一致
  - 用于接收射线点击
```

### 层级结构示例

```
WorldSpaceCanvas (Canvas)
├── Panel
│   ├── Title (TextMeshPro)
│   ├── Button_Play
│   ├── Button_Settings
│   └── Button_Quit
├── BoxCollider（同尺寸，用于射线检测）
└── GraphicRaycaster
```

---

## 3D UI 组件

### 可交互按钮（物理反馈）

按钮在 VR 中需要更直观的视觉和触觉反馈：

```csharp
// VR 3D 按钮 —— 支持手柄射线和直接触摸
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using UnityEngine.XR.Interaction.Toolkit;

public class VR3DButton : MonoBehaviour, IPointerEnterHandler, IPointerClickHandler, IPointerExitHandler
{
    [Header("按钮设置")]
    public Image buttonImage;
    public TMPro.TextMeshProUGUI buttonText;
    public Color normalColor = Color.white;
    public Color hoverColor = new Color(0.9f, 0.9f, 1f);
    public Color pressedColor = new Color(0.7f, 0.7f, 0.8f);

    [Header("物理反馈")]
    public Transform buttonSurface;      // 按钮表面，用于按下位移
    public float pressDepth = 0.01f;      // 按下深度（米）
    public bool useHaptic = true;
    public float hapticIntensity = 0.3f;

    [Header("音效")]
    public AudioSource audioSource;
    public AudioClip hoverSound;
    public AudioClip clickSound;

    [Header("事件")]
    public UnityEngine.Events.UnityEvent OnButtonClicked;

    private Vector3 _restPosition;
    private bool _isPressed;

    void Start()
    {
        if (buttonSurface != null)
            _restPosition = buttonSurface.localPosition;
    }

    public void OnPointerEnter(PointerEventData eventData)
    {
        // 悬停效果
        if (buttonImage != null)
            buttonImage.color = hoverColor;

        if (audioSource != null && hoverSound != null)
            audioSource.PlayOneShot(hoverSound);

        // 触觉反馈
        if (useHaptic)
            TriggerHaptic(hapticIntensity * 0.5f);
    }

    public void OnPointerClick(PointerEventData eventData)
    {
        // 按下效果
        if (buttonImage != null)
            buttonImage.color = pressedColor;

        if (buttonSurface != null)
            buttonSurface.localPosition = _restPosition - Vector3.forward * pressDepth;

        if (audioSource != null && clickSound != null)
            audioSource.PlayOneShot(clickSound);

        if (useHaptic)
            TriggerHaptic(hapticIntensity);

        _isPressed = true;
        OnButtonClicked?.Invoke();

        // 延迟恢复
        Invoke(nameof(ResetButton), 0.15f);
    }

    public void OnPointerExit(PointerEventData eventData)
    {
        if (!_isPressed && buttonImage != null)
            buttonImage.color = normalColor;
    }

    void ResetButton()
    {
        _isPressed = false;
        if (buttonImage != null)
            buttonImage.color = normalColor;
        if (buttonSurface != null)
            buttonSurface.localPosition = _restPosition;
    }

    void TriggerHaptic(float intensity)
    {
        // 触发所有连接的手柄震动
        var devices = new System.Collections.Generic.List<UnityEngine.XR.InputDevice>();
        UnityEngine.XR.InputDevices.GetDevicesAtXRNode(UnityEngine.XR.XRNode.RightHand, devices);
        UnityEngine.XR.InputDevices.GetDevicesAtXRNode(UnityEngine.XR.XRNode.LeftHand, devices);

        foreach (var device in devices)
        {
            if (device.TryGetHapticCapabilities(out var capabilities) && capabilities.supportsImpulse)
                device.SendHapticImpulse(0, intensity, 0.1f);
        }
    }
}
```

### 面板与菜单

```csharp
// VR 菜单面板管理器
using UnityEngine;
using UnityEngine.UI;

public class VRMenuPanel : MonoBehaviour
{
    [Header("面板设置")]
    public CanvasGroup canvasGroup;
    public Transform panelTransform;
    public float fadeDuration = 0.3f;

    [Header("跟随设置")]
    public bool followPlayer = true;
    public Transform playerHead;
    public float followDistance = 1.5f;
    public float followHeight = 0f;       // 相对于眼睛的垂直偏移
    public float followSpeed = 5f;
    public bool facePlayer = true;

    private bool _isVisible;
    private Coroutine _fadeCoroutine;

    void Start()
    {
        // 初始隐藏
        canvasGroup.alpha = 0;
        canvasGroup.interactable = false;
        canvasGroup.blocksRaycasts = false;
        _isVisible = false;
    }

    void Update()
    {
        // 跟随玩家
        if (followPlayer && _isVisible && playerHead != null)
        {
            Vector3 targetPos = playerHead.position + playerHead.forward * followDistance;
            targetPos.y = playerHead.position.y + followHeight;

            panelTransform.position = Vector3.Lerp(
                panelTransform.position,
                targetPos,
                Time.deltaTime * followSpeed
            );

            if (facePlayer)
            {
                panelTransform.LookAt(new Vector3(
                    playerHead.position.x,
                    panelTransform.position.y,
                    playerHead.position.z
                ));
                panelTransform.Rotate(0, 180, 0); // 面向玩家
            }
        }
    }

    public void Show()
    {
        if (_fadeCoroutine != null) StopCoroutine(_fadeCoroutine);
        _fadeCoroutine = StartCoroutine(FadePanel(true));
    }

    public void Hide()
    {
        if (_fadeCoroutine != null) StopCoroutine(_fadeCoroutine);
        _fadeCoroutine = StartCoroutine(FadePanel(false));
    }

    public void Toggle()
    {
        if (_isVisible) Hide();
        else Show();
    }

    System.Collections.IEnumerator FadePanel(bool show)
    {
        float startAlpha = canvasGroup.alpha;
        float targetAlpha = show ? 1f : 0f;
        float elapsed = 0;

        while (elapsed < fadeDuration)
        {
            elapsed += Time.deltaTime;
            canvasGroup.alpha = Mathf.Lerp(startAlpha, targetAlpha, elapsed / fadeDuration);
            yield return null;
        }

        canvasGroup.alpha = targetAlpha;
        canvasGroup.interactable = show;
        canvasGroup.blocksRaycasts = show;
        _isVisible = show;
    }
}
```

### TextMeshPro 文本显示

在 VR 中使用 TextMeshPro 的关键配置：

```
TextMeshPro 设置建议：
──────────────────────
字体大小：根据距离计算
  - 1米距离：推荐 24-36 pt
  - 2米距离：推荐 48-72 pt
  - 公式：字号 ≈ 距离(米) × 30

字体：
  - 使用清晰无衬线字体
  - 避免过细的字重
  - 中文推荐：思源黑体、Noto Sans CJK

渲染：
  - SDF 渲染（默认）
  - Outline 宽度 0.1-0.2（增加可读性）
  - Underlay 偏移 (1, -1) 加阴影

材质：
  - 使用 TextMeshPro/Distance Field 
  - 或自定义 VR 优化材质
```

---

## 手柄激光指针交互 UI

### XR Ray Interactor 配置

XR Interaction Toolkit 自带射线交互器，用于 UI 交互：

```
XR Ray Interactor 设置：
────────────────────────
Line Type: Straight（UI 交互推荐）
Max Raycast Distance: 10-20米
Raycast Mask: 包含 UI 层

UI 交互要求：
- Canvas 上有 GraphicRaycaster
- Canvas 渲染模式为 World Space
- UI 元素勾选 Raycast Target
```

### 自定义射线视觉效果

```csharp
// 自定义激光指针效果
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

[RequireComponent(typeof(XRInteractorLineVisual))]
public class CustomLaserPointer : MonoBehaviour
{
    [Header("激光设置")]
    public Color defaultColor = new Color(0.8f, 0.8f, 1f, 0.6f);
    public Color hoverColor = new Color(1f, 0.8f, 0.3f, 0.8f);
    public Color clickColor = new Color(1f, 0.3f, 0.3f, 1f);

    [Header("末端光点")]
    public GameObject dotPrefab;
    public float dotScale = 0.02f;

    private XRInteractorLineVisual _lineVisual;
    private LineRenderer _lineRenderer;
    private GameObject _dotInstance;
    private Color _currentColor;

    void Start()
    {
        _lineVisual = GetComponent<XRInteractorLineVisual>();
        _lineRenderer = GetComponent<LineRenderer>();

        // 创建末端光点
        if (dotPrefab != null)
        {
            _dotInstance = Instantiate(dotPrefab);
            _dotInstance.transform.localScale = Vector3.one * dotScale;
        }

        SetColor(defaultColor);
    }

    public void SetColor(Color color)
    {
        _currentColor = color;
        if (_lineRenderer != null)
        {
            _lineRenderer.startColor = color;
            _lineRenderer.endColor = color * new Color(1, 1, 1, 0.3f);
        }
    }

    public void SetHover()
    {
        SetColor(hoverColor);
    }

    public void SetClick()
    {
        SetColor(clickColor);
    }

    public void UpdateDotPosition(Vector3 position, bool active)
    {
        if (_dotInstance != null)
        {
            _dotInstance.SetActive(active);
            _dotInstance.transform.position = position;
        }
    }
}
```

### UI 射线检测管理

```csharp
// 管理手柄射线与 UI 的交互状态
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

public class VRUIRayManager : MonoBehaviour
{
    [Header("射线交互器")]
    public XRBaseController leftController;
    public XRBaseController rightController;

    [Header("射线视觉")]
    public CustomLaserPointer leftLaser;
    public CustomLaserPointer rightLaser;

    [Header("震动设置")]
    public float uiHoverHaptic = 0.1f;
    public float uiClickHaptic = 0.4f;
    public float hapticDuration = 0.05f;

    private bool _leftHovering;
    private bool _rightHovering;

    /// <summary>
    /// 当射线悬停在 UI 上时调用
    /// </summary>
    public void OnUIHover(bool isRightHand)
    {
        if (isRightHand)
        {
            if (!_rightHovering)
            {
                _rightHovering = true;
                rightLaser?.SetHover();
                TriggerHaptic(rightController, uiHoverHaptic);
            }
        }
        else
        {
            if (!_leftHovering)
            {
                _leftHovering = true;
                leftLaser?.SetHover();
                TriggerHaptic(leftController, uiHoverHaptic);
            }
        }
    }

    /// <summary>
    /// 当射线离开 UI 时调用
    /// </summary>
    public void OnUIHoverExit(bool isRightHand)
    {
        if (isRightHand)
        {
            _rightHovering = false;
            rightLaser?.SetColor(rightLaser.defaultColor);
        }
        else
        {
            _leftHovering = false;
            leftLaser?.SetColor(leftLaser.defaultColor);
        }
    }

    /// <summary>
    /// 当射线点击 UI 时调用
    /// </summary>
    public void OnUIClick(bool isRightHand)
    {
        var laser = isRightHand ? rightLaser : leftLaser;
        var controller = isRightHand ? rightController : leftController;

        laser?.SetClick();
        TriggerHaptic(controller, uiClickHaptic);

        // 短暂延迟恢复颜色
        Invoke(nameof(ResetColors), 0.15f);
    }

    void ResetColors()
    {
        if (_rightHovering) rightLaser?.SetHover();
        else rightLaser?.SetColor(rightLaser.defaultColor);

        if (_leftHovering) leftLaser?.SetHover();
        else leftLaser?.SetColor(leftLaser.defaultColor);
    }

    void TriggerHaptic(XRBaseController controller, float intensity)
    {
        if (controller != null)
            controller.SendHapticImpulse(intensity, hapticDuration);
    }
}
```

---

## UI 动画与过渡效果

### 常用动画类型

```csharp
// VR UI 动画控制器 —— 支持多种动画效果
using UnityEngine;
using System.Collections;

public class VRUIAnimator : MonoBehaviour
{
    [Header("缩放动画")]
    public bool animateScale = true;
    public AnimationCurve scaleCurve = AnimationCurve.EaseInOut(0, 0, 1, 1);

    [Header("滑入动画")]
    public bool animateSlide = false;
    public Vector3 slideOffset = new Vector3(0, -0.2f, 0);

    [Header("通用设置")]
    public float duration = 0.3f;
    public bool useUnscaledTime = false;

    private Vector3 _originalScale;
    private Vector3 _originalPosition;
    private Coroutine _currentAnimation;

    void Awake()
    {
        _originalScale = transform.localScale;
        _originalPosition = transform.localPosition;
    }

    /// <summary>
    /// 弹出显示
    /// </summary>
    public void PopIn()
    {
        if (_currentAnimation != null) StopCoroutine(_currentAnimation);
        _currentAnimation = StartCoroutine(PopInAnimation());
    }

    /// <summary>
    /// 缩小消失
    /// </summary>
    public void PopOut()
    {
        if (_currentAnimation != null) StopCoroutine(_currentAnimation);
        _currentAnimation = StartCoroutine(PopOutAnimation());
    }

    /// <summary>
    /// 滑入显示
    /// </summary>
    public void SlideIn()
    {
        if (_currentAnimation != null) StopCoroutine(_currentAnimation);
        _currentAnimation = StartCoroutine(SlideInAnimation());
    }

    /// <summary>
    /// 滑出隐藏
    /// </summary>
    public void SlideOut()
    {
        if (_currentAnimation != null) StopCoroutine(_currentAnimation);
        _currentAnimation = StartCoroutine(SlideOutAnimation());
    }

    IEnumerator PopInAnimation()
    {
        float elapsed = 0;
        transform.localScale = Vector3.zero;
        gameObject.SetActive(true);

        while (elapsed < duration)
        {
            elapsed += DeltaTime();
            float t = scaleCurve.Evaluate(elapsed / duration);
            transform.localScale = _originalScale * t;
            yield return null;
        }

        transform.localScale = _originalScale;
    }

    IEnumerator PopOutAnimation()
    {
        float elapsed = 0;
        Vector3 startScale = transform.localScale;

        while (elapsed < duration)
        {
            elapsed += DeltaTime();
            float t = 1f - scaleCurve.Evaluate(elapsed / duration);
            transform.localScale = startScale * t;
            yield return null;
        }

        transform.localScale = Vector3.zero;
        gameObject.SetActive(false);
    }

    IEnumerator SlideInAnimation()
    {
        float elapsed = 0;
        Vector3 startPos = _originalPosition + slideOffset;
        transform.localPosition = startPos;

        if (animateScale)
            transform.localScale = _originalScale * 0.8f;

        gameObject.SetActive(true);

        while (elapsed < duration)
        {
            elapsed += DeltaTime();
            float t = elapsed / duration;
            float easedT = Mathf.SmoothStep(0, 1, t);

            transform.localPosition = Vector3.Lerp(startPos, _originalPosition, easedT);

            if (animateScale)
                transform.localScale = _originalScale * Mathf.Lerp(0.8f, 1f, easedT);

            yield return null;
        }

        transform.localPosition = _originalPosition;
        transform.localScale = _originalScale;
    }

    IEnumerator SlideOutAnimation()
    {
        float elapsed = 0;
        Vector3 startPos = transform.localPosition;
        Vector3 endPos = _originalPosition + slideOffset;

        while (elapsed < duration)
        {
            elapsed += DeltaTime();
            float t = Mathf.SmoothStep(0, 1, elapsed / duration);

            transform.localPosition = Vector3.Lerp(startPos, endPos, t);

            if (animateScale)
                transform.localScale = _originalScale * Mathf.Lerp(1f, 0.8f, t);

            yield return null;
        }

        gameObject.SetActive(false);
    }

    float DeltaTime()
    {
        return useUnscaledTime ? Time.unscaledDeltaTime : Time.deltaTime;
    }
}
```

### 高级动画：弹簧效果

```csharp
// 弹簧动画效果 —— 更自然的 VR 交互感
using UnityEngine;

public class SpringUIEffect : MonoBehaviour
{
    [Header("弹簧参数")]
    public float stiffness = 200f;     // 刚度
    public float damping = 10f;        // 阻尼
    public float mass = 1f;            // 质量

    [Header("限制")]
    public float maxDisplacement = 0.05f;

    private Vector3 _restPosition;
    private Vector3 _velocity;
    private bool _isSpringing;
    private Vector3 _displacement;

    void Start()
    {
        _restPosition = transform.localPosition;
    }

    /// <summary>
    /// 对 UI 施加冲击
    /// </summary>
    public void ApplyImpulse(Vector3 force)
    {
        _velocity += force / mass;
        _isSpringing = true;
    }

    /// <summary>
    /// 按钮按下弹跳
    /// </summary>
    public void Bounce()
    {
        ApplyImpulse(Vector3.forward * 0.02f);
    }

    void Update()
    {
        if (!_isSpringing) return;

        // 弹簧力
        Vector3 springForce = -stiffness * _displacement;
        // 阻尼力
        Vector3 dampingForce = -damping * _velocity;

        // 加速度
        Vector3 acceleration = (springForce + dampingForce) / mass;
        _velocity += acceleration * Time.deltaTime;
        _displacement += _velocity * Time.deltaTime;

        // 限制位移
        _displacement = Vector3.ClampMagnitude(_displacement, maxDisplacement);

        transform.localPosition = _restPosition + _displacement;

        // 停止检测
        if (_displacement.magnitude < 0.0001f && _velocity.magnitude < 0.0001f)
        {
            _isSpringing = false;
            _displacement = Vector3.zero;
            _velocity = Vector3.zero;
            transform.localPosition = _restPosition;
        }
    }
}
```

---

## 最佳实践

### 字体与可读性

```
距离与字体大小对照：
──────────────────────
距离(米) │ 推荐字号(pt) │ 最小字号
  0.5    │   18-24      │   14
  1.0    │   30-36      │   24
  1.5    │   42-54      │   36
  2.0    │   54-72      │   48
  3.0    │   80-100     │   72

字体选择：
─────────
✅ 无衬线字体（Sans-serif）
✅ 中等或粗字重
✅ 避免过细或装饰性字体
✅ 中文推荐：思源黑体、阿里巴巴普惠体

对比度：
─────────
✅ 文字与背景对比度 ≥ 4.5:1（WCAG AA）
✅ 推荐 ≥ 7:1（WCAG AAA）
✅ 深色模式：白字 + 半透明深色背景
✅ 浅色模式：深色字 + 半透明白色背景
```

### 位置与布局

```
UI 放置建议：
──────────────
菜单面板：
  - 距离玩家 1.0 ~ 2.0 米
  - 高度与视线平齐或略低（-15° 以内）
  - 略微倾斜面向玩家（5-10°）

固定 UI（如 HUD）：
  - 距离 0.5 ~ 0.8 米
  - 边缘位置，不遮挡视野中心
  - 尺寸适中，不压迫视野

手腕 UI：
  - 绑定在非主手手腕
  - 仅在翻腕查看时显示
  - 小而精简

布局原则：
  - 按钮尺寸 ≥ 5cm x 5cm（实际空间）
  - 按钮间距 ≥ 1cm
  - 最多 5-7 个并列选项
  - 使用图标 + 文字，不要纯文字按钮
```

### 性能建议

```
VR UI 性能注意事项：
────────────────────
✅ 合批 UI Canvas（相同 Canvas 的元素一起合批）
✅ 减少 Canvas 数量（每个 Canvas 独立渲染）
✅ 避免频繁更新 TextMeshPro（缓存文本）
✅ 少用透明和模糊效果
✅ 使用 CanvasGroup 统一控制透明度
❌ 避免每一帧都更新 UI
❌ 避免大量动态 UI 元素
❌ 避免过度使用 Particle System 作为 UI 特效
```

---

## 完整代码示例

### 综合 VR UI 系统

```csharp
// VRUIManager.cs —— 完整的 VR UI 管理系统
using UnityEngine;
using UnityEngine.UI;
using System.Collections;
using System.Collections.Generic;

public class VRUIManager : MonoBehaviour
{
    // === 单例 ===
    public static VRUIManager Instance { get; private set; }

    [Header("UI 面板")]
    public VRMenuPanel mainMenu;
    public VRMenuPanel settingsMenu;
    public VRMenuPanel pauseMenu;
    public VRMenuPanel infoPanel;

    [Header("通知系统")]
    public Transform notificationParent;
    public GameObject notificationPrefab;
    public float notificationDuration = 3f;
    public int maxNotifications = 3;

    [Header("确认对话框")]
    public VRMenuPanel confirmDialog;
    public TMPro.TextMeshProUGUI confirmText;
    public UnityEngine.UI.Button confirmYesBtn;
    public UnityEngine.UI.Button confirmNoBtn;

    [Header("玩家参考")]
    public Transform playerHead;

    private List<GameObject> _activeNotifications = new List<GameObject>();
    private System.Action _confirmCallback;

    void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
    }

    void Start()
    {
        // 设置玩家头部引用
        if (playerHead == null)
        {
            var camera = Camera.main;
            if (camera != null) playerHead = camera.transform;
        }

        // 传递给各面板
        if (mainMenu != null) mainMenu.playerHead = playerHead;
        if (settingsMenu != null) settingsMenu.playerHead = playerHead;
        if (pauseMenu != null) pauseMenu.playerHead = playerHead;

        // 确认对话框按钮绑定
        if (confirmYesBtn != null)
            confirmYesBtn.onClick.AddListener(OnConfirmYes);
        if (confirmNoBtn != null)
            confirmNoBtn.onClick.AddListener(OnConfirmNo);

        // 初始关闭所有面板
        CloseAllPanels();
    }

    // === 面板控制 ===

    public void ShowMainMenu() { CloseAllPanels(); mainMenu?.Show(); }
    public void ShowSettings() { settingsMenu?.Show(); }
    public void ShowPauseMenu() { CloseAllPanels(); pauseMenu?.Show(); }
    public void ShowInfoPanel() { infoPanel?.Show(); }

    public void CloseAllPanels()
    {
        mainMenu?.Hide();
        settingsMenu?.Hide();
        pauseMenu?.Hide();
        infoPanel?.Hide();
        confirmDialog?.Hide();
    }

    public void ToggleMainMenu()
    {
        if (mainMenu != null)
            mainMenu.Toggle();
    }

    // === 确认对话框 ===

    public void ShowConfirmDialog(string message, System.Action onConfirm)
    {
        if (confirmText != null)
            confirmText.text = message;

        _confirmCallback = onConfirm;
        confirmDialog?.Show();
    }

    void OnConfirmYes()
    {
        _confirmCallback?.Invoke();
        confirmDialog?.Hide();
        _confirmCallback = null;
    }

    void OnConfirmNo()
    {
        confirmDialog?.Hide();
        _confirmCallback = null;
    }

    // === 通知系统 ===

    public void ShowNotification(string message)
    {
        if (notificationPrefab == null || notificationParent == null) return;

        // 移除超出数量的通知
        while (_activeNotifications.Count >= maxNotifications)
        {
            Destroy(_activeNotifications[0]);
            _activeNotifications.RemoveAt(0);
        }

        // 创建新通知
        GameObject notification = Instantiate(notificationPrefab, notificationParent);
        notification.SetActive(true);

        var text = notification.GetComponentInChildren<TMPro.TextMeshProUGUI>();
        if (text != null) text.text = message;

        var animator = notification.GetComponent<VRUIAnimator>();
        if (animator != null) animator.PopIn();

        _activeNotifications.Add(notification);

        // 自动消失
        StartCoroutine(RemoveNotificationAfterDelay(notification, notificationDuration));
    }

    IEnumerator RemoveNotificationAfterDelay(GameObject notification, float delay)
    {
        yield return new WaitForSeconds(delay);

        if (notification != null)
        {
            var animator = notification.GetComponent<VRUIAnimator>();
            if (animator != null)
            {
                animator.PopOut();
                yield return new WaitForSeconds(animator.duration);
            }

            _activeNotifications.Remove(notification);
            Destroy(notification);
        }
    }
}
```

---

## 参考资源

- [Unity UI Manual](https://docs.unity3d.com/Manual/UIToolkits.html)
- [TextMeshPro Documentation](https://docs.unity3d.com/Packages/com.unity.textmeshpro@latest)
- [XR Interaction Toolkit - UI](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- [VR UI Best Practices - Oculus](https://developer.oculus.com/resources/bp-uidev/)

---

> 下一节：[06-VR性能优化](../06-VR性能优化/)
