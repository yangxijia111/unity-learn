# 对象池模式

> 频繁创建和销毁对象会让 GC 爆炸、游戏卡顿。对象池让你"借"和"还"对象，而不是一直 new 和 Destroy。

## 📌 你会学到什么

- 对象池原理（为什么要用）
- 基础实现（手写一个）
- 泛型对象池（通用版本）
- Unity 官方 ObjectPool API
- 实战应用：子弹、敌人、特效

---

## 1. 为什么需要对象池

**问题：** 射击游戏中，一秒可能打出几十上百发子弹。每发子弹 `Instantiate()` + `Destroy()` 会产生大量 GC 垃圾，导致卡顿。

**解决：** 提前创建一批对象存起来，需要时"拿出来用"，用完"放回去"，而不是销毁。

```
普通流程：  new → 使用 → 销毁 → new → 使用 → 销毁  （GC压力大）
对象池流程：取出 → 使用 → 放回 → 取出 → 使用 → 放回  （几乎无GC）
```

---

## 2. 基础对象池实现

```csharp
using System.Collections.Generic;
using UnityEngine;

public class SimpleObjectPool : MonoBehaviour
{
    [SerializeField] private GameObject prefab;   // 要池化的预制体
    [SerializeField] private int initialSize = 20; // 初始数量

    private Queue<GameObject> pool = new Queue<GameObject>();

    void Start()
    {
        // 预热：提前创建一批对象
        for (int i = 0; i < initialSize; i++)
        {
            GameObject obj = CreateNewObject();
            ReturnToPool(obj);
        }
    }

    // 从池中获取对象
    public GameObject GetFromPool(Vector3 position, Quaternion rotation)
    {
        GameObject obj;

        if (pool.Count > 0)
        {
            // 池中有，直接用
            obj = pool.Dequeue();
        }
        else
        {
            // 池空了，创建新的
            obj = CreateNewObject();
        }

        obj.transform.position = position;
        obj.transform.rotation = rotation;
        obj.SetActive(true);

        return obj;
    }

    // 归还对象到池中
    public void ReturnToPool(GameObject obj)
    {
        obj.SetActive(false);
        pool.Enqueue(obj);
    }

    private GameObject CreateNewObject()
    {
        GameObject obj = Instantiate(prefab, transform);
        obj.SetActive(false);

        // 让对象知道自己属于哪个池
        var returnable = obj.AddComponent<PoolableObject>();
        returnable.pool = this;

        return obj;
    }
}

// 挂在池化对象上，负责自动归还
public class PoolableObject : MonoBehaviour
{
    [HideInInspector] public SimpleObjectPool pool;
}
```

---

## 3. 实战：子弹对象池

```csharp
using System.Collections;
using UnityEngine;

// 子弹脚本
public class Bullet : MonoBehaviour
{
    [SerializeField] private float speed = 20f;
    [SerializeField] private float lifeTime = 3f; // 子弹存活时间

    private Rigidbody rb;
    private SimpleObjectPool pool;
    private float timer;

    void Awake()
    {
        rb = GetComponent<Rigidbody>();
    }

    void OnEnable()
    {
        timer = 0;
        // 激活时给一个向前的速度
        rb.linearVelocity = transform.forward * speed;
    }

    void Update()
    {
        // 到时间自动回池
        timer += Time.deltaTime;
        if (timer >= lifeTime)
        {
            ReturnToPool();
        }
    }

    void OnCollisionEnter(Collision collision)
    {
        // 碰到物体也回池
        Debug.Log($"子弹击中：{collision.gameObject.name}");
        ReturnToPool();
    }

    public void SetPool(SimpleObjectPool pool)
    {
        this.pool = pool;
    }

    private void ReturnToPool()
    {
        rb.linearVelocity = Vector3.zero; // 重置速度
        pool.ReturnToPool(gameObject);
    }
}

// 枪的射击脚本
public class Gun : MonoBehaviour
{
    [SerializeField] private SimpleObjectPool bulletPool;
    [SerializeField] private Transform firePoint;
    [SerializeField] private float fireRate = 0.1f; // 每秒10发

    private float nextFireTime;

    void Update()
    {
        if (Input.GetMouseButton(0) && Time.time >= nextFireTime)
        {
            nextFireTime = Time.time + fireRate;
            Fire();
        }
    }

    void Fire()
    {
        // 从池中取出子弹，而不是 Instantiate
        GameObject bullet = bulletPool.GetFromPool(firePoint.position, firePoint.rotation);

        // 设置子弹的池引用
        Bullet bulletScript = bullet.GetComponent<Bullet>();
        bulletScript.SetPool(bulletPool);
    }
}
```

---

## 4. 泛型对象池（通用版本）

```csharp
using System;
using System.Collections.Generic;

// 泛型对象池，不限于 GameObject
public class GenericPool<T> where T : class
{
    private Queue<T> pool = new Queue<T>();
    private Func<T> createFunc;     // 创建新对象的委托
    private Action<T> onGet;        // 取出时的回调
    private Action<T> onReturn;     // 归还时的回调

    public int Count => pool.Count;

    public GenericPool(Func<T> createFunc, Action<T> onGet = null, Action<T> onReturn = null, int initialSize = 10)
    {
        this.createFunc = createFunc;
        this.onGet = onGet;
        this.onReturn = onReturn;

        // 预热
        for (int i = 0; i < initialSize; i++)
        {
            pool.Enqueue(createFunc());
        }
    }

    public T Get()
    {
        T item = pool.Count > 0 ? pool.Dequeue() : createFunc();
        onGet?.Invoke(item);
        return item;
    }

    public void Return(T item)
    {
        onReturn?.Invoke(item);
        pool.Enqueue(item);
    }
}

// 使用示例：池化普通的 C# 对象
public class DamageText
{
    public string text;
    public float duration;
}

// 在代码中使用
// var textPool = new GenericPool<DamageText>(
//     createFunc: () => new DamageText(),
//     onReturn: (t) => { t.text = ""; t.duration = 0; } // 归还时重置
// );
```

---

## 5. Unity 官方 ObjectPool（Unity 2021+）

```csharp
using UnityEngine;
using UnityEngine.Pool;

public class OfficialPoolExample : MonoBehaviour
{
    [SerializeField] private GameObject bulletPrefab;

    private ObjectPool<GameObject> bulletPool;

    void Awake()
    {
        bulletPool = new ObjectPool<GameObject>(
            createFunc: () => Instantiate(bulletPrefab),           // 创建
            actionOnGet: (obj) => obj.SetActive(true),             // 取出时
            actionOnRelease: (obj) => obj.SetActive(false),        // 归还时
            actionOnDestroy: (obj) => Destroy(obj),                // 销毁时
            defaultCapacity: 20,                                   // 默认容量
            maxSize: 100                                           // 最大容量（超出会销毁多余对象）
        );
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            // 从池中取出
            GameObject bullet = bulletPool.Get();
            bullet.transform.position = transform.position;
        }
    }

    // 假设子弹脚本触发归还
    public void OnBulletReturned(GameObject bullet)
    {
        bulletPool.Release(bullet); // 归还到池
    }
}
```

---

## 6. 实战应用汇总

### 6.1 敌人生成池

```csharp
public class EnemySpawner : MonoBehaviour
{
    [SerializeField] private SimpleObjectPool enemyPool;
    [SerializeField] private float spawnInterval = 2f;
    [SerializeField] private Transform[] spawnPoints;

    void Start()
    {
        InvokeRepeating(nameof(SpawnEnemy), 1f, spawnInterval);
    }

    void SpawnEnemy()
    {
        Transform point = spawnPoints[UnityEngine.Random.Range(0, spawnPoints.Length)];
        enemyPool.GetFromPool(point.position, Quaternion.identity);
    }
}
```

### 6.2 特效池

```csharp
public class VFXManager : MonoBehaviour
{
    // 按特效名称缓存多个池
    private Dictionary<string, SimpleObjectPool> vfxPools = new Dictionary<string, SimpleObjectPool>();

    public void PlayVFX(string vfxName, Vector3 position)
    {
        if (vfxPools.TryGetValue(vfxName, out var pool))
        {
            GameObject vfx = pool.GetFromPool(position, Quaternion.identity);
            // 特效播放完自动回池（通过 ParticleSystem 的 Stop Action 或协程）
        }
    }
}
```

---

## 📊 对象池 vs 直接 Instantiate/Destroy

| 对比项 | Instantiate/Destroy | 对象池 |
|--------|-------------------|--------|
| 内存 | 反复分配释放 | 预分配，复用 |
| GC 压力 | 大（每帧可能触发） | 极小 |
| 首次加载 | 快（按需创建） | 慢（预热需要时间） |
| 代码复杂度 | 简单 | 稍复杂 |
| 适用场景 | 少量、低频创建 | 大量、高频创建 |

---

## 🎯 使用建议

1. **子弹、敌人、特效** → 必须用对象池
2. **UI 元素**（如伤害飘字）→ 推荐用池
3. **一次性场景物件**（如通关门）→ 不需要池
4. **池的大小要合理** → 根据游戏最高并发量设定
5. **记得重置状态** → 归还时清空速度、血量等数据

---

> 💡 **一句话总结：** 对象池 = "东西反复用，别一直 new 和 destroy"。射击游戏、弹幕游戏、粒子特效多的项目，对象池是必备技能。
