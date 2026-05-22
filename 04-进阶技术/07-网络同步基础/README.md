# 网络同步基础

> 多人游戏的核心挑战：让每个玩家看到的游戏世界一致。这里讲 Unity 网络多人游戏的基础知识。

## 📌 你会学到什么

- 客户端-服务器架构
- 帧同步 vs 状态同步
- Netcode for GameObjects 基础
- RPC（远程过程调用）
- 同步变量

---

## 1. 客户端-服务器架构

### 两种主流架构

```
P2P（点对点）：  玩家A ←→ 玩家B ←→ 玩家C
- 每个玩家既是客户端也是服务器
- 延迟低，但容易作弊，不适合竞技游戏

C/S（客户端-服务器）：玩家A → 服务器 → 玩家B
- 所有数据经过服务器
- 防作弊好，但需要服务器成本
- Unity 大多数多人游戏采用这种
```

### 基本流程

```
1. 服务器启动，等待连接
2. 客户端连接服务器
3. 服务器分配玩家身份，同步初始状态
4. 游戏中：客户端发送操作 → 服务器处理 → 广播结果给所有客户端
5. 客户端断开 → 服务器清理资源
```

---

## 2. 帧同步 vs 状态同步

| 对比 | 帧同步（Lockstep） | 状态同步（State Sync） |
|------|-------------------|----------------------|
| 同步什么 | 玩家输入（按键操作） | 游戏状态（位置、血量等） |
| 服务器角色 | 转发输入 | 计算并广播结果 |
| 确定性 | 要求完全确定性 | 不要求 |
| 带宽 | 很小（只传输入） | 较大（传状态） |
| 断线重做 | 需要快进所有帧 | 直接同步最新状态 |
| 典型游戏 | RTS、格斗、MOBA（LOL早期） | FPS、MMORPG |

### 帧同步核心思想

```
第1帧：所有玩家输入 → 服务器收集 → 广播第1帧输入 → 各自执行
第2帧：所有玩家输入 → 服务器收集 → 广播第2帧输入 → 各自执行
...
```

每个客户端用**相同的输入** + **相同的逻辑** = **相同的结果**

### 状态同步核心思想

```
服务器维护世界权威状态
→ 定期把状态快照发给客户端
→ 客户端插值平滑显示
```

---

## 3. Netcode for GameObjects 基础

Unity 官方的网络方案（原名 Netcode for GameObjects / NGO）。

### 安装

`Window → Package Manager → Unity Registry → 搜索 "Netcode for GameObjects" → Install`

### 基本组件

```csharp
using Unity.Netcode;

// 1. NetworkManager：挂在一个空物体上，管理整个网络会话
// 2. NetworkObject：挂在需要同步的物体上
// 3. NetworkBehaviour：代替 MonoBehaviour，提供网络功能
```

### 服务器启动与连接

```csharp
using Unity.Netcode;
using Unity.Netcode.Transports.UDPTransport;
using UnityEngine;

public class NetworkManagerUI : MonoBehaviour
{
    void OnGUI()
    {
        // 简单的网络界面
        GUILayout.BeginArea(new Rect(10, 10, 300, 200));

        if (!NetworkManager.Singleton.IsClient && !NetworkManager.Singleton.IsServer)
        {
            // 还没连接，显示按钮
            if (GUILayout.Button("启动主机 (Host)"))
                NetworkManager.Singleton.StartHost();

            if (GUILayout.Button("作为客户端连接"))
                NetworkManager.Singleton.StartClient();

            if (GUILayout.Button("启动专用服务器"))
                NetworkManager.Singleton.StartServer();
        }
        else
        {
            GUILayout.Label($"角色：{(NetworkManager.Singleton.IsHost ? "主机" : "客户端")}");
            GUILayout.Label($"玩家数：{NetworkManager.Singleton.ConnectedClientsList.Count}");

            if (GUILayout.Button("断开"))
                NetworkManager.Singleton.Shutdown();
        }

        GUILayout.EndArea();
    }
}
```

---

## 4. RPC（远程过程调用）

**RPC = 调用一个方法，但它会在另一个机器上执行。**

### 两种 RPC

- `[ServerRpc]`：客户端调用 → 服务器执行
- `[ClientRpc]`：服务器调用 → 所有客户端执行

```csharp
using Unity.Netcode;
using UnityEngine;

public class PlayerNetwork : NetworkBehaviour
{
    // 客户端调用，服务器执行
    [ServerRpc]
    public void ShootServerRpc(Vector3 direction)
    {
        // 这段代码在服务器上运行
        Debug.Log($"服务器收到射击请求，方向：{direction}");

        // 服务器执行后，通知所有客户端播放效果
        ShootEffectClientRpc(direction);
    }

    // 服务器调用，所有客户端执行
    [ClientRpc]
    private void ShootEffectClientRpc(Vector3 direction)
    {
        // 这段代码在每个客户端上运行
        Debug.Log("播放射击特效");
        // Instantiate(muzzleFlashPrefab, firePoint.position, Quaternion.identity);
    }

    void Update()
    {
        // 只有本地玩家才能操作
        if (!IsOwner) return;

        if (Input.GetMouseButtonDown(0))
        {
            // 客户端调用 ServerRpc
            ShootServerRpc(transform.forward);
        }
    }
}
```

### RPC 规则

- `ServerRpc` 方法名必须以 `ServerRpc` 结尾
- `ClientRpc` 方法名必须以 `ClientRpc` 结尾
- 只有拥有该 NetworkObject 的客户端才能调用它的 ServerRpc
- 不要频繁调用 RPC（每帧调用 = 网络洪水）

---

## 5. 同步变量

**同步变量 = 服务器改了值，所有客户端自动更新。**

```csharp
using Unity.Netcode;
using UnityEngine;

public class PlayerStats : NetworkBehaviour
{
    // 同步变量：服务器改值后自动同步到所有客户端
    private NetworkVariable<int> health = new NetworkVariable<int>(
        100,
        NetworkVariableReadPermission.Everyone,    // 谁能读
        NetworkVariableWritePermission.Server      // 只有服务器能写
    );

    // 同步变量可以带 OnValueChanged 回调
    public override void OnNetworkSpawn()
    {
        // 订阅值变化事件
        health.OnValueChanged += OnHealthChanged;

        // 初始化时也更新一次 UI
        UpdateHealthUI(health.Value);
    }

    public override void OnNetworkDespawn()
    {
        health.OnValueChanged -= OnHealthChanged;
    }

    private void OnHealthChanged(int oldValue, int newValue)
    {
        Debug.Log($"血量变化：{oldValue} → {newValue}");
        UpdateHealthUI(newValue);
    }

    private void UpdateHealthUI(int currentHealth)
    {
        // 更新血条 UI
        // healthBar.fillAmount = currentHealth / 100f;
    }

    // 只有服务器能修改同步变量
    [ServerRpc]
    public void TakeDamageServerRpc(int damage)
    {
        health.Value -= damage;
        if (health.Value <= 0)
        {
            DieClientRpc();
        }
    }

    [ClientRpc]
    private void DieClientRpc()
    {
        Debug.Log("玩家死亡");
        // 播放死亡动画等
    }
}
```

### 同步变量 vs RPC 对比

| 对比 | NetworkVariable | RPC |
|------|----------------|-----|
| 用途 | 持续同步的值（血量、分数） | 一次性的事件（射击、爆炸） |
| 自动性 | 自动同步最新值 | 手动调用才触发 |
| 频率 | 每帧检查变化 | 每次调用才发一次 |
| 历史 | 没有历史，只有最新值 | 每次调用都会执行 |

---

## 6. 网络同步注意事项

### 客户端预测与插值

```csharp
// 移动同步：不要等服务器确认再移动（会有明显延迟）
public class PlayerMovement : NetworkBehaviour
{
    private NetworkVariable<Vector3> serverPosition = new NetworkVariable<Vector3>();

    void Update()
    {
        if (IsOwner)
        {
            // 本地玩家：直接移动（客户端预测）
            Vector3 input = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
            transform.position += input * 5f * Time.deltaTime;

            // 把位置同步给服务器
            UpdatePositionServerRpc(transform.position);
        }
        else
        {
            // 远程玩家：插值到服务器位置
            transform.position = Vector3.Lerp(transform.position, serverPosition.Value, 10f * Time.deltaTime);
        }
    }

    [ServerRpc]
    private void UpdatePositionServerRpc(Vector3 position)
    {
        serverPosition.Value = position;
    }
}
```

### 常见坑

1. **不要每帧发 RPC** — 带宽爆炸，用同步变量或合并发送
2. **服务器权威** — 重要逻辑（伤害、得分）必须在服务器执行
3. **IsOwner 检查** — 每个玩家的脚本都在所有客户端跑，要判断是否是本地玩家
4. **断线处理** — 玩家掉线时记得清理 NetworkObject

---

## 📚 推荐学习路径

1. 先用 Unity 官方 Netcode for GameObjects 跑通一个多人 Demo
2. 理解服务器权威的重要性
3. 学习客户端预测和延迟补偿
4. 进阶：考虑用 Mirror 或 Photon（更成熟的网络框架）
5. 大型项目：考虑专用服务器 + 自定义协议

---

> 💡 **核心原则：** 网络游戏中，**永远不要信任客户端**。重要逻辑（判定伤害、生成物品等）必须在服务器执行，客户端只负责显示和收集输入。
