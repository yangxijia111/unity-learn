# 物理系统

## 📋 目录

- [物理系统概述](#物理系统概述)
- [Rigidbody 刚体](#rigidbody-刚体)
- [Collider 碰撞器](#collider-碰撞器)
- [碰撞与触发事件](#碰撞与触发事件)
- [射线检测 Raycast](#射线检测-raycast)
- [物理材质](#物理材质)
- [Layer 与碰撞矩阵](#layer-与碰撞矩阵)
- [完整示例](#完整示例)

---

## 物理系统概述

Unity 内置 PhysX（3D）和 Box2D（2D）物理引擎，处理：

- 重力、碰撞、弹力
- 角色移动与跳跃
- 射击、抛射物
- 触发器检测（门、捡物品）

```
3D 物理（全小写）          2D 物理（带 2D 后缀）
Rigidbody                 Rigidbody2D
BoxCollider               BoxCollider2D
SphereCollider            CircleCollider2D
MeshCollider              PolygonCollider2D
OnCollisionEnter          OnCollisionEnter2D
OnTriggerEnter            OnTriggerEnter2D
```

---

## Rigidbody 刚体

Rigidbody 让物体受物理引擎控制（重力、力、碰撞响应）：

```csharp
using UnityEngine;

public class RigidbodyControl : MonoBehaviour
{
    private Rigidbody rb;

    [Header("移动设置")]
    public float moveSpeed = 5f;
    public float jumpForce = 10f;

    void Start()
    {
        rb = GetComponent<Rigidbody>();

        // 基本属性设置
        rb.mass = 1f;              // 质量
        rb.drag = 0f;              // 线性阻力（空气阻力）
        rb.angularDrag = 0.05f;    // 角阻力
        rb.useGravity = true;      // 是否受重力影响
        rb.isKinematic = false;    // true = 不受物理控制，用代码移动
    }

    void FixedUpdate()  // 物理操作放在 FixedUpdate
    {
        // 获取输入
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");

        // 施加力移动
        Vector3 moveDir = new Vector3(h, 0, v).normalized;
        rb.AddForce(moveDir * moveSpeed, ForceMode.Force);

        // 跳跃
        if (Input.GetKeyDown(KeyCode.Space))
        {
            rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
        }
    }

    // 各种施力方式
    void ForceExamples()
    {
        // ForceMode.Force - 持续力（默认，考虑质量）
        rb.AddForce(Vector3.forward * 10f, ForceMode.Force);

        // ForceMode.Impulse - 瞬间冲力（考虑质量）
        rb.AddForce(Vector3.up * 5f, ForceMode.Impulse);

        // ForceMode.VelocityChange - 直接改变速度（忽略质量）
        rb.AddForce(Vector3.forward * 10f, ForceMode.VelocityChange);

        // ForceMode.Acceleration - 持续加速度（忽略质量）
        rb.AddForce(Vector3.down * 9.8f, ForceMode.Acceleration);

        // 在指定位置施加力（产生扭矩）
        rb.AddForceAtPosition(Vector3.up * 10f, transform.position + Vector3.right);

        // 扭矩（旋转力）
        rb.AddTorque(Vector3.up * 5f);

        // 直接设置速度
        rb.velocity = Vector3.forward * 10f;

        // 直接设置角速度
        rb.angularVelocity = Vector3.up * 5f;
    }

    // 着地检测
    bool IsGrounded()
    {
        // 从脚下发射射线检测地面
        return Physics.Raycast(transform.position, Vector3.down, 1.1f);
    }
}
```

---

## Collider 碰撞器

Collider 定义物体的物理形状，用于碰撞检测：

```csharp
// 常用 Collider 类型：

// BoxCollider - 盒子（箱子、墙壁、地板）
BoxCollider box = gameObject.AddComponent<BoxCollider>();
box.size = new Vector3(1, 1, 1);      // 大小
box.center = Vector3.zero;             // 中心偏移

// SphereCollider - 球体（球、角色简化碰撞）
SphereCollider sphere = gameObject.AddComponent<SphereCollider>();
sphere.radius = 0.5f;

// CapsuleCollider - 胶囊体（最适合角色控制器）
CapsuleCollider capsule = gameObject.AddComponent<CapsuleCollider>();
capsule.height = 2f;
capsule.radius = 0.5f;

// MeshCollider - 精确网格碰撞（性能开销大）
MeshCollider mesh = gameObject.AddComponent<MeshCollider>();
mesh.sharedMesh = GetComponent<MeshFilter>().mesh;
mesh.convex = true;  // 凸包模式（必须勾选才能用于物理碰撞）

// 需要 Rigidbody 才能产生物理碰撞！
// Collider + Rigidbody = 完整的碰撞检测
```

---

## 碰撞与触发事件

### 碰撞事件（需要双方都有 Collider + 至少一方有 Rigidbody）

```csharp
using UnityEngine;

public class CollisionHandler : MonoBehaviour
{
    // 碰撞开始
    void OnCollisionEnter(Collision collision)
    {
        Debug.Log($"碰到: {collision.gameObject.name}");

        // 获取碰撞信息
        ContactPoint contact = collision.contacts[0];
        Vector3 hitPoint = contact.point;          // 碰撞点
        Vector3 hitNormal = contact.normal;         // 碰撞法线
        float impactForce = collision.impulse.magnitude; // 碰撞力

        // 检查碰撞物体的标签
        if (collision.gameObject.CompareTag("Enemy"))
        {
            Debug.Log("碰到了敌人！");
        }

        // 检查碰撞物体的层
        if (collision.gameObject.layer == LayerMask.NameToLayer("Ground"))
        {
            Debug.Log("碰到了地面");
        }
    }

    // 碰撞持续中（每帧调用）
    void OnCollisionStay(Collision collision)
    {
        // 比如：在地面上持续检测
    }

    // 碰撞结束
    void OnCollisionExit(Collision collision)
    {
        Debug.Log($"离开: {collision.gameObject.name}");
    }
}
```

### 触发事件（Collider 设为 isTrigger = true）

```csharp
public class TriggerHandler : MonoBehaviour
{
    // 进入触发区域
    void OnTriggerEnter(Collider other)
    {
        Debug.Log($"进入触发区: {other.gameObject.name}");

        // 捡金币
        if (other.CompareTag("Coin"))
        {
            GameManager.Instance.AddScore(10);
            Destroy(other.gameObject); // 销毁金币
        }

        // 进入危险区域
        if (other.CompareTag("DeathZone"))
        {
            Debug.Log("掉入死亡区域！");
            // 重生或游戏结束
        }

        // 进入传送门
        if (other.CompareTag("Portal"))
        {
            SceneLoader.Instance.LoadScene("Level2");
        }
    }

    // 在触发区域内持续
    void OnTriggerStay(Collider other)
    {
        // 比如：在毒气区域内持续扣血
        if (other.CompareTag("Poison"))
        {
            // 持续伤害逻辑
        }
    }

    // 离开触发区域
    void OnTriggerExit(Collider other)
    {
        Debug.Log($"离开触发区: {other.gameObject.name}");
    }
}
```

---

## 射线检测 Raycast

射线检测是物理系统中非常常用的功能（射击、选择、地面检测）：

```csharp
using UnityEngine;

public class RaycastExamples : MonoBehaviour
{
    [Header("射击设置")]
    public float shootRange = 100f;
    public int damage = 25;
    public LayerMask hitLayers;        // 可射击的层

    void Update()
    {
        // 鼠标左键射击
        if (Input.GetMouseButtonDown(0))
        {
            Shoot();
        }

        // 鼠标右键选择物体
        if (Input.GetMouseButtonDown(1))
        {
            SelectObject();
        }
    }

    void Shoot()
    {
        // 从屏幕中心发射射线（摄像机视角）
        Ray ray = Camera.main.ScreenPointToRay(new Vector3(
            Screen.width / 2f,
            Screen.height / 2f,
            0
        ));

        RaycastHit hit;

        // 射线检测
        if (Physics.Raycast(ray, out hit, shootRange, hitLayers))
        {
            Debug.Log($"击中: {hit.collider.gameObject.name}");
            Debug.Log($"距离: {hit.distance}m");
            Debug.Log($"击中点: {hit.point}");
            Debug.Log($"击中法线: {hit.normal}");

            // 造成伤害
            EnemyHealth enemy = hit.collider.GetComponent<EnemyHealth>();
            if (enemy != null)
            {
                enemy.TakeDamage(damage);
            }

            // 生成弹孔特效
            // Instantiate(bulletHolePrefab, hit.point, Quaternion.LookRotation(hit.normal));
        }
    }

    void SelectObject()
    {
        // 从鼠标位置发射射线
        Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);
        RaycastHit hit;

        if (Physics.Raycast(ray, out hit))
        {
            Selectable obj = hit.collider.GetComponent<Selectable>();
            if (obj != null)
            {
                obj.OnSelected();
            }
        }
    }

    // 地面检测（从角色脚下向下发射射线）
    bool IsGrounded()
    {
        Vector3 origin = transform.position + Vector3.up * 0.1f;
        float distance = 1.2f;

        return Physics.Raycast(origin, Vector3.down, distance);
    }

    // 球形检测（检测范围内的所有物体）
    void SphereCheck()
    {
        float radius = 5f;
        Collider[] hitColliders = Physics.OverlapSphere(transform.position, radius);

        foreach (Collider col in hitColliders)
        {
            if (col.CompareTag("Enemy"))
            {
                Debug.Log($"范围内有敌人: {col.name}");
            }
        }
    }

    // 盒子检测
    void BoxCheck()
    {
        Vector3 halfExtents = new Vector3(1, 1, 1);
        Collider[] hits = Physics.OverlapBox(
            transform.position,
            halfExtents,
            transform.rotation
        );
    }
}
```

---

## 物理材质

控制物体碰撞时的摩擦力和弹力：

```csharp
// 创建物理材质：
// Project 面板 → 右键 → Create → Physic Material

// 代码中使用物理材质
void SetupPhysicsMaterial()
{
    Collider col = GetComponent<Collider>();

    // 创建物理材质
    PhysicMaterial bouncyMat = new PhysicMaterial("Bouncy");
    bouncyMat.bounciness = 0.8f;       // 弹力（0=不弹，1=完美弹性）
    bouncyMat.dynamicFriction = 0.2f;  // 动摩擦
    bouncyMat.staticFriction = 0.2f;   // 静摩擦
    bouncyMat.bounceCombine = PhysicMaterialCombine.Maximum; // 弹力组合方式

    col.material = bouncyMat;
}

// 常见物理材质设置：
// 冰面: 摩擦力 0.05, 弹力 0
// 橡皮球: 摩擦力 0.5, 弹力 0.8
// 地面: 摩擦力 0.6, 弹力 0
```

---

## Layer 与碰撞矩阵

**Layer（层）** 用于分类 GameObject，**碰撞矩阵** 决定哪些层之间可以碰撞：

```csharp
// 设置 Layer（最多 32 层）
// Edit → Project Settings → Tags and Layers

// 在 Inspector 中为 GameObject 指定 Layer
gameObject.layer = LayerMask.NameToLayer("Player");

// LayerMask 用法
public class LayerExamples : MonoBehaviour
{
    public LayerMask groundLayer;      // 在 Inspector 中勾选
    public LayerMask enemyLayer;

    void Start()
    {
        // 通过名称设置层
        int playerLayer = LayerMask.NameToLayer("Player");
        gameObject.layer = playerLayer;

        // 射线检测只检测特定层
        RaycastHit hit;
        // 两种写法效果一样：
        if (Physics.Raycast(transform.position, Vector3.down, out hit, 10f, groundLayer))
        {
            Debug.Log("检测到地面");
        }

        // 手动创建 LayerMask
        int layerMask = (1 << 6) | (1 << 7);  // 检测第6层和第7层
        if (Physics.Raycast(transform.position, Vector3.down, out hit, 10f, layerMask))
        {
            Debug.Log("检测到指定层");
        }
    }

    // 碰撞矩阵设置：
    // Edit → Project Settings → Physics
    // 勾选/取消勾选 = 允许/禁止两层之间碰撞
    // 比如：取消 Player 层和 Player 层的碰撞（玩家之间不碰撞）
}
```

---

## 完整示例

一个完整的玩家控制器（含物理）：

```csharp
using UnityEngine;

/// <summary>
/// 物理驱动的玩家控制器
/// 功能：移动、跳跃、地面检测、斜坡处理
/// </summary>
[RequireComponent(typeof(Rigidbody))]
[RequireComponent(typeof(CapsuleCollider))]
public class PhysicsPlayerController : MonoBehaviour
{
    [Header("移动")]
    public float moveSpeed = 7f;
    public float airControlMultiplier = 0.3f;

    [Header("跳跃")]
    public float jumpForce = 8f;
    public LayerMask groundLayer;

    [Header("斜坡")]
    public float maxSlopeAngle = 45f;

    private Rigidbody rb;
    private CapsuleCollider capsule;
    private bool isGrounded;
    private bool jumpRequested;

    void Start()
    {
        rb = GetComponent<Rigidbody>();
        capsule = GetComponent<CapsuleCollider>();

        // 冻结旋转，防止角色倾倒
        rb.freezeRotation = true;
        rb.interpolation = RigidbodyInterpolation.Interpolate;
    }

    void Update()
    {
        // 在 Update 中检测输入
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            jumpRequested = true;
        }
    }

    void FixedUpdate()
    {
        // 地面检测
        CheckGround();

        // 移动
        Move();

        // 跳跃
        if (jumpRequested)
        {
            Jump();
            jumpRequested = false;
        }
    }

    void CheckGround()
    {
        // 从角色底部发射射线
        float radius = capsule.radius * 0.9f;
        Vector3 origin = transform.position + Vector3.up * 0.1f;
        isGrounded = Physics.SphereCast(origin, radius, Vector3.down, out _, 0.2f, groundLayer);
    }

    void Move()
    {
        float h = Input.GetAxis("Horizontal");
        float v = Input.GetAxis("Vertical");

        Vector3 inputDir = transform.right * h + transform.forward * v;
        inputDir = Vector3.ClampMagnitude(inputDir, 1f);

        // 地面和空中不同控制力
        float multiplier = isGrounded ? 1f : airControlMultiplier;

        // 计算目标速度
        Vector3 targetVel = inputDir * moveSpeed;
        Vector3 currentVel = rb.velocity;
        Vector3 velocityChange = targetVel - new Vector3(currentVel.x, 0, currentVel.z);

        rb.AddForce(velocityChange * multiplier, ForceMode.VelocityChange);
    }

    void Jump()
    {
        // 重置垂直速度再跳跃，保证跳跃高度一致
        rb.velocity = new Vector3(rb.velocity.x, 0, rb.velocity.z);
        rb.AddForce(Vector3.up * jumpForce, ForceMode.Impulse);
    }

    // 调试可视化
    void OnDrawGizmos()
    {
        // 显示地面检测范围
        Gizmos.color = isGrounded ? Color.green : Color.red;
        Gizmos.DrawWireSphere(transform.position + Vector3.up * 0.1f, 0.3f);
    }
}
```

### 要点总结

| 概念 | 说明 |
|------|------|
| Rigidbody | 受物理引擎控制的刚体 |
| Collider | 定义碰撞形状（不产生物理力） |
| isTrigger | 触发器模式，只检测不碰撞 |
| OnCollisionXxx | 碰撞事件 |
| OnTriggerXxx | 触发事件 |
| FixedUpdate | 物理相关逻辑放这里 |
| Raycast | 射线检测 |
| Layer | 分类 + 碰撞过滤 |
| 物理材质 | 控制摩擦力和弹力 |

---

> 📌 下一步：学习 [2D 精灵与动画](../07-2D精灵与动画/README.md)
