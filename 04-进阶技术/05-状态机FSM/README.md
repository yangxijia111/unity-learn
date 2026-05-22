# 状态机 FSM（有限状态机）

> 角色有不同状态（站着、跑步、攻击、死亡），状态之间的切换用状态机来管理，清晰又不容易出 Bug。

## 📌 你会学到什么

- 有限状态机原理
- 基础代码实现
- 状态模式（用类封装每个状态）
- AI 行为树对比
- 敌人 AI 示例
- 玩家状态管理

---

## 1. 什么是有限状态机

**核心思想：** 一个对象在任意时刻只能处于有限个状态中的一个，状态之间通过条件触发切换。

```
         [看到玩家]
  巡逻 ──────────────→ 追击
   ↑                      │
   │   [丢失视野]          │ [进入攻击范围]
   └───────────────────────┼──────────→ 攻击
                           │           │
                           │  [离开范围]│
                           └───────────┘
```

**三要素：**
1. **状态（State）**：当前是什么（站着？跑着？）
2. **事件/条件（Event）**：什么触发了切换（按下按键？看到敌人？）
3. **动作（Action）**：进入/退出状态时做什么

---

## 2. 基础实现（Enum + Switch）

**最简单的写法，适合状态不多的情况。**

```csharp
using UnityEngine;

public class PlayerFSM : MonoBehaviour
{
    // 定义所有状态
    public enum PlayerState
    {
        Idle,
        Run,
        Jump,
        Attack,
        Dead
    }

    private PlayerState currentState = PlayerState.Idle;
    private Animator animator;

    void Start()
    {
        animator = GetComponent<Animator>();
        EnterState(currentState);
    }

    void Update()
    {
        switch (currentState)
        {
            case PlayerState.Idle:
                UpdateIdle();
                break;
            case PlayerState.Run:
                UpdateRun();
                break;
            case PlayerState.Jump:
                UpdateJump();
                break;
            case PlayerState.Attack:
                UpdateAttack();
                break;
            case PlayerState.Dead:
                // 死亡状态不做任何事
                break;
        }
    }

    // ===== 状态更新逻辑 =====

    void UpdateIdle()
    {
        if (Input.GetAxisRaw("Horizontal") != 0)
            ChangeState(PlayerState.Run);
        else if (Input.GetButtonDown("Jump"))
            ChangeState(PlayerState.Jump);
        else if (Input.GetMouseButtonDown(0))
            ChangeState(PlayerState.Attack);
    }

    void UpdateRun()
    {
        float h = Input.GetAxis("Horizontal");
        transform.position += Vector3.right * h * 5f * Time.deltaTime;

        if (h == 0)
            ChangeState(PlayerState.Idle);
        else if (Input.GetButtonDown("Jump"))
            ChangeState(PlayerState.Jump);
    }

    void UpdateJump()
    {
        // 简化：跳跃动画播完后回到 Idle
        // 实际项目中用 Animator 的 Exit Time 或事件触发
    }

    void UpdateAttack()
    {
        // 攻击动画播完后回到 Idle
    }

    // ===== 状态切换 =====

    void ChangeState(PlayerState newState)
    {
        if (currentState == newState) return;

        ExitState(currentState);
        currentState = newState;
        EnterState(newState);
    }

    void EnterState(PlayerState state)
    {
        switch (state)
        {
            case PlayerState.Idle:
                animator?.Play("Idle");
                break;
            case PlayerState.Run:
                animator?.Play("Run");
                break;
            case PlayerState.Jump:
                animator?.Play("Jump");
                break;
            case PlayerState.Attack:
                animator?.Play("Attack");
                break;
        }
        Debug.Log($"进入状态：{state}");
    }

    void ExitState(PlayerState state)
    {
        Debug.Log($"退出状态：{state}");
    }

    // 外部调用：受伤
    public void TakeDamage(int damage)
    {
        if (currentState == PlayerState.Dead) return;
        // 可以在这里检查是否死亡
        // if (health <= 0) ChangeState(PlayerState.Dead);
    }
}
```

---

## 3. 状态模式（用类封装状态）

**状态多了以后，switch 会变成怪物。用状态模式让每个状态独立成类。**

```csharp
// 状态接口
public interface IState
{
    void Enter();    // 进入状态时
    void Update();   // 每帧执行
    void Exit();     // 退出状态时
}

// 状态机管理器
public class StateMachine
{
    private IState currentState;

    public void ChangeState(IState newState)
    {
        currentState?.Exit();
        currentState = newState;
        currentState?.Enter();
    }

    public void Update()
    {
        currentState?.Update();
    }
}
```

### 用状态模式重写敌人 AI

```csharp
using UnityEngine;

// 敌人控制器
public class EnemyController : MonoBehaviour
{
    public StateMachine stateMachine = new StateMachine();
    public Transform player;
    public float chaseRange = 10f;
    public float attackRange = 2f;
    public float moveSpeed = 3f;

    void Start()
    {
        // 初始状态：巡逻
        stateMachine.ChangeState(new PatrolState(this));
    }

    void Update()
    {
        stateMachine.Update();
    }

    // 辅助方法：与玩家的距离
    public float DistanceToPlayer()
    {
        return Vector3.Distance(transform.position, player.position);
    }
}

// ===== 各状态实现 =====

// 巡逻状态
public class PatrolState : IState
{
    private EnemyController enemy;
    private Vector3 patrolTarget;

    public PatrolState(EnemyController enemy)
    {
        this.enemy = enemy;
    }

    public void Enter()
    {
        Debug.Log("敌人开始巡逻");
        // 随机一个巡逻点
        patrolTarget = enemy.transform.position + new Vector3(
            Random.Range(-5f, 5f), 0, Random.Range(-5f, 5f));
    }

    public void Update()
    {
        // 移动到巡逻点
        enemy.transform.position = Vector3.MoveTowards(
            enemy.transform.position, patrolTarget, enemy.moveSpeed * Time.deltaTime);

        // 检查是否到达巡逻点
        if (Vector3.Distance(enemy.transform.position, patrolTarget) < 0.5f)
        {
            patrolTarget = enemy.transform.position + new Vector3(
                Random.Range(-5f, 5f), 0, Random.Range(-5f, 5f));
        }

        // 检测切换条件：看到玩家 → 追击
        if (enemy.DistanceToPlayer() < enemy.chaseRange)
        {
            enemy.stateMachine.ChangeState(new ChaseState(enemy));
        }
    }

    public void Exit()
    {
        Debug.Log("敌人结束巡逻");
    }
}

// 追击状态
public class ChaseState : IState
{
    private EnemyController enemy;

    public ChaseState(EnemyController enemy)
    {
        this.enemy = enemy;
    }

    public void Enter()
    {
        Debug.Log("敌人发现玩家，开始追击！");
    }

    public void Update()
    {
        // 追踪玩家
        enemy.transform.position = Vector3.MoveTowards(
            enemy.transform.position, enemy.player.position, enemy.moveSpeed * 1.5f * Time.deltaTime);

        // 面向玩家
        enemy.transform.LookAt(enemy.player);

        float distance = enemy.DistanceToPlayer();

        // 进入攻击范围 → 攻击
        if (distance < enemy.attackRange)
        {
            enemy.stateMachine.ChangeState(new AttackState(enemy));
        }
        // 丢失目标 → 回到巡逻
        else if (distance > enemy.chaseRange * 1.5f)
        {
            enemy.stateMachine.ChangeState(new PatrolState(enemy));
        }
    }

    public void Exit()
    {
        Debug.Log("敌人停止追击");
    }
}

// 攻击状态
public class AttackState : IState
{
    private EnemyController enemy;
    private float attackCooldown = 1f;
    private float lastAttackTime;

    public AttackState(EnemyController enemy)
    {
        this.enemy = enemy;
    }

    public void Enter()
    {
        Debug.Log("敌人开始攻击！");
        lastAttackTime = Time.time - attackCooldown; // 进入就立即攻击一次
    }

    public void Update()
    {
        // 面向玩家
        enemy.transform.LookAt(enemy.player);

        float distance = enemy.DistanceToPlayer();

        // 攻击冷却
        if (Time.time - lastAttackTime >= attackCooldown)
        {
            Debug.Log("敌人攻击玩家！");
            lastAttackTime = Time.time;
        }

        // 玩家逃出攻击范围 → 追击
        if (distance > enemy.attackRange * 1.5f)
        {
            enemy.stateMachine.ChangeState(new ChaseState(enemy));
        }
        // 玩家跑太远 → 巡逻
        if (distance > enemy.chaseRange)
        {
            enemy.stateMachine.ChangeState(new PatrolState(enemy));
        }
    }

    public void Exit()
    {
        Debug.Log("敌人停止攻击");
    }
}
```

---

## 4. FSM vs 行为树

| 对比 | FSM（状态机） | 行为树（Behavior Tree） |
|------|-------------|----------------------|
| 复杂度 | 简单 | 复杂 |
| 状态数量 | 适合少（<10个） | 适合多 |
| 可读性 | 直观 | 层级化，可视化编辑 |
| 扩展性 | 状态多了难维护 | 模块化，易扩展 |
| 适用场景 | 简单 AI、玩家状态 | 复杂 AI、Boss 行为 |
| Unity 支持 | 自己写 | Animator 状态机、第三方插件 |

**什么时候用 FSM：**
- 玩家角色状态（Idle/Run/Jump/Attack）
- 简单敌人 AI（3-5 个状态）
- UI 面板状态

**什么时候用行为树：**
- Boss 的复杂决策（10+ 个行为节点）
- 需要组合条件的 NPC
- RTS 游戏的单位 AI

---

## 5. Unity Animator 状态机

Unity 的 Animator Controller 本质上就是一个可视化状态机：

```
Any State → Idle → Run → Jump
                ↘ Attack ↗
```

```csharp
// 用代码控制 Animator 状态机
public class PlayerAnimation : MonoBehaviour
{
    private Animator animator;
    private static readonly int SpeedHash = Animator.StringToHash("Speed");
    private static readonly int IsJumpingHash = Animator.StringToHash("IsJumping");
    private static readonly int AttackTriggerHash = Animator.StringToHash("Attack");

    void Start()
    {
        animator = GetComponent<Animator>();
    }

    void Update()
    {
        // 设置参数，让 Animator 自己切换状态
        animator.SetFloat(SpeedHash, Mathf.Abs(Input.GetAxis("Horizontal")));
        animator.SetBool(IsJumpingHash, !IsGrounded());

        if (Input.GetMouseButtonDown(0))
            animator.SetTrigger(AttackTriggerHash);
    }

    bool IsGrounded()
    {
        return Physics.Raycast(transform.position, Vector3.down, 0.2f);
    }
}
```

---

## 🎯 使用建议

1. **状态少于 5 个** → Enum + Switch 就够了
2. **状态 5-10 个** → 用状态模式（IState 接口）
3. **状态超过 10 个** → 考虑行为树或分层状态机
4. **与动画配合** → 优先用 Animator 状态机，代码状态机管逻辑

---

> 💡 **一句话总结：** 状态机让复杂的行为变得可管理。先把状态画在纸上，确认切换逻辑没问题，再写代码。
