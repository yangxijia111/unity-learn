# 06-VR 性能优化

> 稳定高帧率是 VR 体验的生命线

---

## 目录

- [VR 性能要求](#vr-性能要求)
- [Unity Profiler 在 VR 中的使用](#unity-profiler-在-vr-中的使用)
- [渲染优化](#渲染优化)
- [内存与加载优化](#内存与加载优化)
- [Meta Quest 特有优化](#meta-quest-特有优化)
- [性能指标对照表](#性能指标对照表)
- [完整代码示例](#完整代码示例)

---

## VR 性能要求

### 为什么帧率至关重要

```
传统游戏：30 FPS 可接受，60 FPS 舒适
VR 游戏：低于目标帧率 = 立即头晕恶心

帧率目标：
─────────
设备类型              │ 目标帧率
Meta Quest 2/3/Pro   │ 72 / 90 / 120 Hz
Valve Index           │ 80 / 90 / 120 / 144 Hz
PSVR2                 │ 90 / 120 Hz
Apple Vision Pro      │ 90 / 96 Hz
Pico 4                │ 72 / 90 Hz

帧时间预算（以 90Hz 为例）：
──────────────────────────────
总预算：11.11ms
├── CPU 逻辑：~6ms
├── GPU 渲染：~11ms（与 CPU 并行）
├── 同步等待：~0.1ms
└── 提交延迟：预留

关键原则：
✅ 永远不要掉帧 —— 宁可降低画质
✅ 帧时间波动比平均帧率更致命
✅ 一体机（Quest）优化压力远大于 PCVR
```

### Motion-to-Photon 延迟

```
Motion-to-Photon：
  从头部转动到屏幕更新的时间

目标：< 20ms
可接受：< 50ms
不可接受：> 100ms（立即晕动）

降低延迟的方法：
─────────────────
✅ Late Latching（延迟锁定头部数据）
✅ Single Pass Rendering
✅ 减少 CPU/GPU 耗时
✅ 使用 Application SpaceWarp（Quest）
✅ 降低渲染分辨率换取帧率
```

---

## Unity Profiler 在 VR 中的使用

### Profiler 配置

```
打开方式：Window → Analysis → Profiler

关键模块：
─────────
CPU Usage：主线程各系统耗时
  ├── Scripts（游戏逻辑）
  ├── Physics
  ├── Animation
  ├── Rendering（CPU 端）
  └── VSync / Overhead

GPU Usage：渲染耗时（需要 GPU Profiler 模块）
  ├── Opaque
  ├── Transparent
  ├── Shadows
  ├── Post-Processing
  └── UI Rendering

Memory：内存使用
  ├── Total（总内存）
  ├── Used（已使用）
  ├── Reserved（已分配）
  └── Mono（C# 托管内存）
```

### VR 专用分析

```csharp
// 性能监控脚本 —— 实时追踪 VR 帧率
using UnityEngine;
using UnityEngine.XR;

public class VRPerformanceMonitor : MonoBehaviour
{
    [Header("显示设置")]
    public bool showOnStart = false;
    public TMPro.TextMeshProUGUI displayText;

    [Header("警告阈值")]
    public float warningFrameTime = 12f;   // ms
    public float criticalFrameTime = 14f;  // ms

    private float _avgFrameTime;
    private int _frameCount;
    private float _timer;
    private float _worstFrameTime;
    private int _droppedFrames;

    void Start()
    {
        Application.targetFrameRate = -1; // 不限制，由 XR 管理
        QualitySettings.vSyncCount = 0;   // VR 不使用 VSync

        if (displayText != null)
            displayText.gameObject.SetActive(showOnStart);
    }

    void Update()
    {
        float frameTime = Time.unscaledDeltaTime * 1000f; // 转为 ms

        // 追踪最差帧
        if (frameTime > _worstFrameTime)
            _worstFrameTime = frameTime;

        // 检测掉帧（超过目标 50% 算掉帧）
        float targetFrameTime = 1000f / XRDevice.refreshRate;
        if (frameTime > targetFrameTime * 1.5f)
            _droppedFrames++;

        // 滑动平均
        _avgFrameTime = Mathf.Lerp(_avgFrameTime, frameTime, 0.1f);
        _frameCount++;

        // 更新显示
        _timer += Time.unscaledDeltaTime;
        if (_timer >= 0.5f) // 每半秒更新一次
        {
            _timer = 0;
            UpdateDisplay();
        }
    }

    void UpdateDisplay()
    {
        if (displayText == null) return;

        float fps = 1000f / _avgFrameTime;
        string status = _avgFrameTime < warningFrameTime ? "良好" :
                        _avgFrameTime < criticalFrameTime ? "警告" : "严重";

        displayText.text = $"FPS: {fps:F0}\n" +
                           $"帧时间: {_avgFrameTime:F1}ms\n" +
                           $"最差帧: {_worstFrameTime:F1}ms\n" +
                           $"掉帧数: {_droppedFrames}\n" +
                           $"状态: {status}";

        // 颜色指示
        displayText.color = _avgFrameTime < warningFrameTime ? Color.green :
                            _avgFrameTime < criticalFrameTime ? Color.yellow : Color.red;
    }

    public void ResetStats()
    {
        _worstFrameTime = 0;
        _droppedFrames = 0;
        _frameCount = 0;
        _avgFrameTime = 0;
    }

    public void ToggleDisplay()
    {
        if (displayText != null)
            displayText.gameObject.SetActive(!displayText.gameObject.activeSelf);
    }
}
```

---

## 渲染优化

### Single Pass Instanced Rendering

VR 最重要的渲染优化：

```
传统 Multi Pass：
  左眼渲染一次 → 右眼渲染一次
  CPU 提交 2 次 Draw Call

Single Pass Instanced：
  左右眼同时渲染（使用 Instanced Draw Call）
  CPU 提交 1 次 Draw Call
  GPU 通过 Instance ID 区分左右眼

设置方式：
─────────
Project Settings → Player → XR Settings
  → Stereo Rendering Mode: Single Pass Instanced

或代码设置：
```

```csharp
using UnityEngine;
using UnityEngine.XR;

public class SetStereoRendering : MonoBehaviour
{
    void Start()
    {
        // 设置为 Single Pass Instanced
        XRSettings.eyeTextureResolutionScale = 1.0f;
        // PlayerSettings.VRSettings 也可设置
    }
}
```

### Draw Call 合批

```
合批策略：
─────────
1. 静态合批（Static Batching）
   - 标记为 Static 的物体自动合批
   - 适合不移动的环境物件
   - 会增加内存（合并后的网格）

2. 动态合批（Dynamic Batching）
   - 自动对小网格（< 300 顶点）合批
   - 对 Mesh Renderer 有效
   - 有 CPU 开销，仅在 Draw Call 很多时有用

3. GPU Instancing
   - 相同材质、不同变换的物体批量绘制
   - 需要材质勾选 Enable GPU Instancing
   - 适合大量相同物体（草、树、碎片）

4. SRP Batcher（URP/HDRP）
   - 自动生效
   - 不要求相同材质，只要 Shader 变体相同
   - 大幅减少 SetPass Call

实践建议：
─────────
✅ 环境模型合并材质（Texture Atlas）
✅ 尽量使用 URP 并启用 SRP Batcher
✅ 使用 LOD Group 减少远处渲染开销
✅ 相同物体使用 GPU Instancing
❌ 避免大量不同材质的小物体
```

### LOD 在 VR 中的使用

```csharp
// VR 优化的 LOD 配置工具
using UnityEngine;

public class VRLODConfigurator : MonoBehaviour
{
    [Header("LOD 距离（VR 特调）")]
    // VR 中物体看起来更近，LOD 距离应更保守
    public float lod0Distance = 10f;   // 最高精度
    public float lod1Distance = 25f;   // 中等精度
    public float lod2Distance = 50f;   // 低精度
    public float cullDistance = 80f;   // 完全剔除

    [Header("屏幕覆盖率阈值")]
    public float lod0ScreenRatio = 0.6f;
    public float lod1ScreenRatio = 0.3f;
    public float lod2ScreenRatio = 0.1f;

    /// <summary>
    /// 为场景中所有带 LOD Group 的物体配置距离
    /// </summary>
    [ContextMenu("配置所有 LOD")]
    public void ConfigureAllLODs()
    {
        LODGroup[] lodGroups = FindObjectsOfType<LODGroup>();
        int configured = 0;

        foreach (var lg in lodGroups)
        {
            LOD[] lods = lg.GetLODs();
            if (lods.Length >= 3)
            {
                lods[0].screenRelativeTransitionHeight = lod0ScreenRatio;
                lods[1].screenRelativeTransitionHeight = lod1ScreenRatio;
                lods[2].screenRelativeTransitionHeight = lod2ScreenRatio;
                lg.SetLODs(lods);
                configured++;
            }
        }

        Debug.Log($"已配置 {configured} 个 LOD Group");
    }

    /// <summary>
    /// 自动为 GameObject 创建 LOD Group
    /// </summary>
    public static void AutoSetupLOD(GameObject obj, Mesh high, Mesh mid, Mesh low, Material mat)
    {
        var lodGroup = obj.GetComponent<LODGroup>();
        if (lodGroup == null)
            lodGroup = obj.AddComponent<LODGroup>();

        // 创建 LOD 子物体
        GameObject lod0 = CreateLODObject(obj, "LOD0", high, mat);
        GameObject lod1 = CreateLODObject(obj, "LOD1", mid, mat);
        GameObject lod2 = CreateLODObject(obj, "LOD2", low, mat);

        LOD[] lods = new LOD[3];
        lods[0] = new LOD(0.6f, new Renderer[] { lod0.GetComponent<Renderer>() });
        lods[1] = new LOD(0.3f, new Renderer[] { lod1.GetComponent<Renderer>() });
        lods[2] = new LOD(0.1f, new Renderer[] { lod2.GetComponent<Renderer>() });

        lodGroup.SetLODs(lods);
    }

    static GameObject CreateLODObject(GameObject parent, string name, Mesh mesh, Material mat)
    {
        GameObject go = new GameObject(name);
        go.transform.SetParent(parent.transform);
        go.transform.localPosition = Vector3.zero;
        go.transform.localRotation = Quaternion.identity;

        var mf = go.AddComponent<MeshFilter>();
        mf.mesh = mesh;
        var mr = go.AddComponent<MeshRenderer>();
        mr.material = mat;

        return go;
    }
}
```

### Occlusion Culling

```
遮挡剔除设置：
──────────────
1. 标记静态物体为 Occluder Static / Occludee Static
2. Window → Rendering → Occlusion Culling
3. Bake 烘焙遮挡数据
4. 在 VR 中特别有效（减少不可见物体渲染）

VR 注意事项：
─────────────
✅ 大型室内场景（房间、走廊）收益最大
✅ 配合 LOD 使用效果更佳
✅ 注意烘焙质量 vs 文件大小
❌ 室外开阔场景收益有限
❌ 不要对动态物体启用（开销大于收益）
```

### Shader 优化（移动端 VR 特有）

```
Quest / 移动端 Shader 建议：
────────────────────────────
✅ 使用 URP Lit / Simple Lit Shader
✅ 避免 Grab Pass（极昂贵）
✅ 减少 Pass 数量（尽量单 Pass）
✅ 使用 Alpha Cutoff 代替 Alpha Blend
✅ 避免复杂数学运算（sin/cos/pow）
✅ 使用 half 精度代替 float（移动端 GPU 支持好）
✅ 减少纹理采样次数
❌ 避免 Tessellation
❌ 避免 Geometry Shader
❌ 避免实时反射探针
❌ 避免屏幕空间反射（SSR）
```

---

## 内存与加载优化

### AssetBundle / Addressables

```csharp
// Addressables 场景异步加载器（推荐方式）
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

public class VRSceneLoader : MonoBehaviour
{
    [Header("加载界面")]
    public GameObject loadingScreen;
    public TMPro.TextMeshProUGUI loadingText;
    public UnityEngine.UI.Slider progressBar;

    [Header("设置")]
    public float minimumLoadTime = 1.5f; // 最短显示时间（防闪烁）

    private AsyncOperationHandle<SceneInstance> _loadHandle;

    /// <summary>
    /// 异步加载场景
    /// </summary>
    public void LoadScene(string sceneAddress)
    {
        StartCoroutine(LoadSceneAsync(sceneAddress));
    }

    System.Collections.IEnumerator LoadSceneAsync(string sceneAddress)
    {
        // 显示加载界面
        if (loadingScreen != null) loadingScreen.SetActive(true);
        float startTime = Time.time;

        // 释放当前场景（可选）
        // yield return Addressables.UnloadSceneAsync(currentHandle);

        // 异步加载
        _loadHandle = Addressables.LoadSceneAsync(sceneAddress, LoadSceneMode.Single, false);

        while (!_loadHandle.IsDone)
        {
            float progress = _loadHandle.PercentComplete;
            if (progressBar != null) progressBar.value = progress;
            if (loadingText != null) loadingText.text = $"加载中... {progress * 100:F0}%";
            yield return null;
        }

        if (_loadHandle.Status == AsyncOperationStatus.Succeeded)
        {
            // 确保最低加载时间
            float elapsed = Time.time - startTime;
            if (elapsed < minimumLoadTime)
                yield return new WaitForSeconds(minimumLoadTime - elapsed);

            // 激活场景
            yield return _loadHandle.Result.ActivateAsync();

            // 隐藏加载界面
            if (loadingScreen != null) loadingScreen.SetActive(false);
        }
        else
        {
            Debug.LogError($"场景加载失败: {sceneAddress}");
            if (loadingText != null) loadingText.text = "加载失败！";
        }
    }

    /// <summary>
    /// 预加载资源
    /// </summary>
    public void PreloadAssets(string label)
    {
        StartCoroutine(PreloadAsync(label));
    }

    System.Collections.IEnumerator PreloadAsync(string label)
    {
        var handle = Addressables.DownloadDependenciesAsync(label, false);

        while (!handle.IsDone)
        {
            Debug.Log($"预加载进度: {handle.PercentComplete * 100:F0}%");
            yield return null;
        }

        if (handle.Status == AsyncOperationStatus.Succeeded)
            Debug.Log("预加载完成");
        else
            Debug.LogError("预加载失败");

        Addressables.Release(handle);
    }
}
```

### 纹理压缩（ASTC for Quest）

```
Android (Quest) 纹理压缩格式：
──────────────────────────────
格式          │ 大小        │ 质量     │ 推荐场景
ASTC 4x4     │ 1 像素/字节 │ 最高     │ UI、重要角色
ASTC 6x6     │ 0.44 像素/字节│ 高     │ 一般物体
ASTC 8x8     │ 0.25 像素/字节│ 中     │ 环境贴图
ASTC 10x10   │ 0.16 像素/字节│ 低     │ 远处物体
ASTC 12x12   │ 0.11 像素/字节│ 最低   │ 不重要的纹理

设置方式：
─────────
纹理导入设置：
  Texture Type: Default
  Platform: Android
  Override for Android: ✓
  Format: ASTC (自动选择块大小)
  或手动选择：ASTC 4x4 / 6x6 / 8x8

Unity 批量设置：
  Edit → Project Settings → Default Texture Compression

工具：
─────────
  Texture Compression Quality：Unity 内置
  Crunch Compression：无损再压缩，减小包体
  Mip Maps：VR 中大多数纹理应启用 Mip Maps
  （减少远处纹理采样闪烁）
```

---

## Meta Quest 特有优化

### Fixed Foveated Rendering (FFR)

```
FFR 原理：
  屏幕边缘渲染分辨率降低，中心保持高分辨率
  利用人眼中央凹视觉特性

设置级别（0-4）：
─────────────────
Off (0)     │ 无降级，全分辨率
Low (1)     │ 轻微降级，几乎看不出
Medium (2)  │ 中等降级，边缘略模糊
High (3)    │ 较大降级，边缘可见模糊
HighTop (4) │ 最大降级，仅中心清晰

代码设置：
```

```csharp
using UnityEngine;
using UnityEngine.XR;

public class QuestFFRController : MonoBehaviour
{
    // FFR 级别
    public enum FFRLevel
    {
        Off = 0,
        Low = 1,
        Medium = 2,
        High = 3,
        HighTop = 4
    }

    public FFRLevel currentLevel = FFRLevel.Medium;

    void Start()
    {
        SetFFRLevel(currentLevel);
    }

    /// <summary>
    /// 设置 Fixed Foveated Rendering 级别
    /// </summary>
    public void SetFFRLevel(FFRLevel level)
    {
        currentLevel = level;

        #if UNITY_ANDROID && !UNITY_EDITOR
        // Quest 专用 API
        var xrDisplaySubsystems = new System.Collections.Generic.List<XRDisplaySubsystem>();
        SubsystemManager.GetInstances(xrDisplaySubsystems);

        foreach (var subsystem in xrDisplaySubsystems)
        {
            // 设置 FFR（通过 OVR 或 XR Plugin）
            // 不同 SDK 有不同接口
        }

        // 如果使用 OVR：
        // OVRManager.foveatedRenderingLevel = (OVRManager.FoveatedRenderingLevel)level;
        // OVRManager.useDynamicFoveatedRendering = true;
        #endif

        Debug.Log($"FFR 级别设置为: {level}");
    }

    /// <summary>
    /// 根据性能自动调整 FFR
    /// </summary>
    public void AutoAdjustFFR(float currentFrameTime)
    {
        float targetFrameTime = 1000f / 90f; // 90Hz

        if (currentFrameTime > targetFrameTime * 1.15f)
        {
            // 掉帧，提高 FFR
            if (currentLevel < FFRLevel.HighTop)
                SetFFRLevel(currentLevel + 1);
        }
        else if (currentFrameTime < targetFrameTime * 0.85f)
        {
            // 性能富余，降低 FFR
            if (currentLevel > FFRLevel.Off)
                SetFFRLevel(currentLevel - 1);
        }
    }
}
```

### Application SpaceWarp

```
Application SpaceWarp (ASW)：
  当游戏帧率低于目标时，系统自动生成中间帧

原理：
  游戏渲染 45 FPS → ASW 插帧到 90 FPS
  使用深度缓冲和运动矢量生成中间帧

注意事项：
─────────
✅ ASW 是辅助手段，不是优化的替代品
✅ 偶尔掉帧时效果好
✅ 持续依赖 ASW 会出现视觉伪影
✅ 复杂运动（快速旋转）时伪影更明显

最佳实践：
  - 目标帧率仍是 90 FPS（或目标 Hz）
  - ASW 仅作为安全网
  - 监控 ASW 激活频率，过高说明需要优化
```

### Quest 渲染分辨率

```csharp
// Quest 渲染分辨率管理
using UnityEngine;

public class QuestRenderSettings : MonoBehaviour
{
    [Header("分辨率设置")]
    [Tooltip("渲染分辨率缩放（1.0 = 默认）")]
    [Range(0.5f, 1.5f)]
    public float renderScale = 1.0f;

    [Tooltip("眼睛纹理分辨率缩放")]
    [Range(0.5f, 2.0f)]
    public float eyeTextureScale = 1.0f;

    void Start()
    {
        ApplySettings();
    }

    public void ApplySettings()
    {
        // 设置渲染缩放
        UnityEngine.XR.XRSettings.eyeTextureResolutionScale = eyeTextureScale;
        UnityEngine.XR.XRSettings.renderViewportScale = renderScale;

        // Quest 默认分辨率约 1440x1584（每眼）
        // 缩放到 0.85 可显著提升性能，几乎看不出差别
    }

    /// <summary>
    /// 动态调整分辨率以维持帧率
    /// </summary>
    public void DynamicResolution(float currentFPS, float targetFPS)
    {
        float ratio = currentFPS / targetFPS;

        if (ratio < 0.9f) // 低于目标 90%
        {
            eyeTextureScale = Mathf.Max(0.7f, eyeTextureScale - 0.05f);
        }
        else if (ratio > 1.1f) // 高于目标 110%
        {
            eyeTextureScale = Mathf.Min(1.2f, eyeTextureScale + 0.02f);
        }

        ApplySettings();
    }
}
```

---

## 性能指标对照表

### Quest 2 性能预算

```
帧率 90Hz（帧时间 11.11ms）：
══════════════════════════════════════════════════════════
类别            │ 预算(ms) │ 说明
────────────────┼──────────┼──────────────────────────
CPU 总计        │  ≤ 8.0   │ 游戏逻辑 + 物理 + 动画
  游戏逻辑      │  ≤ 4.0   │ C# 脚本
  物理          │  ≤ 2.0   │ 碰撞检测、刚体
  动画          │  ≤ 1.5   │ Animator / 骨骼动画
  其他          │  ≤ 0.5   │ 寻路、音频等
────────────────┼──────────┼──────────────────────────
GPU 总计        │  ≤ 11.0  │ 包含所有渲染
  Opaque        │  ≤ 4.0   │ 不透明物体
  Transparent   │  ≤ 2.0   │ 透明/半透明
  Shadows       │  ≤ 2.0   │ 阴影渲染
  Post-FX       │  ≤ 1.5   │ 后处理
  UI            │  ≤ 0.5   │ Canvas 渲染
  其他          │  ≤ 1.0   │ 粒子、特效
────────────────┼──────────┼──────────────────────────
Draw Calls      │  ≤ 100   │ 单 Pass 后的 DC
Triangles       │  ≤ 750K  │ 两个眼睛总和
Texture Memory  │  ≤ 1GB   │ 纹理占用内存
────────────────┼──────────┼────────────────────────══

PCVR（如 Index 120Hz 帧时间 8.33ms）：
══════════════════════════════════════════════════════════
类别            │ 预算(ms) │ 说明
────────────────┼──────────┼──────────────────────────
CPU 总计        │  ≤ 6.0   │ 更短的帧时间
GPU 总计        │  ≤ 8.0   │ 更强的 GPU
Draw Calls      │  ≤ 2000  │ PC 可承受更多
Triangles       │  ≤ 3M    │ 每眼睛
Texture Memory  │  ≤ 4GB   │ 显存更大
```

### 关键指标检查清单

```
每日检查：
☐ 帧时间稳定在目标值以下
☐ 无持续掉帧
☐ CPU 主线程无超时峰值
☐ GPU 无超时峰值
☐ Draw Call < 预算
☐ 内存无持续增长（泄露检测）

优化阶段检查：
☐ Single Pass Instanced 已启用
☐ LOD 覆盖率 > 80%（可 LOD 的物体）
☐ GPU Instancing 已在相同物体上启用
☐ 纹理已压缩为 ASTC（Quest）
☐ 阴影使用合理距离
☐ 后处理已简化或移除
☐ UI Canvas 合批正常
```

---

## 完整代码示例

### 综合性能优化管理器

```csharp
// VRPerformanceManager.cs —— VR 性能自动优化管理器
using UnityEngine;
using System.Collections.Generic;

public class VRPerformanceManager : MonoBehaviour
{
    public static VRPerformanceManager Instance { get; private set; }

    [Header("目标设置")]
    public float targetFrameRate = 90f;
    public bool autoOptimize = true;

    [Header("采样设置")]
    public int sampleCount = 60;         // 采样帧数
    public float checkInterval = 2f;     // 检查间隔（秒）

    [Header("优化参数")]
    [Range(0.5f, 1.5f)]
    public float minRenderScale = 0.7f;
    [Range(0.5f, 1.5f)]
    public float maxRenderScale = 1.0f;
    public float renderScaleStep = 0.05f;

    [Header("LOD 设置")]
    public float lodBiasNormal = 1.0f;
    public float lodBiasAggressive = 0.5f;

    // 内部状态
    private Queue<float> _frameTimes;
    private float _currentRenderScale = 1.0f;
    private float _checkTimer;
    private int _consecutiveBadFrames;
    private PerformanceLevel _currentLevel;

    public enum PerformanceLevel
    {
        Optimal,     // 性能良好
        Moderate,    // 轻微降级
        Aggressive,  // 大幅降级
        Emergency    // 紧急降级
    }

    void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;

        _frameTimes = new Queue<float>();
        _currentLevel = PerformanceLevel.Optimal;
    }

    void Start()
    {
        QualitySettings.lodBias = lodBiasNormal;
        Application.targetFrameRate = -1;
        QualitySettings.vSyncCount = 0;
    }

    void Update()
    {
        // 记录帧时间
        _frameTimes.Enqueue(Time.unscaledDeltaTime * 1000f);
        if (_frameTimes.Count > sampleCount)
            _frameTimes.Dequeue();

        // 检查是否连续掉帧
        float frameTime = Time.unscaledDeltaTime * 1000f;
        float targetTime = 1000f / targetFrameRate;

        if (frameTime > targetTime * 1.2f)
            _consecutiveBadFrames++;
        else
            _consecutiveBadFrames = Mathf.Max(0, _consecutiveBadFrames - 1);

        // 定期检查并优化
        if (autoOptimize)
        {
            _checkTimer += Time.unscaledDeltaTime;
            if (_checkTimer >= checkInterval)
            {
                _checkTimer = 0;
                EvaluateAndOptimize();
            }
        }
    }

    void EvaluateAndOptimize()
    {
        // 计算平均帧时间
        float avgFrameTime = 0;
        float worstFrameTime = 0;
        foreach (float ft in _frameTimes)
        {
            avgFrameTime += ft;
            if (ft > worstFrameTime) worstFrameTime = ft;
        }
        avgFrameTime /= _frameTimes.Count;

        float targetTime = 1000f / targetFrameRate;
        float ratio = avgFrameTime / targetTime;

        PerformanceLevel newLevel;

        if (ratio <= 0.85f)
            newLevel = PerformanceLevel.Optimal;
        else if (ratio <= 1.0f)
            newLevel = PerformanceLevel.Moderate;
        else if (ratio <= 1.15f)
            newLevel = PerformanceLevel.Aggressive;
        else
            newLevel = PerformanceLevel.Emergency;

        // 连续掉帧加急处理
        if (_consecutiveBadFrames > 30)
            newLevel = PerformanceLevel.Emergency;

        if (newLevel != _currentLevel)
        {
            ApplyOptimizations(newLevel);
            _currentLevel = newLevel;
            Debug.Log($"性能级别变更: {newLevel} (平均帧时间: {avgFrameTime:F1}ms)");
        }
    }

    void ApplyOptimizations(PerformanceLevel level)
    {
        switch (level)
        {
            case PerformanceLevel.Optimal:
                _currentRenderScale = maxRenderScale;
                QualitySettings.lodBias = lodBiasNormal;
                QualitySettings.shadowDistance = 50f;
                QualitySettings.antiAliasing = 4;
                break;

            case PerformanceLevel.Moderate:
                _currentRenderScale = 0.9f;
                QualitySettings.lodBias = lodBiasNormal;
                QualitySettings.shadowDistance = 40f;
                QualitySettings.antiAliasing = 2;
                break;

            case PerformanceLevel.Aggressive:
                _currentRenderScale = 0.8f;
                QualitySettings.lodBias = lodBiasAggressive;
                QualitySettings.shadowDistance = 25f;
                QualitySettings.antiAliasing = 2;
                break;

            case PerformanceLevel.Emergency:
                _currentRenderScale = minRenderScale;
                QualitySettings.lodBias = lodBiasAggressive;
                QualitySettings.shadowDistance = 15f;
                QualitySettings.antiAliasing = 0;
                break;
        }

        // 应用渲染缩放
        _currentRenderScale = Mathf.Clamp(_currentRenderScale, minRenderScale, maxRenderScale);
        UnityEngine.XR.XRSettings.eyeTextureResolutionScale = _currentRenderScale;
    }

    // === 公共 API ===

    public PerformanceLevel GetCurrentLevel() => _currentLevel;

    public float GetAverageFPS()
    {
        if (_frameTimes.Count == 0) return 0;
        float avg = 0;
        foreach (float ft in _frameTimes) avg += ft;
        avg /= _frameTimes.Count;
        return 1000f / avg;
    }

    public void ForceOptimization(PerformanceLevel level)
    {
        ApplyOptimizations(level);
        _currentLevel = level;
    }

    public void ResetToOptimal()
    {
        ForceOptimization(PerformanceLevel.Optimal);
    }
}
```

---

## 参考资源

- [Meta Quest Performance Best Practices](https://developer.oculus.com/resources/pb/)
- [Unity VR Performance Guide](https://docs.unity3d.com/Manual/VROptimization.html)
- [Meta Developer Documentation](https://developer.meta.com/)
- [URP Performance Optimization](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest)

---

> 上一节：[05-VRUI与3D界面](../05-VRUI与3D界面/)
> 下一节：[07-VR实战项目-射击游戏](../07-VR实战项目-射击游戏/)
