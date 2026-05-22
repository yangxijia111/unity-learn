# VR 实战项目：射击游戏

## 📋 目录

- [项目概述](#项目概述)
- [项目架构](#项目架构)
- [武器系统](#武器系统)
- [弹药与射击](#弹药与射击)
- [靶子系统](#靶子系统)
- [音效与手柄反馈](#音效与手柄反馈)
- [完整项目结构](#完整项目结构)

---

## 项目概述

本项目是一个 VR 射击靶场游戏，适合学习 VR 开发的核心交互。玩家可以：
- 抓取武器并瞄准
- 射击不同类型的靶子
- 获得分数反馈
- 体验手柄震动和音效反馈

**技术栈：** Unity 2022 LTS + XR Interaction Toolkit + OpenXR

---

## 项目架构

```
VRShootingRange/
├── Scripts/
│   ├── Weapons/
│   │   ├── WeaponBase.cs          # 武器基类
│   │   ├── Pistol.cs              # 手枪
│   │   ├── Rifle.cs               # 步枪
│   │   └── WeaponMagazine.cs      # 弹夹系统
│   ├── Projectile/
│   │   ├── Bullet.cs              # 子弹
│   │   └── BulletTrail.cs         # 弹道轨迹
│   ├── Targets/
│   │   ├── TargetBase.cs          # 靶子基类
│   │   ├── StaticTarget.cs        # 静态靶
│   │   ├── MovingTarget.cs        # 移动靶
│   │   └── TargetSpawner.cs       # 靶子生成器
│   ├── Scoring/
│   │   ├── ScoreManager.cs        # 分数管理
│   │   └── ScoreDisplay.cs        # 分数显示
│   ├── Audio/
│   │   └── VRAudioManager.cs      # 音频管理
│   └── Haptics/
│       └── HapticFeedback.cs      # 手柄震动
├── Prefabs/
├── Audio/
└── Scenes/
    └── ShootingRange.unity
```

---

## 武器系统

### 武器基类 (WeaponBase.cs)

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 武器基类 - 所有武器的公共逻辑
/// 负责：抓取、射击、换弹、音效、震动
/// </summary>
public abstract class WeaponBase : XRGrabInteractable
{
    [Header("武器设置")]
    [Tooltip("武器名称")]
    public string weaponName = "默认武器";
    
    [Tooltip("弹夹容量")]
    public int magazineCapacity = 12;
    
    [Tooltip("射速（发/秒）")]
    public float fireRate = 5f;
    
    [Tooltip("射击力度")]
    public float shootForce = 50f;
    
    [Tooltip("后坐力强度")]
    public float recoilStrength = 0.5f;

    [Header("引用")]
    [Tooltip("枪口位置")]
    public Transform muzzlePoint;
    
    [Tooltip("子弹预制体")]
    public GameObject bulletPrefab;
    
    [Tooltip("射击音效")]
    public AudioClip shootSound;
    
    [Tooltip("换弹音效")]
    public AudioClip reloadSound;
    
    [Tooltip("空仓音效")]
    public AudioClip emptySound;

    // 内部状态
    protected int currentAmmo;
    protected float nextFireTime;
    protected bool isReloading;
    protected AudioSource audioSource;
    protected XRBaseController controller;

    protected override void Awake()
    {
        base.Awake();
        audioSource = GetComponent<AudioSource>();
        if (audioSource == null)
            audioSource = gameObject.AddComponent<AudioSource>();
        
        currentAmmo = magazineCapacity;
        
        // 监听扳机按下事件
        activated.AddListener(OnActivate);
        deactivated.AddListener(OnDeactivate);
    }

    /// <summary>
    /// 扳机按下 - 开始射击
    /// </summary>
    protected virtual void OnActivate(ActivateEventArgs args)
    {
        // 获取抓取者的手柄控制器
        var interactor = args.interactorObject as XRBaseControllerInteractor;
        if (interactor != null)
            controller = interactor.xrController;
        
        // 开始连续射击
        InvokeRepeating(nameof(Fire), 0f, 1f / fireRate);
    }

    /// <summary>
    /// 扳机松开 - 停止射击
    /// </summary>
    protected virtual void OnDeactivate(DeactivateEventArgs args)
    {
        CancelInvoke(nameof(Fire));
    }

    /// <summary>
    /// 射击方法 - 子类可重写
    /// </summary>
    protected virtual void Fire()
    {
        if (isReloading) return;
        
        // 检查弹药
        if (currentAmmo <= 0)
        {
            PlayEmptySound();
            return;
        }
        
        currentAmmo--;
        
        // 生成子弹
        SpawnBullet();
        
        // 播放音效
        PlayShootSound();
        
        // 手柄震动反馈
        ApplyHapticFeedback();
        
        // 后坐力效果
        ApplyRecoil();
        
        Debug.Log($"[{weaponName}] 射击! 剩余弹药: {currentAmmo}/{magazineCapacity}");
    }

    /// <summary>
    /// 生成子弹
    /// </summary>
    protected virtual void SpawnBullet()
    {
        if (bulletPrefab == null || muzzlePoint == null) return;
        
        GameObject bullet = Instantiate(
            bulletPrefab, 
            muzzlePoint.position, 
            muzzlePoint.rotation
        );
        
        Rigidbody rb = bullet.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.velocity = muzzlePoint.forward * shootForce;
        }
        
        // 5秒后自动销毁
        Destroy(bullet, 5f);
    }

    /// <summary>
    /// 换弹
    /// </summary>
    public virtual void Reload()
    {
        if (isReloading || currentAmmo == magazineCapacity) return;
        
        isReloading = true;
        
        if (reloadSound != null)
            audioSource.PlayOneShot(reloadSound);
        
        // 模拟换弹延时
        Invoke(nameof(FinishReload), 1.5f);
    }

    protected virtual void FinishReload()
    {
        currentAmmo = magazineCapacity;
        isReloading = false;
        Debug.Log($"[{weaponName}] 换弹完成! 弹药: {currentAmmo}/{magazineCapacity}");
    }

    /// <summary>
    /// 手柄震动反馈
    /// </summary>
    protected virtual void ApplyHapticFeedback()
    {
        if (controller != null)
        {
            // 发送脉冲震动，强度0.3，持续0.1秒
            controller.SendHapticImpulse(0.3f, 0.1f);
        }
    }

    /// <summary>
    /// 后坐力效果
    /// </summary>
    protected virtual void ApplyRecoil()
    {
        // 简单的后坐力：向后旋转武器
        transform.Rotate(-recoilStrength, 0, 0, Space.Self);
        
        // 可以用协程做平滑回弹
        StartCoroutine(SmoothRecoilRecovery());
    }

    System.Collections.IEnumerator SmoothRecoilRecovery()
    {
        yield return new WaitForSeconds(0.05f);
        
        float duration = 0.1f;
        float elapsed = 0f;
        Quaternion startRot = transform.localRotation;
        Quaternion endRot = Quaternion.identity; // 原始角度（需根据实际调整）
        
        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            transform.localRotation = Quaternion.Slerp(startRot, endRot, elapsed / duration);
            yield return null;
        }
    }

    protected void PlayShootSound()
    {
        if (shootSound != null)
            audioSource.PlayOneShot(shootSound);
    }

    protected void PlayEmptySound()
    {
        if (emptySound != null)
            audioSource.PlayOneShot(emptySound);
    }

    protected override void OnDestroy()
    {
        base.OnDestroy();
        activated.RemoveListener(OnActivate);
        deactivated.RemoveListener(OnDeactivate);
    }
}
```

### 手枪实现 (Pistol.cs)

```csharp
using UnityEngine;

/// <summary>
/// 手枪 - 半自动武器，每次扣扳机只射一发
/// </summary>
public class Pistol : WeaponBase
{
    [Header("手枪特有设置")]
    [Tooltip("精准度（散布角度）")]
    public float accuracy = 1f;
    
    private bool hasFired = false;

    protected override void OnActivate(ActivateEventArgs args)
    {
        // 手枪不使用 InvokeRepeating，而是单发
        base.OnActivate(args);
        CancelInvoke(nameof(Fire)); // 取消基类的连发
        
        if (!hasFired)
        {
            Fire();
            hasFired = true;
        }
    }

    protected override void OnDeactivate(DeactivateEventArgs args)
    {
        base.OnDeactivate(args);
        hasFired = false; // 松开扳机后可以再次射击
    }

    protected override void SpawnBullet()
    {
        if (bulletPrefab == null || muzzlePoint == null) return;
        
        // 添加精准度散布
        Quaternion spread = Quaternion.Euler(
            Random.Range(-accuracy, accuracy),
            Random.Range(-accuracy, accuracy),
            0
        );
        
        GameObject bullet = Instantiate(
            bulletPrefab,
            muzzlePoint.position,
            muzzlePoint.rotation * spread
        );
        
        Rigidbody rb = bullet.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.velocity = (muzzlePoint.rotation * spread) * Vector3.forward * shootForce;
        }
        
        Destroy(bullet, 5f);
    }
}
```

### 步枪实现 (Rifle.cs)

```csharp
using UnityEngine;

/// <summary>
/// 步枪 - 全自动武器，按住扳机连续射击
/// </summary>
public class Rifle : WeaponBase
{
    [Header("步枪特有设置")]
    [Tooltip("基础散布")]
    public float baseSpread = 2f;
    
    [Tooltip("最大散布（持续射击时增大）")]
    public float maxSpread = 8f;
    
    [Tooltip("散布增长速度")]
    public float spreadIncreaseRate = 0.5f;
    
    private float currentSpread;
    private bool isFiring;

    protected override void OnActivate(ActivateEventArgs args)
    {
        base.OnActivate(args);
        isFiring = true;
        currentSpread = baseSpread;
    }

    protected override void OnDeactivate(DeactivateEventArgs args)
    {
        base.OnDeactivate(args);
        isFiring = false;
    }

    protected override void Fire()
    {
        // 持续射击时散布增大
        if (isFiring)
        {
            currentSpread = Mathf.Min(currentSpread + spreadIncreaseRate, maxSpread);
        }
        
        base.Fire();
    }

    void Update()
    {
        // 不射击时散布恢复
        if (!isFiring && currentSpread > baseSpread)
        {
            currentSpread = Mathf.Max(currentSpread - spreadIncreaseRate * 2, baseSpread);
        }
    }

    protected override void SpawnBullet()
    {
        if (bulletPrefab == null || muzzlePoint == null) return;
        
        Quaternion spread = Quaternion.Euler(
            Random.Range(-currentSpread, currentSpread),
            Random.Range(-currentSpread, currentSpread),
            0
        );
        
        GameObject bullet = Instantiate(
            bulletPrefab,
            muzzlePoint.position,
            muzzlePoint.rotation * spread
        );
        
        Rigidbody rb = bullet.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.velocity = (muzzlePoint.rotation * spread) * Vector3.forward * shootForce;
        }
        
        Destroy(bullet, 3f);
    }
}
```

---

## 弹药与射击

### 子弹 (Bullet.cs)

```csharp
using UnityEngine;

/// <summary>
/// 子弹 - 物理弹丸方式
/// 碰撞到目标时造成伤害并触发反馈
/// </summary>
public class Bullet : MonoBehaviour
{
    [Tooltip("伤害值")]
    public float damage = 25f;
    
    [Tooltip("命中特效")]
    public GameObject hitEffectPrefab;
    
    [Tooltip("命中音效")]
    public AudioClip hitSound;

    void OnCollisionEnter(Collision collision)
    {
        // 检查是否击中靶子
        TargetBase target = collision.gameObject.GetComponent<TargetBase>();
        if (target != null)
        {
            // 获取碰撞点
            ContactPoint contact = collision.GetContact(0);
            
            // 造成伤害
            target.TakeDamage(damage, contact.point);
            
            // 生成命中特效
            if (hitEffectPrefab != null)
            {
                GameObject effect = Instantiate(
                    hitEffectPrefab,
                    contact.point,
                    Quaternion.LookRotation(contact.normal)
                );
                Destroy(effect, 2f);
            }
            
            // 播放命中音效
            if (hitSound != null)
            {
                AudioSource.PlayClipAtPoint(hitSound, contact.point);
            }
        }
        
        // 子弹碰到任何东西都销毁
        Destroy(gameObject);
    }
}
```

### 弹道轨迹 (BulletTrail.cs)

```csharp
using UnityEngine;

/// <summary>
/// 弹道轨迹 - 射线检测方式的可视化轨迹
/// 适用于不需要物理碰撞的快速射击
/// </summary>
public class BulletTrail : MonoBehaviour
{
    [Tooltip("轨迹宽度")]
    public float trailWidth = 0.02f;
    
    [Tooltip("轨迹持续时间")]
    public float trailDuration = 0.1f;
    
    [Tooltip("最大射程")]
    public float maxDistance = 100f;
    
    LineRenderer lineRenderer;

    void Awake()
    {
        lineRenderer = gameObject.AddComponent<LineRenderer>();
        lineRenderer.startWidth = trailWidth;
        lineRenderer.endWidth = trailWidth * 0.5f;
        lineRenderer.material = new Material(Shader.Find("Sprites/Default"));
        lineRenderer.startColor = Color.yellow;
        lineRenderer.endColor = new Color(1, 1, 0, 0);
    }

    /// <summary>
    /// 显示从枪口到命中点的弹道轨迹
    /// </summary>
    public void ShowTrail(Vector3 startPos, Vector3 endPos)
    {
        lineRenderer.SetPosition(0, startPos);
        lineRenderer.SetPosition(1, endPos);
        
        // 延时隐藏
        Destroy(gameObject, trailDuration);
    }
}
```

---

## 靶子系统

### 靶子基类 (TargetBase.cs)

```csharp
using UnityEngine;
using UnityEngine.Events;

/// <summary>
/// 靶子基类 - 所有靶子的公共逻辑
/// </summary>
public abstract class TargetBase : MonoBehaviour
{
    [Header("靶子设置")]
    [Tooltip("靶子生命值")]
    public float maxHealth = 100f;
    
    [Tooltip("命中不同区域的分数倍率")]
    public float[] ringMultipliers = { 10f, 5f, 3f, 1f };
    
    [Tooltip("被击中事件")]
    public UnityEvent<float, Vector3> onHit;
    
    [Tooltip("被摧毁事件")]
    public UnityEvent onDestroyed;

    protected float currentHealth;
    protected bool isAlive;

    protected virtual void Start()
    {
        currentHealth = maxHealth;
        isAlive = true;
    }

    /// <summary>
    /// 受到伤害
    /// </summary>
    /// <param name="damage">伤害值</param>
    /// <param name="hitPoint">命中世界坐标</param>
    public virtual void TakeDamage(float damage, Vector3 hitPoint)
    {
        if (!isAlive) return;
        
        currentHealth -= damage;
        
        // 计算得分（根据命中位置）
        float score = CalculateScore(hitPoint);
        
        // 触发命中事件
        onHit?.Invoke(score, hitPoint);
        
        // 通知分数管理器
        ScoreManager.Instance?.AddScore(score);
        
        Debug.Log($"靶子被击中! 伤害:{damage} 得分:{score} 剩余血量:{currentHealth}");
        
        // 检查是否被摧毁
        if (currentHealth <= 0)
        {
            DestroyTarget();
        }
    }

    /// <summary>
    /// 根据命中位置计算得分
    /// 越靠近中心分数越高
    /// </summary>
    protected virtual float CalculateScore(Vector3 hitPoint)
    {
        float distance = Vector3.Distance(hitPoint, transform.position);
        float maxRadius = 0.5f; // 靶子半径
        
        // 根据距离判断命中区域
        float normalizedDist = distance / maxRadius;
        
        if (normalizedDist < 0.25f) return 100 * ringMultipliers[0]; // 中心
        if (normalizedDist < 0.5f) return 100 * ringMultipliers[1];  // 内环
        if (normalizedDist < 0.75f) return 100 * ringMultipliers[2]; // 中环
        return 100 * ringMultipliers[3]; // 外环
    }

    /// <summary>
    /// 销毁靶子
    /// </summary>
    protected virtual void DestroyTarget()
    {
        isAlive = false;
        onDestroyed?.Invoke();
        
        // 播放销毁动画后销毁
        // 这里简单直接销毁
        Destroy(gameObject, 0.5f);
    }

    /// <summary>
    /// 重置靶子
    /// </summary>
    public virtual void ResetTarget()
    {
        currentHealth = maxHealth;
        isAlive = true;
    }
}
```

### 静态靶 (StaticTarget.cs)

```csharp
using UnityEngine;

/// <summary>
/// 静态靶 - 固定不动的靶子
/// </summary>
public class StaticTarget : TargetBase
{
    [Header("静态靶设置")]
    [Tooltip("是否可重生")]
    public bool canRespawn = true;
    
    [Tooltip("重生延迟（秒）")]
    public float respawnDelay = 3f;

    protected override void DestroyTarget()
    {
        base.DestroyTarget();
        
        if (canRespawn)
        {
            Invoke(nameof(Respawn), respawnDelay);
        }
    }

    void Respawn()
    {
        ResetTarget();
        
        // 恢复外观
        MeshRenderer renderer = GetComponent<MeshRenderer>();
        if (renderer != null)
            renderer.enabled = true;
        
        Debug.Log("静态靶已重生!");
    }
}
```

### 移动靶 (MovingTarget.cs)

```csharp
using UnityEngine;

/// <summary>
/// 移动靶 - 沿路径移动的靶子，增加射击难度
/// </summary>
public class MovingTarget : TargetBase
{
    [Header("移动设置")]
    [Tooltip("移动路径点")]
    public Transform[] waypoints;
    
    [Tooltip("移动速度")]
    public float moveSpeed = 2f;
    
    [Tooltip("到达路径点后的等待时间")]
    public float waitTime = 1f;

    private int currentWaypointIndex = 0;
    private float waitTimer = 0f;
    private bool isWaiting = false;

    protected override void Start()
    {
        base.Start();
        
        if (waypoints.Length > 0)
        {
            transform.position = waypoints[0].position;
        }
    }

    void Update()
    {
        if (!isAlive || waypoints.Length == 0) return;
        
        if (isWaiting)
        {
            waitTimer -= Time.deltaTime;
            if (waitTimer <= 0)
            {
                isWaiting = false;
                currentWaypointIndex = (currentWaypointIndex + 1) % waypoints.Length;
            }
            return;
        }
        
        // 向下一个路径点移动
        Transform targetWaypoint = waypoints[currentWaypointIndex];
        transform.position = Vector3.MoveTowards(
            transform.position,
            targetWaypoint.position,
            moveSpeed * Time.deltaTime
        );
        
        // 面向移动方向
        Vector3 direction = targetWaypoint.position - transform.position;
        if (direction.magnitude > 0.01f)
        {
            transform.forward = direction;
        }
        
        // 到达路径点
        if (Vector3.Distance(transform.position, targetWaypoint.position) < 0.01f)
        {
            isWaiting = true;
            waitTimer = waitTime;
        }
    }
}
```

### 靶子生成器 (TargetSpawner.cs)

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// 靶子生成器 - 控制靶子的生成波次
/// </summary>
public class TargetSpawner : MonoBehaviour
{
    [Header("生成设置")]
    [Tooltip("靶子预制体列表")]
    public GameObject[] targetPrefabs;
    
    [Tooltip("生成点")]
    public Transform[] spawnPoints;
    
    [Tooltip("每波生成数量")]
    public int targetsPerWave = 5;
    
    [Tooltip("波次间隔")]
    public float waveInterval = 10f;
    
    [Tooltip("是否自动生成下一波")]
    public bool autoNextWave = true;

    private List<GameObject> activeTargets = new List<GameObject>();
    private int currentWave = 0;
    private float waveTimer;

    void Start()
    {
        StartNextWave();
    }

    void Update()
    {
        // 清理已销毁的靶子
        activeTargets.RemoveAll(t => t == null);
        
        // 检查是否所有靶子都被消灭
        if (activeTargets.Count == 0 && autoNextWave)
        {
            waveTimer += Time.deltaTime;
            if (waveTimer >= waveInterval)
            {
                StartNextWave();
                waveTimer = 0f;
            }
        }
    }

    /// <summary>
    /// 开始下一波
    /// </summary>
    public void StartNextWave()
    {
        currentWave++;
        Debug.Log($"=== 第 {currentWave} 波 ===");
        
        int spawnCount = Mathf.Min(targetsPerWave + currentWave, spawnPoints.Length);
        
        for (int i = 0; i < spawnCount; i++)
        {
            SpawnTarget(i);
        }
    }

    /// <summary>
    /// 在指定生成点生成一个随机靶子
    /// </summary>
    void SpawnTarget(int spawnIndex)
    {
        if (targetPrefabs.Length == 0) return;
        
        GameObject prefab = targetPrefabs[Random.Range(0, targetPrefabs.Length)];
        Transform spawnPoint = spawnPoints[spawnIndex % spawnPoints.Length];
        
        GameObject target = Instantiate(prefab, spawnPoint.position, spawnPoint.rotation);
        
        // 注册销毁事件，从活跃列表移除
        TargetBase targetBase = target.GetComponent<TargetBase>();
        if (targetBase != null)
        {
            targetBase.onDestroyed.AddListener(() => activeTargets.Remove(target));
        }
        
        activeTargets.Add(target);
    }
}
```

---

## 音效与手柄反馈

### 手柄震动反馈 (HapticFeedback.cs)

```csharp
using UnityEngine;
using UnityEngine.XR.Interaction.Toolkit;

/// <summary>
/// 手柄震动反馈工具类
/// 提供各种预设的震动效果
/// </summary>
public static class HapticFeedback
{
    /// <summary>
    /// 轻微反馈 - UI交互、悬停
    /// </summary>
    public static void Light(XRBaseController controller)
    {
        controller.SendHapticImpulse(0.15f, 0.05f);
    }

    /// <summary>
    /// 中等反馈 - 抓取、碰撞
    /// </summary>
    public static void Medium(XRBaseController controller)
    {
        controller.SendHapticImpulse(0.4f, 0.1f);
    }

    /// <summary>
    /// 强烈反馈 - 射击、爆炸
    /// </summary>
    public static void Strong(XRBaseController controller)
    {
        controller.SendHapticImpulse(0.8f, 0.15f);
    }

    /// <summary>
    /// 射击反馈 - 模拟扳机扣下的感觉
    /// </summary>
    public static void Shoot(XRBaseController controller)
    {
        controller.SendHapticImpulse(0.6f, 0.08f);
    }

    /// <summary>
    /// 命中反馈 - 子弹命中目标
    /// </summary>
    public static void Hit(XRBaseController controller)
    {
        controller.SendHapticImpulse(0.3f, 0.12f);
    }

    /// <summary>
    /// 自定义震动
    /// </summary>
    /// <param name="amplitude">振幅 0-1</param>
    /// <param name="duration">持续时间（秒）</param>
    public static void Custom(XRBaseController controller, float amplitude, float duration)
    {
        controller.SendHapticImpulse(amplitude, duration);
    }
}
```

### VR 音频管理器 (VRAudioManager.cs)

```csharp
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// VR 音频管理器
/// 管理3D空间音效、背景音乐、音效池
/// </summary>
public class VRAudioManager : MonoBehaviour
{
    public static VRAudioManager Instance { get; private set; }

    [Header("音效库")]
    public AudioClip shootSound;
    public AudioClip reloadSound;
    public AudioClip emptyMagSound;
    public AudioClip hitSound;
    public AudioClip targetDestroySound;
    public AudioClip bgm;

    [Header("设置")]
    [Tooltip("音效池大小")]
    public int poolSize = 10;
    
    [Range(0, 1)]
    public float masterVolume = 1f;
    
    [Range(0, 1)]
    public float sfxVolume = 0.8f;
    
    [Range(0, 1)]
    public float bgmVolume = 0.5f;

    private AudioSource bgmSource;
    private Queue<AudioSource> sfxPool = new Queue<AudioSource>();

    void Awake()
    {
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject);
            return;
        }
        Instance = this;
        
        // 初始化BGM播放器
        bgmSource = gameObject.AddComponent<AudioSource>();
        bgmSource.loop = true;
        bgmSource.spatialBlend = 0f; // 2D音效
        bgmSource.volume = bgmVolume * masterVolume;
        
        // 初始化音效池
        for (int i = 0; i < poolSize; i++)
        {
            GameObject sfxObj = new GameObject($"SFX_{i}");
            sfxObj.transform.SetParent(transform);
            AudioSource source = sfxObj.AddComponent<AudioSource>();
            source.spatialBlend = 1f; // 3D音效
            sfxPool.Enqueue(source);
        }
    }

    void Start()
    {
        // 播放背景音乐
        if (bgm != null)
        {
            bgmSource.clip = bgm;
            bgmSource.Play();
        }
    }

    /// <summary>
    /// 播放3D音效（从音效池取）
    /// </summary>
    public void Play3DSound(AudioClip clip, Vector3 position, float volume = 1f)
    {
        if (clip == null || sfxPool.Count == 0) return;
        
        AudioSource source = sfxPool.Dequeue();
        source.transform.position = position;
        source.clip = clip;
        source.volume = volume * sfxVolume * masterVolume;
        source.Play();
        
        // 播放完毕后归还音效池
        StartCoroutine(ReturnToPool(source, clip.length));
    }

    /// <summary>
    /// 播放射击音效
    /// </summary>
    public void PlayShootSound(Vector3 position)
    {
        Play3DSound(shootSound, position, 0.8f);
    }

    /// <summary>
    /// 播放命中音效
    /// </summary>
    public void PlayHitSound(Vector3 position)
    {
        Play3DSound(hitSound, position, 0.6f);
    }

    System.Collections.IEnumerator ReturnToPool(AudioSource source, float delay)
    {
        yield return new WaitForSeconds(delay + 0.1f);
        sfxPool.Enqueue(source);
    }
}
```

---

## 完整项目结构

### 使用说明

1. **创建 Unity 项目**：Unity 2022 LTS + 3D (URP) 模板
2. **安装依赖**：
   - XR Interaction Toolkit (2.3+)
   - XR Plugin Management
   - OpenXR Plugin
3. **导入代码**：将 Scripts 文件夹放入 Assets
4. **搭建场景**：
   - 添加 XR Origin
   - 创建射击场环境
   - 放置靶子和生成点
   - 配置武器预制体
5. **配置 Input Actions**：参考 XRI 默认配置
6. **测试运行**

### 扩展建议

- **增加武器类型**：霰弹枪、狙击枪、弓箭
- **增加敌人AI**：简单的行为树敌人
- **增加关卡系统**：不同难度的靶场
- **增加排行榜**：本地/在线分数排行
- **多人模式**：合作或对战射击

---

*更多 VR 开发内容请参考本目录其他章节。*
