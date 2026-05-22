# 寻路系统 NavMesh

> 让 NPC 和敌人能够自动在场景中找到路到达目标点，这就是寻路。Unity 内置的 NavMesh 系统是最常用的方案。

## 📌 你会学到什么

- NavMesh 烘焙（标记可行走区域）
- NavMeshAgent（控制角色移动）
- 寻路控制（速度、停止距离、路径更新）
- 动态障碍物
- NavMeshLink（跳跃、掉落点）
- 寻路优化

---

## 1. NavMesh 烘焙

**烘焙 = 告诉 Unity "哪些地方可以走"。**

### 操作步骤

1. 把场景中标记为导航区域的物体设置为 **Navigation Static**
   - 选中物体 → Inspector 右上角 Static 下拉 → 勾选 Navigation Static
2. 打开导航窗口：`Window → AI → Navigation`
3. 选 Bake 标签页，调整参数：
   - **Agent Radius**：角色碰撞半径
   - **Agent Height**：角色高度（决定能不能钻过缝隙）
   - **Max Slope**：最大可行走坡度（默认 45°）
   - **Step Height**：最大台阶高度
4. 点击 **Bake**

### 烘焙后你会看到

场景地面覆盖了一层蓝色区域 = 可以走的地方。

---

## 2. NavMeshAgent 基础

```csharp
using UnityEngine;
using UnityEngine.AI;

public class ClickToMove : MonoBehaviour
{
    private NavMeshAgent agent;

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();
    }

    void Update()
    {
        // 鼠标点击地面移动
        if (Input.GetMouseButtonDown(0))
        {
            Ray ray = Camera.main.ScreenPointToRay(Input.mousePosition);

            if (Physics.Raycast(ray, out RaycastHit hit))
            {
                // 设置目标点，NavMeshAgent 自动计算路径并移动
                agent.SetDestination(hit.point);
            }
        }
    }
}
```

**NavMeshAgent 常用属性：**

```csharp
agent.speed = 5f;             // 移动速度
agent.angularSpeed = 120f;    // 转向速度
agent.acceleration = 8f;      // 加速度
agent.stoppingDistance = 1.5f; // 到目标点多近停下
agent.isStopped = false;       // 是否停止
agent.velocity                 // 当前速度向量（可手动覆盖）
```

---

## 3. 追踪目标（敌人追玩家）

```csharp
using UnityEngine;
using UnityEngine.AI;

public class EnemyChase : MonoBehaviour
{
    private NavMeshAgent agent;
    [SerializeField] private Transform target; // 玩家
    [SerializeField] private float updateInterval = 0.3f; // 路径更新间隔

    private float timer;

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();
    }

    void Update()
    {
        // 不要每帧都 SetDestination！很耗性能
        timer += Time.deltaTime;
        if (timer >= updateInterval)
        {
            timer = 0;
            agent.SetDestination(target.position);
        }

        // 到达目标附近 → 停下
        if (agent.remainingDistance <= agent.stoppingDistance)
        {
            agent.isStopped = true;
            // 攻击逻辑...
        }
        else
        {
            agent.isStopped = false;
        }
    }
}
```

---

## 4. 动态障碍物

**场景中会移动或开关的物体，用 NavMeshObstacle 组件。**

### 设置方式

1. 给动态物体添加 `NavMeshObstacle` 组件
2. 勾选 **Carve**（雕刻）— 这会让 NavMesh 实时挖洞
3. 当障碍物移动/销毁时，NavMesh 会自动更新

```csharp
// 代码控制障碍物开关
public class Gate : MonoBehaviour
{
    private NavMeshObstacle obstacle;

    void Start()
    {
        obstacle = GetComponent<NavMeshObstacle>();
    }

    public void OpenGate()
    {
        obstacle.enabled = false; // 关闭障碍，寻路可以通过
    }

    public void CloseGate()
    {
        obstacle.enabled = true; // 开启障碍，寻路绕行
    }
}
```

**⚠️ 注意：** Carve 模式有性能开销。场景里动态障碍物不要太多（< 10 个建议）。

---

## 5. NavMeshLink（跳跃、掉落点）

**让角色能在 NavMesh 不相连的区域之间跳跃或掉落。**

### 设置方式

1. 创建一个空物体
2. 添加 `NavMeshLink` 组件
3. 设置起点（Start）和终点（End）
4. 设置宽度、是否双向等

```csharp
// 代码动态创建 NavMeshLink
using UnityEngine.AI;

public class DynamicLink : MonoBehaviour
{
    [SerializeField] private Transform startPoint;
    [SerializeField] private Transform endPoint;

    void Start()
    {
        NavMeshLink link = gameObject.AddComponent<NavMeshLink>();
        link.startPoint = startPoint.localPosition;
        link.endPoint = endPoint.localPosition;
        link.width = 2f;
        link.bidirectional = true;
    }

    // 开关 Link（比如门打开时才能通过）
    public void SetLinkActive(bool active)
    {
        GetComponent<NavMeshLink>().enabled = active;
    }
}
```

---

## 6. 寻路进阶控制

### 6.1 脱离 NavMesh（跳跃时）

```csharp
public class JumpController : MonoBehaviour
{
    private NavMeshAgent agent;
    private bool isJumping;

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space) && !isJumping)
        {
            StartCoroutine(Jump());
        }
    }

    System.Collections.IEnumerator Jump()
    {
        isJumping = true;

        // 1. 先记录目标点
        Vector3 destination = agent.destination;

        // 2. 禁用 NavMeshAgent，用物理控制跳跃
        agent.enabled = false;
        Rigidbody rb = GetComponent<Rigidbody>();
        rb.isKinematic = false;
        rb.AddForce(Vector3.up * 5f, ForceMode.Impulse);

        // 3. 等落地
        yield return new WaitForSeconds(1f);

        // 4. 落地后重新启用 NavMeshAgent
        rb.isKinematic = true;
        agent.enabled = true;
        agent.SetDestination(destination);

        isJumping = false;
    }
}
```

### 6.2 路径可视化（调试用）

```csharp
public class PathDebugger : MonoBehaviour
{
    private NavMeshAgent agent;

    void Start()
    {
        agent = GetComponent<NavMeshAgent>();
    }

    void OnDrawGizmos()
    {
        if (agent == null || agent.path == null) return;

        // 画出当前路径
        Gizmos.color = Color.yellow;
        Vector3[] corners = agent.path.corners;
        for (int i = 0; i < corners.Length - 1; i++)
        {
            Gizmos.DrawLine(corners[i], corners[i + 1]);
            Gizmos.DrawSphere(corners[i], 0.2f);
        }
    }
}
```

---

## 7. 寻路优化

| 优化手段 | 说明 |
|---------|------|
| 减少路径更新频率 | 不要每帧 SetDestination，0.3-0.5 秒一次 |
| 使用 Off-Mesh Link 替代大范围 NavMesh | 减少烘焙数据量 |
| 分层 NavMesh | 不同 Agent 用不同 Agent Type |
| 减少动态障碍物数量 | Carve 有性能开销 |
| AI 分帧更新 | 100 个敌人不要同帧寻路，分批处理 |

### AI 分帧更新示例

```csharp
public class EnemyManager : MonoBehaviour
{
    private List<EnemyChase> enemies = new List<EnemyChase>();
    [SerializeField] private int updatesPerFrame = 20; // 每帧更新多少个敌人

    void Update()
    {
        // 分批更新，避免同帧卡顿
        int start = (Time.frameCount * updatesPerFrame) % enemies.Count;
        int end = Mathf.Min(start + updatesPerFrame, enemies.Count);

        for (int i = start; i < end; i++)
        {
            enemies[i].UpdatePath();
        }
    }
}
```

---

## 📚 常见问题

**Q: 角色卡在墙角不动？**
A: 检查 Agent Radius 是否太小，或者 NavMesh 烘焙时 Agent Radius 设置太大，导致缝隙无法通过。

**Q: 寻路绕远路？**
A: 检查 NavMeshAreas 的 Cost 设置。不同区域可以设置不同代价，让角色优先走特定路径。

**Q: 动态物体不避开？**
A: 确认动态物体有 NavMeshObstacle 组件，且勾选了 Carve。

---

> 💡 **核心原则：** NavMesh 是 Unity 寻路的标配方案。大多数情况够用，但如果需要 2D 寻路或更复杂的地形，考虑 A* Pathfinding Project（第三方插件）。
