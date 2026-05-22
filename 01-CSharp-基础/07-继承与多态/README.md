# 07 - 继承与多态

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 继承基础](#1-继承基础)
- [2. base 关键字](#2-base-关键字)
- [3. virtual 与 override（多态）](#3-virtual-与-override多态)
- [4. sealed 密封](#4-sealed-密封)
- [5. is 与 as 运算符](#5-is-与-as-运算符)
- [6. Unity 实际应用](#6-unity-实际应用)

---

## 1. 继承基础

继承让子类**复用父类的代码**，并可以扩展新功能。

### 1.1 基本语法

```csharp
// 父类（基类）
public class Character
{
    public string name;
    public int hp;
    public int damage;
    
    public void Move()
    {
        Debug.Log($"{name} 移动");
    }
    
    public void TakeDamage(int amount)
    {
        hp -= amount;
        Debug.Log($"{name} 受到 {amount} 伤害，HP: {hp}");
    }
}

// 子类（派生类）使用 : 继承
public class Player : Character
{
    public int level;
    
    // Player 自己的方法
    public void GainExp(int exp)
    {
        Debug.Log($"{name} 获得 {exp} 经验");
    }
}

// 使用
Player player = new Player();
player.name = "萃n";
player.hp = 100;
player.Move();           // 可以调用父类方法
player.TakeDamage(20);   // 继承自 Character
player.GainExp(50);      // Player 自己的方法
```

### 1.2 继承链

```csharp
// 父类
public class Character { }

// 继承 Character
public class Player : Character { }

// 继承 Player（多层继承）
public class Warrior : Player { }

// 继承 Character
public class Enemy : Character { }
```

```
        Character
        /       \
    Player     Enemy
      |
   Warrior
```

### 1.3 ⚠️ C# 只支持单继承

```csharp
// ❌ 错误！不能继承多个类
// public class Warrior : Player, Enemy { }

// ✅ 正确：只能继承一个类，但可以实现多个接口
// public class Warrior : Player, IAttackable, IDefendable { }
```

---

## 2. base 关键字

`base` 用于访问**父类的成员**。

### 2.1 调用父类构造函数

```csharp
public class Character
{
    public string name;
    public int hp;
    
    // 父类构造函数
    public Character(string name, int hp)
    {
        this.name = name;
        this.hp = hp;
    }
}

public class Player : Character
{
    public int level;
    
    // 子类构造函数：用 base 调用父类构造函数
    public Player(string name, int hp, int level) : base(name, hp)
    {
        // base(name, hp) 先执行，设置 name 和 hp
        this.level = level;  // 然后设置 level
    }
}

Player p = new Player("萃n", 100, 5);
// 执行顺序：Character构造 → Player构造
```

### 2.2 调用父类方法

```csharp
public class Character
{
    public string name;
    
    public virtual void Attack()
    {
        Debug.Log($"{name} 进行普通攻击");
    }
    
    public void Heal(int amount)
    {
        Debug.Log($"{name} 恢复了 {amount} HP");
    }
}

public class Warrior : Character
{
    public override void Attack()
    {
        base.Attack();  // 先执行父类的攻击
        Debug.Log($"{name} 追加旋风斩！");  // 再执行自己的
    }
    
    public void BattleCry()
    {
        base.Heal(10);  // 调用父类的 Heal 方法
        Debug.Log($"{name} 发出战吼！");
    }
}
```

### 2.3 Unity 中的 base 用法

```csharp
public class Enemy : MonoBehaviour
{
    protected int hp = 100;
    
    public virtual void Die()
    {
        Debug.Log("敌人死亡");
        Destroy(gameObject);
    }
}

public class Boss : Enemy
{
    public override void Die()
    {
        base.Die();  // 执行父类的销毁逻辑
        Debug.Log("Boss 被击败！播放过场动画...");
        SpawnReward();
    }
    
    void SpawnReward() { }
}
```

---

## 3. virtual 与 override（多态）

多态 = **同一个方法，不同的表现**。让子类可以有自己的实现。

### 3.1 基本用法

```csharp
// 父类：用 virtual 标记可被重写的方法
public class Character
{
    public string name;
    
    public virtual void Attack()
    {
        Debug.Log($"{name} 挥拳攻击");
    }
    
    public virtual string GetRole()
    {
        return "角色";
    }
}

// 子类：用 override 重写
public class Warrior : Character
{
    public override void Attack()
    {
        Debug.Log($"{name} 挥剑猛砍！");
    }
    
    public override string GetRole()
    {
        return "战士";
    }
}

public class Mage : Character
{
    public override void Attack()
    {
        Debug.Log($"{name} 释放火球术！");
    }
    
    public override string GetRole()
    {
        return "法师";
    }
}
```

### 3.2 多态的威力

```csharp
// 用父类类型引用子类对象
Character[] party = new Character[]
{
    new Warrior { name = "亚瑟" },
    new Mage { name = "梅林" },
    new Character { name = "路人" }
};

// 同一个方法调用，不同行为！
foreach (Character c in party)
{
    c.Attack();
    // 输出：
    // 亚瑟 挥剑猛砍！
    // 梅林 释放火球术！
    // 路人 挥拳攻击
}
```

### 3.3 Unity 中的多态

```csharp
// 父类：所有敌人共用
public class Enemy : MonoBehaviour
{
    public float speed = 3f;
    public int damage = 10;
    
    public virtual void Attack(Player player)
    {
        player.TakeDamage(damage);
    }
    
    public virtual void Die()
    {
        // 基础死亡逻辑
        DropLoot();
        Destroy(gameObject, 0.5f);
    }
    
    protected virtual void DropLoot()
    {
        // 基础掉落
    }
}

// 子类：近战敌人
public class MeleeEnemy : Enemy
{
    public override void Attack(Player player)
    {
        // 近战攻击逻辑
        if (Vector3.Distance(transform.position, player.transform.position) < 2f)
        {
            base.Attack(player);
            PlaySlashEffect();
        }
    }
}

// 子类：远程敌人
public class RangedEnemy : Enemy
{
    public GameObject projectilePrefab;
    
    public override void Attack(Player player)
    {
        // 远程攻击逻辑：发射投射物
        Vector3 dir = (player.transform.position - transform.position).normalized;
        Instantiate(projectilePrefab, transform.position, Quaternion.LookRotation(dir));
    }
}

// 子类：Boss
public class Boss : Enemy
{
    public override void Die()
    {
        base.Die();  // 先执行基础死亡
        GameManager.Instance.OnBossDefeated();  // 额外：通知游戏管理器
        PlayEpicDeathAnimation();
    }
    
    protected override void DropLoot()
    {
        // Boss 掉落更好的物品
        SpawnRareItems();
    }
}
```

### 3.4 abstract 抽象方法

没有默认实现，**子类必须重写**。

```csharp
public abstract class Weapon
{
    public string weaponName;
    public int damage;
    
    // 抽象方法：没有方法体，子类必须实现
    public abstract void Attack();
    
    // 抽象属性
    public abstract float AttackSpeed { get; }
    
    // 普通方法：子类可以继承
    public void ShowInfo()
    {
        Debug.Log($"{weaponName} - 伤害:{damage} 速度:{AttackSpeed}");
    }
}

public class Sword : Weapon
{
    public override float AttackSpeed => 1.5f;
    
    public override void Attack()
    {
        Debug.Log("挥砍！");
    }
}

// ❌ 不能实例化抽象类
// Weapon w = new Weapon();  // 编译错误

// ✅ 可以用抽象类作为引用类型
Weapon weapon = new Sword();
weapon.Attack();  // 挥砍！
```

---

## 4. sealed 密封

`sealed` 阻止进一步继承/重写。

### 4.1 密封类

```csharp
// 密封类：不能被继承
public sealed class GameManager : MonoBehaviour
{
    // 这个类的逻辑是最终的，不需要子类扩展
}

// ❌ 错误：不能继承密封类
// public class MyGameManager : GameManager { }
```

### 4.2 密封方法

```csharp
public class Character
{
    public virtual void Attack()
    {
        Debug.Log("基础攻击");
    }
}

public class Player : Character
{
    // sealed：禁止子类再重写这个方法
    public sealed override void Attack()
    {
        Debug.Log("玩家攻击（最终版本）");
    }
}

public class Warrior : Player
{
    // ❌ 错误：Player.Attack 是 sealed 的，不能重写
    // public override void Attack() { }
}
```

---

## 5. is 与 as 运算符

用于**类型检查和转换**。

### 5.1 is 运算符

检查对象是否是某个类型，返回 `bool`。

```csharp
Character character = new Warrior();

// is 检查
bool isWarrior = character is Warrior;      // true
bool isMage = character is Mage;            // false
bool isCharacter = character is Character;  // true（继承关系）

// 模式匹配（C# 7.0+）
if (character is Warrior w)
{
    // 自动转换并赋值给 w
    Debug.Log($"{w.name} 是战士");
}
```

### 5.2 as 运算符

安全的类型转换，失败返回 `null`（不会报错）。

```csharp
Character character = new Warrior();

// as 转换
Warrior warrior = character as Warrior;  // 成功
Mage mage = character as Mage;          // 失败，mage = null

// 安全使用
if (mage != null)
{
    mage.CastSpell();  // 不会执行，因为 mage 是 null
}
```

### 5.3 Unity 中的典型用法

```csharp
// 检测碰撞对象的类型
void OnTriggerEnter(Collider other)
{
    // 检查是不是玩家
    if (other.gameObject is Player)  // 注意：Unity 中更常用 GetComponent
    {
        Debug.Log("碰到玩家了");
    }
    
    // 用 as 安全获取组件
    Player player = other.GetComponent<Player>() as Player;
    if (player != null)
    {
        player.TakeDamage(10);
    }
    
    // 模式匹配（推荐）
    if (other.TryGetComponent(out IDamageable damageable))
    {
        damageable.TakeDamage(10);
    }
}

// 处理不同类型的对象
void ProcessCharacter(Character c)
{
    switch (c)
    {
        case Warrior w:
            w.Charge();
            break;
        case Mage m:
            m.CastFireball();
            break;
        case Character generic:
            generic.Attack();
            break;
    }
}
```

---

## 6. Unity 实际应用

```csharp
using UnityEngine;
using System.Collections.Generic;

// ========== 基类 ==========
public abstract class BaseEnemy : MonoBehaviour
{
    [Header("基础属性")]
    [SerializeField] protected int maxHP = 100;
    [SerializeField] protected int damage = 10;
    [SerializeField] protected float moveSpeed = 3f;
    
    protected int currentHP;
    protected bool isDead;
    
    // 子类必须实现
    public abstract string EnemyType { get; }
    protected abstract void SpecialAttack();
    
    // 子类可以重写
    public virtual void TakeDamage(int amount)
    {
        currentHP -= amount;
        SpawnDamageNumber(amount);
        
        if (currentHP <= 0 && !isDead)
        {
            Die();
        }
    }
    
    protected virtual void Die()
    {
        isDead = true;
        GameManager.Instance.AddScore(GetScoreValue());
        DropLoot();
        Destroy(gameObject, 2f);
    }
    
    protected virtual int GetScoreValue() => 10;
    protected virtual void DropLoot() { }
    void SpawnDamageNumber(int amount) { }
    
    void Start()
    {
        currentHP = maxHP;
    }
}

// ========== 近战敌人 ==========
public class MeleeEnemy : BaseEnemy
{
    public override string EnemyType => "近战";
    
    protected override void SpecialAttack()
    {
        // 冲锋攻击
        Debug.Log("冲锋！");
    }
    
    protected override int GetScoreValue() => 15;
}

// ========== 远程敌人 ==========
public class RangedEnemy : BaseEnemy
{
    [SerializeField] private GameObject projectilePrefab;
    
    public override string EnemyType => "远程";
    
    protected override void SpecialAttack()
    {
        // 发射投射物
        Instantiate(projectilePrefab, transform.position, Quaternion.identity);
    }
}

// ========== Boss ==========
public class BossEnemy : BaseEnemy
{
    [SerializeField] private int phaseCount = 3;
    private int currentPhase = 1;
    
    public override string EnemyType => "Boss";
    
    public override void TakeDamage(int amount)
    {
        base.TakeDamage(amount);  // 调用父类的受伤逻辑
        
        // Boss 额外逻辑：检查阶段转换
        float hpPercent = (float)currentHP / maxHP;
        if (hpPercent < 0.66f && currentPhase == 1) EnterPhase(2);
        if (hpPercent < 0.33f && currentPhase == 2) EnterPhase(3);
    }
    
    protected override void Die()
    {
        base.Die();
        GameManager.Instance.OnBossDefeated();
        CameraShake.Instance.Shake(1f, 0.5f);
    }
    
    protected override void DropLoot()
    {
        // Boss 掉落稀有物品
    }
    
    protected override int GetScoreValue() => 1000;
    
    void EnterPhase(int phase)
    {
        currentPhase = phase;
        Debug.Log($"Boss 进入第 {phase} 阶段！");
    }
}

// ========== 游戏管理器中使用多态 ==========
public class WaveSpawner : MonoBehaviour
{
    private List<BaseEnemy> activeEnemies = new();
    
    public void SpawnWave()
    {
        // 多态：统一管理不同类型的敌人
        SpawnEnemy<MeleeEnemy>(5);
        SpawnEnemy<RangedEnemy>(3);
    }
    
    void SpawnEnemy<T>(int count) where T : BaseEnemy
    {
        for (int i = 0; i < count; i++)
        {
            Vector3 pos = GetRandomPosition();
            T enemy = Instantiate(enemyPrefab, pos, Quaternion.identity).GetComponent<T>();
            activeEnemies.Add(enemy);
        }
    }
    
    // 统一伤害所有敌人
    public void DamageAllEnemies(int damage)
    {
        foreach (var enemy in activeEnemies)
        {
            if (enemy != null && !enemy.IsDead)
            {
                enemy.TakeDamage(damage);  // 多态调用
            }
        }
    }
    
    // 清理死亡敌人
    public void CleanupDead()
    {
        activeEnemies.RemoveAll(e => e == null);
    }
    
    [SerializeField] private GameObject enemyPrefab;
    Vector3 GetRandomPosition() => new(Random.Range(-10f, 10f), 0, Random.Range(-10f, 10f));
}
```

---

## 📝 练习题

1. 创建 `Character` 基类和 `Warrior`、`Mage`、`Archer` 子类，每个子类重写 `Attack()` 方法
2. 用 `base` 关键字在子类构造函数中调用父类构造函数
3. 用 `is` 和 `as` 运算符实现一个 `HealCharacter(Character c)` 方法，只有 Player 类型才治疗
4. 创建一个抽象类 `Item`，子类 `Weapon`、`Armor`、`Potion` 分别实现 `Use()` 方法
5. 实现一个多态示例：用一个 `Character[]` 数组存储不同角色，循环调用 `Attack()` 看不同表现

---

> 🔗 上一节：[06 - 面向对象编程（OOP）](../06-面向对象编程（OOP）/README.md)
> 🔗 下一节：[08 - 接口与抽象类](../08-接口与抽象类/README.md)
