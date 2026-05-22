# 04 - 数组与集合

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. 数组 Array](#1-数组-array)
- [2. List 列表](#2-list-列表)
- [3. Dictionary 字典](#3-dictionary-字典)
- [4. Queue 队列](#4-queue-队列)
- [5. Stack 栈](#5-stack-栈)
- [6. HashSet 集合](#6-hashset-集合)
- [7. 集合选择指南](#7-集合选择指南)
- [8. Unity 实际应用](#8-unity-实际应用)

---

## 1. 数组 Array

固定大小，声明后不可增减元素。访问速度快。

### 1.1 一维数组

```csharp
// 声明和初始化
int[] scores = new int[5];              // 创建大小为5的数组，默认值为0
int[] hp = { 100, 80, 60, 40, 20 };    // 直接初始化
string[] items = new string[] { "剑", "盾", "药水" };

// 访问元素（索引从0开始）
int first = hp[0];       // 100
int last = hp[4];        // 20
hp[2] = 100;             // 修改第三个元素

// 数组长度
int length = hp.Length;  // 5

// 遍历数组
for (int i = 0; i < hp.Length; i++)
{
    Debug.Log($"第{i}个元素: {hp[i]}");
}

foreach (int value in hp)
{
    Debug.Log(value);
}
```

### 1.2 多维数组

```csharp
// 二维数组（矩形）：适合表示棋盘、地图
int[,] grid = new int[3, 3];  // 3×3 的网格
grid[0, 0] = 1;  // 第0行第0列
grid[1, 2] = 5;  // 第1行第2列

// 初始化
int[,] map =
{
    { 1, 0, 1 },
    { 0, 1, 0 },
    { 1, 1, 0 }
};

// 遍历二维数组
for (int row = 0; row < map.GetLength(0); row++)       // GetLength(0) = 行数
{
    for (int col = 0; col < map.GetLength(1); col++)   // GetLength(1) = 列数
    {
        Debug.Log($"map[{row},{col}] = {map[row, col]}");
    }
}
```

### 1.3 交错数组（数组的数组）

```csharp
// 交错数组：每行可以有不同的长度
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2 };
jagged[1] = new int[] { 3, 4, 5 };
jagged[2] = new int[] { 6 };

Debug.Log(jagged[1][2]);  // 5
```

### 1.4 数组常用方法

```csharp
int[] arr = { 5, 3, 1, 4, 2 };

Array.Sort(arr);                    // 排序：{1, 2, 3, 4, 5}
Array.Reverse(arr);                 // 反转：{5, 4, 3, 2, 1}
int index = Array.IndexOf(arr, 3);  // 查找：返回索引1
Array.Clear(arr, 0, arr.Length);    // 清空：全部变为0
```

---

## 2. List 列表

**动态数组**，可随时增减元素。最常用的集合！

### 2.1 基本操作

```csharp
using System.Collections.Generic;

// 创建
List<int> scores = new List<int>();
List<string> items = new List<string> { "剑", "盾", "药水" };

// 添加
scores.Add(100);           // 添加到末尾
scores.Add(85);
scores.Insert(0, 999);     // 在索引0处插入

// 删除
scores.Remove(85);         // 删除第一个值为85的元素
scores.RemoveAt(0);        // 删除索引0的元素
scores.RemoveAll(x => x < 50);  // 删除所有小于50的

// 访问
int first = scores[0];     // 通过索引访问
scores[1] = 200;           // 修改

// 查询
bool has999 = scores.Contains(999);     // 是否包含
int idx = scores.IndexOf(999);          // 查找索引
int count = scores.Count;               // 元素数量

// 遍历
foreach (int score in scores)
{
    Debug.Log(score);
}
```

### 2.2 容量管理

```csharp
List<int> list = new List<int>();

// 容量 vs 数量
int count = list.Count;      // 实际元素数量
int capacity = list.Capacity; // 内部数组容量（自动扩容）

// 预设容量（减少扩容次数，提升性能）
List<int> bigList = new List<int>(1000);  // 预分配1000个位置

// 清空
list.Clear();  // 清空所有元素，Count变为0
```

### 2.3 常用操作

```csharp
List<int> numbers = new List<int> { 5, 3, 8, 1, 9, 2 };

// 排序
numbers.Sort();              // {1, 2, 3, 5, 8, 9}
numbers.Reverse();           // {9, 8, 5, 3, 2, 1}

// 转换
int[] array = numbers.ToArray();   // List → 数组
List<int> copy = new List<int>(numbers);  // 复制

// 查找（Lambda 表达式）
int found = numbers.Find(x => x > 5);           // 找第一个 >5 的
List<int> allFound = numbers.FindAll(x => x > 5); // 找所有 >5 的
```

---

## 3. Dictionary 字典

**键值对**存储，通过 Key 快速查找 Value。

### 3.1 基本操作

```csharp
using System.Collections.Generic;

// 创建
Dictionary<string, int> inventory = new Dictionary<string, int>();

// 添加
inventory.Add("金币", 100);
inventory["宝石"] = 5;         // 另一种添加方式
inventory["药水"] = 3;

// 访问
int gold = inventory["金币"];  // 100

// ✅ 安全访问（推荐）
if (inventory.TryGetValue("宝石", out int gems))
{
    Debug.Log($"宝石数量: {gems}");
}
else
{
    Debug.Log("没有宝石");
}

// 修改
inventory["金币"] = 200;       // 已存在则修改

// 删除
inventory.Remove("药水");

// 查询
bool hasKey = inventory.ContainsKey("金币");  // true
int count = inventory.Count;                  // 键值对数量
```

### 3.2 遍历

```csharp
// 遍历键值对
foreach (var kvp in inventory)
{
    Debug.Log($"{kvp.Key}: {kvp.Value}");
}

// 只遍历键
foreach (string key in inventory.Keys)
{
    Debug.Log(key);
}

// 只遍历值
foreach (int value in inventory.Values)
{
    Debug.Log(value);
}
```

---

## 4. Queue 队列

**先进先出（FIFO）**，像排队一样。

### 4.1 基本操作

```csharp
using System.Collections.Generic;

// 创建
Queue<string> taskQueue = new Queue<string>();

// 入队（添加到队尾）
taskQueue.Enqueue("任务A");
taskQueue.Enqueue("任务B");
taskQueue.Enqueue("任务C");

// 出队（从队头取出并移除）
string next = taskQueue.Dequeue();  // "任务A"

// 查看队头（不移除）
string peek = taskQueue.Peek();     // "任务B"

// 查询
int count = taskQueue.Count;        // 2
bool hasC = taskQueue.Contains("任务C");  // true

// 遍历
foreach (string task in taskQueue)
{
    Debug.Log(task);
}
```

### 4.2 Unity 典型用法

```csharp
// 伤害数字队列（依次显示）
Queue<int> damageNumbers = new Queue<int>();

void TakeDamage(int damage)
{
    damageNumbers.Enqueue(damage);
}

void Update()
{
    if (damageNumbers.Count > 0)
    {
        int damage = damageNumbers.Dequeue();
        ShowDamageNumber(damage);
    }
}
```

---

## 5. Stack 栈

**后进先出（LIFO）**，像叠盘子一样。

### 5.1 基本操作

```csharp
using System.Collections.Generic;

// 创建
Stack<string> history = new Stack<string>();

// 压栈（添加到栈顶）
history.Push("页面A");
history.Push("页面B");
history.Push("页面C");

// 弹栈（取出栈顶并移除）
string top = history.Pop();    // "页面C"

// 查看栈顶（不移除）
string peek = history.Peek();  // "页面B"

// 查询
int count = history.Count;     // 2
```

### 5.2 典型用法：撤销功能

```csharp
// 撤销/重做系统
Stack<Vector3> positionHistory = new Stack<Vector3>();

void MovePlayer(Vector3 newPos)
{
    positionHistory.Push(transform.position);  // 保存当前位置
    transform.position = newPos;
}

void Undo()
{
    if (positionHistory.Count > 0)
    {
        transform.position = positionHistory.Pop();  // 回到上一个位置
    }
}
```

---

## 6. HashSet 集合

**不重复的元素集合**，查找速度极快。

### 6.1 基本操作

```csharp
using System.Collections.Generic;

// 创建
HashSet<string> unlockedLevels = new HashSet<string>();

// 添加（重复添加会被忽略）
unlockedLevels.Add("关卡1");
unlockedLevels.Add("关卡2");
unlockedLevels.Add("关卡1");  // 重复，不会添加

bool added = unlockedLevels.Add("关卡1");  // false（已存在）

// 删除
unlockedLevels.Remove("关卡1");

// 查询
bool hasLevel = unlockedLevels.Contains("关卡2");  // true
int count = unlockedLevels.Count;                  // 唯一元素数量
```

### 6.2 集合运算

```csharp
HashSet<int> setA = new HashSet<int> { 1, 2, 3, 4, 5 };
HashSet<int> setB = new HashSet<int> { 3, 4, 5, 6, 7 };

// 交集：共同拥有的
setA.IntersectWith(setB);       // setA = {3, 4, 5}

// 并集：合并
setA.UnionWith(setB);           // setA = {1, 2, 3, 4, 5, 6, 7}

// 差集：A有B没有的
setA.ExceptWith(setB);          // setA = {1, 2}

// 对称差集：只在一个集合中的
setA.SymmetricExceptWith(setB); // setA = {1, 2, 6, 7}
```

### 6.3 Unity 典型用法

```csharp
// 去重：收集不重复的材料
HashSet<string> collectedMaterials = new HashSet<string>();

void OnMaterialCollected(string material)
{
    if (collectedMaterials.Add(material))  // 返回true表示新收集
    {
        Debug.Log($"新材料: {material}");
        UpdateUI();
    }
}

// 碰撞过滤：防止重复伤害
HashSet<GameObject> hitTargets = new HashSet<GameObject>();

void OnTriggerEnter(Collider other)
{
    if (hitTargets.Add(other.gameObject))  // 第一次碰到才伤害
    {
        other.GetComponent<Health>().TakeDamage(damage);
    }
}

void OnAttackEnd()
{
    hitTargets.Clear();  // 攻击结束，清空记录
}
```

---

## 7. 集合选择指南

| 需求 | 推荐 | 原因 |
|------|------|------|
| 固定大小，高效访问 | `数组[]` | 内存连续，最快 |
| 需要动态增减 | `List<T>` | 最灵活，最常用 |
| 键值查找 | `Dictionary<K,V>` | O(1) 查找速度 |
| 排队处理 | `Queue<T>` | FIFO 语义 |
| 历史记录/撤销 | `Stack<T>` | LIFO 语义 |
| 不重复元素 | `HashSet<T>` | 自动去重 + 快速查找 |

### 性能对比

```
操作          数组    List    Dictionary  HashSet
──────────────────────────────────────────────
按索引访问    O(1)    O(1)    -           -
按值查找      O(n)    O(n)    O(1)        O(1)
添加元素      -       O(1)*   O(1)*       O(1)*
删除元素      -       O(n)    O(1)        O(1)
排序          O(n²)   O(nlogn) -          -

* 均摊复杂度
```

---

## 8. Unity 实际应用

```csharp
using UnityEngine;
using System.Collections.Generic;

public class InventorySystem : MonoBehaviour
{
    // List：背包物品
    [SerializeField] private List<ItemData> items = new List<ItemData>();
    
    // Dictionary：物品ID → 数量映射
    private Dictionary<string, int> itemCounts = new Dictionary<string, int>();
    
    // Queue：待处理的掉落物
    private Queue<ItemData> dropQueue = new Queue<ItemData>();
    
    // HashSet：已解锁的配方
    private HashSet<string> unlockedRecipes = new HashSet<string>();

    public bool AddItem(ItemData item)
    {
        if (items.Count >= maxSlots)
        {
            Debug.Log("背包已满");
            return false;
        }

        items.Add(item);
        
        // 更新计数
        if (itemCounts.ContainsKey(item.id))
            itemCounts[item.id]++;
        else
            itemCounts[item.id] = 1;

        return true;
    }

    public bool RemoveItem(string itemId, int count = 1)
    {
        if (!itemCounts.ContainsKey(itemId) || itemCounts[itemId] < count)
            return false;

        itemCounts[itemId] -= count;
        if (itemCounts[itemId] <= 0)
            itemCounts.Remove(itemId);

        // 从列表中移除
        items.RemoveAll(item => item.id == itemId);
        return true;
    }

    public int GetItemCount(string itemId)
    {
        return itemCounts.TryGetValue(itemId, out int count) ? count : 0;
    }

    public void QueueDrop(ItemData item)
    {
        dropQueue.Enqueue(item);
    }

    void Update()
    {
        // 每帧处理一个掉落
        if (dropQueue.Count > 0)
        {
            var item = dropQueue.Dequeue();
            SpawnDropItem(item);
        }
    }

    public bool UnlockRecipe(string recipeId)
    {
        return unlockedRecipes.Add(recipeId);  // 返回是否成功（不重复）
    }

    [SerializeField] private int maxSlots = 20;
    void SpawnDropItem(ItemData item) { }
}

[System.Serializable]
public class ItemData
{
    public string id;
    public string name;
    public int rarity;
}
```

---

## 📝 练习题

1. 用 `List<int>` 存储考试分数，实现添加、删除不及格分数（<60）、排序输出
2. 用 `Dictionary<string, int>` 实现一个简单的物品商店，支持查看价格、购买（扣金币）
3. 用 `Queue` 模拟排队系统：3个窗口，顾客先到先服务
4. 用 `HashSet` 实现好友列表：添加好友、删除好友、查看共同好友（交集）

---

> 🔗 上一节：[03 - 流程控制](../03-流程控制/README.md)
> 🔗 下一节：[05 - 函数与方法](../05-函数与方法/README.md)
