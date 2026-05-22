# Transform 系统

## 📋 目录

- [基本属性](#基本属性)
- [Local 与 World 空间](#local-与-world-空间)
- [移动与旋转](#移动与旋转)
- [LookAt 朝向](#lookat-朝向)
- [父子关系](#父子关系)
- [综合示例](#综合示例)

---

## 基本属性

Transform 是每个 GameObject 必有的组件，控制物体在空间中的状态：

```csharp
// 三个核心属性
transform.position      // 世界空间位置 (Vector3)
transform.rotation      // 世界空间旋转 (Quaternion)
transform.localScale    // 本地缩放 (Vector3)

// 本地空间（相对于父物体）
transform.localPosition
transform.localRotation
```

```csharp
void Start()
{
    // 设置位置
    transform.position = new Vector3(0, 2, 5);

    // 设置旋转（四元数方式）
    transform.rotation = Quaternion.Euler(0, 45, 0);

    // 设置缩放
    transform.localScale = new Vector3(2, 2, 2);
}
```

---

## Local 与 World 空间

**World（世界空间）**：整个场景的统一坐标系，原点在场景中心。

**Local（本地空间）**：相对于父物体的坐标系。

```csharp
void SpaceExample()
{
    // World 空间：物体在世界中的绝对位置
    Vector3 worldPos = transform.position;

    // Local 空间：物体相对于父物体的位置
    Vector3 localPos = transform.localPosition;

    // 将本地坐标转为世界坐标
    Vector3 worldPoint = transform.TransformPoint(new Vector3(1, 0, 0));

    // 将世界坐标转为本地坐标
    Vector3 localPoint = transform.InverseTransformPoint(worldPoint);

    // 方向向量的转换（不受位置影响）
    Vector3 worldDir = transform.TransformDirection(Vector3.forward);
    Vector3 localDir = transform.InverseTransformDirection(worldDir);
}
```

---

## 移动与旋转

### Translate（平移）

```csharp
void Update()
{
    // 按世界坐标移动（每秒移动 5 个单位）
    transform.Translate(Vector3.forward * 5f * Time.deltaTime);

    // 按自身坐标移动（物体面朝的方向）
    transform.Translate(Vector3.forward * 5f * Time.deltaTime, Space.Self);

    // 按世界坐标移动
    transform.Translate(Vector3.forward * 5f * Time.deltaTime, Space.World);
}
```

### Rotate（旋转）

```csharp
void Update()
{
    // 绕 Y 轴旋转（每秒 90 度）
    transform.Rotate(Vector3.up, 90f * Time.deltaTime);

    // 绕指定轴旋转
    transform.Rotate(0, 90f * Time.deltaTime, 0);

    // 绕指定点旋转
    transform.RotateAround(
        Vector3.zero,          // 围绕点
        Vector3.up,            // 旋转轴
        90f * Time.deltaTime   // 旋转角度
    );
}
```

### 平滑移动（插值）

```csharp
public Transform target;        // 目标位置
public float moveSpeed = 5f;
public float rotateSpeed = 5f;

void Update()
{
    // 平滑移动到目标位置
    transform.position = Vector3.Lerp(
        transform.position,
        target.position,
        moveSpeed * Time.deltaTime
    );

    // 平滑旋转到目标朝向
    transform.rotation = Quaternion.Slerp(
        transform.rotation,
        target.rotation,
        rotateSpeed * Time.deltaTime
    );
}
```

---

## LookAt 朝向

让物体面朝某个方向或目标：

```csharp
public Transform target;

void Update()
{
    // 面朝目标（瞬间转向）
    transform.LookAt(target);

    // 只在 Y 轴上朝向（适合坦克、人物在地面移动）
    transform.LookAt(new Vector3(
        target.position.x,
        transform.position.y,
        target.position.z
    ));

    // 面朝指定方向
    transform.forward = Vector3.forward; // 面朝 Z 轴正方向
}
```

### 平滑朝向

```csharp
public Transform target;
public float rotateSpeed = 5f;

void Update()
{
    // 计算朝向目标的方向
    Vector3 direction = target.position - transform.position;
    direction.y = 0; // 忽略高度差

    if (direction != Vector3.zero)
    {
        // 平滑旋转
        Quaternion targetRotation = Quaternion.LookRotation(direction);
        transform.rotation = Quaternion.Slerp(
            transform.rotation,
            targetRotation,
            rotateSpeed * Time.deltaTime
        );
    }
}
```

---

## 父子关系

父子关系让物体跟随移动，形成层级结构：

```
父物体 (Parent)
  ├── 子物体1 (Child)    ← 跟随父物体移动
  └── 子物体2 (Child)
        └── 孙子物体     ← 跟随子物体2移动
```

```csharp
public Transform parentObject;
public Transform childObject;

void ParentingExamples()
{
    // 设置父物体
    childObject.SetParent(parentObject);

    // 设置父物体，保持世界坐标不变
    childObject.SetParent(parentObject, true);

    // 取消父子关系（成为顶级物体）
    childObject.SetParent(null);

    // 遍历所有子物体
    foreach (Transform child in transform)
    {
        Debug.Log("子物体: " + child.name);
    }

    // 查找指定子物体
    Transform found = transform.Find("子物体名称");

    // 获取子物体数量
    int childCount = transform.childCount;

    // 获取自身在父物体中的索引
    int siblingIndex = transform.GetSiblingIndex();

    // 设置同级顺序
    transform.SetSiblingIndex(0); // 移到最前面
}
```

---

## 综合示例

一个简易的轨道系统（行星绕太阳旋转）：

```csharp
using UnityEngine;

/// <summary>
/// 简易轨道系统
/// 太阳（空物体） → 行星（子物体）
/// 行星自转 + 绕太阳公转
/// </summary>
public class SimpleOrbit : MonoBehaviour
{
    [Header("公转")]
    public Transform centerPoint;       // 公转中心（太阳）
    public float orbitSpeed = 30f;      // 公转速度（度/秒）

    [Header("自转")]
    public float selfRotateSpeed = 100f; // 自转速度
    public Vector3 selfRotateAxis = Vector3.up; // 自转轴

    [Header("轨道参数")]
    public float orbitRadius = 5f;       // 轨道半径
    public float orbitTilt = 0f;         // 轨道倾斜角度

    private float currentAngle = 0f;

    void Start()
    {
        // 如果有父物体，用父物体作为中心
        if (centerPoint == null && transform.parent != null)
        {
            centerPoint = transform.parent;
        }

        // 设置初始轨道位置
        currentAngle = Random.Range(0f, 360f);
    }

    void Update()
    {
        // 自转
        transform.Rotate(selfRotateAxis, selfRotateSpeed * Time.deltaTime);

        // 公转
        if (centerPoint != null)
        {
            currentAngle += orbitSpeed * Time.deltaTime;

            // 计算轨道位置
            float rad = currentAngle * Mathf.Deg2Rad;
            float x = centerPoint.position.x + Mathf.Cos(rad) * orbitRadius;
            float z = centerPoint.position.z + Mathf.Sin(rad) * orbitRadius;
            float y = centerPoint.position.y + Mathf.Sin(orbitTilt * Mathf.Deg2Rad) * orbitRadius;

            transform.position = new Vector3(x, y, z);
        }
    }
}
```

### 要点总结

| 概念 | 说明 |
|------|------|
| `position` | 世界空间坐标 |
| `localPosition` | 相对于父物体的坐标 |
| `TransformPoint` | 本地 → 世界坐标转换 |
| `Translate` | 移动物体 |
| `Rotate` | 旋转物体 |
| `LookAt` | 让物体面朝目标 |
| `SetParent` | 建立/解除父子关系 |
| `Time.deltaTime` | 帧率无关的移动（必须乘以它） |

---

> 📌 下一步：学习 [预制体 Prefab](../04-预制体Prefab/README.md)
