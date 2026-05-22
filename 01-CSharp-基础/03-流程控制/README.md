# 03 - 流程控制

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 条件语句 if-else](#1-条件语句-if-else)
- [2. switch 语句](#2-switch-语句)
- [3. for 循环](#3-for-循环)
- [4. while 循环](#4-while-循环)
- [5. do-while 循环](#5-do-while-循环)
- [6. foreach 循环](#6-foreach-循环)
- [7. break 与 continue](#7-break-与-continue)
- [8. return 语句](#8-return-语句)
- [9. Unity 实际应用](#9-unity-实际应用)

---

## 1. 条件语句 if-else

### 1.1 基本语法

```csharp
// 最简单的 if
int hp = 30;
if (hp <= 0)
{
    Debug.Log("玩家死亡");
}

// if-else
if (hp > 50)
{
    Debug.Log("状态良好");
}
else
{
    Debug.Log("需要治疗");
}

// if - else if - else（多条件）
if (hp > 80)
{
    Debug.Log("满血");
}
else if (hp > 50)
{
    Debug.Log("受了点轻伤");
}
else if (hp > 20)
{
    Debug.Log("伤势严重！");
}
else
{
    Debug.Log("濒死！快吃药！");
}
```

### 1.2 单行写法（省略大括号）

```csharp
// 如果只有一行，可以省略大括号（不推荐）
if (isDead) return;

// ✅ 推荐始终加大括号，避免 bug
if (isDead)
{
    return;
}
```

### 1.3 嵌套 if

```csharp
if (isAlive)
{
    if (hasWeapon)
    {
        if (currentAmmo > 0)
        {
            Fire();
        }
        else
        {
            Reload();
        }
    }
    else
    {
        PickUpWeapon();
    }
}
```

### 1.4 Unity 典型用法

```csharp
void Update()
{
    // 检测输入
    if (Input.GetKeyDown(KeyCode.Space))
    {
        if (isGrounded)
        {
            Jump();
        }
        else if (hasDoubleJump)
        {
            DoubleJump();
        }
    }
}
```

---

## 2. switch 语句

适合**一个变量有多个固定值**的情况。

### 2.1 基本语法

```csharp
int weaponType = 2;

switch (weaponType)
{
    case 0:
        Debug.Log("拳头");
        break;  // 必须有 break
    case 1:
        Debug.Log("剑");
        break;
    case 2:
        Debug.Log("弓箭");
        break;
    case 3:
        Debug.Log("法杖");
        break;
    default:
        Debug.Log("未知武器");
        break;
}
```

### 2.2 合并 case

```csharp
// 多个 case 执行相同逻辑
KeyCode key = Input.GetKeyDown(KeyCode.W) ? KeyCode.W : KeyCode.None;

switch (key)
{
    case KeyCode.W:
    case KeyCode.UpArrow:
        MoveForward();    // W 和 上箭头 都执行
        break;
    case KeyCode.S:
    case KeyCode.DownArrow:
        MoveBackward();
        break;
    case KeyCode.Space:
        Jump();
        break;
    default:
        Idle();
        break;
}
```

### 2.3 switch 表达式（C# 8.0+）

```csharp
// 现代写法：更简洁
string weaponName = weaponType switch
{
    0 => "拳头",
    1 => "剑",
    2 => "弓箭",
    3 => "法杖",
    _ => "未知"  // _ 代替 default
};

// Unity 中的用法
Color rarityColor = itemRarity switch
{
    1 => Color.white,      // 普通
    2 => Color.green,      // 优秀
    3 => Color.blue,       // 稀有
    4 => new Color(0.6f, 0, 0.8f),  // 史诗（紫色）
    5 => new Color(1f, 0.5f, 0f),   // 传说（橙色）
    _ => Color.gray
};
```

### 2.4 Unity 典型用法

```csharp
// 游戏状态机
public enum GameState { Menu, Playing, Paused, GameOver }

GameState currentState = GameState.Playing;

switch (currentState)
{
    case GameState.Menu:
        ShowMainMenu();
        break;
    case GameState.Playing:
        UpdateGame();
        break;
    case GameState.Paused:
        ShowPauseMenu();
        break;
    case GameState.GameOver:
        ShowGameOverScreen();
        break;
}
```

---

## 3. for 循环

**已知循环次数**时使用。

### 3.1 基本语法

```csharp
// for (初始化; 条件; 更新)
for (int i = 0; i < 5; i++)
{
    Debug.Log($"第 {i} 次循环");  // 输出 0, 1, 2, 3, 4
}

// 逆序
for (int i = 10; i > 0; i--)
{
    Debug.Log(i);  // 10, 9, 8, ... 1
}

// 步长为2
for (int i = 0; i < 10; i += 2)
{
    Debug.Log(i);  // 0, 2, 4, 6, 8
}
```

### 3.2 嵌套 for 循环

```csharp
// 乘法表
for (int i = 1; i <= 9; i++)
{
    for (int j = 1; j <= i; j++)
    {
        Console.Write($"{j}×{i}={i * j}\t");
    }
    Console.WriteLine();
}

// 二维地图遍历
int width = 10, height = 10;
for (int y = 0; y < height; y++)
{
    for (int x = 0; x < width; x++)
    {
        // 处理地图 tile[x, y]
        ProcessTile(x, y);
    }
}
```

### 3.3 Unity 典型用法

```csharp
// 批量生成敌人
void SpawnEnemies(int count)
{
    for (int i = 0; i < count; i++)
    {
        Vector3 pos = new Vector3(Random.Range(-10f, 10f), 0, Random.Range(-10f, 10f));
        Instantiate(enemyPrefab, pos, Quaternion.identity);
    }
}

// 遍历子物体
for (int i = 0; i < transform.childCount; i++)
{
    Transform child = transform.GetChild(i);
    child.gameObject.SetActive(true);
}
```

---

## 4. while 循环

**条件为 true 就继续执行**，先判断再执行。

### 4.1 基本语法

```csharp
// 基本 while
int countdown = 5;
while (countdown > 0)
{
    Debug.Log(countdown);
    countdown--;  // 别忘了更新条件！否则死循环
}
Debug.Log("发射！");

// 随机生成直到满足条件
int roll = 0;
while (roll != 6)
{
    roll = Random.Range(1, 7);  // 1-6
    Debug.Log($"掷骰子: {roll}");
}
Debug.Log("掷到6了！");
```

### 4.2 ⚠️ 死循环警告

```csharp
// ❌ 危险！死循环
int x = 0;
while (x < 10)
{
    Debug.Log(x);
    // 忘了 x++！程序会卡死
}

// ✅ 安全写法
int x = 0;
while (x < 10)
{
    Debug.Log(x);
    x++;  // 确保条件最终会变为 false
}
```

### 4.3 Unity 典型用法

```csharp
// 协程中等待条件满足
IEnumerator WaitForPlayerReady()
{
    while (!isPlayerReady)
    {
        Debug.Log("等待玩家准备...");
        yield return null;  // 等待下一帧
    }
    Debug.Log("玩家已准备，开始游戏！");
}

// 查找最近的敌人
Transform FindNearestEnemy()
{
    Transform nearest = null;
    float minDist = float.MaxValue;
    int i = 0;
    
    while (i < enemies.Length)
    {
        float dist = Vector3.Distance(transform.position, enemies[i].position);
        if (dist < minDist)
        {
            minDist = dist;
            nearest = enemies[i];
        }
        i++;
    }
    return nearest;
}
```

---

## 5. do-while 循环

**先执行一次，再判断条件**。至少执行一次。

### 5.1 基本语法

```csharp
// do-while：至少执行一次
int input;
do
{
    Debug.Log("请输入 1-3 的选项:");
    input = GetPlayerInput();  // 假设的获取输入方法
} while (input < 1 || input > 3);

Debug.Log($"你选择了: {input}");
```

### 5.2 while vs do-while

```csharp
// while：可能一次都不执行
int x = 10;
while (x < 5)  // 条件一开始就不满足
{
    Debug.Log(x);  // 不会执行
    x++;
}

// do-while：至少执行一次
int y = 10;
do
{
    Debug.Log(y);  // 会执行一次，输出 10
    y++;
} while (y < 5);
```

---

## 6. foreach 循环

**遍历集合中的每个元素**，最简洁。

### 6.1 基本语法

```csharp
// 遍历数组
int[] scores = { 90, 85, 70, 95, 60 };
foreach (int score in scores)
{
    Debug.Log($"分数: {score}");
}

// 遍历 List
List<string> items = new List<string> { "剑", "盾", "药水" };
foreach (string item in items)
{
    Debug.Log($"物品: {item}");
}

// 遍历 Dictionary
Dictionary<string, int> inventory = new Dictionary<string, int>
{
    { "金币", 100 },
    { "宝石", 5 }
};
foreach (var kvp in inventory)
{
    Debug.Log($"{kvp.Key}: {kvp.Value}");
}
```

### 6.2 ⚠️ 注意事项

```csharp
// ❌ foreach 中不能修改集合
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
foreach (int n in numbers)
{
    // numbers.Remove(n);  // 运行时错误！
}

// ✅ 需要修改时用 for 循环（倒序遍历）
for (int i = numbers.Count - 1; i >= 0; i--)
{
    if (numbers[i] < 3)
    {
        numbers.RemoveAt(i);
    }
}
```

### 6.3 Unity 典型用法

```csharp
// 遍历所有敌人
void DamageAllEnemies(int damage)
{
    GameObject[] enemies = GameObject.FindGameObjectsWithTag("Enemy");
    foreach (GameObject enemy in enemies)
    {
        enemy.GetComponent<Health>().TakeDamage(damage);
    }
}

// 遍历子物体并重置
void ResetAllChildren()
{
    foreach (Transform child in transform)
    {
        child.localPosition = Vector3.zero;
        child.localRotation = Quaternion.identity;
    }
}
```

---

## 7. break 与 continue

### 7.1 break：立即退出循环

```csharp
// 在数组中查找第一个偶数
int[] numbers = { 3, 7, 2, 9, 4, 6 };
foreach (int num in numbers)
{
    if (num % 2 == 0)
    {
        Debug.Log($"找到第一个偶数: {num}");
        break;  // 找到就退出
    }
}
// 输出: 找到第一个偶数: 2
```

### 7.2 continue：跳过本次循环

```csharp
// 只输出奇数
for (int i = 0; i < 10; i++)
{
    if (i % 2 == 0)
    {
        continue;  // 跳过偶数，进入下一次循环
    }
    Debug.Log(i);  // 1, 3, 5, 7, 9
}
```

### 7.3 Unity 典型用法

```csharp
// 查找可攻击的敌人（跳过已死的）
foreach (GameObject enemy in enemies)
{
    Health hp = enemy.GetComponent<Health>();
    if (hp == null || hp.IsDead)
    {
        continue;  // 跳过无效目标
    }
    
    float dist = Vector3.Distance(transform.position, enemy.transform.position);
    if (dist <= attackRange)
    {
        Attack(enemy);
        break;  // 攻击一个就够了
    }
}
```

---

## 8. return 语句

### 8.1 基本用法

```csharp
// 有返回值的方法
int Add(int a, int b)
{
    return a + b;  // 返回结果，退出方法
}

// 无返回值的方法（void）
void Die()
{
    if (isDead) return;  // 提前退出，防止重复执行
    
    isDead = true;
    PlayDeathAnimation();
    ShowGameOver();
}
```

### 8.2 Unity 典型用法

```csharp
// 早期退出模式（卫语句）—— 推荐写法
void Update()
{
    if (isPaused) return;      // 游戏暂停，什么都不做
    if (!isAlive) return;      // 玩家已死，不处理输入
    
    HandleInput();
    UpdateMovement();
    CheckCollisions();
}

// 带返回值的查找方法
GameObject FindClosestEnemy()
{
    GameObject closest = null;
    float minDist = Mathf.Infinity;
    
    foreach (GameObject enemy in enemies)
    {
        float dist = Vector3.Distance(transform.position, enemy.transform.position);
        if (dist < minDist)
        {
            minDist = dist;
            closest = enemy;
        }
    }
    
    return closest;  // 可能返回 null
}
```

---

## 9. Unity 实际应用

```csharp
using UnityEngine;
using System.Collections.Generic;

public class GameLoop : MonoBehaviour
{
    public enum GamePhase { Preparation, Battle, Victory, Defeat }

    [SerializeField] private List<GameObject> enemies = new List<GameObject>();
    private GamePhase currentPhase = GamePhase.Preparation;
    private float phaseTimer;

    void Update()
    {
        if (currentPhase == GamePhase.Victory || currentPhase == GamePhase.Defeat)
            return;

        // switch 处理不同阶段
        switch (currentPhase)
        {
            case GamePhase.Preparation:
                UpdatePreparation();
                break;
            case GamePhase.Battle:
                UpdateBattle();
                break;
        }
    }

    void UpdatePreparation()
    {
        phaseTimer -= Time.deltaTime;
        if (phaseTimer <= 0)
        {
            SpawnWave();
            currentPhase = GamePhase.Battle;
        }
    }

    void UpdateBattle()
    {
        // 倒序遍历，安全移除死亡敌人
        for (int i = enemies.Count - 1; i >= 0; i--)
        {
            if (enemies[i] == null)  // 已被销毁
            {
                enemies.RemoveAt(i);
                continue;
            }

            var health = enemies[i].GetComponent<Health>();
            if (health != null && health.IsDead)
            {
                GiveReward(enemies[i]);
                Destroy(enemies[i]);
                enemies.RemoveAt(i);
            }
        }

        // 检查胜利条件
        if (enemies.Count == 0)
        {
            currentPhase = GamePhase.Victory;
            return;
        }

        // foreach 检查敌人是否靠近玩家
        foreach (GameObject enemy in enemies)
        {
            if (enemy == null) continue;

            float dist = Vector3.Distance(transform.position, enemy.transform.position);
            if (dist <= dangerRadius)
            {
                OnEnemyApproach(enemy);
                break;  // 一次只处理一个
            }
        }
    }

    void SpawnWave()
    {
        for (int i = 0; i < 5; i++)
        {
            var enemy = Instantiate(enemyPrefab, GetRandomPos(), Quaternion.identity);
            enemies.Add(enemy);
        }
    }

    // 其他辅助方法省略...
    Vector3 GetRandomPos() => new Vector3(Random.Range(-10f, 10f), 0, Random.Range(-10f, 10f));
    void GiveReward(GameObject e) { }
    void OnEnemyApproach(GameObject e) { }

    [SerializeField] private GameObject enemyPrefab;
    [SerializeField] private float dangerRadius = 5f;
}
```

---

## 📝 练习题

1. 写一个方法，用 for 循环计算 1 到 100 的总和
2. 用 while 循环模拟掷骰子，直到掷出 6 为止，记录掷了多少次
3. 用 foreach 遍历一个字符串数组，只输出长度大于 3 的字符串（用 continue）
4. 写一个简单的猜数字游戏，用 do-while 循环实现

---

> 🔗 上一节：[02 - 运算符与表达式](../02-运算符与表达式/README.md)
> 🔗 下一节：[04 - 数组与集合](../04-数组与集合/README.md)
