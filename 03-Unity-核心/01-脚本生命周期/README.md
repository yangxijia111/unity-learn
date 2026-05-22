# 01 - 脚本生命周期 (Script Lifecycle)

Unity MonoBehaviour 的生命周期决定了脚本中各个方法的执行顺序，理解它是写出稳定代码的基础。

## 执行顺序图解

```
Reset
  ↓
Awake          ← 对象激活时调用一次（最早）
  ↓
OnEnable       ← 每次启用时调用
  ↓
Start          ← 第一次 Update 之前调用一次
  ↓
FixedUpdate    ← 固定时间间隔（物理帧）
  ↓
Update         ← 每帧调用
  ↓
LateUpdate     ← Update 之后调用
  ↓
OnDisable      ← 禁用时调用
  ↓
OnDestroy      ← 销毁时调用
  ↓
OnApplicationQuit ← 应用退出时调用
```

## 核心方法详解

### Awake()
- 对象实例化后**最先**调用，只调用一次
- 适合做初始化，尤其是引用同一物体上的其他组件
- 此时其他脚本的 Start 还没执行，**不能依赖其他脚本的初始化结果**

### Start()
- 在第一次 Update 之前调用，只调用一次
- 此时所有 Awake 都已执行完毕，适合做跨脚本的初始化

### Update()
- 每帧调用，帧率不固定
- 适合处理**非物理**的游戏逻辑（输入检测、动画状态等）

### FixedUpdate()
- 固定时间间隔调用（默认 0.02s，可在 Time 设置中修改）
- 适合处理**物理相关**逻辑（Rigidbody 操作）
- 帧率可能与 Update 不同

### LateUpdate()
- 在所有 Update 执行完之后调用
- 适合做**跟随逻辑**（如摄像机跟随）

### OnEnable() / OnDisable()
- 组件启用/禁用时调用，可多次触发
- 适合注册/注销事件

### OnDestroy()
- 对象被销毁时调用
- 适合清理资源、取消订阅事件

## 代码示例

```csharp
using UnityEngine;

public class LifecycleDemo : MonoBehaviour
{
    // 最早执行，只一次
    void Awake()
    {
        Debug.Log("Awake - 初始化自身组件引用");
    }

    // 启用时调用（包括首次）
    void OnEnable()
    {
        Debug.Log("OnEnable - 注册事件");
    }

    // 第一帧之前，只一次
    void Start()
    {
        Debug.Log("Start - 跨脚本初始化完成");
    }

    // 每帧调用
    void Update()
    {
        // 处理输入、非物理逻辑
        if (Input.GetKeyDown(KeyCode.Space))
        {
            Debug.Log("空格键按下");
        }
    }

    // 固定时间步长，用于物理
    void FixedUpdate()
    {
        // Rigidbody 移动放这里
        Rigidbody rb = GetComponent<Rigidbody>();
        rb.AddForce(Vector3.forward * 10f);
    }

    // 所有 Update 之后
    void LateUpdate()
    {
        // 摄像机跟随放这里
    }

    // 禁用时调用
    void OnDisable()
    {
        Debug.Log("OnDisable - 注销事件");
    }

    // 销毁时调用
    void OnDestroy()
    {
        Debug.Log("OnDestroy - 清理资源");
    }

    // 应用退出
    void OnApplicationQuit()
    {
        Debug.Log("OnApplicationQuit - 保存数据");
    }
}
```

## 实际应用场景

| 场景 | 推荐方法 |
|------|----------|
| 初始化自身组件引用 | Awake |
| 注册全局事件 | OnEnable / Start |
| 处理玩家输入 | Update |
| 移动刚体角色 | FixedUpdate |
| 摄像机跟随目标 | LateUpdate |
| 注销事件、释放资源 | OnDisable / OnDestroy |
| 退出时保存游戏进度 | OnApplicationQuit |

## 注意事项

- `Awake` 和 `Start` 都只执行一次，区别在于 **执行时机**
- 不要在 `Update` 里做物理操作，去 `FixedUpdate`
- 销毁对象用 `Destroy(gameObject)`，禁用用 `SetActive(false)`
- 多个脚本的执行顺序可在 **Project Settings → Script Execution Order** 中手动设置
