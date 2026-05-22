# 性能优化

> 游戏卡顿、掉帧、内存飙升？性能优化是每个游戏开发者的必修课。这里整理了 Unity 中最实用的优化技巧。

## 📌 你会学到什么

- Profiler 工具使用（找到性能瓶颈）
- DrawCall 优化（渲染层面）
- 内存管理与 GC 优化（避免卡顿）
- LOD（层级细节）
- Occlusion Culling（遮挡剔除）
- 代码层面优化技巧

---

## 1. Profiler 使用

**先找到问题，再优化。不要盲目优化！**

### 打开方式
`Window → Analysis → Profiler`（快捷键 `Ctrl+7`）

### 关键指标

| 指标 | 含义 | 关注点 |
|------|------|--------|
| CPU Usage | 每帧 CPU 耗时 | 主线程、渲染、物理 |
| GPU Usage | 每帧 GPU 耗时 | 渲染、Shader 复杂度 |
| Memory | 内存占用 | Mono 堆、纹理、网格 |
| Batches | 批处理数量 | 越少越好 |
| SetPass Calls | 材质切换次数 | 越少越好 |

### 自定义 Profiler 采样

```csharp
using UnityEngine.Profiling;

public class PerformanceTest : MonoBehaviour
{
    void Update()
    {
        // 标记一段代码，方便在 Profiler 中查看
        Profiler.BeginSample("EnemyAI.Update");
        UpdateAllEnemies(); // 你想分析的代码
        Profiler.EndSample();
    }

    void UpdateAllEnemies()
    {
        // 模拟敌人更新逻辑
    }
}
```

---

## 2. DrawCall 优化

**DrawCall = CPU 通知 GPU "画一次" 的指令。次数越多，CPU 越忙。**

### 什么是批处理（Batching）

Unity 会把能合并的渲染请求合并成一次 DrawCall。

### 优化手段

```csharp
// 1. 使用 Static Batching
// 在 Inspector 中勾选物体的 "Static"，勾选 "Batching Static"
// 适用于：不会移动的场景物体（建筑、地形装饰）

// 2. 使用 Dynamic Batching
// 自动合并小网格（< 300 顶点）的动态物体
// Project Settings → Player → Other Settings → 勾选 Dynamic Batching

// 3. 减少材质种类 —— 使用 Texture Atlas（图集）
// 多个小贴图合成一张大贴图，减少材质切换

// 4. 使用 GPU Instancing（实例化）
// 大量相同物体时非常有效
// 在 Material 上勾选 "Enable GPU Instancing"
```

### GPU Instancing 示例

```csharp
public class GrassSpawner : MonoBehaviour
{
    [SerializeField] private GameObject grassPrefab;
    [SerializeField] private int count = 1000;

    void Start()
    {
        // 确保材质启用了 GPU Instancing
        for (int i = 0; i < count; i++)
        {
            Vector3 pos = new Vector3(
                Random.Range(-50f, 50f), 0, Random.Range(-50f, 50f));
            Instantiate(grassPrefab, pos, Quaternion.identity, transform);
        }
        // 如果材质启用了 Instancing，1000 棵草可能只需要几次 DrawCall
    }
}
```

---

## 3. 内存管理与 GC 优化

### GC（垃圾回收）为什么会卡？

C# 的 Mono/IL2CPP 使用自动内存管理。当堆内存积累到一定量，GC 会暂停游戏来回收垃圾。**这就是突然卡顿的原因！**

### 减少 GC 的技巧

```csharp
public class GCOptimization : MonoBehaviour
{
    // ❌ 坏习惯：每帧产生垃圾
    void BadUpdate()
    {
        // 每帧 new 一个字符串，产生垃圾
        string text = "Score: " + score.ToString();

        // 每帧 new 一个数组
        RaycastHit[] hits = Physics.RaycastAll(ray);
    }

    // ✅ 好习惯：复用对象，缓存结果
    private StringBuilder sb = new StringBuilder(); // 复用 StringBuilder
    private RaycastHit[] hitBuffer = new RaycastHit[10]; // 预分配数组
    private int hitCount;

    void GoodUpdate()
    {
        // 使用 StringBuilder 复用
        sb.Clear();
        sb.Append("Score: ");
        sb.Append(score);

        // 使用 NonAlloc 版本的物理检测
        hitCount = Physics.RaycastNonAlloc(ray, hitBuffer);
    }

    // ✅ 缓存组件引用
    private Rigidbody rb;

    void Start()
    {
        // 一次获取，反复使用
        rb = GetComponent<Rigidbody>();
    }
}
```

### GC 优化要点速查

| ❌ 产生垃圾 | ✅ 替代方案 |
|------------|-----------|
| `string + string` | `StringBuilder` 或 `string.Format` |
| `new List<>()` 每帧 | 缓存 List，使用 `.Clear()` |
| `foreach` 在 List 上 | `for` 循环（避免迭代器分配） |
| `GetComponent<>()` | 缓存到字段 |
| `Physics.RaycastAll()` | `RaycastNonAlloc()` |
| `LINQ` | 手写循环（LINQ 会产生闭包和迭代器） |

---

## 4. LOD（层级细节）

**远处的物体用低模，近处用高模。用画质换性能。**

### 设置步骤

1. 准备同一物体的 3 个版本（高、中、低精度模型）
2. 给父物体添加 `LOD Group` 组件
3. 把 3 个模型拖进去，设置距离阈值

```csharp
// 代码控制 LOD 切换距离
using UnityEngine;

public class CustomLOD : MonoBehaviour
{
    private LODGroup lodGroup;

    void Start()
    {
        lodGroup = GetComponent<LODGroup>();

        // 设置 LOD 距离（屏幕占比）
        LOD[] lods = new LOD[3];
        lods[0] = new LOD(0.6f, GetRenderers(0)); // 近处：60% 屏幕占比
        lods[1] = new LOD(0.3f, GetRenderers(1)); // 中间：30%
        lods[2] = new LOD(0.1f, GetRenderers(2)); // 远处：10%

        lodGroup.SetLODs(lods);
    }

    private Renderer[] GetRenderers(int lodIndex)
    {
        // 返回对应 LOD 级别的渲染器
        return transform.GetChild(lodIndex).GetComponentsInChildren<Renderer>();
    }
}
```

---

## 5. Occlusion Culling（遮挡剔除）

**被墙挡住的物体就不画。省渲染，不省 CPU。**

### 设置步骤

1. 标记场景物体为 `Occluder Static` 或 `Occludee Static`
2. `Window → Rendering → Occlusion Culling`
3. 点 Bake 烘焙
4. Play 测试

```csharp
// Occlusion Culling 是自动运行的，但你可以手动调试
void OnDrawGizmos()
{
    // 在 Scene 视图查看被剔除的物体
    if (Camera.main != null)
    {
        Camera.main.useOcclusionCulling = true; // 默认开启
    }
}
```

**适用场景：** 室内场景、迷宫、建筑内部（效果明显）
**不适用：** 开放世界、空旷地形（几乎没有遮挡）

---

## 6. 代码层面优化

### 6.1 避免在 Update 中做重操作

```csharp
// ❌ 每帧检查
void Update()
{
    GameObject player = GameObject.Find("Player"); // 严禁！
    float dist = Vector3.Distance(transform.position, player.transform.position);
}

// ✅ 缓存引用 + 减少频率
private Transform playerTransform;
private float checkInterval = 0.5f; // 每 0.5 秒检查一次
private float timer;

void Start()
{
    playerTransform = GameObject.Find("Player").transform;
}

void Update()
{
    timer += Time.deltaTime;
    if (timer >= checkInterval)
    {
        timer = 0;
        float dist = Vector3.SqrMagnitude(transform.position - playerTransform.position);
        // 用 SqrMagnitude 代替 Distance，省掉开方运算
    }
}
```

### 6.2 对象池（见单独章节）

### 6.3 使用值类型减少 GC

```csharp
// ✅ struct 替代 class（小对象用值类型）
public struct Point
{
    public float x, y, z;
}

// ✅ 使用 Vector3 而不是自定义 class
// Unity 的 Vector3/Quaternion/Color 都是 struct，不产生 GC
```

### 6.4 字符串优化

```csharp
// ❌ 字符串拼接
void Bad()
{
    for (int i = 0; i < 100; i++)
    {
        string log = "Enemy " + i + " spawned"; // 每次都 new string
    }
}

// ✅ StringBuilder
private StringBuilder sb = new StringBuilder();

void Good()
{
    for (int i = 0; i < 100; i++)
    {
        sb.Clear();
        sb.Append("Enemy ").Append(i).Append(" spawned");
    }
}
```

---

## 📊 优化优先级速查

```
性能问题排查顺序：
1. 先开 Profiler 看瓶颈在哪
2. CPU 密集 → 优化算法、减少 Update、对象池
3. GPU 密集 → 降低 DrawCall、LOD、Shader 优化
4. 内存问题 → GC 优化、纹理压缩、资源释放
5. 物理问题 → 减少 Collider、简化 MeshCollider、分层碰撞
```

---

## 🎯 性能优化黄金法则

1. **先测量，再优化** —— 不要猜
2. **优化最慢的部分** —— 木桶效应
3. **不要过度优化** —— 手机上 60 帧就够了
4. **目标平台决定一切** —— PC 能跑不代表手机能跑

---

> 💡 **记住：** "过早优化是万恶之源" —— Donald Knuth。先让游戏好玩，再让它流畅。
