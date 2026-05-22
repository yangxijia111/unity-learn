# 03 - ScriptableObject

ScriptableObject 是 Unity 中用于存储**数据资产**的基类，不依赖场景和 GameObject，适合做配置表、共享数据。

## 与 MonoBehaviour 的区别

| 特性 | ScriptableObject | MonoBehaviour |
|------|-------------------|---------------|
| 需要挂载到物体 | ❌ 不需要 | ✅ 需要 |
| 有自己的生命周期 | ❌ 没有 Update 等 | ✅ 有完整生命周期 |
| 本质 | 数据容器 | 组件/行为 |
| 内存 | 共享引用，节省内存 | 每个实例独立 |
| 持久化 | 可保存为 .asset 文件 | 依赖场景 |

## 创建 ScriptableObject

### 1. 定义数据类

```csharp
using UnityEngine;

// 创建右键菜单入口
[CreateAssetMenu(fileName = "New Item", menuName = "GameData/Item")]
public class ItemData : ScriptableObject
{
    public string itemName;
    public string description;
    public Sprite icon;
    public int price;
    public ItemType type;

    public enum ItemType
    {
        Weapon,
        Armor,
        Consumable,
        Material
    }
}
```

### 2. 在编辑器中创建

- **Project 窗口** → 右键 → Create → GameData → Item
- 填写数据后就是一个 `.asset` 文件

## 实战示例

### 角色属性配置

```csharp
[CreateAssetMenu(fileName = "New Character", menuName = "GameData/Character")]
public class CharacterData : ScriptableObject
{
    public string characterName;
    public float maxHP;
    public float attack;
    public float defense;
    public float moveSpeed;
    public RuntimeAnimatorController animator;
}
```

### 游戏配置（全局共享）

```csharp
[CreateAssetMenu(fileName = "GameConfig", menuName = "GameData/GameConfig")]
public class GameConfig : ScriptableObject
{
    [Header("玩家设置")]
    public float playerSpeed = 5f;
    public float jumpForce = 10f;
    public int maxLives = 3;

    [Header("敌人设置")]
    public float enemySpawnRate = 2f;
    public float enemySpeed = 3f;

    [Header("UI设置")]
    public Color healthBarColor = Color.red;
    public float uiFadeTime = 0.3f;
}
```

### 在 MonoBehaviour 中使用

```csharp
public class PlayerController : MonoBehaviour
{
    // 在 Inspector 中拖入配置资产
    public CharacterData characterData;
    public GameConfig gameConfig;

    private float currentHP;

    void Start()
    {
        // 直接读取 ScriptableObject 中的数据
        currentHP = characterData.maxHP;
        Debug.Log($"角色: {characterData.characterName}, HP: {currentHP}");
    }

    void Update()
    {
        // 使用共享配置
        float speed = gameConfig.playerSpeed;
        transform.Translate(Vector3.forward * speed * Time.deltaTime);
    }
}
```

### 运行时创建（代码生成）

```csharp
public class RuntimeSOCreator : MonoBehaviour
{
    void Start()
    {
        // 运行时创建 SO 实例（不会保存为文件）
        ItemData potion = ScriptableObject.CreateInstance<ItemData>();
        potion.itemName = "生命药水";
        potion.description = "恢复 50 点生命值";
        potion.price = 100;
        potion.type = ItemData.ItemType.Consumable;

        Debug.Log($"创建物品: {potion.itemName}");
    }
}
```

### 事件系统（SO 当消息总线）

```csharp
[CreateAssetMenu(fileName = "GameEvent", menuName = "GameData/GameEvent")]
public class GameEvent : ScriptableObject
{
    private List<System.Action> listeners = new List<System.Action>();

    public void Raise()
    {
        // 反向遍历，防止 listener 在回调中移除自己
        for (int i = listeners.Count - 1; i >= 0; i--)
        {
            listeners[i]?.Invoke();
        }
    }

    public void Register(System.Action listener)
    {
        if (!listeners.Contains(listener))
            listeners.Add(listener);
    }

    public void Unregister(System.Action listener)
    {
        listeners.Remove(listener);
    }
}

// 使用示例
public class GameEventListener : MonoBehaviour
{
    public GameEvent gameEvent;
    public UnityEvent response;

    void OnEnable() => gameEvent.Register(OnEventRaised);
    void OnDisable() => gameEvent.Unregister(OnEventRaised);

    void OnEventRaised() => response.Invoke();
}
```

## 使用场景总结

| 场景 | 说明 |
|------|------|
| **配置表** | 角色属性、物品数据、关卡配置 |
| **共享数据** | 多个对象引用同一份数据，修改一处全局生效 |
| **解耦通信** | 作为事件总线，减少脚本间的直接引用 |
| **编辑器工具** | 存储编辑器状态、构建设置 |
| **存档系统** | 临时存储玩家进度数据 |

## 注意事项

1. **SO 的修改在运行时不会持久化** —— 退出 Play 模式后恢复原值（这是特性，不是 Bug）
2. **多个场景共享的配置适合用 SO**，避免重复配置
3. **SO 不能加到预制体的组件列表**，但可以作为字段引用
4. **不要在 SO 里存储运行时状态**（如当前 HP），那是 MonoBehaviour 的活
