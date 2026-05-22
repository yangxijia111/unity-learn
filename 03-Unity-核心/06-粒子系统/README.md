# 06 - 粒子系统 (Particle System)

Unity 的 Particle System 用于创建火焰、烟雾、魔法特效、爆炸等视觉效果。

## 创建方式

- **GameObject → Effects → Particle System**
- 或给物体添加 **Particle System** 组件

## 核心模块概览

粒子系统由多个模块组成，在 Inspector 中展开/折叠：

| 模块 | 作用 |
|------|------|
| **Main** | 基础属性（时长、循环、起始大小/颜色/速度/生命周期） |
| **Emission** | 发射速率、爆发点 |
| **Shape** | 发射形状（球、圆锥、盒子、圆环等） |
| **Velocity over Lifetime** | 速度随时间变化 |
| **Color over Lifetime** | 颜色随时间渐变（淡入淡出） |
| **Size over Lifetime** | 大小随时间变化 |
| **Rotation over Lifetime** | 旋转随时间变化 |
| **Collision** | 碰撞检测 |
| **Renderer** | 渲染模式（Billboard、Mesh）、材质 |

## Inspector 常用设置

### Main 模块
- `Duration`：单次循环时长
- `Looping`：是否循环
- `Start Lifetime`：粒子存活时间
- `Start Speed`：初始速度
- `Start Size`：初始大小
- `Start Color`：初始颜色
- `Simulation Space`：Local（跟随物体）或 World（不跟随）

### Emission 模块
- `Rate over Time`：每秒发射粒子数
- `Bursts`：在指定时间点爆发 N 个粒子

### Shape 模块
- `Shape`：Sphere、Hemisphere、Cone、Box、Circle、Edge 等
- `Radius`：发射范围

### Color over Lifetime
- 勾选后用渐变条设置颜色变化
- 常用于：**Alpha 从 1→0 实现淡出效果**

## 代码控制

```csharp
using UnityEngine;

public class ParticleControl : MonoBehaviour
{
    private ParticleSystem ps;

    void Start()
    {
        ps = GetComponent<ParticleSystem>();

        // 基础属性
        var main = ps.main;
        main.duration = 2f;
        main.loop = true;
        main.startLifetime = 1.5f;
        main.startSpeed = 5f;
        main.startSize = 0.5f;
        main.startColor = Color.yellow;
        main.simulationSpace = ParticleSystemSimulationSpace.World;

        // 发射设置
        var emission = ps.emission;
        emission.rateOverTime = 20f;

        // 形状
        var shape = ps.shape;
        shape.shapeType = ParticleSystemShapeType.Cone;
        shape.angle = 25f;
        shape.radius = 0.5f;
    }

    void Update()
    {
        // 按空格手动发射一次
        if (Input.GetKeyDown(KeyCode.Space))
        {
            ps.Emit(30); // 立即发射 30 个粒子
        }

        // 停止发射
        if (Input.GetKeyDown(KeyCode.S))
        {
            ps.Stop();          // 停止发射，现有粒子继续播放
            // ps.Stop(true, ParticleSystemStopBehavior.StopEmittingAndClear); // 立即清除
        }
    }

    // 检查粒子是否播放完毕
    bool IsFinished()
    {
        return !ps.isPlaying && !ps.isEmitting;
    }
}
```

## 实战特效示例

### 爆炸特效

```csharp
public class ExplosionEffect : MonoBehaviour
{
    public ParticleSystem explosionPrefab;

    public void Explode(Vector3 position)
    {
        ParticleSystem explosion = Instantiate(explosionPrefab, position, Quaternion.identity);

        // 播放完自动销毁
        Destroy(explosion.gameObject, explosion.main.duration + explosion.main.startLifetime.constantMax);
    }
}
```

### 受击火花

```csharp
public class HitEffect : MonoBehaviour
{
    public ParticleSystem hitSpark;

    void OnCollisionEnter(Collision collision)
    {
        // 在碰撞点生成火花
        ContactPoint contact = collision.GetContact(0);
        ParticleSystem spark = Instantiate(hitSpark, contact.point, Quaternion.identity);

        // 让火花朝碰撞法线方向发射
        var shape = spark.shape;
        shape.rotation = Quaternion.LookRotation(contact.normal).eulerAngles;

        Destroy(spark.gameObject, 2f);
    }
}
```

### 拖尾特效（Trail Renderer + Particle）

```csharp
public class ProjectileTrail : MonoBehaviour
{
    public ParticleSystem trailEffect;

    void Start()
    {
        // 让粒子跟随物体移动
        var main = trailEffect.main;
        main.simulationSpace = ParticleSystemSimulationSpace.Local;

        // 调整拖尾形状
        var shape = trailEffect.shape;
        shape.enabled = false; // 关闭形状，粒子从中心发射

        // 速度为零，只做拖尾
        main.startSpeed = 0f;
        main.startLifetime = 0.5f;
    }
}
```

### 循环渐变颜色

```csharp
public class ColorCycleParticle : MonoBehaviour
{
    public ParticleSystem ps;

    void Start()
    {
        // 通过脚本设置颜色渐变
        var colorOverLifetime = ps.colorOverLifetime;
        colorOverLifetime.enabled = true;

        Gradient gradient = new Gradient();
        gradient.SetKeys(
            new GradientColorKey[] {
                new GradientColorKey(Color.red, 0f),
                new GradientColorKey(Color.yellow, 0.5f),
                new GradientColorKey(Color.blue, 1f)
            },
            new GradientAlphaKey[] {
                new GradientAlphaKey(1f, 0f),
                new GradientAlphaKey(0f, 1f) // 逐渐消失
            }
        );

        colorOverLifetime.color = new ParticleSystem.MinMaxGradient(gradient);
    }
}
```

## 性能优化建议

1. **控制粒子数量** —— 移动端建议单个系统不超过 100 个粒子
2. **用 Texture Sheet Animation** —— 用序列帧动画代替多个粒子实现复杂效果
3. **及时销毁** —— 一次性特效播放完立即销毁
4. **使用对象池** —— 高频创建/销毁的特效（如子弹命中）用对象池
5. **减少 Overdraw** —— 降低粒子大小或数量，避免半透明大面积叠加

## 注意事项

- `Simulation Space = World` 适合爆炸、技能等不需要跟随物体的效果
- `Simulation Space = Local` 适合拖尾、光环等需要跟随物体的效果
- 粒子渲染顺序由 **Sorting Layer / Order in Layer** 控制
- 粒子材质需要使用 **Particles/Standard Unlit** 或对应 Shader
- 使用 **Particle System Force Field** 可以实现风力、漩涡等环境效果
