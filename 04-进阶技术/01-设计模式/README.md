# 设计模式

> 在 Unity 游戏开发中常用的几种设计模式，理解它们能让你的代码更优雅、更好维护。

## 📌 你会学到什么

- 单例模式（Singleton）—— 全局唯一的管理者
- 观察者模式（Observer）—— 事件驱动的解耦利器
- 工厂模式（Factory）—— 对象创建的流水线
- 命令模式（Command）—— 把操作封装成对象
- 策略模式（Strategy）—— 算法自由切换
- Unity 中的实际应用示例

---

## 1. 单例模式 Singleton

**核心思想：** 一个类在整个游戏中只有一个实例。

**什么时候用：** 游戏管理器（GameManager）、音频管理器、存档系统等全局访问点。

```csharp
public class GameManager : MonoBehaviour
{
    // 静态实例，全局唯一
    public static GameManager Instance { get; private set; }

    private int score = 0;

    void Awake()
    {
        // 单例检查：如果已有实例则销毁重复的
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        DontDestroyOnLoad(gameObject); // 跨场景不销毁
    }

    public void AddScore(int points)
    {
        score += points;
        Debug.Log($"当前分数：{score}");
    }
}

// 使用方式：任何地方都能直接调用
// GameManager.Instance.AddScore(10);
```

**⚠️ 注意事项：**
- 不要滥用单例，用多了会让代码耦合度变高
- `DontDestroyOnLoad` 要配合场景管理做好清理

---

## 2. 观察者模式 Observer

**核心思想：** 当某个对象状态改变时，自动通知所有"关注者"。

**Unity 内置实现：** UnityEvent / C# 事件（event / delegate）

```csharp
// 发布者：玩家
public class Player : MonoBehaviour
{
    // 定义事件
    public event System.Action<int> OnHealthChanged;

    private int health = 100;

    public void TakeDamage(int damage)
    {
        health -= damage;
        // 触发事件，通知所有订阅者
        OnHealthChanged?.Invoke(health);
    }
}

// 订阅者：UI 血条
public class HealthBarUI : MonoBehaviour
{
    [SerializeField] private Player player;
    [SerializeField] private Image healthBar;

    void OnEnable()
    {
        // 订阅事件
        player.OnHealthChanged += UpdateHealthBar;
    }

    void OnDisable()
    {
        // 取消订阅（很重要！防止内存泄漏）
        player.OnHealthChanged -= UpdateHealthBar;
    }

    void UpdateHealthBar(int currentHealth)
    {
        healthBar.fillAmount = currentHealth / 100f;
    }
}
```

**常见场景：** 血条更新、成就解锁、任务系统、音效触发。

---

## 3. 工厂模式 Factory

**核心思想：** 把对象的创建逻辑封装起来，调用者不用关心具体怎么 new。

```csharp
// 敌人基类
public abstract class Enemy : MonoBehaviour
{
    public string enemyName;
    public abstract void Attack();
}

// 具体敌人
public class Goblin : Enemy
{
    public override void Attack() => Debug.Log("哥布林挥舞棍棒！");
}

public class Dragon : Enemy
{
    public override void Attack() => Debug.Log("龙喷出火焰！");
}

// 工厂类
public static class EnemyFactory
{
    public static Enemy CreateEnemy(string type, Vector3 position)
    {
        Enemy enemy = type switch
        {
            "Goblin" => new GameObject("Goblin").AddComponent<Goblin>(),
            "Dragon" => new GameObject("Dragon").AddComponent<Dragon>(),
            _ => throw new System.ArgumentException($"未知敌人类型：{type}")
        };
        enemy.transform.position = position;
        enemy.enemyName = type;
        return enemy;
    }
}

// 使用
// Enemy goblin = EnemyFactory.CreateEnemy("Goblin", Vector3.zero);
```

---

## 4. 命令模式 Command

**核心思想：** 把"操作"封装成对象，支持撤销、重做、排队执行。

**经典场景：** 回放系统、技能系统、撤销功能。

```csharp
// 命令接口
public interface ICommand
{
    void Execute();
    void Undo();
}

// 具体命令：移动
public class MoveCommand : ICommand
{
    private Transform target;
    private Vector3 direction;
    private float distance;

    public MoveCommand(Transform target, Vector3 direction, float distance)
    {
        this.target = target;
        this.direction = direction;
        this.distance = distance;
    }

    public void Execute()
    {
        target.position += direction * distance;
    }

    public void Undo()
    {
        target.position -= direction * distance;
    }
}

// 命令管理器
public class CommandManager : MonoBehaviour
{
    private Stack<ICommand> history = new Stack<ICommand>();

    public void ExecuteCommand(ICommand command)
    {
        command.Execute();
        history.Push(command);
    }

    public void Undo()
    {
        if (history.Count > 0)
        {
            ICommand cmd = history.Pop();
            cmd.Undo();
        }
    }
}
```

---

## 5. 策略模式 Strategy

**核心思想：** 定义一族算法，把它们封装起来，让它们可以互相替换。

```csharp
// 移动策略接口
public interface IMoveStrategy
{
    void Move(Transform transform, float speed);
}

// 具体策略：地面移动
public class GroundMove : IMoveStrategy
{
    public void Move(Transform transform, float speed)
    {
        Vector3 input = new Vector3(Input.GetAxis("Horizontal"), 0, Input.GetAxis("Vertical"));
        transform.position += input * speed * Time.deltaTime;
    }
}

// 具体策略：飞行移动
public class FlyMove : IMoveStrategy
{
    public void Move(Transform transform, float speed)
    {
        Vector3 input = new Vector3(Input.GetAxis("Horizontal"), Input.GetAxis("Jump"), Input.GetAxis("Vertical"));
        transform.position += input * speed * Time.deltaTime;
    }
}

// 使用策略的类
public class PlayerController : MonoBehaviour
{
    private IMoveStrategy moveStrategy;
    [SerializeField] private float speed = 5f;

    // 可以随时切换移动方式
    public void SetMoveStrategy(IMoveStrategy strategy)
    {
        moveStrategy = strategy;
    }

    void Update()
    {
        moveStrategy?.Move(transform, speed);
    }
}
```

---

## 🔗 Unity 中的设计模式速查

| 模式 | Unity 典型应用 |
|------|--------------|
| 单例 | GameManager、AudioManager、SaveManager |
| 观察者 | UnityEvent、C# Event、消息系统 |
| 工厂 | 敌人生成器、道具生成器、子弹池 |
| 命令 | 技能系统、操作回放、输入缓冲 |
| 策略 | AI 行为切换、不同武器的攻击方式 |

---

## 📚 建议学习顺序

1. **单例** → 最简单，先理解全局访问的概念
2. **观察者** → Unity 事件系统的基础，必须掌握
3. **工厂** → 对象创建的优雅解法
4. **策略** → 理解"面向接口编程"
5. **命令** → 进阶技巧，适合复杂交互系统

---

> 💡 **核心原则：** 设计模式是工具，不是目标。用对了让代码清晰，用错了反而复杂。先让代码能跑，再考虑模式。
