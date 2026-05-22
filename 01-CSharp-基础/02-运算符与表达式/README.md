# 02 - 运算符与表达式

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 算术运算符](#1-算术运算符)
- [2. 比较运算符](#2-比较运算符)
- [3. 逻辑运算符](#3-逻辑运算符)
- [4. 赋值运算符](#4-赋值运算符)
- [5. 位运算符](#5-位运算符)
- [6. 三元运算符](#6-三元运算符)
- [7. 运算符优先级](#7-运算符优先级)
- [8. Unity 实际应用](#8-unity-实际应用)

---

## 1. 算术运算符

```csharp
// 基本算术运算
int a = 10, b = 3;

int add = a + b;      // 13  加法
int sub = a - b;      // 7   减法
int mul = a * b;      // 30  乘法
int div = a / b;      // 3   除法（整数除法，去掉小数）
int mod = a % b;      // 1   取余（模运算）

// 浮点数除法
float fa = 10f, fb = 3f;
float result = fa / fb;  // 3.3333... 保留小数

// ⚠️ 整数除法陷阱
int x = 5 / 2;        // 结果是 2，不是 2.5！
float y = 5f / 2f;    // 结果是 2.5f

// 自增自减
int score = 0;
score++;    // score = 1（后置：先使用再+1）
++score;    // score = 2（前置：先+1再使用）
score--;    // score = 1
```

### Unity 中的典型用法

```csharp
// 计算伤害（带暴击倍率）
float baseDamage = 50f;
float critMultiplier = 1.5f;
float finalDamage = baseDamage * critMultiplier;  // 75

// 计算经验升级（取余判断）
int currentExp = 150;
int expPerLevel = 100;
int level = currentExp / expPerLevel;      // 1级
int overflow = currentExp % expPerLevel;   // 50点溢出经验

// 百分比计算
float currentHP = 75f;
float maxHP = 100f;
float hpPercent = currentHP / maxHP * 100f;  // 75%
```

---

## 2. 比较运算符

比较运算的结果都是 `bool` 类型（true/false）。

```csharp
int hp = 50, maxHP = 100;

bool isEqual = (hp == maxHP);     // false  等于
bool notEqual = (hp != maxHP);    // true   不等于
bool greater = (hp > 30);         // true   大于
bool less = (hp < 30);            // false  小于
bool greaterEq = (hp >= 50);      // true   大于等于
bool lessEq = (hp <= 50);         // true   小于等于
```

### 字符串比较

```csharp
string a = "hello";
string b = "Hello";

bool equal = a == b;                          // false（区分大小写）
bool ignoreCase = a.Equals(b, 
    StringComparison.OrdinalIgnoreCase);      // true（忽略大小写）
```

### Unity 中的典型用法

```csharp
// 判断玩家是否可以攻击
bool canAttack = currentHP > 0 && attackCooldown <= 0;

// 判断是否在范围内
float distance = Vector3.Distance(playerPos, enemyPos);
bool inRange = distance <= attackRange;

// 判断等级是否达标
bool canEnterDungeon = playerLevel >= 10;
```

---

## 3. 逻辑运算符

```csharp
bool a = true, b = false;

// 逻辑与：两个都为 true 才为 true
bool and = a && b;     // false

// 逻辑或：有一个为 true 就为 true
bool or = a || b;      // true

// 逻辑非：取反
bool not = !a;         // false
```

### 短路求值（重要！）

```csharp
// && 短路：左边为 false，右边不执行
if (player != null && player.hp > 0)
{
    // 如果 player 是 null，不会执行 player.hp，避免空引用错误
}

// || 短路：左边为 true，右边不执行
if (hasShield || hasArmor)
{
    // 如果有盾牌，不会检查盔甲
}
```

### 常见逻辑组合

```csharp
// 判断玩家是否存活且可操作
bool canControl = isAlive && !isStunned && !isPaused;

// 判断是否可以使用技能
bool canUseSkill = currentMP >= skillCost && cooldownTimer <= 0;

// 判断游戏结束条件
bool isGameOver = allPlayersDead || timeUp || bossDefeated;
```

---

## 4. 赋值运算符

```csharp
int x = 10;

// 复合赋值运算符
x += 5;    // x = x + 5  → 15
x -= 3;    // x = x - 3  → 12
x *= 2;    // x = x * 2  → 24
x /= 4;    // x = x / 4  → 6
x %= 4;    // x = x % 4  → 2

// 链式赋值
int a, b, c;
a = b = c = 0;  // 三个都赋值为 0
```

### Unity 中的典型用法

```csharp
// 移动速度加成
moveSpeed *= speedBuff;     // 速度加倍
currentHP -= damage;        // 扣血
currentHP += healAmount;    // 回血
cooldownTimer -= Time.deltaTime;  // 冷却倒计时

// 夹值（限制范围）
currentHP = Mathf.Clamp(currentHP, 0, maxHP);
```

---

## 5. 位运算符

位运算直接操作二进制位，在游戏开发中常用于**标志位（Flags）**。

```csharp
// 基本位运算
int a = 5;   // 二进制: 0101
int b = 3;   // 二进制: 0011

int and = a & b;   // 0001 = 1   按位与
int or = a | b;    // 0111 = 7   按位或
int xor = a ^ b;   // 0110 = 6   按位异或
int not = ~a;      // ...11111010  按位取反
int left = a << 1; // 1010 = 10  左移（×2）
int right = a >> 1;// 0010 = 2   右移（÷2）
```

### Flags 枚举（Unity 常用）

```csharp
// 定义权限标志
[System.Flags]
public enum PlayerState
{
    None    = 0,       // 0000
    Moving  = 1 << 0,  // 0001 = 1
    Jumping = 1 << 1,  // 0010 = 2
    Attacking = 1 << 2,// 0100 = 4
    Dead    = 1 << 3   // 1000 = 8
}

// 使用位运算组合状态
PlayerState state = PlayerState.Moving | PlayerState.Attacking; // 0101 = 5

// 检查是否包含某个状态
bool isMoving = (state & PlayerState.Moving) != 0;       // true
bool isDead = (state & PlayerState.Dead) != 0;           // false

// 添加状态
state |= PlayerState.Jumping;   // 添加跳跃状态

// 移除状态
state &= ~PlayerState.Moving;   // 移除移动状态

// 切换状态
state ^= PlayerState.Attacking; // 切换攻击状态
```

### LayerMask（Unity 层级遮罩）

```csharp
// LayerMask 也是位运算的典型应用
LayerMask groundLayer = 1 << 6;           // 第6层
LayerMask enemyLayer = 1 << 8;            // 第8层
LayerMask both = groundLayer | enemyLayer; // 合并

// 检查是否在某层
bool hitGround = (hitLayer & groundLayer) != 0;
```

---

## 6. 三元运算符

```csharp
// 语法：条件 ? 值1 : 值2
// 条件为 true → 值1，否则 → 值2

int hp = 30;
string status = hp > 50 ? "健康" : "危险";  // "危险"

// 等价于：
string status2;
if (hp > 50)
    status2 = "健康";
else
    status2 = "危险";
```

### 嵌套三元（不推荐过度嵌套）

```csharp
// 嵌套三元运算符（可读性差，慎用）
int score = 85;
string grade = score >= 90 ? "A" 
             : score >= 80 ? "B" 
             : score >= 60 ? "C" 
             : "D";  // "B"

// ✅ 推荐用 if-else 或 switch 替代嵌套三元
```

### Unity 中的典型用法

```csharp
// 根据血量显示颜色
Color hpColor = currentHP > 50 ? Color.green : Color.red;

// 选择音效
string sound = isCritical ? "crit_hit" : "normal_hit";

// 选择动画
string anim = isRunning ? "Run" : "Idle";

// 设置文本
text.text = isAlive ? $"HP: {hp}" : "已死亡";
```

---

## 7. 运算符优先级

从高到低排列（同一行优先级相同）：

```
优先级    运算符              说明
─────────────────────────────────────
最高    ()                  括号
        ++ -- (后置)        自增自减
        ! ~                 逻辑非、按位取反
        ++ -- (前置)        自增自减
        * / %               乘除取余
        + -                 加减
        << >>               位移
        < <= > >=           比较
        == !=               等于不等于
        &                   按位与
        ^                   按位异或
        |                   按位或
        &&                  逻辑与
        ||                  逻辑或
        ?:                  三元运算符
最低    = += -= *= /= %=   赋值
```

### 建议：用括号明确意图

```csharp
// ❌ 不清楚优先级
bool result = a + b * c > d && e || f;

// ✅ 用括号明确（推荐）
bool result = ((a + (b * c)) > d) && (e || f);
```

---

## 8. Unity 实际应用

```csharp
using UnityEngine;

public class CombatSystem : MonoBehaviour
{
    [SerializeField] private float attackRange = 3f;
    [SerializeField] private float attackCooldown = 1f;
    [SerializeField] private int baseDamage = 20;
    
    private float cooldownTimer;
    private bool canAttack;

    void Update()
    {
        // 赋值 + 比较
        cooldownTimer -= Time.deltaTime;
        canAttack = cooldownTimer <= 0f;

        // 三元运算符：选择动画
        string state = canAttack ? "Idle" : "Attacking";
    }

    public int CalculateDamage(bool isCritical, int defense)
    {
        // 三元运算符 + 算术运算
        float multiplier = isCritical ? 2.0f : 1.0f;
        int rawDamage = (int)(baseDamage * multiplier);
        
        // 减法 + 钳制
        int finalDamage = Mathf.Max(1, rawDamage - defense);
        
        return finalDamage;
    }

    public bool CanTargetEnemy(Transform enemy)
    {
        if (enemy == null) return false;  // 短路保护

        // 多条件判断
        float distance = Vector3.Distance(transform.position, enemy.position);
        bool inRange = distance <= attackRange;
        bool isReady = canAttack && cooldownTimer <= 0;
        
        return inRange && isReady;
    }

    public string GetHealthStatus(int hp, int maxHP)
    {
        // 嵌套三元（简化示例）
        float ratio = (float)hp / maxHP;
        return ratio > 0.7f ? "满血" 
             : ratio > 0.3f ? "受伤" 
             : "濒死";
    }
}
```

---

## 📝 练习题

1. 写一个判断：血量 > 0 **且** 没有被眩晕 **且** 不在冷却中，才能攻击
2. 用三元运算符，根据分数返回 "及格" 或 "不及格"（60分及格）
3. 定义 `[Flags]` 枚举 `ItemType`，包含 Weapon、Armor、Potion、Key，用位运算检查玩家是否同时拥有 Weapon 和 Key

---

> 🔗 上一节：[01 - 变量与数据类型](../01-变量与数据类型/README.md)
> 🔗 下一节：[03 - 流程控制](../03-流程控制/README.md)
