# 预制体 Prefab

## 📋 目录

- [什么是 Prefab](#什么是-prefab)
- [创建 Prefab](#创建-prefab)
- [实例化 Instantiate](#实例化-instantiate)
- [销毁与对象池](#销毁与对象池)
- [Prefab 覆盖与应用](#prefab-覆盖与应用)
- [预制体变体](#预制体变体)
- [嵌套预制体](#嵌套预制体)
- [完整示例](#完整示例)

---

## 什么是 Prefab

**Prefab（预制体）** 是 GameObject 的模板/蓝图：

- 在 Project 面板中存储为 `.prefab` 文件
- 可以在场景中重复使用（实例化）
- 修改 Prefab 会自动更新所有实例（除非实例有覆盖）
- 适合：敌人、子弹、金币、树木、UI 面板等重复出现的对象

```
Prefab（模板）──────→ 实例1（场景中）
    │
    ├──→ 实例2（场景中）
    └──→ 实例3（场景中）
         修改 Prefab → 所有实例同步更新
```

---

## 创建 Prefab

### 方法一：拖拽创建（最常用）

```
1. 在 Hierarchy 中搭建好一个完整的 GameObject（含所有组件和子物体）
2. 直接从 Hierarchy 拖到 Project 面板
3. 完成！现在它就是一个 Prefab 了
```

### 方法二：代码创建

```csharp
// Prefab 不能直接用代码在运行时创建资产文件
// 但可以在编辑器脚本中创建（见 Editor 目录下的示例）

#if UNITY_EDITOR
using UnityEditor;

public class PrefabCreator
{
    [MenuItem("Tools/Create Prefab From Selection")]
    static void CreatePrefab()
    {
        GameObject selected = Selection.activeGameObject;
        if (selected == null) return;

        string path = "Assets/Prefabs/" + selected.name + ".prefab";
        // 确保目录存在
        System.IO.Directory.CreateDirectory("Assets/Prefabs");
        PrefabUtility.SaveAsPrefabAsset(selected, path);
    }
}
#endif
```

---

## 实例化 Instantiate

在运行时生成 Prefab 的副本：

```csharp
using UnityEngine;

public class Spawner : MonoBehaviour
{
    [Header("预制体引用")]
    public GameObject enemyPrefab;      // 在 Inspector 中拖入
    public GameObject bulletPrefab;

    [Header("生成设置")]
    public Transform spawnPoint;         // 生成位置
    public int spawnCount = 5;

    void Start()
    {
        // 方法一：在指定位置实例化
        Instantiate(enemyPrefab, spawnPoint.position, Quaternion.identity);

        // 方法二：先实例化，再设置位置
        GameObject obj = Instantiate(enemyPrefab);
        obj.transform.position = new Vector3(1, 0, 3);

        // 方法三：实例化并设置父物体
        GameObject uiElement = Instantiate(enemyPrefab, transform);
    }

    void Update()
    {
        // 点击鼠标生成子弹
        if (Input.GetMouseButtonDown(0))
        {
            SpawnBullet();
        }
    }

    void SpawnBullet()
    {
        // 实例化并获取组件
        GameObject bullet = Instantiate(
            bulletPrefab,
            transform.position,
            transform.rotation
        );

        // 直接获取刚体并施加力
        Rigidbody rb = bullet.GetComponent<Rigidbody>();
        rb.AddForce(transform.forward * 1000f);
    }

    // 批量生成
    public void SpawnWave()
    {
        for (int i = 0; i < spawnCount; i++)
        {
            Vector3 randomPos = new Vector3(
                Random.Range(-10f, 10f),
                0,
                Random.Range(-10f, 10f)
            );
            Instantiate(enemyPrefab, randomPos, Quaternion.identity);
        }
    }
}
```

---

## 销毁与对象池

### Destroy 销毁

```csharp
// 立即销毁
Destroy(gameObject);

// 延迟销毁（比如播放完死亡动画后）
Destroy(gameObject, 2f);

// 销毁指定组件（不销毁整个物体）
Destroy(GetComponent<Rigidbody>());
```

### 对象池（Object Pooling）

频繁 `Instantiate` + `Destroy` 会造成性能问题。对象池复用已有对象：

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// 简易对象池
/// </summary>
public class SimpleObjectPool : MonoBehaviour
{
    public GameObject prefab;
    public int poolSize = 20;

    private Queue<GameObject> pool = new Queue<GameObject>();

    void Start()
    {
        // 预先创建对象
        for (int i = 0; i < poolSize; i++)
        {
            GameObject obj = Instantiate(prefab);
            obj.SetActive(false);
            pool.Enqueue(obj);
        }
    }

    // 从池中获取对象
    public GameObject Get()
    {
        if (pool.Count > 0)
        {
            GameObject obj = pool.Dequeue();
            obj.SetActive(true);
            return obj;
        }

        // 池不够用，临时创建
        return Instantiate(prefab);
    }

    // 归还对象到池中
    public void Return(GameObject obj)
    {
        obj.SetActive(false);
        pool.Enqueue(obj);
    }
}
```

---

## Prefab 覆盖与应用

在场景中修改了 Prefab 实例后，可以选择：

```csharp
// 覆盖类型（在 Inspector 面板中操作）：
// - Instance Overrides（实例覆盖）：只影响当前实例
// - 通过 "Apply" 按钮将修改应用到 Prefab 源文件（影响所有实例）
// - 通过 "Revert" 按钮放弃修改，恢复 Prefab 默认值

// 实际操作步骤：
// 1. 选中场景中的 Prefab 实例
// 2. Inspector 顶部会出现 Overrides 选项
// 3. "Apply All" → 把所有修改写回 Prefab
// 4. "Revert All" → 放弃所有修改
```

---

## 预制体变体

**Prefab Variant（预制体变体）** 基于另一个 Prefab，继承其基础但有自己独特的修改：

```
BaseEnemy（基础预制体）
  ├── RedEnemy（变体：红色材质 + 更多生命值）
  ├── BlueEnemy（变体：蓝色材质 + 更快速度）
  └── BossEnemy（变体：大型化 + 特殊攻击）
```

```
创建变体的方法：
1. 在 Project 中右键基础 Prefab
2. 选择 "Create → Prefab Variant"
3. 修改变体的独特属性
4. 基础 Prefab 更新时，变体自动继承变化
```

---

## 嵌套预制体

一个 Prefab 可以包含其他 Prefab 的实例：

```
坦克 Prefab
  ├── 炮塔 Prefab（子预制体）
  │     ├── 炮管 Prefab
  │     └── 机枪 Prefab
  └── 履带 Prefab（子预制体）
        ├── 左履带
        └── 右履带
```

```csharp
// 嵌套预制体的优势：
// - 模块化设计：炮塔可以用在不同坦克上
// - 分离更新：修改炮塔 Prefab，所有使用它的坦克自动更新
// - 团队协作：不同人负责不同模块

// 嵌套预制体注意事项：
// - 修改子预制体会影响所有包含它的父预制体
// - 可以在父级中覆盖子预制体的属性
```

---

## 完整示例

一个完整的敌人生成系统：

```csharp
using UnityEngine;

/// <summary>
/// 敌人波次生成器
/// 使用 Prefab + 对象池 + 协程
/// </summary>
public class EnemyWaveSpawner : MonoBehaviour
{
    [Header("预制体设置")]
    public GameObject[] enemyPrefabs;       // 多种敌人预制体
    public Transform[] spawnPoints;          // 多个生成点

    [Header("波次设置")]
    public float timeBetweenWaves = 10f;     // 波次间隔
    public int enemiesPerWave = 5;           // 每波敌人数量
    public float spawnInterval = 0.5f;       // 生成间隔

    private int currentWave = 0;
    private bool isSpawning = false;

    void Start()
    {
        // 开始第一波
        StartCoroutine(WaveRoutine());
    }

    System.Collections.IEnumerator WaveRoutine()
    {
        while (true)
        {
            currentWave++;
            Debug.Log($"第 {currentWave} 波来袭！");

            // 等待波次间隔
            yield return new WaitForSeconds(timeBetweenWaves);

            // 生成当前波次的敌人
            yield return StartCoroutine(SpawnWave());

            // 等待所有敌人被消灭（简化处理）
            yield return new WaitForSeconds(5f);
        }
    }

    System.Collections.IEnumerator SpawnWave()
    {
        isSpawning = true;

        for (int i = 0; i < enemiesPerWave; i++)
        {
            SpawnEnemy();
            yield return new WaitForSeconds(spawnInterval);
        }

        isSpawning = false;
    }

    void SpawnEnemy()
    {
        // 随机选择敌人类型
        int enemyIndex = Random.Range(0, enemyPrefabs.Length);

        // 随机选择生成点
        int spawnIndex = Random.Range(0, spawnPoints.Length);

        // 实例化敌人
        GameObject enemy = Instantiate(
            enemyPrefabs[enemyIndex],
            spawnPoints[spawnIndex].position,
            Quaternion.identity
        );

        // 随机缩放，增加视觉变化
        float scale = Random.Range(0.8f, 1.2f);
        enemy.transform.localScale = Vector3.one * scale;
    }
}
```

### 要点总结

| 操作 | 方法 |
|------|------|
| 创建 Prefab | 从 Hierarchy 拖到 Project |
| 实例化 | `Instantiate(prefab)` |
| 销毁 | `Destroy(gameObject)` |
| 修改 → 应用 | Inspector 中 "Apply All" |
| 创建变体 | 右键 → Create → Prefab Variant |
| 对象池 | 复用对象避免频繁创建销毁 |

---

> 📌 下一步：学习 [场景管理](../05-场景管理/README.md)
