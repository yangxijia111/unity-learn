# 09 - 异常处理

> C# 基础入门 | 适合 Unity 开发者

---

## 📑 目录

- [1. try-catch-finally](#1-try-catch-finally)
- [2. throw 抛出异常](#2-throw-抛出异常)
- [3. 常见异常类型](#3-常见异常类型)
- [4. 自定义异常](#4-自定义异常)
- [5. 异常处理最佳实践](#5-异常处理最佳实践)
- [6. Unity 中的异常处理](#6-unity-中的异常处理)

---

## 1. try-catch-finally

### 1.1 基本语法

```csharp
try
{
    // 可能出错的代码
    int[] arr = { 1, 2, 3 };
    Debug.Log(arr[10]);  // 越界！会抛出异常
}
catch (Exception e)
{
    // 捕获异常后执行
    Debug.LogError($"出错了: {e.Message}");
}
finally
{
    // 无论是否异常，都会执行（可选）
    Debug.Log("清理工作完成");
}
```

### 1.2 捕获特定异常

```csharp
try
{
    string input = "abc";
    int number = int.Parse(input);  // FormatException
}
catch (FormatException e)
{
    Debug.LogError("格式错误：不是有效数字");
}
catch (OverflowException e)
{
    Debug.LogError("溢出错误：数字太大");
}
catch (Exception e)
{
    // 兜底：捕获所有其他异常
    Debug.LogError($"未知错误: {e.Message}");
}
```

### 1.3 异常对象的信息

```csharp
try
{
    int x = 0;
    int result = 10 / x;
}
catch (DivideByZeroException e)
{
    Debug.LogError($"消息: {e.Message}");           // 错误描述
    Debug.LogError($"类型: {e.GetType().Name}");    // 异常类型
    Debug.LogError($"堆栈: {e.StackTrace}");        // 调用栈
    Debug.LogError($"内部: {e.InnerException}");    // 内部异常
}
```

### 1.4 finally 的作用

```csharp
// finally 常用于释放资源
StreamReader reader = null;
try
{
    reader = new StreamReader("data.txt");
    string content = reader.ReadToEnd();
    Debug.Log(content);
}
catch (FileNotFoundException)
{
    Debug.LogError("文件不存在");
}
finally
{
    // 无论成功还是失败，都关闭文件
    reader?.Close();
    Debug.Log("文件已关闭");
}
```

### 1.5 using 语句（资源管理）

```csharp
// using：自动释放资源（等价于 try-finally）
using (StreamReader reader = new StreamReader("data.txt"))
{
    string content = reader.ReadToEnd();
    Debug.Log(content);
}  // 自动调用 reader.Close() 和 reader.Dispose()

// C# 8.0+ 简化写法
using StreamReader reader = new StreamReader("data.txt");
string content = reader.ReadToEnd();
// 方法结束时自动释放
```

---

## 2. throw 抛出异常

### 2.1 抛出异常

```csharp
public void SetHP(int value)
{
    if (value < 0)
    {
        throw new ArgumentException("HP 不能为负数");
    }
    if (value > maxHP)
    {
        throw new ArgumentOutOfRangeException(nameof(value), 
            $"HP 不能超过 {maxHP}");
    }
    hp = value;
}
```

### 2.2 重新抛出异常

```csharp
try
{
    DangerousOperation();
}
catch (Exception e)
{
    Debug.LogError("记录日志...");
    
    // 重新抛出，让上层处理
    throw;           // 保留原始堆栈信息（推荐）
    // throw e;      // 会重置堆栈（不推荐）
}
```

### 2.3 异常过滤器（C# 6.0+）

```csharp
try
{
    CallAPI();
}
catch (HttpRequestException e) when (e.Message.Contains("404"))
{
    Debug.LogError("资源不存在");
}
catch (HttpRequestException e) when (e.Message.Contains("500"))
{
    Debug.LogError("服务器错误");
}
catch (HttpRequestException e)
{
    Debug.LogError($"网络错误: {e.Message}");
}
```

---

## 3. 常见异常类型

| 异常类型 | 触发场景 |
|---------|---------|
| `NullReferenceException` | 对象为 null 时访问其成员 |
| `IndexOutOfRangeException` | 数组/列表索引越界 |
| `ArgumentException` | 方法参数不合法 |
| `ArgumentNullException` | 参数不能为 null 却传了 null |
| `InvalidOperationException` | 对象状态不允许当前操作 |
| `FormatException` | 字符串格式错误（如 "abc" 转 int） |
| `OverflowException` | 数值运算溢出 |
| `DivideByZeroException` | 除以零 |
| `FileNotFoundException` | 文件不存在 |
| `KeyNotFoundException` | Dictionary 中键不存在 |

### Unity 中常见的异常

```csharp
// 1. NullReferenceException —— 最常见！
GameObject enemy = GameObject.Find("Enemy");
// 如果找不到 "Enemy"，enemy 就是 null
enemy.SetActive(true);  // ❌ NullReferenceException

// ✅ 防御
if (enemy != null)
{
    enemy.SetActive(true);
}

// 2. MissingReferenceException —— Unity 特有
// 对象已被 Destroy，但还有引用
Destroy(enemy);
// 后面访问 enemy 就会报 MissingReferenceException

// 3. MissingComponentException
var rb = GetComponent<Rigidbody>();
rb.AddForce(Vector3.up);  // ❌ 如果没挂 Rigidbody 组件
```

---

## 4. 自定义异常

### 4.1 创建自定义异常

```csharp
// 自定义异常类：继承 Exception
public class GameException : Exception
{
    public GameException() { }
    public GameException(string message) : base(message) { }
    public GameException(string message, Exception inner) : base(message, inner) { }
}

// 具体异常
public class NotEnoughGoldException : GameException
{
    public int Required { get; }
    public int Current { get; }
    
    public NotEnoughGoldException(int required, int current)
        : base($"金币不足：需要 {required}，当前 {current}")
    {
        Required = required;
        Current = current;
    }
}

public class PlayerDeadException : GameException
{
    public string PlayerName { get; }
    
    public PlayerDeadException(string playerName)
        : base($"玩家 {playerName} 已死亡，无法执行操作")
    {
        PlayerName = playerName;
    }
}

public class SkillOnCooldownException : GameException
{
    public float RemainingTime { get; }
    
    public SkillOnCooldownException(float remaining)
        : base($"技能冷却中，还需 {remaining:F1} 秒")
    {
        RemainingTime = remaining;
    }
}
```

### 4.2 使用自定义异常

```csharp
public class ShopSystem
{
    public void BuyItem(Item item, Player player)
    {
        if (!player.IsAlive)
            throw new PlayerDeadException(player.Name);
        
        if (player.Gold < item.Price)
            throw new NotEnoughGoldException(item.Price, player.Gold);
        
        player.Gold -= item.Price;
        player.Inventory.Add(item);
    }
}

// 调用方
try
{
    shop.BuyItem(sword, player);
    Debug.Log("购买成功！");
}
catch (NotEnoughGoldException e)
{
    Debug.LogWarning($"还差 {e.Required - e.Current} 金币");
}
catch (PlayerDeadException e)
{
    Debug.LogError(e.Message);
}
```

---

## 5. 异常处理最佳实践

### 5.1 应该做的

```csharp
// ✅ 验证参数并抛出明确的异常
public void SetLevel(int level)
{
    if (level < 1)
        throw new ArgumentOutOfRangeException(nameof(level), "等级不能小于1");
    if (level > 99)
        throw new ArgumentOutOfRangeException(nameof(level), "等级不能超过99");
    
    this.level = level;
}

// ✅ 用 Try 方法避免异常
if (int.TryParse(input, out int result))
{
    // 成功
}
else
{
    // 失败，但没有异常开销
}

// ✅ 用 TryGetValue 避免 KeyNotFoundException
if (dict.TryGetValue(key, out var value))
{
    // 安全访问
}

// ✅ 捕获后记录足够信息
catch (Exception e)
{
    Debug.LogError($"[{GetType().Name}] 操作失败: {e.Message}\n{e.StackTrace}");
}
```

### 5.2 不应该做的

```csharp
// ❌ 用异常控制正常流程
try
{
    int val = dict[key];  // 可能抛出 KeyNotFoundException
}
catch (KeyNotFoundException)
{
    val = defaultValue;  // 应该用 TryGetValue
}

// ❌ 吞掉异常（什么都不做）
try
{
    DangerousOperation();
}
catch (Exception) { }  // 错误被隐藏，以后很难调试

// ❌ 过于宽泛的捕获
try
{
    // 很多代码...
}
catch (Exception)  // 捕获所有异常，可能隐藏真正的 bug
{
    // 随便处理
}

// ❌ 在循环中使用异常控制流程
for (int i = 0; ; i++)
{
    try
    {
        DoSomething(arr[i]);  // 用异常判断结束
    }
    catch (IndexOutOfRangeException)
    {
        break;  // 应该用 i < arr.Length
    }
}
```

### 5.3 异常处理原则

```
1. 只在异常情况下使用异常（不用来控制正常流程）
2. 尽量捕获具体的异常类型
3. 异常发生后要记录足够信息
4. 不要吞掉异常（至少记录日志）
5. 用 Try 方法代替 try-catch（如 TryParse、TryGetValue）
6. 参数验证尽早抛出异常（fail fast）
```

---

## 6. Unity 中的异常处理

### 6.1 全局异常处理

```csharp
public class ExceptionHandler : MonoBehaviour
{
    void Awake()
    {
        // 注册全局异常处理器
        Application.logMessageReceived += OnLogMessage;
    }
    
    void OnDestroy()
    {
        Application.logMessageReceived -= OnLogMessage;
    }
    
    void OnLogMessage(string message, string stackTrace, LogType type)
    {
        if (type == LogType.Exception)
        {
            // 上报到错误追踪服务
            ErrorReporter.Instance?.ReportException(message, stackTrace);
            
            // 或者显示友好的错误提示
            UIManager.Instance?.ShowError("游戏遇到问题，请重启");
        }
    }
}
```

### 6.2 协程中的异常处理

```csharp
IEnumerator LoadData()
{
    UnityWebRequest request = null;
    try
    {
        request = UnityWebRequest.Get("https://api.example.com/data");
        yield return request.SendWebRequest();
        
        if (request.result != UnityWebRequest.Result.Success)
        {
            throw new Exception($"网络错误: {request.error}");
        }
        
        string json = request.downloadHandler.text;
        ProcessData(json);
    }
    catch (Exception e)
    {
        Debug.LogError($"加载失败: {e.Message}");
        // 协程中的异常不会冒泡到调用者，需要在这里处理
    }
    finally
    {
        request?.Dispose();
    }
}
```

### 6.3 实际游戏代码示例

```csharp
public class SaveSystem : MonoBehaviour
{
    public void SaveGame(GameData data)
    {
        string path = GetSavePath();
        string json = JsonUtility.ToJson(data, true);
        
        try
        {
            // 确保目录存在
            string dir = Path.GetDirectoryName(path);
            if (!Directory.Exists(dir))
                Directory.CreateDirectory(dir);
            
            // 先写临时文件，再重命名（防止写入中断导致数据损坏）
            string tempPath = path + ".tmp";
            File.WriteAllText(tempPath, json);
            
            if (File.Exists(path))
                File.Delete(path);
            File.Move(tempPath, path);
            
            Debug.Log("保存成功");
        }
        catch (UnauthorizedAccessException e)
        {
            Debug.LogError($"无权限写入: {e.Message}");
        }
        catch (IOException e)
        {
            Debug.LogError($"IO 错误: {e.Message}");
        }
        catch (Exception e)
        {
            Debug.LogError($"保存失败: {e.Message}");
            throw;  // 重新抛出，让调用者知道
        }
    }

    public GameData LoadGame()
    {
        string path = GetSavePath();
        
        if (!File.Exists(path))
        {
            Debug.LogWarning("存档文件不存在，创建新游戏");
            return new GameData();  // 返回默认数据
        }
        
        try
        {
            string json = File.ReadAllText(path);
            GameData data = JsonUtility.FromJson<GameData>(json);
            
            if (data == null)
                throw new InvalidDataException("存档数据为空");
            
            return data;
        }
        catch (InvalidDataException e)
        {
            Debug.LogError($"存档损坏: {e.Message}");
            // 尝试从备份恢复
            return TryLoadBackup() ?? new GameData();
        }
        catch (Exception e)
        {
            Debug.LogError($"加载失败: {e.Message}");
            return new GameData();
        }
    }

    string GetSavePath() => Path.Combine(Application.persistentDataPath, "save.json");
    GameData TryLoadBackup() => null;
}

[System.Serializable]
public class GameData { }
```

---

## 📝 练习题

1. 写一个 `Divide(int a, int b)` 方法，处理除零异常并返回安全结果
2. 自定义 `InventoryFullException` 异常，在背包已满时抛出
3. 写一个安全的 `ParseVector3(string input)` 方法，用 try-catch 处理格式错误
4. 重构以下代码，避免用异常控制流程：

```csharp
// ❌ 有问题的代码，请修复
try { var x = dict[key]; } catch { x = default; }
```

---

> 🔗 上一节：[08 - 接口与抽象类](../08-接口与抽象类/README.md)
> 🔗 下一节：[10 - 委托与事件](../10-委托与事件/README.md)
