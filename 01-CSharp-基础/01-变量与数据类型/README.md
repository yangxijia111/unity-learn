# 01 - 变量与数据类型

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 基本数据类型](#1-基本数据类型)
- [2. 变量声明与命名规范](#2-变量声明与命名规范)
- [3. 类型转换](#3-类型转换)
- [4. 常量 const](#4-常量-const)
- [5. 隐式类型 var](#5-隐式类型-var)
- [6. Unity 中的实际应用](#6-unity-中的实际应用)

---

## 1. 基本数据类型

### 1.1 整数类型

| 类型 | 大小 | 范围 | 说明 |
|------|------|------|------|
| `byte` | 8位 | 0 ~ 255 | 小整数，常用于颜色值 |
| `short` | 16位 | -32,768 ~ 32,767 | 较小整数 |
| `int` | 32位 | -21亿 ~ 21亿 | **最常用** |
| `long` | 64位 | 非常大 | 大数计算 |

```csharp
// 整数类型示例
int hp = 100;                    // 玩家生命值
int damage = 25;                 // 伤害值
byte playerLevel = 1;            // 等级（0-255足够了）
long totalExperience = 9999999L; // 经验值（加L后缀表示long）
short ammo = 30;                 // 弹药数量
```

### 1.2 浮点类型（小数）

| 类型 | 精度 | 说明 |
|------|------|------|
| `float` | 6-7位 | Unity **最常用**，加 `f` 后缀 |
| `double` | 15-16位 | 默认小数类型，精度更高 |
| `decimal` | 28-29位 | 金融计算用，游戏开发很少用 |

```csharp
// 浮点类型示例
float speed = 5.5f;              // 移动速度（必须加f）
float gravity = -9.81f;          // 重力
double pi = 3.141592653589793;   // 高精度计算
decimal price = 19.99m;          // 商城价格（加m后缀）
```

> ⚠️ **注意：** Unity 中几乎所有数值都用 `float`，因为 Unity API 默认接受 float 类型。

### 1.3 布尔类型

```csharp
// bool 类型：只有 true 和 false 两个值
bool isAlive = true;             // 是否存活
bool isGameOver = false;         // 游戏是否结束
bool hasKey = true;              // 是否有钥匙
bool canJump = isAlive && !isGameOver; // 逻辑运算结果
```

### 1.4 字符与字符串

```csharp
// char：单个字符，用单引号
char grade = 'A';
char key = 'W';                  // 按键绑定

// string：字符串，用双引号
string playerName = "萃n";
string weaponName = "光剑";
string empty = "";               // 空字符串
string nullStr = null;           // 空引用（不同于空字符串）

// 字符串拼接
string greeting = "你好，" + playerName;  // "你好，萃n"

// 字符串插值（推荐）
string message = $"玩家 {playerName} 的等级是 {playerLevel}";
```

---

## 2. 变量声明与命名规范

### 2.1 声明语法

```csharp
// 基本语法：类型 变量名 = 初始值;
int score = 0;
float time = 0.0f;
string name = "小萃";

// 先声明，后赋值
int a;
a = 10;

// 同时声明多个同类型变量
int x = 1, y = 2, z = 3;
```

### 2.2 命名规范

```csharp
// ✅ 驼峰命名法（camelCase）—— 用于局部变量和参数
int playerHealth;
float moveSpeed;
string weaponName;
bool isJumping;

// ✅ 帕斯卡命名法（PascalCase）—— 用于类名、方法名、属性
public class PlayerController { }
public void TakeDamage() { }
public int MaxHealth { get; set; }

// ❌ 不好的命名
int a;           // 含义不清
int x1;          // 无意义
int theValueOfHP; // 太长
```

### 2.3 变量的默认值

```csharp
// 不同类型的默认值
int defaultInt;      // 默认 0
float defaultFloat;  // 默认 0.0f
bool defaultBool;    // 默认 false
string defaultStr;   // 默认 null（空引用！）
char defaultChar;    // 默认 '\0'（空字符）
```

---

## 3. 类型转换

### 3.1 隐式转换（自动）

小类型 → 大类型，编译器自动完成，**不会丢失数据**。

```csharp
// 隐式转换：小 → 大
int myInt = 42;
float myFloat = myInt;    // int → float（自动）
double myDouble = myFloat; // float → double（自动）

byte b = 10;
int i = b;                 // byte → int（自动）
```

### 3.2 显式转换（强制）

大类型 → 小类型，需要手动转换，**可能丢失数据**。

```csharp
// 显式转换：大 → 小（可能截断）
double pi = 3.14159;
int truncated = (int)pi;   // 结果是 3（小数部分丢失）

float bigNum = 300.5f;
byte smallNum = (byte)bigNum; // 结果是 44（溢出截断！）

// Unity 实际场景：Vector3 的坐标转换
float x = (int)transform.position.x; // 取整数部分
```

### 3.3 Parse 和 TryParse（字符串 → 数值）

```csharp
// Parse：字符串转数值（格式错误会报错）
string strNum = "123";
int parsed = int.Parse(strNum);     // 123
float parsedF = float.Parse("3.14"); // 3.14f

// TryParse：安全转换（推荐）
string input = "abc";
if (int.TryParse(input, out int result))
{
    Console.WriteLine($"转换成功：{result}");
}
else
{
    Console.WriteLine("转换失败，不是有效数字");
}
```

### 3.4 ToString（数值 → 字符串）

```csharp
// 数值转字符串
int score = 999;
string scoreStr = score.ToString();      // "999"
string formatted = score.ToString("D5"); // "00999"（补零）
string money = (19.99).ToString("F2");   // "19.99"
```

---

## 4. 常量 const

```csharp
// const：编译时常量，声明时必须赋值，之后不能修改
const float GRAVITY = -9.81f;
const int MAX_PLAYERS = 4;
const string GAME_VERSION = "1.0.0";

// ❌ 这样会报错：
// GRAVITY = -10f;  // 编译错误！常量不能修改

// Unity 中的典型用法
public class GameConfig
{
    public const int MAX_INVENTORY_SIZE = 20;  // 背包最大容量
    public const float RESPAWN_TIME = 3.0f;    // 重生时间
    public const string SCENE_MAIN_MENU = "MainMenu"; // 场景名
}
```

> 💡 **`const` vs `readonly`**：`const` 是编译时常量（更快），`readonly` 是运行时常量（可以在构造函数中赋值）。

---

## 5. 隐式类型 var

```csharp
// var：让编译器自动推断类型（必须有初始值）
var health = 100;           // 推断为 int
var name = "小萃";          // 推断为 string
var speed = 5.5f;           // 推断为 float
var isActive = true;        // 推断为 bool

// ❌ 这样会报错：
// var x;       // 错误！var 必须有初始值
// var y = null; // 错误！无法推断类型

// var 在复杂类型中特别好用
var enemies = new List<GameObject>();  // 比写两边 List<GameObject> 简洁
var playerData = new Dictionary<string, int>();

// Unity 中的典型用法
var rb = GetComponent<Rigidbody>();    // 不用写完整类型名
var hit = Physics.Raycast(ray, out RaycastHit info);
```

> ⚠️ **建议**：类型明显时用 `var`（如 `new` 创建对象），类型不明显时用显式类型，保持代码可读性。

---

## 6. Unity 中的实际应用

```csharp
using UnityEngine;

public class PlayerStats : MonoBehaviour
{
    // 常量：游戏配置
    private const int MAX_HP = 100;
    private const float SPEED = 5.0f;

    // 变量：玩家状态
    [SerializeField] private int currentHP = MAX_HP;  // 在 Inspector 中可见
    [SerializeField] private float moveSpeed = SPEED;
    
    private bool isAlive = true;
    private string playerName = "Player";

    void Start()
    {
        // var 推断类型
        var rb = GetComponent<Rigidbody>();
        var collider = GetComponent<Collider>();
        
        Debug.Log($"玩家 {playerName} 初始化完成，HP: {currentHP}");
    }

    public void TakeDamage(int damage)
    {
        // 类型转换：确保伤害为正数
        currentHP -= Mathf.Abs(damage);
        
        if (currentHP <= 0)
        {
            currentHP = 0;
            isAlive = false;
            Debug.Log("玩家死亡");
        }
    }

    public void Heal(string healItemName)
    {
        // 字符串插值
        int healAmount = 25;
        currentHP = Mathf.Min(currentHP + healAmount, MAX_HP);
        Debug.Log($"使用 {healItemName}，恢复 {healAmount} HP，当前 HP: {currentHP}");
    }
}
```

---

## 📝 练习题

1. 声明一个 `Player` 类的变量，包含：名字(string)、等级(int)、血量(float)、是否在线(bool)
2. 写一个方法，把字符串 "100" 转换为 int，然后乘以 2 输出
3. 定义一个常量 `PI = 3.14159f`，计算半径为 5 的圆面积

---

> 🔗 下一节：[02 - 运算符与表达式](../02-运算符与表达式/README.md)
