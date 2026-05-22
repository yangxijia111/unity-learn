# 数据结构与算法

> 游戏开发离不开数据结构和算法。这里整理游戏开发中最常用的那些，用 Unity/C# 实际代码演示。

## 📌 你会学到什么

- 游戏常用数据结构（链表、树、图、堆）
- 排序算法及实际应用
- 查找算法
- 时间复杂度分析（大O表示法）
- Unity 中的实际应用场景

---

## 1. 常用数据结构

### 1.1 链表 LinkedList

**特点：** 插入/删除快，查找慢。适合频繁增删的场景。

```csharp
// Unity 中的 LinkedList 应用：任务队列
using System.Collections.Generic;

public class TaskQueue : MonoBehaviour
{
    private LinkedList<string> tasks = new LinkedList<string>();

    // 添加任务到队尾
    public void AddTask(string task)
    {
        tasks.AddLast(task);
        Debug.Log($"任务已添加：{task}");
    }

    // 插队：添加到指定任务前面
    public void InsertBefore(string targetTask, string newTask)
    {
        var node = tasks.Find(targetTask);
        if (node != null)
        {
            tasks.AddBefore(node, newTask);
            Debug.Log($"插队成功：{newTask} 在 {targetTask} 之前");
        }
    }

    // 执行队首任务
    public void ExecuteNext()
    {
        if (tasks.Count > 0)
        {
            string task = tasks.First.Value;
            Debug.Log($"执行任务：{task}");
            tasks.RemoveFirst();
        }
    }
}
```

### 1.2 树 Tree（二叉搜索树）

**特点：** 层级结构，查找效率高。用于场景树、UI 层级、技能树等。

```csharp
// 二叉搜索树节点
public class TreeNode
{
    public int value;
    public TreeNode left;   // 左子树（比当前节点小）
    public TreeNode right;  // 右子树（比当前节点大）

    public TreeNode(int value)
    {
        this.value = value;
    }
}

// 简单的二叉搜索树
public class BST
{
    public TreeNode root;

    // 插入节点
    public void Insert(int value)
    {
        root = InsertRecursive(root, value);
    }

    private TreeNode InsertRecursive(TreeNode node, int value)
    {
        if (node == null) return new TreeNode(value);

        if (value < node.value)
            node.left = InsertRecursive(node.left, value);
        else if (value > node.value)
            node.right = InsertRecursive(node.right, value);

        return node;
    }

    // 查找节点
    public bool Search(int value)
    {
        return SearchRecursive(root, value);
    }

    private bool SearchRecursive(TreeNode node, int value)
    {
        if (node == null) return false;
        if (value == node.value) return true;
        return value < node.value
            ? SearchRecursive(node.left, value)
            : SearchRecursive(node.right, value);
    }
}

// 游戏应用：技能树系统
// 你可以把技能的解锁顺序用树来表示
```

### 1.3 图 Graph

**特点：** 表达复杂关系。用于寻路、社交关系、关卡连接。

```csharp
// 邻接表实现图
public class Graph
{
    // 节点 -> 邻居列表
    private Dictionary<string, List<string>> adjacencyList = new Dictionary<string, List<string>>();

    // 添加节点
    public void AddNode(string node)
    {
        if (!adjacencyList.ContainsKey(node))
            adjacencyList[node] = new List<string>();
    }

    // 添加边（无向图）
    public void AddEdge(string from, string to)
    {
        AddNode(from);
        AddNode(to);
        adjacencyList[from].Add(to);
        adjacencyList[to].Add(from);
    }

    // BFS 广度优先搜索（找最短路径）
    public List<string> BFS(string start, string end)
    {
        Queue<string> queue = new Queue<string>();
        Dictionary<string, string> parent = new Dictionary<string, string>();
        HashSet<string> visited = new HashSet<string>();

        queue.Enqueue(start);
        visited.Add(start);

        while (queue.Count > 0)
        {
            string current = queue.Dequeue();

            if (current == end)
                return ReconstructPath(parent, start, end);

            foreach (string neighbor in adjacencyList[current])
            {
                if (!visited.Contains(neighbor))
                {
                    visited.Add(neighbor);
                    parent[neighbor] = current;
                    queue.Enqueue(neighbor);
                }
            }
        }

        return null; // 没找到路径
    }

    private List<string> ReconstructPath(Dictionary<string, string> parent, string start, string end)
    {
        List<string> path = new List<string>();
        string current = end;
        while (current != start)
        {
            path.Add(current);
            current = parent[current];
        }
        path.Add(start);
        path.Reverse();
        return path;
    }
}

// 游戏应用：地图关卡连接、NPC社交关系网
```

### 1.4 堆 Heap（优先队列）

**特点：** 快速获取最大/最小值。用于任务优先级、A*寻路。

```csharp
// 最小堆实现（简化版，用于A*寻路的开放列表）
public class MinHeap<T> where T : IComparable<T>
{
    private List<T> items = new List<T>();

    public int Count => items.Count;

    public void Add(T item)
    {
        items.Add(item);
        HeapifyUp(items.Count - 1);
    }

    public T ExtractMin()
    {
        if (items.Count == 0) throw new System.InvalidOperationException("堆为空");

        T min = items[0];
        items[0] = items[items.Count - 1];
        items.RemoveAt(items.Count - 1);

        if (items.Count > 0)
            HeapifyDown(0);

        return min;
    }

    // 上浮操作
    private void HeapifyUp(int index)
    {
        while (index > 0)
        {
            int parent = (index - 1) / 2;
            if (items[index].CompareTo(items[parent]) >= 0) break;
            (items[index], items[parent]) = (items[parent], items[index]);
            index = parent;
        }
    }

    // 下沉操作
    private void HeapifyDown(int index)
    {
        int last = items.Count - 1;
        while (true)
        {
            int left = index * 2 + 1;
            int right = index * 2 + 2;
            int smallest = index;

            if (left <= last && items[left].CompareTo(items[smallest]) < 0)
                smallest = left;
            if (right <= last && items[right].CompareTo(items[smallest]) < 0)
                smallest = right;

            if (smallest == index) break;
            (items[index], items[smallest]) = (items[smallest], items[index]);
            index = smallest;
        }
    }
}

// 游戏应用：A*寻路的开放列表、任务优先级队列、定时器
```

---

## 2. 排序算法

### 快速排序 Quick Sort（最常用）

```csharp
public static class SortAlgorithms
{
    // 快速排序 - 平均 O(n log n)
    public static void QuickSort(int[] arr, int low, int high)
    {
        if (low < high)
        {
            int pivotIndex = Partition(arr, low, high);
            QuickSort(arr, low, pivotIndex - 1);
            QuickSort(arr, pivotIndex + 1, high);
        }
    }

    private static int Partition(int[] arr, int low, int high)
    {
        int pivot = arr[high]; // 选最后一个元素作为基准
        int i = low - 1;

        for (int j = low; j < high; j++)
        {
            if (arr[j] <= pivot)
            {
                i++;
                (arr[i], arr[j]) = (arr[j], arr[i]); // 交换
            }
        }

        (arr[i + 1], arr[high]) = (arr[high], arr[i + 1]);
        return i + 1;
    }

    // 内置排序（实际开发中优先用这个）
    public static void UseBuiltInSort(int[] arr)
    {
        System.Array.Sort(arr); // 内部是内省排序 Introsort
    }
}

// 游戏应用：排行榜排序、装备属性排序、敌人优先级排序
```

---

## 3. 查找算法

### 二分查找 Binary Search

```csharp
public static class SearchAlgorithms
{
    // 二分查找 - O(log n)，前提：数组已排序
    public static int BinarySearch(int[] sortedArr, int target)
    {
        int left = 0, right = sortedArr.Length - 1;

        while (left <= right)
        {
            int mid = left + (right - left) / 2;

            if (sortedArr[mid] == target)
                return mid;
            else if (sortedArr[mid] < target)
                left = mid + 1;
            else
                right = mid - 1;
        }

        return -1; // 未找到
    }

    // 实际应用：在排序后的经验值表里查找当前等级
    public static int GetLevelByExp(int[] expTable, int currentExp)
    {
        // expTable[i] = 升到 i+1 级需要的总经验
        // 用二分查找找到当前经验对应的等级
        int level = System.Array.BinarySearch(expTable, currentExp);
        if (level < 0)
            level = ~level; // 按位取反，得到插入点
        return level;
    }
}
```

---

## 4. 时间复杂度（大O表示法）

| 复杂度 | 名称 | 例子 | 游戏中的感受 |
|--------|------|------|-------------|
| O(1) | 常数 | 数组按索引访问 | 瞬间完成 |
| O(log n) | 对数 | 二分查找 | 100万数据约20步 |
| O(n) | 线性 | 遍历列表 | 数据翻倍，时间翻倍 |
| O(n log n) | 线性对数 | 快速排序 | 排序的及格线 |
| O(n²) | 平方 | 冒泡排序、双重循环 | 数据多了会卡 |
| O(2ⁿ) | 指数 | 某些递归 | 别用，会爆栈 |

**🎯 游戏开发经验：**
- 主循环（Update）里的操作，每帧都要跑，必须高效
- 玩家少时 O(n²) 没事，玩家多了就炸
- 能用 Dictionary（O(1)查找）就别用 List 遍历

---

## 5. Unity 实际应用场景

### 场景 1：背包系统的数据结构选择

```csharp
// 背包：用 List 或 Dictionary
public class Inventory : MonoBehaviour
{
    // 方案1：有序列表（适合有格子限制的背包）
    public List<Item> items = new List<Item>();

    // 方案2：字典（适合堆叠型道具，如金币、药水）
    public Dictionary<string, int> stackableItems = new Dictionary<string, int>();

    // 字典查找 O(1)，比遍历 List O(n) 快得多
    public bool HasItem(string itemId)
    {
        return stackableItems.ContainsKey(itemId);
    }
}

[System.Serializable]
public class Item
{
    public string id;
    public string name;
    public int count;
}
```

### 场景 2：排行榜排序

```csharp
public class Leaderboard : MonoBehaviour
{
    private List<PlayerScore> scores = new List<PlayerScore>();

    public void AddScore(string playerName, int score)
    {
        scores.Add(new PlayerScore { name = playerName, score = score });

        // 按分数降序排列
        scores.Sort((a, b) => b.score.CompareTo(a.score));

        // 只保留前10名
        if (scores.Count > 10)
            scores.RemoveRange(10, scores.Count - 10);
    }
}

public class PlayerScore
{
    public string name;
    public int score;
}
```

---

## 📚 学习建议

1. **先理解概念，再背代码** —— 知道"为什么用"比"怎么写"更重要
2. **优先用内置 API** —— `System.Array.Sort()`、`Dictionary`、`HashSet` 都是优化过的
3. **关注性能瓶颈** —— 用 Profiler 找到真正的慢点，别盲目优化
4. **刷 LeetCode 简单题** —— 每天1-2道，保持手感

---

> 💡 **核心原则：** 游戏开发中，数据结构的选择直接影响性能。100个敌人用 List 遍历没问题，10000个就必须考虑空间分区（如四叉树）了。
