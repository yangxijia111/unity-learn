# 10 - 委托与事件

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 委托 Delegate](#1-委托-delegate)
- [2. Action 与 Func](#2-action-与-func)
- [3. Lambda 表达式](#3-lambda-表达式)
- [4. 事件 Event](#4-事件-event)
- [5. 多播委托](#5-多播委托)
- [6. Unity 中的事件模式](#6-unity-中的事件模式)
- [7. Unity 实际应用](#7-unity-实际应用)

---

## 1. 委托 Delegate

委托是**方法的引用类型**——可以把方法当作变量传递。

### 1.1 基本概念

```csharp
// 委托就像一个"方法签名模板"
// 下面这个委托可以引用任何「返回void，接受一个int参数」的方法
delegate void DamageHandler(int damage);
```

### 1.2 定义和使用委托

```csharp
// 1. 定义委托类型
delegate int MathOperation(int a, int b);

// 2. 定义匹配的方法
int Add(int a, int b) => a + b;
int Subtract(int a, int b) => a - b;
int Multiply(int a, int b) => a * b;

// 3. 创建委托实例（引用方法）
MathOperation op = Add;

// 4. 调用委托
int result = op(3, 5);  // 调用 Add(3, 5)，返回 8

// 5. 切换方法
op = Subtract;
result = op(10, 3);  // 调用 Subtract(10, 3)，返回 7

// 6. 也可以这样写
MathOperation op2 = new MathOperation(Add);
```

### 1.3 委托作为参数

```csharp
delegate void OnComplete(string message);

// 委托作为方法参数
void DoWork(OnComplete callback)
{
    Debug.Log("工作中...");
    // 做一些操作
    callback?.Invoke("工作完成！");
}

// 传递方法
void ShowMessage(string msg) => Debug.Log(msg);
DoWork(ShowMessage);

// 匿名方法
DoWork(delegate (string msg) { Debug.Log(msg); });
```

### 1.4 委托的空检查

```csharp
delegate void Notify();

Notify onNotify = null;

// ❌ 如果 onNotify 为空会报错
// onNotify();  // NullReferenceException

// ✅ 安全调用方式 1：空条件运算符
onNotify?.Invoke();

// ✅ 安全调用方式 2：if 检查
if (onNotify != null)
{
    onNotify();
}
```

---

## 2. Action 与 Func

系统内置委托，不用自己定义。

### 2.1 Action（无返回值）

```csharp
// Action：返回 void 的委托
// Action              → 无参数
// Action<T>           → 1个参数
// Action<T1, T2>      → 2个参数
// 最多支持 16 个参数

// 无参数
Action sayHello = () => Debug.Log("你好！");
sayHello();

// 1个参数
Action<string> greet = (name) => Debug.Log($"你好，{name}！");
greet("萃n");

// 2个参数
Action<string, int> showScore = (name, score) =>
    Debug.Log($"{name} 的分数是 {score}");
showScore("萃n", 100);
```

### 2.2 Func（有返回值）

```csharp
// Func：最后一个类型参数是返回值类型
// Func<TResult>             → 无参数，返回 TResult
// Func<T, TResult>          → 1个参数
// Func<T1, T2, TResult>     → 2个参数

// 无参数
Func<int> getRandom = () => UnityEngine.Random.Range(1, 7);
int roll = getRandom();  // 掷骰子

// 1个参数
Func<int, int> doubleIt = (x) => x * 2;
int result = doubleIt(5);  // 10

// 2个参数
Func<int, int, int> add = (a, b) => a + b;
int sum = add(3, 5);  // 8

// 3个参数
Func<float, float, float, Vector3> createVector =
    (x, y, z) => new Vector3(x, y, z);
Vector3 pos = createVector(1f, 2f, 3f);
```

### 2.3 Unity 中的 Action/Func

```csharp
// Unity 常用模式
public class Enemy : MonoBehaviour
{
    // 用 Action 定义回调
    public Action<Enemy> OnDeath;
    public Action<int> OnDamageTaken;
    
    public void TakeDamage(int amount)
    {
        currentHP -= amount;
        OnDamageTaken?.Invoke(amount);
        
        if (currentHP <= 0)
        {
            Die();
        }
    }
    
    void Die()
    {
        OnDeath?.Invoke(this);  // 通知所有监听者
        Destroy(gameObject);
    }
}

// 监听
void Start()
{
    enemy.OnDeath += HandleEnemyDeath;
    enemy.OnDamageTaken += HandleDamage;
}

void HandleEnemyDeath(Enemy deadEnemy)
{
    Debug.Log($"{deadEnemy.name} 被消灭了！");
    score += 100;
}

void HandleDamage(int amount)
{
    // 显示伤害数字
}
```

---

## 3. Lambda 表达式

用简洁的语法定义匿名方法。

### 3.1 基本语法

```csharp
// 完整写法
Func<int, int, int> add = (int a, int b) => { return a + b; };

// 省略参数类型（编译器推断）
Func<int, int, int> add2 = (a, b) => { return a + b; };

// 单行表达式（省略大括号和 return）
Func<int, int, int> add3 = (a, b) => a + b;

// 无参数
Action sayHi = () => Debug.Log("Hi！");

// 单参数（可以省略括号）
Action<string> log = msg => Debug.Log(msg);
```

### 3.2 Lambda 在集合操作中的应用

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// FindAll：查找所有满足条件的
List<int> evens = numbers.FindAll(n => n % 2 == 0);  // {2, 4, 6, 8, 10}

// Find：找第一个
int first = numbers.Find(n => n > 5);  // 6

// TrueForAll：是否全部满足
bool allPositive = numbers.TrueForAll(n => n > 0);  // true

// ForEach：遍历执行
numbers.ForEach(n => Debug.Log(n));

// 转换
List<string> strings = numbers.ConvertAll(n => $"数字{n}");

// 排序
numbers.Sort((a, b) => b.CompareTo(a));  // 降序

// LINQ 中大量使用 Lambda
using System.Linq;
var result = numbers.Where(n => n > 5)
                    .Select(n => n * 2)
                    .ToList();  // {12, 14, 16, 18, 20}
```

### 3.3 Unity 中的 Lambda

```csharp
// 协程回调
IEnumerator LoadData(string url, Action<string> onComplete)
{
    var request = UnityWebRequest.Get(url);
    yield return request.SendWebRequest();
    onComplete?.Invoke(request.downloadHandler.text);
}

// 使用 Lambda
StartCoroutine(LoadData("https://api.example.com", json =>
{
    Debug.Log($"收到数据: {json}");
    ProcessData(json);
}));

// 事件监听
button.onClick.AddListener(() => OnButtonClick("按钮1"));

// 查找
var enemy = enemies.Find(e => e.hp > 0 && e.distance < 10f);

// 排序
enemies.Sort((a, b) => a.distance.CompareTo(b.distance));
```

---

## 4. 事件 Event

事件是**受限制的委托**——只能在类内部触发，外部只能 `+=` 和 `-=`。

### 4.1 基本语法

```csharp
public class Publisher
{
    // 用 event 关键字声明
    public event Action<string> OnMessage;
    
    public void SendMessage(string msg)
    {
        // 只能在类内部触发
        OnMessage?.Invoke(msg);
    }
}

public class Subscriber
{
    public void Subscribe(Publisher pub)
    {
        // 外部只能 += 和 -=
        pub.OnMessage += HandleMessage;
        // pub.OnMessage = HandleMessage;  // ❌ 错误！不能直接赋值
    }
    
    void HandleMessage(string msg)
    {
        Debug.Log($"收到消息: {msg}");
    }
}
```

### 4.2 事件 vs 委托

```csharp
public class Player
{
    // 委托：外部可以随意调用（危险！）
    public Action<int> OnDamageDelegate;
    
    // 事件：外部只能订阅/取消（安全！）
    public event Action<int> OnDamageEvent;
    
    public void TakeDamage(int amount)
    {
        // 内部可以调用
        OnDamageDelegate?.Invoke(amount);
        OnDamageEvent?.Invoke(amount);
    }
}

// 外部使用
Player player = new Player();

// 委托：外部可以调用（不应该这样做）
player.OnDamageDelegate?.Invoke(999);  // 可以调用，但不应该

// 事件：外部不能调用
// player.OnDamageEvent?.Invoke(999);  // ❌ 编译错误！
player.OnDamageEvent += (dmg) => Debug.Log(dmg);  // ✅ 只能订阅
```

### 4.3 标准事件模式（EventArgs）

```csharp
// 定义事件参数类
public class DamageEventArgs : EventArgs
{
    public int DamageAmount { get; }
    public DamageType DamageType { get; }
    public GameObject Attacker { get; }
    
    public DamageEventArgs(int amount, DamageType type, GameObject attacker)
    {
        DamageAmount = amount;
        DamageType = type;
        Attacker = attacker;
    }
}

public enum DamageType { Physical, Magic, True }

// 发布者
public class Character : MonoBehaviour
{
    // 标准事件声明：EventHandler<T>
    public event EventHandler<DamageEventArgs> OnDamaged;
    
    public void TakeDamage(int amount, DamageType type, GameObject attacker)
    {
        OnDamaged?.Invoke(this, new DamageEventArgs(amount, type, attacker));
    }
}

// 订阅者
public class UIManager : MonoBehaviour
{
    [SerializeField] private Character player;
    
    void OnEnable()
    {
        player.OnDamaged += HandlePlayerDamaged;
    }
    
    void OnDisable()
    {
        player.OnDamaged -= HandlePlayerDamaged;  // 别忘了取消订阅！
    }
    
    void HandlePlayerDamaged(object sender, DamageEventArgs e)
    {
        Debug.Log($"玩家受到 {e.DamageAmount} 点 {e.DamageType} 伤害");
        ShowDamageNumber(e.DamageAmount);
    }
    
    void ShowDamageNumber(int amount) { }
}
```

---

## 5. 多播委托

一个委托可以引用**多个方法**，调用时依次执行。

```csharp
// 多播委托：+= 添加方法
Action<string> log = null;
log += (msg) => Debug.Log($"控制台: {msg}");
log += (msg) => Debug.LogWarning($"警告: {msg}");
log += (msg) => SaveToLog(msg);

// 调用时依次执行所有方法
log?.Invoke("测试消息");
// 输出：
// 控制台: 测试消息
// 警告: 测试消息
// （并保存到日志文件）

// -= 移除方法
log -= (msg) => Debug.LogWarning($"警告: {msg}");
```

### 注意事项

```csharp
// ⚠️ 多播委托的返回值
Func<int, int> calc = null;
calc += (x) => x * 2;
calc += (x) => x * 3;

// 只返回最后一个方法的结果！
int result = calc(5);  // 15（不是 10）

// ⚠️ 异常中断
Action act = null;
act += () => Debug.Log("1");
act += () => throw new Exception("出错了！");  // 异常！
act += () => Debug.Log("3");  // 不会执行！

try { act?.Invoke(); }
catch { /* 只能在这里处理 */ }
```

---

## 6. Unity 中的事件模式

### 6.1 UnityEvent（Inspector 可配置）

```csharp
using UnityEngine.Events;

public class Health : MonoBehaviour
{
    [SerializeField] private int maxHP = 100;
    private int currentHP;
    
    // UnityEvent：可以在 Inspector 中配置
    [SerializeField] private UnityEvent<int> OnHealthChanged;
    [SerializeField] private UnityEvent OnDeath;
    
    public void TakeDamage(int amount)
    {
        currentHP -= amount;
        OnHealthChanged?.Invoke(currentHP);
        
        if (currentHP <= 0)
        {
            OnDeath?.Invoke();
        }
    }
    
    void Start() => currentHP = maxHP;
}

// 在 Inspector 中：
// - 拖入要调用的 GameObject
// - 选择要调用的方法
// - 设置参数
```

### 6.2 C# 事件模式

```csharp
public class GameManager : MonoBehaviour
{
    // 事件总线模式
    public static event Action<int> OnScoreChanged;
    public static event Action<string> OnLevelCompleted;
    public static event Action OnGameOver;
    
    private int score;
    
    public void AddScore(int points)
    {
        score += points;
        OnScoreChanged?.Invoke(score);
    }
    
    public void CompleteLevel(string levelName)
    {
        OnLevelCompleted?.Invoke(levelName);
    }
    
    public void GameOver()
    {
        OnGameOver?.Invoke();
    }
}

// 任何地方都可以订阅
public class ScoreUI : MonoBehaviour
{
    void OnEnable()
    {
        GameManager.OnScoreChanged += UpdateScoreDisplay;
    }
    
    void OnDisable()
    {
        GameManager.OnScoreChanged -= UpdateScoreDisplay;
    }
    
    void UpdateScoreDisplay(int newScore) { }
}
```

### 6.3 自定义事件系统

```csharp
// 简单的事件总线
public static class EventManager
{
    private static Dictionary<Type, Delegate> events = new();
    
    public static void Subscribe<T>(Action<T> handler) where T : struct
    {
        var type = typeof(T);
        if (events.ContainsKey(type))
            events[type] = Delegate.Combine(events[type], handler);
        else
            events[type] = handler;
    }
    
    public static void Unsubscribe<T>(Action<T> handler) where T : struct
    {
        var type = typeof(T);
        if (events.ContainsKey(type))
            events[type] = Delegate.Remove(events[type], handler);
    }
    
    public static void Publish<T>(T eventData) where T : struct
    {
        var type = typeof(T);
        if (events.TryGetValue(type, out var handler))
            (handler as Action<T>)?.Invoke(eventData);
    }
}

// 定义事件数据
public struct EnemyKilledEvent
{
    public string EnemyName;
    public int Score;
}

public struct PlayerLevelUpEvent
{
    public int NewLevel;
}

// 发布事件
EventManager.Publish(new EnemyKilledEvent { EnemyName = "哥布林", Score = 100 });

// 订阅事件
EventManager.Subscribe<EnemyKilledEvent>(OnEnemyKilled);

void OnEnemyKilled(EnemyKilledEvent e)
{
    Debug.Log($"{e.EnemyName} 被消灭！获得 {e.Score} 分");
}
```

---

## 7. Unity 实际应用

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;

// ========== 事件参数 ==========
public class PlayerEventArgs : EventArgs
{
    public Player Player { get; }
    public int Amount { get; }
    
    public PlayerEventArgs(Player player, int amount)
    {
        Player = player;
        Amount = amount;
    }
}

// ========== Player 类 ==========
public class Player : MonoBehaviour
{
    [SerializeField] private int maxHP = 100;
    private int currentHP;
    private int gold;
    
    // 事件
    public event EventHandler<PlayerEventArgs> OnDamaged;
    public event EventHandler<PlayerEventArgs> OnHealed;
    public event Action<int> OnGoldChanged;
    public event Action OnDeath;
    
    // 只读属性
    public int HP => currentHP;
    public int Gold => gold;
    public bool IsAlive => currentHP > 0;
    
    void Start()
    {
        currentHP = maxHP;
    }
    
    public void TakeDamage(int amount)
    {
        if (!IsAlive) return;
        
        currentHP = Mathf.Max(0, currentHP - amount);
        OnDamaged?.Invoke(this, new PlayerEventArgs(this, amount));
        
        if (!IsAlive)
            OnDeath?.Invoke();
    }
    
    public void Heal(int amount)
    {
        if (!IsAlive) return;
        
        int actualHeal = Mathf.Min(amount, maxHP - currentHP);
        currentHP += actualHeal;
        
        if (actualHeal > 0)
            OnHealed?.Invoke(this, new PlayerEventArgs(this, actualHeal));
    }
    
    public bool TrySpendGold(int amount)
    {
        if (gold < amount) return false;
        gold -= amount;
        OnGoldChanged?.Invoke(gold);
        return true;
    }
    
    public void AddGold(int amount)
    {
        gold += amount;
        OnGoldChanged?.Invoke(gold);
    }
}

// ========== UI 监听 ==========
public class PlayerHUD : MonoBehaviour
{
    [SerializeField] private Player player;
    
    void OnEnable()
    {
        player.OnDamaged += HandleDamaged;
        player.OnHealed += HandleHealed;
        player.OnGoldChanged += HandleGoldChanged;
        player.OnDeath += HandleDeath;
    }
    
    void OnDisable()
    {
        player.OnDamaged -= HandleDamaged;
        player.OnHealed -= HandleHealed;
        player.OnGoldChanged -= HandleGoldChanged;
        player.OnDeath -= HandleDeath;
    }
    
    void HandleDamaged(object sender, PlayerEventArgs e)
    {
        UpdateHPBar(e.Player.HP);
        ShowDamageNumber(e.Amount, Color.red);
        ShakeScreen(0.1f);
    }
    
    void HandleHealed(object sender, PlayerEventArgs e)
    {
        UpdateHPBar(e.Player.HP);
        ShowDamageNumber(e.Amount, Color.green);
    }
    
    void HandleGoldChanged(int newGold) { }
    void HandleDeath() { }
    void UpdateHPBar(int hp) { }
    void ShowDamageNumber(int amount, Color color) { }
    void ShakeScreen(float duration) { }
}

// ========== 成就系统（订阅事件）==========
public class AchievementSystem : MonoBehaviour
{
    private int totalDamageDealt;
    private int enemiesKilled;
    
    [SerializeField] private Player player;
    
    void OnEnable()
    {
        player.OnDamaged += TrackDamage;
    }
    
    void OnDisable()
    {
        player.OnDamaged -= TrackDamage;
    }
    
    void TrackDamage(object sender, PlayerEventArgs e)
    {
        totalDamageDealt += e.Amount;
        CheckAchievements();
    }
    
    void CheckAchievements()
    {
        if (totalDamageDealt >= 1000)
            UnlockAchievement("承受1000点伤害");
    }
    
    void UnlockAchievement(string name)
    {
        Debug.Log($"🏆 解锁成就: {name}");
    }
}

// ========== 音效系统 ==========
public class SFXManager : MonoBehaviour
{
    [SerializeField] private AudioClip hitSound;
    [SerializeField] private AudioClip healSound;
    [SerializeField] private AudioClip deathSound;
    
    [SerializeField] private Player player;
    
    void OnEnable()
    {
        player.OnDamaged += (s, e) => PlaySound(hitSound);
        player.OnHealed += (s, e) => PlaySound(healSound);
        player.OnDeath += () => PlaySound(deathSound);
    }
    
    void PlaySound(AudioClip clip)
    {
        AudioSource.PlayClipAtPoint(clip, transform.position);
    }
}
```

---

## 📝 练习题

1. 定义一个 `delegate int Operation(int a, int b)`，实现加减乘除，用委托切换操作
2. 用 `Action` 和 `Func` 改写一个回调系统
3. 用 Lambda 表达式对 `List<int>` 实现过滤（>50）、排序、转换（×2）
4. 实现一个事件系统：`Enemy` 死亡时触发事件，`UIManager` 和 `LootSystem` 分别订阅
5. 用 `UnityEvent` 实现一个可配置的触发器：进入触发区域时调用指定方法

---

> 🔗 上一节：[09 - 异常处理](../09-异常处理/README.md)
> 🔗 返回目录：[01 - C# 基础](../README.md)
