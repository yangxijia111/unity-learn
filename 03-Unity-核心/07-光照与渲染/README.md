# 07 - 光照与渲染 (Lighting & Rendering)

Unity 的光照和渲染管线决定了游戏的视觉表现，从基础灯光到后处理都需要理解。

## Light 灯光类型

### Directional Light（方向光）
- 模拟**太阳/月亮**，影响整个场景
- 位置无关，只看旋转方向
- 场景中通常放一个就够了

```csharp
// 代码动态修改方向光
Light sun = FindObjectOfType<Light>();
sun.type = LightType.Directional;
sun.color = Color.yellow;
sun.intensity = 1.2f;
sun.transform.rotation = Quaternion.Euler(50f, -30f, 0f);
```

### Point Light（点光源）
- 像灯泡，向四周发光
- 受位置和范围影响

```csharp
Light point = gameObject.AddComponent<Light>();
point.type = LightType.Point;
point.color = Color.red;
point.range = 10f;       // 照射范围
point.intensity = 2f;
```

### Spot Light（聚光灯）
- 像手电筒/舞台灯，锥形照射
- 受位置、方向、角度影响

```csharp
Light spot = gameObject.AddComponent<Light>();
spot.type = LightType.Spot;
spot.color = Color.white;
spot.range = 20f;
spot.spotAngle = 45f;    // 锥形角度
spot.intensity = 3f;
```

### Area Light（区域光）
- 仅支持 **Baked GI**（烘焙光照贴图）
- 模拟面光源（如窗户、天花板灯管）
- 不能实时使用

## 渲染管线 (Render Pipeline)

### Built-in Render Pipeline（内置）
- Unity 默认管线，兼容性最好
- 功能有限，性能一般
- 适合学习和小型项目

### URP（Universal Render Pipeline）
- 轻量级可编程管线，**推荐大多数项目使用**
- 支持 2D/3D、移动端优化、Shader Graph
- 后处理效果内置

### HDRP（High Definition Render Pipeline）
- 高保真管线，PC/主机 3A 级画质
- 性能开销大，不适合移动端

| 管线 | 适用平台 | 画质 | 性能 |
|------|---------|------|------|
| Built-in | 全平台 | 中 | 中 |
| URP | 全平台（含移动端） | 中高 | 优 |
| HDRP | PC/主机 | 极高 | 高开销 |

## Camera 摄像机

```csharp
public class CameraSetup : MonoBehaviour
{
    private Camera cam;

    void Start()
    {
        cam = Camera.main;

        // 基础设置
        cam.clearFlags = CameraClearFlags.SolidColor; // Skybox / SolidColor / Depth
        cam.backgroundColor = Color.black;
        cam.fieldOfView = 60f;   // 视野角度
        cam.nearClipPlane = 0.1f; // 近裁剪面
        cam.farClipPlane = 1000f; // 远裁剪面

        // 正交模式（2D 游戏）
        cam.orthographic = true;
        cam.orthographicSize = 5f;

        // 渲染层级（只渲染指定层）
        cam.cullingMask = ~LayerMask.GetMask("UI"); // 不渲染 UI 层
    }

    // 屏幕坐标转世界坐标
    Vector3 ScreenToWorld(Vector3 screenPos)
    {
        screenPos.z = 10f; // 距离摄像机的距离
        return cam.ScreenToWorldPoint(screenPos);
    }

    // 世界坐标转屏幕坐标
    Vector3 WorldToScreen(Vector3 worldPos)
    {
        return cam.WorldToScreenPoint(worldPos);
    }
}
```

## 光照模式

| 模式 | 说明 |
|------|------|
| **Realtime** | 每帧实时计算，效果最好但开销最大 |
| **Mixed** | 静态物体烘焙，动态物体实时计算，**推荐** |
| **Baked** | 预先烘焙到光照贴图，运行时零开销 |

## 光照贴图（Lightmapping）

1. 将不动的物体标记为 **Static**
2. **Window → Rendering → Lighting** 打开 Lighting 窗口
3. 点击 **Generate Lighting** 烘焙
4. 烘焙后灯光信息保存在光照贴图纹理中

## 后处理（Post-Processing）

### URP 后处理

1. 添加 **Global Volume** 组件
2. 创建 **Volume Profile** 资产
3. 添加效果覆盖

### 常用后处理效果

```csharp
// URP 后处理通过 Volume 组件控制，代码修改示例：
using UnityEngine;
using UnityEngine.Rendering;
using UnityEngine.Rendering.Universal;

public class PostProcessControl : MonoBehaviour
{
    private Volume volume;
    private Bloom bloom;
    private Vignette vignette;
    private ColorAdjustments colorAdjustments;

    void Start()
    {
        volume = GetComponent<Volume>();
        volume.profile.TryGet(out bloom);
        volume.profile.TryGet(out vignette);
        volume.profile.TryGet(out colorAdjustments);
    }

    // 受伤效果：暗角 + 红色偏移
    public void HitEffect()
    {
        vignette.intensity.value = 0.5f;
        colorAdjustments.colorFilter.value = Color.red;
    }

    public void ResetEffect()
    {
        vignette.intensity.value = 0f;
        colorAdjustments.colorFilter.value = Color.white;
    }

    // 发光增强（拾取道具时）
    public void BoostBloom(float duration)
    {
        StartCoroutine(BloomPulse(duration));
    }

    System.Collections.IEnumerator BloomPulse(float duration)
    {
        float original = bloom.intensity.value;
        bloom.intensity.value = 3f;
        yield return new WaitForSeconds(duration);
        bloom.intensity.value = original;
    }
}
```

### 常见后处理效果一览

| 效果 | 用途 |
|------|------|
| **Bloom** | 发光/泛光（灯光、特效边缘发光） |
| **Vignette** | 暗角（受击、紧张氛围） |
| **Color Grading** | 色彩调整（电影感、冷暖色调） |
| **Depth of Field** | 景深模糊（过场动画、焦点效果） |
| **Motion Blur** | 运动模糊（高速移动） |
| **Chromatic Aberration** | 色差（受击、CRT 风格） |

## 实用技巧

- **Light Probe**：给动态物体提供烘焙光照信息（否则动态物体在烘焙场景中会很暗）
- **Reflection Probe**：提供环境反射（金属/光滑材质需要）
- **Shadow Distance**：在 Quality Settings 中设置阴影距离，远了不渲染阴影以提升性能
- **Light Layer**：让灯光只影响指定物体（URP 支持）

## 注意事项

1. 方向光的 **Shadow Type** 开启后性能开销较大，移动端慎重
2. 实时光源数量要控制，**移动端建议不超过 2-3 个实时灯光**
3. 2D 游戏用 **2D Renderer** + **2D Light** 代替 3D 灯光
4. 切换渲染管线需要在 **Graphics** 和 **Quality** 设置中同时指定 Pipeline Asset
5. 后处理效果在 **URP/HDRP** 中实现方式不同，不要混用代码
