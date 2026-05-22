# UI 系统（UGUI）

## 📋 目录

- [UGUI 概述](#ugui-概述)
- [Canvas 画布](#canvas-画布)
- [EventSystem 事件系统](#eventsystem-事件系统)
- [核心 UI 组件](#核心-ui-组件)
- [Anchor 锚点系统](#anchor-锚点系统)
- [Layout Group 布局](#layout-group-布局)
- [事件绑定](#事件绑定)
- [完整示例](#完整示例)

---

## UGUI 概述

**UGUI**（Unity GUI）是 Unity 内置的 UI 系统，基于 Canvas 渲染：

```
创建 UI 的方式：
1. Hierarchy → 右键 → UI → 选择组件
2. 自动创建 Canvas 和 EventSystem（如果没有）

UI 层级结构：
Canvas（画布）
  ├── Panel（面板）
  │     ├── Text（文字）
  │     ├── Button（按钮）
  │     └── Image（图片）
  └── Panel2
        ├── Slider（滑块）
        └── InputField（输入框）
```

---

## Canvas 画布

Canvas 是所有 UI 元素的根容器：

```
Canvas 渲染模式：

1. Screen Space - Overlay（最常用）
   - UI 直接覆盖在屏幕上
   - 不需要摄像机
   - 适合：菜单、HUD

2. Screen Space - Camera
   - UI 渲染在指定摄像机前方
   - UI 可以有 3D 透视效果
   - 适合：有 3D 效果的 UI

3. World Space
   - UI 作为场景中的 3D 物体
   - 可以自由移动旋转
   - 适合：血条、3D 按钮、VR UI
```

```csharp
using UnityEngine;

public class CanvasControl : MonoBehaviour
{
    private Canvas canvas;

    void Start()
    {
        canvas = GetComponent<Canvas>();

        // Canvas 属性
        canvas.renderMode = RenderMode.ScreenSpaceOverlay;
        canvas.sortingOrder = 0;              // 多个 Canvas 的渲染顺序
        canvas.scaleFactor = 1f;              // 缩放因子
        canvas.pixelPerfect = true;           // 像素完美渲染

        // Canvas Scaler（缩放适配）
        CanvasScaler scaler = GetComponent<CanvasScaler>();
        scaler.uiScaleMode = CanvasScaler.ScaleMode.ScaleWithScreenSize;
        scaler.referenceResolution = new Vector2(1920, 1080);  // 参考分辨率
        scaler.matchWidthOrHeight = 0.5f;     // 0=宽度适配, 1=高度适配
    }
}
```

---

## EventSystem 事件系统

EventSystem 处理所有 UI 交互事件（点击、拖拽、输入等）：

```
创建 UI 时自动添加：
EventSystem（事件系统）
  └── Standalone Input Module（PC 输入模块）
      - 处理鼠标、键盘、手柄输入
      - 可以添加 Touch Input Module（移动端）

EventSystem 职责：
- 检测鼠标悬停/点击的 UI 元素
- 管理当前选中的 UI 元素
- 分发事件到对应的 UI 组件
```

```csharp
// EventSystem 的注意事项：
// 1. 场景中只能有一个 EventSystem
// 2. 删除 EventSystem = UI 点击全部失效
// 3. 可以通过 EventSystem.current 访问

using UnityEngine.EventSystems;

public class EventSystemExample : MonoBehaviour
{
    void CheckSelectedUI()
    {
        // 获取当前选中的 UI
        GameObject selected = EventSystem.current.currentSelectedGameObject;
        Debug.Log("当前选中: " + selected?.name);
    }

    void SetSelectedUI(GameObject uiElement)
    {
        // 设置选中状态
        EventSystem.current.SetSelectedGameObject(uiElement);
    }
}
```

---

## 核心 UI 组件

### Button 按钮

```csharp
using UnityEngine;
using UnityEngine.UI;

public class ButtonControl : MonoBehaviour
{
    public Button myButton;

    void Start()
    {
        // 代码创建按钮
        // Button 在 ButtonExamples 中详细演示

        // 按钮属性
        myButton.interactable = true;          // 是否可交互
        myButton.transition = Selectable.Transition.ColorTint; // 过渡方式

        // Navigation（键盘导航）
        Navigation nav = myButton.navigation;
        nav.mode = Navigation.Mode.Explicit;
        nav.selectOnDown = null;  // 按下键跳转到哪个按钮
    }
}
```

### Text 文本

```csharp
using UnityEngine;
using UnityEngine.UI;

public class TextControl : MonoBehaviour
{
    public Text scoreText;
    public Text timerText;

    void Update()
    {
        // 更新分数显示
        scoreText.text = "分数: " + GameManager.Instance.playerScore;

        // Text 属性
        scoreText.fontSize = 32;                         // 字体大小
        scoreText.color = Color.white;                   // 字体颜色
        scoreText.alignment = TextAnchor.MiddleCenter;   // 对齐方式
        scoreText.fontStyle = FontStyle.Bold;            // 字体样式
        scoreText.horizontalOverflow = HorizontalWrapMode.Wrap; // 换行模式
    }
}
```

### Image 图片

```csharp
using UnityEngine;
using UnityEngine.UI;

public class ImageControl : MonoBehaviour
{
    public Image iconImage;
    public Image backgroundImage;

    void Start()
    {
        // Image 属性
        iconImage.sprite = null;                         // 设置精灵
        iconImage.color = Color.white;                   // 颜色叠加
        iconImage.raycastTarget = true;                  // 是否接收射线（点击）
        iconImage.type = Image.Type.Simple;              // 显示类型
        iconImage.preserveAspect = true;                 // 保持宽高比

        // Image 类型：
        // Simple：简单拉伸
        // Sliced：九宫格拉伸（需要 Sprite 设置 Border）
        // Tiled：平铺
        // Filled：填充（用于冷却、进度条）
    }

    // 填充效果（冷却计时器）
    public void SetFillAmount(float amount)
    {
        iconImage.type = Image.Type.Filled;
        iconImage.fillMethod = Image.FillMethod.Radial360;
        iconImage.fillAmount = amount;  // 0~1
    }
}
```

### Slider 滑块

```csharp
using UnityEngine;
using UnityEngine.UI;

public class SliderControl : MonoBehaviour
{
    public Slider healthBar;
    public Slider volumeSlider;

    void Start()
    {
        // Slider 属性
        healthBar.minValue = 0;
        healthBar.maxValue = 100;
        healthBar.value = 100;
        healthBar.wholeNumbers = true;  // 只取整数

        // 监听值变化
        healthBar.onValueChanged.AddListener(OnHealthChanged);
    }

    void OnHealthChanged(float value)
    {
        Debug.Log("生命值: " + value);
    }

    public void UpdateHealth(int current, int max)
    {
        healthBar.value = (float)current / max * healthBar.maxValue;
    }
}
```

### 其他常用组件

```csharp
// InputField - 输入框
// Toggle - 开关（复选框）
// Dropdown - 下拉菜单
// ScrollRect - 滚动区域
// RawImage - 原始图片（用于 RenderTexture、视频）
// Panel - 空白面板（用于背景、分组）
```

---

## Anchor 锚点系统

**Anchor（锚点）** 决定 UI 元素相对于父容器的定位方式：

```
预设锚点位置（Inspector 中点开 Anchor 预设）：

左上角    上居中    右上角
左居中    拉伸全屏   右居中
左下角    下居中    右下角

拉伸模式：UI 会跟随父容器缩放
固定模式：UI 保持固定大小和相对位置
```

```csharp
using UnityEngine;

public class AnchorExamples : MonoBehaviour
{
    private RectTransform rt;

    void Start()
    {
        rt = GetComponent<RectTransform>();

        // 锚点（归一化坐标 0~1）
        // 左下角 (0,0)，右上角 (1,1)
        rt.anchorMin = new Vector2(0, 0);     // 锚点最小值
        rt.anchorMax = new Vector2(1, 1);     // 锚点最大值

        // 轴心点（UI 自身的旋转/缩放中心）
        rt.pivot = new Vector2(0.5f, 0.5f);  // 中心

        // 位置（锚点相关的偏移）
        rt.anchoredPosition = new Vector2(0, 0);
        rt.sizeDelta = new Vector2(100, 50);  // 大小（拉伸模式下是额外偏移）
    }

    // 代码动态设置锚点的常见场景：

    // 固定在屏幕左上角
    void SetTopLeft()
    {
        rt.anchorMin = new Vector2(0, 1);
        rt.anchorMax = new Vector2(0, 1);
        rt.pivot = new Vector2(0, 1);
        rt.anchoredPosition = new Vector2(10, -10); // 距离边缘 10 像素
    }

    // 拉伸填满父容器
    void SetStretchAll()
    {
        rt.anchorMin = Vector2.zero;
        rt.anchorMax = Vector2.one;
        rt.sizeDelta = Vector2.zero; // 偏移为0，完全填满
    }

    // 固定在屏幕底部居中
    void SetBottomCenter()
    {
        rt.anchorMin = new Vector2(0.5f, 0);
        rt.anchorMax = new Vector2(0.5f, 0);
        rt.pivot = new Vector2(0.5f, 0);
    }
}
```

---

## Layout Group 布局

Layout Group 自动排列子 UI 元素：

```csharp
using UnityEngine;
using UnityEngine.UI;

// HorizontalLayoutGroup - 水平排列
// VerticalLayoutGroup - 垂直排列
// GridLayoutGroup - 网格排列

public class LayoutExamples : MonoBehaviour
{
    // 在 Inspector 中设置，或代码设置
    void SetupHorizontalLayout()
    {
        HorizontalLayoutGroup hlg = gameObject.AddComponent<HorizontalLayoutGroup>();
        hlg.spacing = 10;                   // 间距
        hlg.childAlignment = TextAnchor.MiddleCenter; // 子物体对齐
        hlg.childForceExpandWidth = true;   // 强制扩展宽度
        hlg.padding = new RectOffset(10, 10, 10, 10); // 内边距
    }

    void SetupGridLayout()
    {
        GridLayoutGroup glg = gameObject.AddComponent<GridLayoutGroup>();
        glg.cellSize = new Vector2(80, 80);     // 每个格子大小
        glg.spacing = new Vector2(5, 5);        // 间距
        glg.constraint = GridLayoutGroup.Constraint.FixedColumnCount;
        glg.constraintCount = 4;                // 固定 4 列
    }
}
```

### Content Size Fitter

让 UI 元素根据内容自动调整大小：

```csharp
void SetupAutoSizeText()
{
    // Content Size Fitter
    ContentSizeFitter fitter = gameObject.AddComponent<ContentSizeFitter>();
    fitter.horizontalFit = ContentSizeFitter.FitMode.PreferredSize;
    fitter.verticalFit = ContentSizeFitter.FitMode.PreferredSize;

    // 搭配 Layout Element 使用
    LayoutElement layoutElement = gameObject.AddComponent<LayoutElement>();
    layoutElement.minWidth = 100;          // 最小宽度
    layoutElement.preferredWidth = 200;    // 首选宽度
    layoutElement.flexibleWidth = 0;       // 弹性宽度（0=不弹性）
}
```

---

## 事件绑定

将 UI 事件连接到脚本逻辑：

### 方法一：Inspector 拖拽（最简单）

```csharp
// 1. 创建一个公共方法
public class MyUIHandler : MonoBehaviour
{
    public void OnStartButtonClick()
    {
        Debug.Log("开始游戏！");
        SceneLoader.Instance.LoadScene("Level1");
    }

    public void OnSliderValueChanged(float value)
    {
        Debug.Log("音量: " + value);
    }
}

// 2. 在 Inspector 中：
// 选中 Button → On Click() → + → 拖入脚本所在物体 → 选择方法
```

### 方法二：代码绑定（推荐）

```csharp
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.Events;

public class EventBinding : MonoBehaviour
{
    [Header("UI 引用")]
    public Button startButton;
    public Button settingButton;
    public Slider volumeSlider;
    public Toggle fullscreenToggle;

    void Start()
    {
        // 绑定按钮点击事件
        startButton.onClick.AddListener(OnStartGame);
        settingButton.onClick.AddListener(() => {
            Debug.Log("打开设置面板");
        });

        // 绑定值变化事件
        volumeSlider.onValueChanged.AddListener(OnVolumeChanged);

        // 绑定 Toggle 事件
        fullscreenToggle.onValueChanged.AddListener(OnFullscreenToggled);
    }

    void OnDestroy()
    {
        // 清理事件（防止内存泄漏）
        startButton.onClick.RemoveListener(OnStartGame);
        volumeSlider.onValueChanged.RemoveListener(OnVolumeChanged);
    }

    void OnStartGame()
    {
        Debug.Log("开始游戏！");
    }

    void OnVolumeChanged(float value)
    {
        AudioListener.volume = value;
    }

    void OnFullscreenToggled(bool isOn)
    {
        Screen.fullScreen = isOn;
    }
}
```

### 方法三：IPointerXxxHandler 接口

```csharp
using UnityEngine;
using UnityEngine.EventSystems;

/// <summary>
/// 实现指针事件接口
/// 适合自定义交互效果
/// </summary>
public class CustomUIButton : MonoBehaviour,
    IPointerEnterHandler,
    IPointerExitHandler,
    IPointerDownHandler,
    IPointerUpHandler,
    IPointerClickHandler,
    IDragHandler
{
    public void OnPointerEnter(PointerEventData eventData)
    {
        Debug.Log("鼠标进入");
        // 放大效果
        transform.localScale = Vector3.one * 1.1f;
    }

    public void OnPointerExit(PointerEventData eventData)
    {
        Debug.Log("鼠标离开");
        transform.localScale = Vector3.one;
    }

    public void OnPointerDown(PointerEventData eventData)
    {
        Debug.Log("按下");
        transform.localScale = Vector3.one * 0.95f;
    }

    public void OnPointerUp(PointerEventData eventData)
    {
        Debug.Log("松开");
        transform.localScale = Vector3.one * 1.1f;
    }

    public void OnPointerClick(PointerEventData eventData)
    {
        Debug.Log($"点击了！按钮: {eventData.button}");

        if (eventData.button == PointerEventData.InputButton.Left)
            Debug.Log("左键点击");
        if (eventData.button == PointerEventData.InputButton.Right)
            Debug.Log("右键点击");
    }

    public void OnDrag(PointerEventData eventData)
    {
        // 拖拽 UI 元素
        GetComponent<RectTransform>().anchoredPosition += eventData.delta;
    }
}
```

---

## 完整示例

一个完整的游戏 UI 系统：

```csharp
using UnityEngine;
using UnityEngine.UI;

/// <summary>
/// 游戏 HUD（抬头显示）
/// 显示：生命值、分数、技能冷却
/// </summary>
public class GameHUD : MonoBehaviour
{
    [Header("生命值")]
    public Slider healthBar;
    public Text healthText;
    public Image healthFill;    // 用于变色

    [Header("分数")]
    public Text scoreText;
    public Text comboText;

    [Header("技能")]
    public Image[] skillIcons;  // 技能图标数组
    public Image[] skillCooldowns; // 冷却遮罩
    private float[] cooldownTimers;

    [Header("小地图")]
    public RawImage minimapImage;

    [Header("提示")]
    public Text hintText;
    public float hintDuration = 3f;

    private int currentScore = 0;
    private int comboCount = 0;

    void Start()
    {
        // 初始化技能冷却计时器
        cooldownTimers = new float[skillIcons.Length];

        // 绑定事件
        // 通常这些事件会由 GameManager 或 PlayerController 触发
    }

    void Update()
    {
        // 更新技能冷却显示
        UpdateSkillCooldowns();
    }

    // === 公共方法 ===

    public void UpdateHealth(int current, int max)
    {
        // 更新滑块
        healthBar.value = (float)current / max;

        // 更新文字
        healthText.text = $"{current} / {max}";

        // 根据生命值变色
        if (healthBar.value > 0.5f)
            healthFill.color = Color.green;
        else if (healthBar.value > 0.25f)
            healthFill.color = Color.yellow;
        else
            healthFill.color = Color.red;
    }

    public void AddScore(int points)
    {
        currentScore += points;
        comboCount++;

        scoreText.text = $"分数: {currentScore}";

        // 显示连击
        if (comboCount > 1)
        {
            comboText.text = $"连击 x{comboCount}!";
            comboText.gameObject.SetActive(true);
            CancelInvoke(nameof(HideCombo));
            Invoke(nameof(HideCombo), 2f);
        }
    }

    void HideCombo()
    {
        comboCount = 0;
        comboText.gameObject.SetActive(false);
    }

    public void UseSkill(int skillIndex)
    {
        if (skillIndex < 0 || skillIndex >= cooldownTimers.Length) return;
        cooldownTimers[skillIndex] = 5f; // 5秒冷却
    }

    void UpdateSkillCooldowns()
    {
        for (int i = 0; i < cooldownTimers.Length; i++)
        {
            if (cooldownTimers[i] > 0)
            {
                cooldownTimers[i] -= Time.deltaTime;
                // 更新冷却遮罩填充
                skillCooldowns[i].fillAmount = cooldownTimers[i] / 5f;
                skillCooldowns[i].gameObject.SetActive(true);
            }
            else
            {
                skillCooldowns[i].gameObject.SetActive(false);
            }
        }
    }

    public void ShowHint(string message)
    {
        hintText.text = message;
        hintText.gameObject.SetActive(true);
        CancelInvoke(nameof(HideHint));
        Invoke(nameof(HideHint), hintDuration);
    }

    void HideHint()
    {
        hintText.gameObject.SetActive(false);
    }
}
```

### 要点总结

| 概念 | 说明 |
|------|------|
| Canvas | UI 根容器，三种渲染模式 |
| EventSystem | 处理 UI 交互事件 |
| RectTransform | UI 专用的 Transform |
| Anchor | 锚点，控制 UI 相对定位 |
| Layout Group | 自动排列子 UI |
| onClick | 按钮点击事件 |
| onValueChanged | 值变化事件 |
| IPointerXxx | 指针事件接口 |

### UI 设计小贴士

1. **先搭结构再调细节**：先把 UI 层级和布局搭好
2. **善用 Layout Group**：避免手动排列
3. **锚点要设对**：不同分辨率下 UI 位置才正确
4. **Canvas 分层**：不同功能的 UI 放不同 Canvas
5. **事件要解绑**：`OnDestroy` 中 RemoveListener 防止内存泄漏

---

> 📌 回到 [Unity 入门目录](../README.md)
