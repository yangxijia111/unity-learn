# 06 - 面向对象编程（OOP）

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 类与对象](#1-类与对象)
- [2. 构造函数](#2-构造函数)
- [3. 属性（get/set）](#3-属性getset)
- [4. 访问修饰符](#4-访问修饰符)
- [5. 静态成员](#5-静态成员)
- [6. 结构体 vs 类](#6-结构体-vs-类)
- [7. Unity 实际应用](#7-unity-实际应用)

---

## 1. 类与对象

### 1.1 什么是类和对象

- **类（Class）**：蓝图/模板，描述事物的特征和行为
- **对象（Object）**：根据蓝图创建的实例

```
类 = 汽车设计图纸
对象 = 根据图纸造出来的具体的车
```

### 1.2 定义类

```csharp
// 定义一个「玩家」类
public class Player
{
    // 字段（特征/数据）
    public string name;
    public int hp;
    public int level;
    
    // 方法（行为/功能）
    public void Attack()
    {
        Debug.Log($"{name} 发起攻击！");
    }
    
    public void TakeDamage(int damage)
    {
        hp -= damage;
        Debug.Log($"{name} 受到 {damage} 点伤害，剩余 HP: {hp}");
    }
    
    public void ShowInfo()
    {
        Debug.Log($"玩家: {name} | 等级: {level} | HP: {hp}");
    }
}
```

### 1.3 创建和使用对象

```csharp
// 创建对象（实例化）
Player player1 = new Player();
player1.name = "萃n";
player1.hp = 100;
player1.level = 1;

// 调用方法
player1.Attack();        // 萃n 发起攻击！
player1.TakeDamage(20);  // 萃n 受到 20 点伤害，剩余 HP: 80
player1.ShowInfo();      // 玩家: 萃n | 等级: 1 | HP: 80

// 创建多个对象（互不影响）
Player player2 = new Player();
player2.name = "敌人";
player2.hp = 50;
player2.level = 3;
```

### 1.4 Unity 中的类

```csharp
// MonoBehaviour 就是一个类，Unity 的脚本都继承自它
public class Enemy : MonoBehaviour  // Enemy 类继承 MonoBehaviour
{
    public float speed = 3f;
    
    void Update()
    {
        transform.Translate(Vector3.forward * speed * Time.deltaTime);
    }
}

// GameObject 也是类
GameObject player = new GameObject("Player");  // 创建空的游戏对象
player.AddComponent<Player>();                  // 添加组件
```

---

## 2. 构造函数

构造函数在**创建对象时自动调用**，用于初始化。

### 2.1 基本构造函数

```csharp
public class Enemy
{
    public string name;
    public int hp;
    public int damage;
    
    // 构造函数：与类同名，无返回值
    public Enemy(string name, int hp, int damage)
    {
        this.name = name;      // this 指当前对象
        this.hp = hp;
        this.damage = damage;
    }
    
    public void ShowInfo()
    {
        Debug.Log($"{name} - HP:{hp} 攻击:{damage}");
    }
}

// 创建时必须传入参数
Enemy goblin = new Enemy("哥布林", 30, 5);
Enemy dragon = new Enemy("巨龙", 500, 50);
goblin.ShowInfo();  // 哥布林 - HP:30 攻击:5
```

### 2.2 默认构造函数

```csharp
public class Item
{
    public string name;
    public int price;
    
    // 无参构造函数（如果没写任何构造函数，编译器会自动生成一个）
    public Item()
    {
        name = "未知物品";
        price = 0;
    }
    
    // 有参构造函数
    public Item(string name, int price)
    {
        this.name = name;
        this.price = price;
    }
}

Item item1 = new Item();             // name="未知", price=0
Item item2 = new Item("光剑", 999);  // name="光剑", price=999
```

### 2.3 构造函数重载和链式调用

```csharp
public class Weapon
{
    public string name;
    public int damage;
    public float speed;
    
    // 基础构造函数
    public Weapon(string name, int damage, float speed)
    {
        this.name = name;
        this.damage = damage;
        this.speed = speed;
    }
    
    // 重载：只传名称，使用默认值
    public Weapon(string name) : this(name, 10, 1.0f) { }
    
    // 重载：传名称和伤害
    public Weapon(string name, int damage) : this(name, damage, 1.0f) { }
}

Weapon sword = new Weapon("铁剑");            // 伤害10，速度1
Weapon bow = new Weapon("长弓", 15);          // 伤害15，速度1
Weapon dagger = new Weapon("匕首", 8, 2.5f);  // 伤害8，速度2.5
```

---

## 3. 属性（get/set）

属性提供**受控的访问方式**，替代直接暴露字段。

### 3.1 基本属性

```csharp
public class Player
{
    // 私有字段
    private int hp;
    
    // 属性（公开的访问接口）
    public int HP
    {
        get { return hp; }           // 读取时调用
        set { hp = value; }          // 赋值时调用（value 是关键字）
    }
}

Player p = new Player();
p.HP = 100;           // 调用 set，value = 100
int current = p.HP;   // 调用 get，返回 100
```

### 3.2 自动属性（推荐）

```csharp
public class Player
{
    // 自动属性：编译器自动生成私有字段
    public string Name { get; set; }
    public int Level { get; set; }
    public float MoveSpeed { get; set; }
    
    // 只读属性（只能在构造函数中赋值）
    public int MaxHP { get; }
    
    // 只读自动属性 + 默认值
    public int MaxLevel { get; } = 99;
    
    public Player(int maxHP)
    {
        MaxHP = maxHP;  // 只读属性可以在构造函数中赋值
    }
}
```

### 3.3 带逻辑的属性

```csharp
public class Player
{
    private int hp;
    private int maxHP = 100;
    
    public int HP
    {
        get { return hp; }
        set
        {
            // 限制范围：0 ~ maxHP
            hp = Mathf.Clamp(value, 0, maxHP);
            
            // 触发事件
            if (hp <= 0)
            {
                OnDeath?.Invoke();
            }
        }
    }
    
    // 只读属性（计算属性）
    public float HPPercent => (float)hp / maxHP * 100f;
    public bool IsAlive => hp > 0;
    public bool IsFullHP => hp >= maxHP;
    
    public event System.Action OnDeath;
}

Player p = new Player();
p.HP = 150;   // 实际设为 100（被 Clamp）
p.HP -= 30;   // 70
Debug.Log(p.HPPercent);  // 70%
Debug.Log(p.IsAlive);    // true
```

### 3.4 访问控制

```csharp
public class Inventory
{
    // 公开读，私有写（外部只能读，内部可以改）
    public int Gold { get; private set; }
    
    // 公开读，保护写（子类可以改）
    public int Level { get; protected set; }
    
    public void AddGold(int amount)
    {
        if (amount > 0)
            Gold += amount;
    }
}
```

---

## 4. 访问修饰符

控制类和成员的**可见性**。

### 4.1 五种访问级别

| 修饰符 | 同一类 | 同一程序集 | 子类 | 任何地方 |
|--------|:------:|:---------:|:----:|:-------:|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `private` | ✅ | ❌ | ❌ | ❌ |
| `protected` | ✅ | ❌ | ✅ | ❌ |
| `internal` | ✅ | ✅ | ❌ | ❌ |
| `protected internal` | ✅ | ✅ | ✅ | ❌ |

### 4.2 实际使用

```csharp
public class Character
{
    // 公开：任何地方都可以访问
    public string characterName;
    
    // 私有：只能在本类中访问（最常用）
    private int secretValue;
    
    // 保护：本类和子类可以访问
    protected int baseDamage;
    
    // 公开属性访问私有字段
    public int SecretValue
    {
        get { return secretValue; }
        private set { secretValue = value; }
    }
    
    // 私有方法：内部逻辑，不对外暴露
    private void CalculateStats()
    {
        // 内部计算
    }
    
    // 公开方法：对外接口
    public void Attack()
    {
        CalculateStats();  // 内部调用私有方法
    }
}
```

### 4.3 Unity 中的访问控制

```csharp
public class GameManager : MonoBehaviour
{
    // [SerializeField]：私有字段但在 Inspector 中可见
    [SerializeField] private int maxEnemies = 10;
    [SerializeField] private GameObject enemyPrefab;
    
    // public：Inspector 中可见 + 代码中可访问
    public float gameSpeed = 1f;
    
    // private：仅代码内部使用
    private int currentEnemies;
    private List<GameObject> activeEnemies = new();
    
    // protected：子类可以访问
    protected int score;
    
    // 公开属性
    public int CurrentEnemies => currentEnemies;
    public bool CanSpawn => currentEnemies < maxEnemies;
}
```

---

## 5. 静态成员

属于**类本身**，而不是某个对象。

### 5.1 静态字段

```csharp
public class Player
{
    // 静态字段：所有 Player 实例共享
    public static int playerCount = 0;
    
    public string name;
    
    public Player(string name)
    {
        this.name = name;
        playerCount++;  // 每创建一个玩家，计数+1
    }
}

// 通过类名访问（不是对象）
Player p1 = new Player("玩家1");
Player p2 = new Player("玩家2");
Debug.Log(Player.playerCount);  // 2
```

### 5.2 静态方法

```csharp
public class MathHelper
{
    // 静态方法：不需要创建对象就能调用
    public static float Clamp(float value, float min, float max)
    {
        if (value < min) return min;
        if (value > max) return max;
        return value;
    }
    
    public static float Distance(Vector3 a, Vector3 b)
    {
        return Vector3.Distance(a, b);
    }
}

// 通过类名调用
float result = MathHelper.Clamp(150, 0, 100);  // 100
```

### 5.3 静态类

整个类都是静态的，不能创建实例。

```csharp
// 静态类：工具类/帮助类
public static class GameUtils
{
    // 常量（隐式静态）
    public const int MAX_PLAYERS = 4;
    
    // 静态方法
    public static float GetDistance(Vector3 a, Vector3 b)
    {
        return Vector3.Distance(a, b);
    }
    
    public static bool IsInRange(Vector3 origin, Vector3 target, float range)
    {
        return GetDistance(origin, target) <= range;
    }
    
    // 静态随机数生成器（避免种子重复问题）
    private static System.Random rng = new System.Random();
    public static int RandomRange(int min, int max)
    {
        return rng.Next(min, max);
    }
}

// 不能 new GameUtils()，只能通过类名调用
float dist = GameUtils.GetDistance(pos1, pos2);
```

### 5.4 Unity 中的单例模式

```csharp
public class GameManager : MonoBehaviour
{
    // 单例：全局唯一实例
    public static GameManager Instance { get; private set; }
    
    public int score;
    public bool isGameOver;
    
    void Awake()
    {
        if (Instance == null)
        {
            Instance = this;
            DontDestroyOnLoad(gameObject);  // 场景切换不销毁
        }
        else
        {
            Destroy(gameObject);  // 销毁重复实例
            return;
        }
    }
    
    public void AddScore(int points)
    {
        score += points;
    }
}

// 任何地方都可以访问
GameManager.Instance.AddScore(100);
Debug.Log(GameManager.Instance.score);
```

---

## 6. 结构体 vs 类

| 特性 | struct（结构体） | class（类） |
|------|:--------------:|:----------:|
| 类型 | 值类型 | 引用类型 |
| 存储 | 栈上 | 堆上 |
| 赋值 | 复制整个数据 | 复制引用（指向同一对象） |
| 默认值 | 不能为 null | 可以为 null |
| 继承 | 不支持 | 支持 |
| 适用 | 小型数据结构 | 复杂对象 |

```csharp
// 结构体：适合小型数据
public struct Point3D
{
    public float x, y, z;
    
    public Point3D(float x, float y, float z)
    {
        this.x = x;
        this.y = y;
        this.z = z;
    }
    
    public float DistanceTo(Point3D other)
    {
        float dx = x - other.x;
        float dy = y - other.y;
        float dz = z - other.z;
        return Mathf.Sqrt(dx * dx + dy * dy + dz * dz);
    }
}

// 使用
Point3D p1 = new Point3D(1, 2, 3);
Point3D p2 = p1;    // 复制整个数据（不是引用）
p2.x = 10;          // p1 不受影响
```

> 💡 Unity 的 `Vector3`, `Vector2`, `Quaternion`, `Color` 等都是结构体。

---

## 7. Unity 实际应用

```csharp
using UnityEngine;
using System.Collections.Generic;

// ========== 数据类 ==========
[System.Serializable]
public class WeaponData
{
    public string weaponName;
    public int baseDamage;
    public float attackSpeed;
    public float range;
    public WeaponType type;
    
    // 自动属性
    public float DPS => baseDamage * attackSpeed;
    
    public WeaponData(string name, int damage, float speed, float range, WeaponType type)
    {
        weaponName = name;
        baseDamage = damage;
        attackSpeed = speed;
        this.range = range;
        this.type = type;
    }
}

public enum WeaponType { Sword, Bow, Staff, Dagger }

// ========== 工具类（静态）==========
public static class DamageCalculator
{
    public static int Calculate(int baseDmg, int defense, float multiplier = 1f)
    {
        int raw = Mathf.Max(1, baseDmg - defense);
        return Mathf.RoundToInt(raw * multiplier);
    }
    
    public static bool IsCritical(float critChance)
    {
        return Random.value < critChance;
    }
}

// ========== 管理器（单例）==========
public class WeaponManager : MonoBehaviour
{
    public static WeaponManager Instance { get; private set; }
    
    [SerializeField] private List<WeaponData> allWeapons = new();
    private Dictionary<string, WeaponData> weaponDict = new();
    
    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        
        // 初始化字典
        foreach (var weapon in allWeapons)
        {
            weaponDict[weapon.weaponName] = weapon;
        }
    }
    
    public WeaponData GetWeapon(string name)
    {
        return weaponDict.TryGetValue(name, out var weapon) ? weapon : null;
    }
    
    public int WeaponCount => allWeapons.Count;
}

// ========== 使用示例 ==========
public class PlayerCombat : MonoBehaviour
{
    private WeaponData currentWeapon;
    
    void Attack(GameObject target)
    {
        if (currentWeapon == null) return;
        
        var defense = target.GetComponent<Defense>();
        int def = defense != null ? defense.Value : 0;
        
        float multiplier = DamageCalculator.IsCritical(0.2f) ? 2f : 1f;
        int finalDamage = DamageCalculator.Calculate(currentWeapon.baseDamage, def, multiplier);
        
        target.GetComponent<Health>().TakeDamage(finalDamage);
    }
    
    void EquipWeapon(string weaponName)
    {
        currentWeapon = WeaponManager.Instance.GetWeapon(weaponName);
    }
}
```

---

## 📝 练习题

1. 创建一个 `Enemy` 类，包含 name、hp、damage 字段，Attack() 和 TakeDamage() 方法
2. 给 `Enemy` 类添加构造函数（3个重载版本）
3. 把 hp 改成属性，添加验证（不能超过 maxHP，不能低于 0）
4. 创建一个静态工具类 `MathUtils`，包含 Clamp、Lerp、RandomRange 方法
5. 实现一个简单的 GameManager 单例

---

> 🔗 上一节：[05 - 函数与方法](../05-函数与方法/README.md)
> 🔗 下一节：[07 - 继承与多态](../07-继承与多态/README.md)
