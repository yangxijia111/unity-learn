# VR 多平台发布指南

## 📋 目录

- [VR 平台概览](#vr-平台概览)
- [平台适配要点](#平台适配要点)
- [Meta Quest 发布流程](#meta-quest-发布流程)
- [SteamVR 发布流程](#steamvr-发布流程)
- [Pico 发布简介](#pico-发布简介)
- [Apple Vision Pro](#apple-vision-pro)
- [跨平台代码架构](#跨平台代码架构)
- [常见问题排查](#常见问题排查)
- [参考资源](#参考资源)

---

## VR 平台概览

### 主流 VR 平台对比

| 平台 | 类型 | 头显 | 渲染要求 | 帧率要求 |
|------|------|------|----------|----------|
| **Meta Quest** | 一体机 | Quest 2/3/3S/Pro | 移动GPU | 72/90/120Hz |
| **SteamVR** | PC VR | 各品牌头显 | PC显卡 | 90Hz |
| **Pico** | 一体机 | Pico 4/4 Ultra | 移动GPU | 72/90Hz |
| **Apple Vision Pro** | 空间计算 | Vision Pro | M2芯片 | 90Hz |
| **PS VR2** | 主机VR | PS VR2 | PS5 | 90/120Hz |

### 一体机 vs PCVR

**一体机（Quest/Pico）：**
- 无需电脑，独立运行
- 性能受限（移动芯片）
- 需要针对移动端优化
- 用户基数大，市场大

**PCVR（SteamVR）：**
- 需要连接PC
- 性能强劲（取决于显卡）
- 画面质量更高
- 开发自由度大

---

## 平台适配要点

### 输入映射差异

不同平台的手柄按键布局不同，需要使用 **Input System** 的 **Action Map** 做抽象映射：

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

/// <summary>
/// 跨平台输入管理器
/// 抽象化不同手柄的输入差异
/// </summary>
public class CrossPlatformInput : MonoBehaviour
{
    [Header("输入动作（通过 Input Actions Asset 配置）")]
    [Tooltip("移动输入")]
    public InputActionReference moveAction;
    
    [Tooltip("转向输入")]
    public InputActionReference turnAction;
    
    [Tooltip("抓取输入")]
    public InputActionReference grabAction;
    
    [Tooltip("扳机输入")]
    public InputActionReference triggerAction;
    
    [Tooltip("菜单按钮")]
    public InputActionReference menuAction;

    // 获取移动方向
    public Vector2 GetMoveInput()
    {
        return moveAction.action.ReadValue<Vector2>();
    }

    // 获取转向输入
    public Vector2 GetTurnInput()
    {
        return turnAction.action.ReadValue<Vector2>();
    }

    // 是否按下抓取
    public bool IsGrabPressed()
    {
        return grabAction.action.IsPressed();
    }

    // 扳机值（0-1）
    public float GetTriggerValue()
    {
        return triggerAction.action.ReadValue<float>();
    }
}
```

### 性能差异对照表

```csharp
/// <summary>
/// 平台性能适配配置
/// 根据目标平台自动调整画质设置
/// </summary>
public static class PlatformPerformanceConfig
{
    // Quest 一体机设置
    private static readonly VRQualitySettings questSettings = new VRQualitySettings
    {
        renderScale = 1.0f,
        antiAliasing = 2,        // MSAA 2x
        shadowQuality = 0,       // 关闭阴影
        textureQuality = 1,      // 中等纹理
        particleMaxCount = 100,
        targetFrameRate = 72,
        useFixedFoveatedRendering = true,
        singlePassRendering = true
    };

    // PC VR 设置
    private static readonly VRQualitySettings pcVrSettings = new VRQualitySettings
    {
        renderScale = 1.2f,
        antiAliasing = 4,        // MSAA 4x
        shadowQuality = 2,       // 中等阴影
        textureQuality = 0,      // 高质量纹理
        particleMaxCount = 1000,
        targetFrameRate = 90,
        useFixedFoveatedRendering = false,
        singlePassRendering = true
    };

    /// <summary>
    /// 根据当前设备自动应用设置
    /// </summary>
    public static void ApplySettings()
    {
        VRQualitySettings settings;

#if UNITY_ANDROID
        // Quest / Pico 等安卓一体机
        settings = questSettings;
        Debug.Log("应用 Quest 性能设置");
#elif UNITY_STANDALONE_WIN
        // PC VR
        settings = pcVrSettings;
        Debug.Log("应用 PC VR 性能设置");
#else
        settings = pcVrSettings;
#endif
        ApplyQualitySettings(settings);
    }

    static void ApplyQualitySettings(VRQualitySettings settings)
    {
        QualitySettings.antiAliasing = settings.antiAliasing;
        QualitySettings.shadowQuality = (ShadowQuality)settings.shadowQuality;
        Application.targetFrameRate = settings.targetFrameRate;
        
        // 设置渲染分辨率
        UnityEngine.XR.XRSettings.eyeTextureResolutionScale = settings.renderScale;
    }
}

[System.Serializable]
public struct VRQualitySettings
{
    public float renderScale;
    public int antiAliasing;
    public int shadowQuality;
    public int textureQuality;
    public int particleMaxCount;
    public int targetFrameRate;
    public bool useFixedFoveatedRendering;
    public bool singlePassRendering;
}
```

---

## Meta Quest 发布流程

### 1. Meta Developer 账号注册

1. 访问 [developer.meta.com](https://developer.meta.com)
2. 使用 Meta/Facebook 账号登录
3. 注册为开发者（个人或组织）
4. 支付一次性注册费 $99（如发布到 Quest Store）

### 2. 创建应用

1. 进入 **My Apps** 页面
2. 点击 **Create New App**
3. 选择平台：**Meta Quest** → **Quest 2/3/3S**
4. 填写应用名称和描述

### 3. Unity 构建设置

```csharp
// 构建前检查清单（编辑器扩展）
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;

public class QuestBuildChecker
{
    [MenuItem("VR Tools/检查 Quest 构建设置")]
    static void CheckQuestBuildSettings()
    {
        bool allGood = true;

        // 1. 检查平台
        if (EditorUserBuildSettings.activeBuildTarget != BuildTarget.Android)
        {
            Debug.LogWarning("⚠️ 目标平台需要切换到 Android");
            allGood = false;
        }

        // 2. 检查最低 API 等级
        if (PlayerSettings.Android.minSdkVersion < AndroidSdkVersions.AndroidApiLevel29)
        {
            Debug.LogWarning("⚠️ 最低 API 等级建议设为 29 (Android 10)");
            allGood = false;
        }

        // 3. 检查 XR Plugin
        if (!UnityEditor.PackageManager.PackageInfo.FindForAssembly(
            System.Reflection.Assembly.Load("Unity.XR.OpenXR")))
        {
            Debug.LogWarning("⚠️ 未检测到 OpenXR Plugin");
            allGood = false;
        }

        // 4. 检查渲染设置
        if (PlayerSettings.GetGraphicsAPIs(BuildTarget.Android)[0] 
            != GraphicsDeviceType.OpenGLES3)
        {
            Debug.Log("ℹ️ 建议使用 OpenGLES3 或 Vulkan");
        }

        if (allGood)
        {
            Debug.Log("✅ Quest 构建设置检查通过!");
        }
    }
}
#endif
```

**Player Settings 关键配置：**
- **Company Name / Product Name**：填写你的工作室名和应用名
- **Default Icon**：512x512 PNG
- **Minimum API Level**：Android 10 (API 29)
- **Target API Level**：Automatic
- **Scripting Backend**：IL2CPP
- **Target Architectures**：ARM64 ✅
- **Graphics APIs**：Vulkan（优先）或 OpenGLES3

### 4. 签名密钥

Quest 要求使用 Oculus 签名密钥：

1. 在 Meta Developer 后台 → **Build Tools** → **Oculus Developer Hub**
2. 或者使用 `ovr-platform-util` 命令行工具
3. 生成 `.keystore` 文件
4. 在 Unity 的 **Publishing Settings** 中配置

### 5. App Lab vs Quest Store

| | App Lab | Quest Store |
|---|---------|-------------|
| **门槛** | 低，几乎自动通过 | 高，需要审核 |
| **可见性** | 需直接链接访问 | 商店搜索可见 |
| **审核** | 基本内容审核 | 严格质量审核 |
| **适合** | 测试、独立开发者 | 正式商业发布 |

**建议路线**：先上 App Lab 测试 → 收集反馈 → 优化后申请 Quest Store

### 6. 提交审核

1. 上传 APK（通过 Meta Developer Hub 或命令行）
2. 填写商店信息（描述、截图、视频）
3. 设置价格（免费或付费）
4. 提交审核（通常 1-3 个工作日 App Lab / 1-2 周 Quest Store）

---

## SteamVR 发布流程

### 1. Steamworks 账号

1. 注册 [partner.steamgames.com](https://partner.steamgames.com)
2. 支付 $100 Steam Direct 费用
3. 创建应用，获取 App ID

### 2. Unity 构建设置

**平台**：PC, Standalone Windows 64-bit

**Player Settings：**
- **Scripting Backend**：Mono 或 IL2CPP
- **Resolution**：不影响（VR 运行在头显内）

### 3. SteamVR Plugin 配置

```csharp
using UnityEngine;
using Valve.VR;

/// <summary>
/// SteamVR 特有功能管理器
/// 处理 Steam 特有的成就、排行榜等
/// </summary>
public class SteamVRManager : MonoBehaviour
{
    void Start()
    {
        // 初始化 SteamVR
        if (SteamVR.instance == null)
        {
            Debug.LogError("SteamVR 未初始化！请确保 SteamVR 正在运行");
            return;
        }
        
        Debug.Log($"SteamVR 运行中 - HMD: {SteamVR.instance.hmd_TrackingSystemName}");
    }

    /// <summary>
    /// 解锁 Steam 成就
    /// </summary>
    public static void UnlockAchievement(string achievementId)
    {
        if (SteamManager.Initialized)
        {
            Steamworks.SteamUserStats.SetAchievement(achievementId);
            Steamworks.SteamUserStats.StoreStats();
            Debug.Log($"成就解锁: {achievementId}");
        }
    }
}
```

### 4. 打包与上传

1. **构建**：Build Settings → Build → 生成 .exe + _Data 文件夹
2. **使用 SteamPipe**：
   - Steamworks 后台 → **SteamPipe** → **Builds**
   - 上传到 Depot
   - 设置默认分支
3. **测试**：通过 Steam 客户端下载测试版

---

## Pico 发布简介

### 与 Quest 的主要区别

- **SDK 不同**：使用 Pico XR SDK（但都支持 OpenXR）
- **商店不同**：Pico Store（国内主要渠道）
- **开发文档**：[developer-global.pico-interactive.com](https://developer-global.pico-interactive.com)

### 构建注意事项

```csharp
/// <summary>
/// 平台检测工具
/// 区分 Quest、Pico、PC 等不同平台
/// </summary>
public static class PlatformDetector
{
    public enum VRPlatform
    {
        Quest,
        Pico,
        SteamVR,
        Unknown
    }

    public static VRPlatform CurrentPlatform
    {
        get
        {
#if UNITY_ANDROID
            // 检测设备型号
            string deviceModel = SystemInfo.deviceModel.ToLower();
            
            if (deviceModel.Contains("quest") || deviceModel.Contains("oculus"))
                return VRPlatform.Quest;
            
            if (deviceModel.Contains("pico"))
                return VRPlatform.Pico;
            
            return VRPlatform.Unknown;
#elif UNITY_STANDALONE_WIN
            return VRPlatform.SteamVR;
#else
            return VRPlatform.Unknown;
#endif
        }
    }

    /// <summary>
    /// 检查是否是一体机
    /// </summary>
    public static bool IsStandaloneHeadset => CurrentPlatform == VRPlatform.Quest 
                                            || CurrentPlatform == VRPlatform.Pico;

    /// <summary>
    /// 检查是否是PC VR
    /// </summary>
    public static bool IsPcVr => CurrentPlatform == VRPlatform.SteamVR;
}
```

---

## Apple Vision Pro

### 特点

- **空间计算**：不完全是传统 VR，更偏向混合现实
- **输入方式**：眼动追踪 + 手势（无手柄）
- **平台限制**：不支持 OpenXR，使用 Apple 的 RealityKit/ARKit
- **Unity 支持**：Unity PolySpatial 扩展

### 注意事项

- Unity 对 Vision Pro 的支持还在发展中
- 需要 Unity Pro/Enterprise 许可
- 建议观望，等工具链成熟后再适配

---

## 跨平台代码架构

```csharp
using UnityEngine;

/// <summary>
/// 跨平台 VR 管理器
/// 根据运行平台自动配置对应设置
/// </summary>
public class CrossPlatformVRManager : MonoBehaviour
{
    public static CrossPlatformVRManager Instance { get; private set; }

    [Header("平台特定预制体")]
    [Tooltip("Quest 输入控制器")]
    public GameObject questControllerPrefab;
    
    [Tooltip("SteamVR 输入控制器")]
    public GameObject steamvrControllerPrefab;
    
    [Header("平台配置")]
    public PlatformConfig questConfig;
    public PlatformConfig steamvrConfig;
    public PlatformConfig picoConfig;

    public PlatformDetector.VRPlatform CurrentPlatform { get; private set; }
    public PlatformConfig CurrentConfig { get; private set; }

    void Awake()
    {
        if (Instance != null)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject);

        // 检测平台
        CurrentPlatform = PlatformDetector.CurrentPlatform;
        
        // 应用平台配置
        ApplyPlatformConfig();
    }

    void ApplyPlatformConfig()
    {
        switch (CurrentPlatform)
        {
            case PlatformDetector.VRPlatform.Quest:
                CurrentConfig = questConfig;
                Debug.Log("🎮 平台: Meta Quest");
                break;
                
            case PlatformDetector.VRPlatform.Pico:
                CurrentConfig = picoConfig;
                Debug.Log("🎮 平台: Pico");
                break;
                
            case PlatformDetector.VRPlatform.SteamVR:
                CurrentConfig = steamvrConfig;
                Debug.Log("🎮 平台: SteamVR");
                break;
                
            default:
                Debug.LogWarning("⚠️ 未知 VR 平台");
                break;
        }

        // 应用性能设置
        if (CurrentConfig != null)
        {
            CurrentConfig.Apply();
        }
    }
}

/// <summary>
/// 平台配置数据
/// </summary>
[System.Serializable]
public class PlatformConfig
{
    public string platformName;
    public int targetFrameRate = 72;
    public float renderScale = 1.0f;
    public int msaaLevel = 2;
    public bool enableShadows = false;
    public bool enableFoveatedRendering = false;
    [Range(0, 1)]
    public float foveationLevel = 0.5f;

    public void Apply()
    {
        Application.targetFrameRate = targetFrameRate;
        QualitySettings.antiAliasing = msaaLevel;
        
        if (enableShadows)
            QualitySettings.shadowQuality = ShadowQuality.All;
        else
            QualitySettings.shadowQuality = ShadowQuality.Disable;
            
        Debug.Log($"平台配置已应用: {platformName} @ {targetFrameRate}Hz");
    }
}
```

---

## 常见问题排查

### Quest 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 黑屏 | XR Plugin 未配置 | 检查 XR Plugin Management → OpenXR |
| 手柄不响应 | Input Actions 未启用 | 确认 Action Asset 中的 Action Map 已启用 |
| 画面卡顿 | 性能不足 | 降低渲染分辨率、关闭阴影 |
| 签名错误 | keystore 问题 | 重新生成签名密钥 |
| 安装失败 | ABI 不匹配 | 确保只勾选 ARM64 |

### SteamVR 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| SteamVR 未启动 | 进程未运行 | 先启动 SteamVR 再运行游戏 |
| 手柄追踪丢失 | USB/蓝牙问题 | 检查基站和接收器 |
| 画面撕裂 | 帧率不足 | 降低画质设置 |

### 通用排查步骤

1. **检查 Unity Console**：有无报错或警告
2. **检查 XR Plugin Management**：确保目标平台的 Plugin 已启用
3. **检查 Input System**：确保 Action Asset 已配置且启用
4. **检查场景中的 XR Origin**：确保存在且配置正确
5. **尝试最小场景**：新建空场景只放 XR Origin 测试

---

## 参考资源

### 官方文档
- [Unity XR 开发文档](https://docs.unity3d.com/Manual/XR.html)
- [XR Interaction Toolkit 文档](https://docs.unity3d.com/Packages/com.unity.xr.interaction.toolkit@latest)
- [Meta Quest 开发者文档](https://developer.meta.com/horizon/documentation/)
- [SteamVR Unity Plugin](https://valvesoftware.github.io/steamvr_unity_plugin/)
- [Pico 开发者文档](https://developer-global.pico-interactive.com/documentation)

### 社区资源
- [Unity VR 开发论坛](https://forum.unity.com/forums/ar-vr-xr-discussion.80/)
- [Reddit r/Unity3D](https://www.reddit.com/r/unity3d/)
- [VR 开发者社区 Discord](https://discord.gg/vrdev)

### 推荐教程
- Unity Learn 官方 VR 课程
- Valem Tutorials (YouTube)
- Justin P. Barnett VR 教程系列

---

*建议先在 App Lab 发布测试版，收集用户反馈后再考虑正式商店发布。*
