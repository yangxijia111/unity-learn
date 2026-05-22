# 08 - 接口与抽象类

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 接口 Interface](#1-接口-interface)
- [2. 抽象类 abstract class](#2-抽象类-abstract-class)
- [3. 接口 vs 抽象类](#3-接口-vs-抽象类)
- [4. 多接口实现](#4-多接口实现)
- [5. 接口的默认实现（C# 8.0+）](#5-接口的默认实现c-80)
- [6. Unity 实际应用](#6-unity-实际应用)

---

## 1. 接口 Interface

接口定义**行为契约**——只规定"能做什么"，不规定"怎么做"。

### 1.1 定义接口

```csharp
// 接口命名惯例：以 I 开头
public interface IDamageable
{
    // 只声明方法签名，没有实现
    void TakeDamage(int damage);
    bool IsDead { get; }
}

public interface IHealable
{
    void Heal(int amount);
}

public interface IInteractable
{
    string InteractionPrompt { get; }
    void Interact(Player player);
}
```

### 1.2 实现接口

```csharp
// 用 : 接口名 实现
public class Player : MonoBehaviour, IDamageable, IHealable
{
    private int hp = 100;
    private int maxHP = 100;
    
    // 必须实现接口的所有成员
    public bool IsDead => hp <= 0;
    
    public void TakeDamage(int damage)
    {
        hp -= damage;
        Debug.Log($"玩家受到 {damage} 伤害，HP: {hp}");
    }
    
    public void Heal(int amount)
    {
        hp = Mathf.Min(hp + amount, maxHP);
        Debug.Log($"玩家恢复 {amount} HP，HP: {hp}");
    }
}

public class Crate : MonoBehaviour, IDamageable
{
    private int hp = 30;
    public bool IsDead => hp <= 0;
    
    public void TakeDamage(int damage)
    {
        hp -= damage;
        if (IsDead)
        {
            SpawnItems();
            Destroy(gameObject);
        }
    }
    
    void SpawnItems() { }
}
```

### 1.3 使用接口

```csharp
// 接口作为类型：统一处理不同对象
public class Weapon : MonoBehaviour
{
    [SerializeField] private int damage = 20;
    
    void OnTriggerEnter(Collider other)
    {
        // 任何实现了 IDamageable 的对象都能受到伤害
        IDamageable target = other.GetComponent<IDamageable>();
        if (target != null && !target.IsDead)
        {
            target.TakeDamage(damage);
        }
    }
}
```

### 1.4 接口的好处

```csharp
// 没有接口：需要为每种类型写重复代码
void DamageEnemy(Enemy e, int dmg) { e.TakeDamage(dmg); }
void DamageBoss(Boss b, int dmg) { b.TakeDamage(dmg); }
void DamageCrate(Crate c, int dmg) { c.TakeDamage(dmg); }

// 有接口：一个方法搞定所有
void DamageTarget(IDamageable target, int damage)
{
    if (target != null && !target.IsDead)
    {
        target.TakeDamage(damage);
    }
}
```

---

## 2. 抽象类 abstract class

抽象类是**不能直接实例化**的类，可以包含抽象方法（子类必须实现）和普通方法（子类可以继承）。

### 2.1 定义抽象类

```csharp
public abstract class Enemy
{
    // 普通字段
    public string enemyName;
    protected int hp;
    protected int damage;
    
    // 普通方法（有实现，子类直接继承）
    public void TakeDamage(int amount)
    {
        hp -= amount;
        if (hp <= 0) Die();
    }
    
    protected virtual void Die()
    {
        Debug.Log($"{enemyName} 死亡");
        Destroy(gameObject);
    }
    
    // 抽象方法（没有实现，子类必须重写）
    public abstract void Attack();
    public abstract float GetAttackRange();
    
    // 虚方法（有默认实现，子类可以选择重写）
    public virtual void Move()
    {
        Debug.Log($"{enemyName} 基础移动");
    }
}
```

### 2.2 继承抽象类

```csharp
public class Goblin : Enemy
{
    // 必须实现所有抽象方法
    public override void Attack()
    {
        Debug.Log("哥布林挥舞木棒！");
    }
    
    public override float GetAttackRange()
    {
        return 2f;
    }
    
    // 可以选择重写虚方法
    public override void Move()
    {
        Debug.Log("哥布林偷偷摸摸地移动");
    }
}

public class Dragon : Enemy
{
    public override void Attack()
    {
        Debug.Log("巨龙喷火！");
    }
    
    public override float GetAttackRange()
    {
        return 15f;
    }
    
    protected override void Die()
    {
        base.Die();  // 调用父类的 Die
        Debug.Log("巨龙轰然倒地，大地震颤！");
    }
}
```

### 2.3 不能实例化抽象类

```csharp
// ❌ 错误
// Enemy e = new Enemy();  // 编译错误

// ✅ 可以用抽象类作为引用类型
Enemy enemy1 = new Goblin();
Enemy enemy2 = new Dragon();

// 多态
enemy1.Attack();  // 哥布林挥舞木棒！
enemy2.Attack();  // 巨龙喷火！
```

---

## 3. 接口 vs 抽象类

| 特性 | 接口 Interface | 抽象类 Abstract Class |
|------|:-------------:|:-------------------:|
| 实例化 | ❌ | ❌ |
| 继承数量 | 多个 | 只能一个 |
| 字段 | ❌ | ✅ |
| 构造函数 | ❌ | ✅ |
| 方法实现 | ❌（C# 8.0+ 支持） | ✅ |
| 访问修饰符 | 默认 public | 任意 |
| 关键字 | `: 接口` | `: 抽象类` |
| 设计理念 | "能做什么"（能力） | "是什么"（身份） |

### 3.1 什么时候用哪个

```csharp
// ✅ 接口：定义能力/行为（不相关的东西可能共享的能力）
// 能被伤害 → IDamageable（敌人、箱子、门、玩家都能被伤害）
// 能被拾取 → ICollectible（金币、药水、武器都能被拾取）
// 能被存储 → ISaveable（玩家数据、设置、关卡进度都能存储）

// ✅ 抽象类：定义身份（相关的类共享的基础）
// 敌人 → Enemy（所有敌人共享：血量、伤害、受伤逻辑）
// 武器 → Weapon（所有武器共享：攻击力、攻击方法）
// 角色 → Character（所有角色共享：名字、移动、攻击）
```

### 3.2 组合使用

```csharp
// 接口定义能力
public interface IDamageable { void TakeDamage(int amount); }
public interface IMoveable { void Move(Vector3 direction); }
public interface ISaveable { string Save(); void Load(string data); }

// 抽象类定义身份
public abstract class Enemy : MonoBehaviour, IDamageable, IMoveable
{
    protected int hp;
    
    // 共享实现
    public void TakeDamage(int amount) { hp -= amount; }
    public void Move(Vector3 direction) { transform.Translate(direction); }
    
    // 子类必须实现
    public abstract void Attack();
}

// 具体子类
public class Zombie : Enemy, ISaveable
{
    public override void Attack() { Debug.Log("僵尸咬人！"); }
    public string Save() => $"Zombie:{hp}";
    public void Load(string data) { /* 解析数据 */ }
}
```

---

## 4. 多接口实现

C# 不支持多继承，但支持**实现多个接口**。

### 4.1 基本多接口

```csharp
public interface IAttackable { void Attack(); }
public interface IDefendable { void Defend(); }
public interface ICastable { void CastSpell(); }

// 实现多个接口
public class Paladin : MonoBehaviour, IAttackable, IDefendable
{
    public void Attack() { Debug.Log("圣骑士挥剑！"); }
    public void Defend() { Debug.Log("圣骑士举盾！"); }
}

public class BattleMage : MonoBehaviour, IAttackable, ICastable
{
    public void Attack() { Debug.Log("战斗法师挥杖！"); }
    public void CastSpell() { Debug.Log("战斗法师施法！"); }
}
```

### 4.2 接口冲突的解决

```csharp
public interface IAnimal
{
    void MakeSound();
}

public interface IPet
{
    void MakeSound();  // 同名方法
}

// 显式接口实现：解决同名方法冲突
public class Dog : IAnimal, IPet
{
    // 隐式实现（默认）
    public void MakeSound()
    {
        Debug.Log("汪汪！");
    }
    
    // 显式实现（只能通过接口调用）
    void IPet.MakeSound()
    {
        Debug.Log("（温柔地叫）呜~");
    }
}

// 使用
Dog dog = new Dog();
dog.MakeSound();              // 汪汪！

IAnimal animal = dog;
animal.MakeSound();           // 汪汪！

IPet pet = dog;
pet.MakeSound();              // （温柔地叫）呜~
```

### 4.3 Unity 中的多接口模式

```csharp
public interface IDamageable
{
    void TakeDamage(int amount, DamageType type);
}

public interface IHealable
{
    void Heal(int amount);
}

public interface IStatusEffect
{
    void ApplyStatus(StatusEffect effect);
    void RemoveStatus(StatusEffect effect);
}

// 玩家实现了所有能力
public class Player : MonoBehaviour, IDamageable, IHealable, IStatusEffect
{
    public void TakeDamage(int amount, DamageType type) { }
    public void Heal(int amount) { }
    public void ApplyStatus(StatusEffect effect) { }
    public void RemoveStatus(StatusEffect effect) { }
}

// 箱子只实现被伤害
public class Crate : MonoBehaviour, IDamageable
{
    public void TakeDamage(int amount, DamageType type) { }
}

// 治疗水晶实现被交互和治疗
public class HealCrystal : MonoBehaviour, IHealable, IInteractable
{
    public void Heal(int amount) { }
    public string InteractionPrompt => "按 E 回复生命";
    public void Interact(Player player) { Heal(50); }
}

public enum DamageType { Physical, Fire, Ice }
public class StatusEffect { }
public interface IInteractable { string InteractionPrompt { get; } void Interact(Player player); }
```

---

## 5. 接口的默认实现（C# 8.0+）

接口可以包含**带默认实现**的方法。

```csharp
public interface IEnemy
{
    string Name { get; }
    int HP { get; set; }
    
    // 抽象方法（必须实现）
    void Attack();
    
    // 默认实现（可以不重写）
    void TakeDamage(int amount)
    {
        HP -= amount;
        Debug.Log($"{Name} 受到 {amount} 伤害");
    }
    
    void Die()
    {
        Debug.Log($"{Name} 死亡");
    }
}

public class Slime : IEnemy
{
    public string Name => "史莱姆";
    public int HP { get; set; } = 30;
    
    public void Attack()
    {
        Debug.Log("史莱姆撞击！");
    }
    
    // TakeDamage 和 Die 使用默认实现，不需要重写
}
```

---

## 6. Unity 实际应用

```csharp
using UnityEngine;
using System.Collections.Generic;

// ========== 接口定义 ==========
public interface IDamageable
{
    void TakeDamage(int amount);
    bool IsAlive { get; }
}

public interface IInteractable
{
    string Prompt { get; }
    void Interact(GameObject interactor);
}

public interface IInventoryItem
{
    string ItemName { get; }
    Sprite Icon { get; }
    void OnPickup(GameObject collector);
}

// ========== 实现示例：可破坏物 ==========
public class Destructible : MonoBehaviour, IDamageable, IInteractable
{
    [SerializeField] private int maxHP = 50;
    [SerializeField] private GameObject lootPrefab;
    
    private int currentHP;
    public bool IsAlive => currentHP > 0;
    public string Prompt => "按 E 搜索";

    void Start() => currentHP = maxHP;

    public void TakeDamage(int amount)
    {
        if (!IsAlive) return;
        currentHP -= amount;
        
        if (!IsAlive)
        {
            if (lootPrefab) Instantiate(lootPrefab, transform.position, Quaternion.identity);
            Destroy(gameObject, 0.5f);
        }
    }

    public void Interact(GameObject interactor)
    {
        if (IsAlive)
            Debug.Log("里面有东西...");
        else
            Debug.Log("已经被打烂了");
    }
}

// ========== 实现示例：可收集物 ==========
public class Coin : MonoBehaviour, IInventoryItem
{
    [SerializeField] private int value = 10;
    [SerializeField] private Sprite icon;
    
    public string ItemName => "金币";
    public Sprite Icon => icon;

    public void OnPickup(GameObject collector)
    {
        GameManager.Instance.AddGold(value);
        Destroy(gameObject);
    }
}

// ========== 实现示例：NPC ==========
public class NPC : MonoBehaviour, IInteractable
{
    [SerializeField] private string[] dialogue;
    private int dialogueIndex;
    
    public string Prompt => "按 E 对话";

    public void Interact(GameObject interactor)
    {
        if (dialogueIndex < dialogue.Length)
        {
            DialogueManager.Instance.ShowText(dialogue[dialogueIndex]);
            dialogueIndex++;
        }
    }
}

// ========== 统一交互系统 ==========
public class InteractionSystem : MonoBehaviour
{
    [SerializeField] private float interactRange = 3f;
    [SerializeField] private LayerMask interactLayer;
    
    private IInteractable currentTarget;

    void Update()
    {
        // 射线检测可交互对象
        if (Physics.Raycast(Camera.main.transform.position, Camera.main.transform.forward,
            out RaycastHit hit, interactRange, interactLayer))
        {
            // 用接口获取交互组件
            currentTarget = hit.collider.GetComponent<IInteractable>();
            
            if (currentTarget != null)
            {
                UIManager.Instance.ShowPrompt(currentTarget.Prompt);
                
                if (Input.GetKeyDown(KeyCode.E))
                {
                    currentTarget.Interact(gameObject);
                }
            }
        }
        else
        {
            currentTarget = null;
            UIManager.Instance.HidePrompt();
        }
    }
}

// ========== 统一伤害系统 ==========
public class DamageArea : MonoBehaviour
{
    [SerializeField] private int damagePerSecond = 10;
    [SerializeField] private float radius = 5f;
    
    void OnTriggerStay(Collider other)
    {
        // 任何实现了 IDamageable 的都受伤
        IDamageable target = other.GetComponent<IDamageable>();
        if (target != null && target.IsAlive)
        {
            target.TakeDamage(Mathf.RoundToInt(damagePerSecond * Time.deltaTime));
        }
    }
}

// ========== 占位类（使代码完整可编译）==========
public class GameManager : MonoBehaviour
{
    public static GameManager Instance;
    public void AddGold(int amount) { }
}
public class DialogueManager : MonoBehaviour
{
    public static DialogueManager Instance;
    public void ShowText(string text) { }
}
public class UIManager : MonoBehaviour
{
    public static UIManager Instance;
    public void ShowPrompt(string text) { }
    public void HidePrompt() { }
}
```

---

## 📝 练习题

1. 创建 `IAttackable` 接口，定义 `Attack()` 方法；创建 `IDefendable` 接口，定义 `Defend()` 方法
2. 创建 `Warrior` 类同时实现两个接口，创建 `Mage` 类只实现 `IAttackable`
3. 创建抽象类 `Item`，定义 `Use()` 抽象方法和 `ShowInfo()` 普通方法
4. 设计一个 `ISaveable` 接口，让 Player、Settings、Inventory 都能被存储/加载
5. 用接口重构一个伤害系统：让敌人、箱子、门都能受到不同类型的伤害

---

> 🔗 上一节：[07 - 继承与多态](../07-继承与多态/README.md)
> 🔗 下一节：[09 - 异常处理](../09-异常处理/README.md)
