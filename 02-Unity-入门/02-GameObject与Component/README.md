# GameObject 与 Component

## 📋 目录

- [GameObject 概念](#gameobject-概念)
- [Component 组件系统](#component-组件系统)
- [添加组件 AddComponent](#添加组件-addcomponent)
- [常用组件介绍](#常用组件介绍)
- [获取组件 GetComponent](#获取组件-getcomponent)
- [完整示例](#完整示例)

---

## GameObject 概念

**GameObject（游戏对象）** 是 Unity 中最基本的构建块。

- 场景中的**一切**都是 GameObject（角色、灯光、摄像机、空物体…）
- GameObject 本身是"空的容器"，靠**组件（Component）**赋予功能
- 每个 GameObject 至少有一个 `Transform` 组件（确定位置）

```csharp
// 创建一个空的 GameObject
GameObject emptyObj = new GameObject("MyEmptyObject");

// 也可以通过预制体实例化（见第04章）
```

---

## Component 组件系统

Unity 使用 **ECS（实体-组件系统）** 思想：

```
GameObject（实体）
  ├── Transform（位置/旋转/缩放）  ← 始终存在
  ├── Rigidbody（物理）
  ├── Collider（碰撞检测）
  ├── MeshRenderer（渲染网格）
  └── 自定义脚本（你写的 C# MonoBehaviour）
```

**核心原则：组合优于继承**
- 不用继承一个复杂的"Player"类
- 而是给空 GameObject 挂上不同的组件来组合功能

---

## 添加组件 AddComponent

```csharp
using UnityEngine;

public class AddComponentExample : MonoBehaviour
{
    void Start()
    {
        // 创建一个新对象
        GameObject obj = new GameObject("Cube");

        // 添加 Rigidbody 组件（3D 刚体）
        Rigidbody rb = obj.AddComponent<Rigidbody>();

        // 添加自定义脚本组件
        MyCustomScript script = obj.AddComponent<MyCustomScript>();

        // 添加碰撞器
        BoxCollider collider = obj.AddComponent<BoxCollider>();

        Debug.Log("组件添加完成！");
    }
}
```

---

## 常用组件介绍

### Transform（变换）
每个 GameObject 必有的组件，控制位置、旋转、缩放。

```csharp
// Transform 基本操作
transform.position = new Vector3(0, 1, 0);    // 设置世界坐标
transform.rotation = Quaternion.identity;       // 重置旋转
transform.localScale = Vector3.one;             // 重置缩放为(1,1,1)

Debug.Log("当前位置: " + transform.position);
```

### Renderer（渲染器）
控制物体的外观显示。

```csharp
// MeshRenderer - 渲染 3D 网格
MeshRenderer renderer = GetComponent<MeshRenderer>();
renderer.material.color = Color.red;  // 改变材质颜色

// SpriteRenderer - 渲染 2D 精灵（第07章详述）
// SpriteRenderer sr = GetComponent<SpriteRenderer>();
```

### Collider（碰撞器）
让物体能被物理系统检测到。

```csharp
// BoxCollider - 盒子碰撞器
BoxCollider box = gameObject.AddComponent<BoxCollider>();
box.size = new Vector3(1, 1, 1);      // 碰撞器大小
box.center = Vector3.zero;             // 碰撞器中心偏移
box.isTrigger = true;                  // 设为触发器（不产生物理碰撞）
```

### Rigidbody（刚体）
让物体受物理引擎控制。

```csharp
// Rigidbody - 3D 刚体
Rigidbody rb = gameObject.AddComponent<Rigidbody>();
rb.mass = 1.0f;                        // 质量
rb.drag = 0.5f;                        // 空气阻力
rb.useGravity = true;                  // 是否受重力影响

// 用代码施加力
rb.AddForce(Vector3.up * 10f, ForceMode.Impulse); // 向上冲力
```

---

## 获取组件 GetComponent

脚本经常需要访问同一 GameObject 上的其他组件：

```csharp
using UnityEngine;

public class GetComponentExample : MonoBehaviour
{
    private Rigidbody rb;
    private MeshRenderer meshRenderer;

    void Start()
    {
        // 获取自身上的组件
        rb = GetComponent<Rigidbody>();
        meshRenderer = GetComponent<MeshRenderer>();

        if (rb == null)
        {
            Debug.LogWarning("没找到 Rigidbody 组件！");
        }
    }

    void Update()
    {
        // 使用获取到的组件
        if (Input.GetKeyDown(KeyCode.Space) && rb != null)
        {
            rb.AddForce(Vector3.up * 5f, ForceMode.Impulse);
        }
    }

    // 获取子对象上的组件
    void FindChildComponent()
    {
        // 在所有子对象中搜索
        Collider[] childColliders = GetComponentsInChildren<Collider>();

        // 获取指定子对象上的组件
        Transform child = transform.Find("ChildName");
        if (child != null)
        {
            Rigidbody childRb = child.GetComponent<Rigidbody>();
        }
    }

    // 通过名称查找其他 GameObject 上的组件
    void FindOtherObjectComponent()
    {
        GameObject otherObj = GameObject.Find("Enemy");
        if (otherObj != null)
        {
            EnemyScript enemy = otherObj.GetComponent<EnemyScript>();
        }

        // 用标签查找
        GameObject player = GameObject.FindGameObjectWithTag("Player");
    }
}
```

---

## 完整示例

一个简单的可交互对象，演示组件组合的威力：

```csharp
using UnityEngine;

/// <summary>
/// 一个可以被推动的箱子
/// 需要组件：Rigidbody + BoxCollider
/// </summary>
[RequireComponent(typeof(Rigidbody))]        // 确保挂载时自动添加 Rigidbody
[RequireComponent(typeof(BoxCollider))]      // 确保挂载时自动添加 BoxCollider
public class PushableBox : MonoBehaviour
{
    [Header("设置")]
    public float pushForce = 5f;
    public Color highlightColor = Color.yellow;

    private Rigidbody rb;
    private MeshRenderer meshRenderer;
    private Color originalColor;

    void Start()
    {
        // 获取所需组件
        rb = GetComponent<Rigidbody>();
        meshRenderer = GetComponent<MeshRenderer>();

        // 保存原始颜色
        if (meshRenderer != null)
        {
            originalColor = meshRenderer.material.color;
        }
    }

    // 当有物体碰到这个箱子时
    void OnCollisionEnter(Collision collision)
    {
        // 检查是否被玩家推动
        if (collision.gameObject.CompareTag("Player"))
        {
            // 计算推力方向
            Vector3 pushDir = collision.transform.position - transform.position;
            pushDir.y = 0; // 不在 Y 轴施力

            // 施加推力
            rb.AddForce(-pushDir.normalized * pushForce, ForceMode.Impulse);

            Debug.Log("箱子被推动了！");
        }
    }

    // 鼠标悬停高亮（需要 Collider 组件）
    void OnMouseEnter()
    {
        if (meshRenderer != null)
            meshRenderer.material.color = highlightColor;
    }

    void OnMouseExit()
    {
        if (meshRenderer != null)
            meshRenderer.material.color = originalColor;
    }
}
```

### 知识点总结

- **GameObject** 是容器，**Component** 是功能
- `[RequireComponent]` 确保依赖组件自动存在
- `GetComponent<T>()` 是组件之间通信的核心 API
- 组合 > 继承，这是 Unity 设计哲学

---

> 📌 下一步：学习 [Transform 系统](../03-Transform系统/README.md)
