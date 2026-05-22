# 05 - 函数与方法

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 方法的定义与调用](#1-方法的定义与调用)
- [2. 参数类型](#2-参数类型)
- [3. 返回值](#3-返回值)
- [4. 方法重载](#4-方法重载)
- [5. 可选参数与命名参数](#5-可选参数与命名参数)
- [6. 递归](#6-递归)
- [7. Unity 中的特殊方法](#7-unity-中的特殊方法)
- [8. Unity 实际应用](#8-unity-实际应用)

---

## 1. 方法的定义与调用

### 1.1 基本语法

```csharp
// 语法：访问修饰符 返回类型 方法名(参数列表) { 方法体 }

// 无参数无返回值
void SayHello()
{
    Debug.Log("你好！");
}

// 有参数无返回值
void Greet(string name)
{
    Debug.Log($"你好，{name}！");
}

// 有参数有返回值
int Add(int a, int b)
{
    return a + b;
}

// 调用
SayHello();                   // 你好！
Greet("萃n");                  // 你好，萃n！
int sum = Add(3, 5);          // sum = 8
```

### 1.2 方法的作用

```csharp
// ❌ 没有方法：代码重复，难以维护
void Update()
{
    // 玩家1攻击
    int damage1 = 20 * 2 - 5;
    hp1 -= damage1;
    Debug.Log($"造成 {damage1} 点伤害");
    PlaySound("hit");
    
    // 玩家2攻击（完全重复的代码）
    int damage2 = 20 * 2 - 5;
    hp2 -= damage2;
    Debug.Log($"造成 {damage2} 点伤害");
    PlaySound("hit");
}

// ✅ 使用方法：代码复用，逻辑清晰
void Attack(int baseDamage, ref int targetHP)
{
    int damage = baseDamage * 2 - 5;
    targetHP -= damage;
    Debug.Log($"造成 {damage} 点伤害");
    PlaySound("hit");
}

void Update()
{
    Attack(20, ref hp1);
    Attack(20, ref hp2);
}
```

---

## 2. 参数类型

### 2.1 值参数（默认）

传递的是**副本**，方法内修改不影响原变量。

```csharp
void Double(int num)
{
    num *= 2;  // 只修改副本
    Debug.Log($"方法内: {num}");
}

int x = 5;
Double(x);
Debug.Log($"方法外: {x}");  // x 仍然是 5
```

### 2.2 ref 引用参数

传递的是**引用**，方法内修改会影响原变量。**调用前必须赋值**。

```csharp
void Double(ref int num)
{
    num *= 2;  // 修改原变量
    Debug.Log($"方法内: {num}");
}

int x = 5;
Double(ref x);   // 调用时也要加 ref
Debug.Log($"方法外: {x}");  // x 变成 10

// Unity 典型用法：修改玩家属性
void ApplyDamage(ref int hp, int damage)
{
    hp -= damage;
    if (hp < 0) hp = 0;
}

int playerHP = 100;
ApplyDamage(ref playerHP, 30);  // playerHP = 70
```

### 2.3 out 输出参数

方法必须对其赋值。用于**返回多个值**。

```csharp
// 返回两个值：商和余数
void Divide(int a, int b, out int quotient, out int remainder)
{
    quotient = a / b;    // 必须赋值
    remainder = a % b;   // 必须赋值
}

int q, r;
Divide(17, 5, out q, out r);
Debug.Log($"商: {q}, 余数: {r}");  // 商: 3, 余数: 2

// C# 7.0+ 内联声明
Divide(17, 5, out int q2, out int r2);

// Unity 典型用法
bool TryGetEnemyInRange(float range, out GameObject enemy)
{
    enemy = null;
    foreach (var e in enemies)
    {
        if (Vector3.Distance(transform.position, e.transform.position) <= range)
        {
            enemy = e;
            return true;
        }
    }
    return false;
}

// 使用
if (TryGetEnemyInRange(10f, out GameObject target))
{
    Attack(target);
}
```

### 2.4 params 可变参数

接收**不定数量**的参数，必须是最后一个参数。

```csharp
// 可以传任意多个 int
int Sum(params int[] numbers)
{
    int total = 0;
    foreach (int n in numbers)
    {
        total += n;
    }
    return total;
}

// 各种调用方式
int s1 = Sum(1, 2, 3);           // 6
int s2 = Sum(1, 2, 3, 4, 5);     // 15
int s3 = Sum(new int[] { 1, 2 }); // 3  也可以传数组

// Unity 典型用法：批量禁用对象
void DisableAll(params GameObject[] objects)
{
    foreach (var obj in objects)
    {
        obj.SetActive(false);
    }
}

DisableAll(player, enemy1, enemy2, boss);  // 一次禁用多个
```

### 2.5 参数类型对比

| 类型 | 调用前赋值？ | 方法内必须赋值？ | 影响原变量？ | 用途 |
|------|:-----------:|:---------------:|:----------:|------|
| 值参数 | ✅ | ❌ | ❌ | 默认，传递数据 |
| `ref` | ✅ | ❌ | ✅ | 修改原变量 |
| `out` | ❌ | ✅ | ✅ | 返回多个值 |
| `params` | ✅ | ❌ | ❌ | 不定数量参数 |

---

## 3. 返回值

### 3.1 无返回值 void

```csharp
void Die()
{
    isDead = true;
    PlayDeathAnimation();
    // 没有 return 语句，或 return; 提前退出
}

void ShowMessage(string msg, bool isError)
{
    if (string.IsNullOrEmpty(msg)) return;  // 提前退出

    Color color = isError ? Color.red : Color.white;
    Debug.Log($"<color={color}>{msg}</color>");
}
```

### 3.2 有返回值

```csharp
// 返回单个值
float GetDistance(Vector3 a, Vector3 b)
{
    return Vector3.Distance(a, b);
}

// 返回 bool（判断类方法，通常以 Is/Can/Has 开头）
bool IsInRange(Vector3 target, float range)
{
    return GetDistance(transform.position, target) <= range;
}

bool CanAttack()
{
    return isAlive && !isStunned && cooldownTimer <= 0;
}

// 返回对象
GameObject FindClosestEnemy()
{
    GameObject closest = null;
    float minDist = Mathf.Infinity;
    
    foreach (var enemy in enemies)
    {
        float dist = GetDistance(transform.position, enemy.transform.position);
        if (dist < minDist)
        {
            minDist = dist;
            closest = enemy;
        }
    }
    return closest;
}
```

### 3.3 使用元组返回多个值（C# 7.0+）

```csharp
// 返回元组
(string name, int level, float hp) GetPlayerInfo()
{
    return (playerName, playerLevel, currentHP);
}

// 接收
var info = GetPlayerInfo();
Debug.Log($"{info.name} Lv.{info.level} HP:{info.hp}");

// 解构
(var name, var level, var hp) = GetPlayerInfo();
```

---

## 4. 方法重载

**同名方法，不同参数列表**。编译器根据参数自动选择。

```csharp
// 三个重载版本
int Add(int a, int b)
{
    return a + b;
}

float Add(float a, float b)           // 参数类型不同
{
    return a + b;
}

int Add(int a, int b, int c)          // 参数数量不同
{
    return a + b + c;
}

// 调用
int r1 = Add(1, 2);          // 调用 int Add(int, int)
float r2 = Add(1.5f, 2.5f);  // 调用 float Add(float, float)
int r3 = Add(1, 2, 3);       // 调用 int Add(int, int, int)
```

### Unity 中的重载

```csharp
// Unity API 自身就大量使用重载
// Debug.Log 的重载
Debug.Log("消息");
Debug.Log("消息", gameObject);

// Vector3 构造函数重载
Vector3 v1 = new Vector3(1, 2);        // z默认为0
Vector3 v2 = new Vector3(1, 2, 3);

// Instantiate 的重载
Instantiate(prefab);
Instantiate(prefab, position, rotation);
Instantiate(prefab, parent);
```

### 自定义重载示例

```csharp
public class DamageSystem
{
    // 基础伤害
    int CalculateDamage(int baseDamage)
    {
        return baseDamage;
    }

    // 带防御的伤害
    int CalculateDamage(int baseDamage, int defense)
    {
        return Mathf.Max(1, baseDamage - defense);
    }

    // 带暴击的伤害
    int CalculateDamage(int baseDamage, int defense, bool isCritical)
    {
        int raw = CalculateDamage(baseDamage, defense);
        return isCritical ? raw * 2 : raw;
    }

    // 带属性的伤害
    int CalculateDamage(int baseDamage, int defense, ElementType element, float elementMultiplier)
    {
        int raw = CalculateDamage(baseDamage, defense);
        return (int)(raw * elementMultiplier);
    }
}
```

---

## 5. 可选参数与命名参数

### 5.1 可选参数

给参数默认值，调用时可以不传。

```csharp
void SpawnEnemy(string type, int count = 1, float hp = 100f, bool isBoss = false)
{
    for (int i = 0; i < count; i++)
    {
        // 生成敌人
    }
}

// 各种调用方式
SpawnEnemy("Goblin");                          // 1只，100血，非Boss
SpawnEnemy("Dragon", 3);                       // 3只，100血，非Boss
SpawnEnemy("Dragon", 1, 500f);                 // 1只，500血，非Boss
SpawnEnemy("FinalBoss", 1, 10000f, true);      // 1只，10000血，是Boss
```

### 5.2 命名参数

按名称指定参数，可以不按顺序。

```csharp
// 跳过中间的可选参数
SpawnEnemy("Dragon", isBoss: true, hp: 500f);

// 提高可读性
Vector3 pos = new Vector3(x: 0f, y: 5f, z: 10f);
```

---

## 6. 递归

方法**调用自身**。需要有退出条件，否则会栈溢出。

### 6.1 基本示例

```csharp
// 阶乘：5! = 5 × 4 × 3 × 2 × 1 = 120
int Factorial(int n)
{
    if (n <= 1) return 1;       // 退出条件
    return n * Factorial(n - 1); // 递归调用
}

// 调用过程
// Factorial(5) = 5 * Factorial(4)
//              = 5 * 4 * Factorial(3)
//              = 5 * 4 * 3 * Factorial(2)
//              = 5 * 4 * 3 * 2 * Factorial(1)
//              = 5 * 4 * 3 * 2 * 1
//              = 120
```

### 6.2 斐波那契数列

```csharp
// 1, 1, 2, 3, 5, 8, 13, 21, ...
int Fibonacci(int n)
{
    if (n <= 1) return n;  // 退出条件
    return Fibonacci(n - 1) + Fibonacci(n - 2);
}
```

### 6.3 Unity 中的递归应用

```csharp
// 递归查找子物体（包括嵌套的子物体）
Transform FindChildRecursive(Transform parent, string childName)
{
    // 先检查直接子物体
    foreach (Transform child in parent)
    {
        if (child.name == childName)
            return child;
        
        // 递归检查子物体的子物体
        Transform found = FindChildRecursive(child, childName);
        if (found != null)
            return found;
    }
    return null;
}

// 递归计算树的深度
int GetTreeDepth(Transform node)
{
    if (node.childCount == 0) return 1;
    
    int maxDepth = 0;
    foreach (Transform child in node)
    {
        int depth = GetTreeDepth(child);
        if (depth > maxDepth) maxDepth = depth;
    }
    return maxDepth + 1;
}

// 递归处理文件夹
void ProcessFolder(string path)
{
    foreach (string file in Directory.GetFiles(path))
    {
        ProcessFile(file);
    }
    foreach (string dir in Directory.GetDirectories(path))
    {
        ProcessFolder(dir);  // 递归处理子文件夹
    }
}
```

### 6.4 ⚠️ 递归注意事项

```csharp
// ❌ 没有退出条件 → 栈溢出！
void BadRecursion(int n)
{
    Debug.Log(n);
    BadRecursion(n + 1);  // 永远不会停止
}

// ❌ 递归太深 → 栈溢出！
// 对于大量数据，改用循环
int FactorialIterative(int n)
{
    int result = 1;
    for (int i = 2; i <= n; i++)
    {
        result *= i;
    }
    return result;
}
```

---

## 7. Unity 中的特殊方法

### 7.1 生命周期方法

```csharp
public class Player : MonoBehaviour
{
    // 游戏开始时调用一次
    void Awake()
    {
        // 初始化（在Start之前）
    }

    // 第一次 Update 之前调用
    void Start()
    {
        // 依赖其他对象的初始化
    }

    // 每帧调用
    void Update()
    {
        // 处理输入、游戏逻辑
    }

    // 固定时间间隔调用（物理计算）
    void FixedUpdate()
    {
        // 物理相关操作
    }

    // Update 之后调用
    void LateUpdate()
    {
        // 跟随摄像机等
    }

    // 销毁时调用
    void OnDestroy()
    {
        // 清理资源
    }
}
```

### 7.2 协程方法

```csharp
// 返回类型是 IEnumerator
IEnumerator DelayedAttack(float delay)
{
    Debug.Log("准备攻击...");
    yield return new WaitForSeconds(delay);  // 等待 delay 秒
    Debug.Log("攻击！");
    PerformAttack();
}

// 启动协程
StartCoroutine(DelayedAttack(2f));

// 带返回值的协程（通过回调）
IEnumerator LoadData(string url, System.Action<string> callback)
{
    UnityWebRequest request = UnityWebRequest.Get(url);
    yield return request.SendWebRequest();
    
    if (request.result == UnityWebRequest.Result.Success)
    {
        callback?.Invoke(request.downloadHandler.text);
    }
}
```

---

## 8. Unity 实际应用

```csharp
using UnityEngine;

public class SkillSystem : MonoBehaviour
{
    [Header("技能配置")]
    [SerializeField] private float skillCooldown = 5f;
    [SerializeField] private int skillDamage = 50;
    [SerializeField] private float skillRange = 10f;
    [SerializeField] private float skillDuration = 3f;

    private float cooldownTimer;
    private bool isSkillActive;

    // ========== 判断方法（bool 返回）==========
    public bool CanUseSkill()
    {
        return cooldownTimer <= 0 && !isSkillActive;
    }

    public bool IsTargetInRange(Vector3 targetPos)
    {
        return Vector3.Distance(transform.position, targetPos) <= skillRange;
    }

    // ========== 行为方法（void）==========
    public void UseSkill(Vector3 targetPos)
    {
        if (!CanUseSkill()) return;
        if (!IsTargetInRange(targetPos))
        {
            Debug.Log("目标超出范围");
            return;
        }

        cooldownTimer = skillCooldown;
        StartCoroutine(ExecuteSkill(targetPos));
    }

    private IEnumerator ExecuteSkill(Vector3 targetPos)
    {
        isSkillActive = true;
        
        // 技能特效
        SpawnEffect(targetPos);
        yield return new WaitForSeconds(0.5f);
        
        // 造成伤害
        DealDamageInArea(targetPos, skillRange, skillDamage);
        yield return new WaitForSeconds(skillDuration);
        
        isSkillActive = false;
    }

    // ========== 计算方法（有返回值）==========
    private int CalculateFinalDamage(int baseDmg, params float[] multipliers)
    {
        float final = baseDmg;
        foreach (float m in multipliers)
        {
            final *= m;
        }
        return Mathf.RoundToInt(final);
    }

    private GameObject[] GetTargetsInRange(Vector3 center, float range)
    {
        Collider[] hits = Physics.OverlapSphere(center, range);
        System.Collections.Generic.List<GameObject> targets = new();
        
        foreach (var hit in hits)
        {
            if (hit.CompareTag("Enemy"))
                targets.Add(hit.gameObject);
        }
        return targets.ToArray();
    }

    // ========== 重载方法 ==========
    private void DealDamageInArea(Vector3 center, float range, int damage)
    {
        var targets = GetTargetsInRange(center, range);
        foreach (var target in targets)
        {
            ApplyDamage(target, damage);
        }
    }

    private void DealDamageInArea(Vector3 center, float range, int damage, ElementType element)
    {
        float multiplier = GetElementMultiplier(element);
        int finalDmg = CalculateFinalDamage(damage, multiplier);
        DealDamageInArea(center, range, finalDmg);
    }

    // ========== out 参数应用 ==========
    private bool TryGetBestTarget(Vector3 center, float range, out GameObject bestTarget)
    {
        bestTarget = null;
        float minHP = float.MaxValue;
        
        foreach (var target in GetTargetsInRange(center, range))
        {
            var health = target.GetComponent<Health>();
            if (health != null && health.CurrentHP < minHP)
            {
                minHP = health.CurrentHP;
                bestTarget = target;
            }
        }
        return bestTarget != null;
    }

    void Update()
    {
        if (cooldownTimer > 0)
            cooldownTimer -= Time.deltaTime;
    }

    // 辅助方法（省略实现）
    void SpawnEffect(Vector3 pos) { }
    void ApplyDamage(GameObject target, int dmg) { }
    float GetElementMultiplier(ElementType e) => 1f;
}

public enum ElementType { Fire, Ice, Thunder }
public class Health : MonoBehaviour { public float CurrentHP; }
```

---

## 📝 练习题

1. 写一个方法 `int Clamp(int value, int min, int max)` 限制数值范围
2. 用 `out` 参数实现 `bool TryParseVector3(string input, out Vector3 result)`
3. 用 `params` 实现一个 `int Max(params int[] numbers)` 找最大值
4. 用递归实现字符串反转 `string Reverse(string s)`
5. 重载一个 `Heal` 方法：分别支持传入固定值、百分比、和物品名

---

> 🔗 上一节：[04 - 数组与集合](../04-数组与集合/README.md)
> 🔗 下一节：[06 - 面向对象编程（OOP）](../06-面向对象编程（OOP）/README.md)
