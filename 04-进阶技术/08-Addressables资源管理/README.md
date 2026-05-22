# Addressables 资源管理

> 传统方式打 AssetBundle 太麻烦？Addressables 是 Unity 官方的资源管理方案，简化了打包、加载和热更新流程。

## 📌 你会学到什么

- Addressables 系统概述
- 资源打包（标记与分组）
- 异步加载（核心 API）
- 依赖管理（自动处理依赖关系）
- 热更新思路
- CDN 分发

---

## 1. Addressables 系统概述

**Addressables 是什么：** Unity 官方的资源管理框架，基于 Address（地址）来加载资源，不用关心文件路径和依赖关系。

**解决的问题：**
- 不用手动管理 AssetBundle 依赖
- 异步加载，不会卡主线程
- 支持本地和远程资源
- 热更新友好

### 安装

`Window → Package Manager → Unity Registry → 搜索 "Addressables" → Install`

---

## 2. 资源打包

### 2.1 标记资源

1. 选中资源（预制体、纹理等）
2. Inspector 中勾选 **Addressable**
3. 设置地址名（默认是文件路径，可以改成好记的名字）

```
示例地址：
- Prefabs/Player       → 玩家预制体
- Textures/Environment → 环境贴图
- Audio/BGM/Battle     → 战斗背景音乐
```

### 2.2 分组管理

`Window → Asset Management → Addressables → Groups`

- 默认所有资源在一个组里
- 可以创建多个组，按需加载
- 组可以设置为 **本地** 或 **远程**

```
分组示例：
├── Local（本地，随包发布）
│   ├── 核心UI
│   ├── 基础角色
│   └── 基础场景
├── Remote（远程，可热更新）
│   ├── 活动资源
│   ├── 新角色皮肤
│   └── DLC内容
```

### 2.3 构建

```
Addressables Groups 窗口 → Build → New Build → Default Build Script
```

构建后会在 `Assets/StreamingAssets/aa` 生成本地资源，远程资源在 `ServerData` 目录。

---

## 3. 异步加载（核心 API）

### 3.1 基本加载

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class LoadExample : MonoBehaviour
{
    // 方式 1：通过地址名加载
    async void LoadByAddress()
    {
        // 异步加载预制体
        AsyncOperationHandle<GameObject> handle =
            Addressables.LoadAssetAsync<GameObject>("Prefabs/Player");

        await handle.Task; // 等待加载完成

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            GameObject player = handle.Result;
            Instantiate(player);
        }

        // 用完记得释放！
        Addressables.Release(handle);
    }

    // 方式 2：通过 AssetReference（推荐，在 Inspector 中拖拽）
    [SerializeField] private AssetReference playerPrefab;

    async void LoadByReference()
    {
        AsyncOperationHandle<GameObject> handle = playerPrefab.LoadAssetAsync<GameObject>();
        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            GameObject player = Instantiate(handle.Result);
        }
    }

    void OnDestroy()
    {
        // 释放资源
        playerPrefab.ReleaseAsset();
    }
}
```

### 3.2 批量加载

```csharp
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class BatchLoadExample : MonoBehaviour
{
    // 按标签批量加载
    async void LoadAllEnemies()
    {
        // 加载所有标记了 "Enemy" 标签的资源
        AsyncOperationHandle<IList<GameObject>> handle =
            Addressables.LoadAssetsAsync<GameObject>("Enemy", (obj) =>
            {
                // 每加载一个就执行的回调
                Debug.Log($"加载了敌人资源：{obj.name}");
            });

        await handle.Task;

        if (handle.Status == AsyncOperationStatus.Succeeded)
        {
            foreach (var enemy in handle.Result)
            {
                Instantiate(enemy);
            }
        }

        Addressables.Release(handle);
    }

    // 按地址列表加载
    async void LoadMultipleAssets()
    {
        var addresses = new List<string>
        {
            "Prefabs/Enemy/Goblin",
            "Prefabs/Enemy/Dragon",
            "Prefabs/Enemy/Skeleton"
        };

        AsyncOperationHandle<IList<GameObject>> handle =
            Addressables.LoadAssetsAsync<GameObject>(addresses, null);

        await handle.Task;

        foreach (var enemy in handle.Result)
        {
            Instantiate(enemy);
        }

        Addressables.Release(handle);
    }
}
```

### 3.3 场景加载

```csharp
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.ResourceProviders;
using UnityEngine.SceneManagement;

public class SceneLoader : MonoBehaviour
{
    // 加载 Addressable 场景
    async void LoadGameScene()
    {
        AsyncOperationHandle<SceneInstance> handle =
            Addressables.LoadSceneAsync("Scenes/GameLevel1", LoadSceneMode.Single);

        await handle.Task;
        Debug.Log("场景加载完成！");
    }

    // 叠加加载场景（不卸载当前场景）
    async void LoadUIOverlay()
    {
        await Addressables.LoadSceneAsync("Scenes/HUD", LoadSceneMode.Additive).Task;
    }
}
```

---

## 4. 依赖管理

**Addressables 自动处理依赖关系！**

比如一个预制体引用了材质 A，材质 A 引用了贴图 B。当你加载预制体时，Addressables 自动把材质 A 和贴图 B 一起加载，不用你手动管。

### 查看依赖

`Addressables Groups 窗口 → 右键一个资源 → View Dependencies`

### 最佳实践

```
✅ 推荐做法：
- 把共享资源（材质、Shader）放到独立的组
- 预制体和它独有的贴图放一组
- 依赖关系越简单越好

❌ 避免：
- 一个组太大（加载太慢）
- 资源之间循环依赖
- 把场景直接打包到组里（应该用场景的 Addressable 设置）
```

---

## 5. 热更新思路

**热更新 = 不重新发包，就能更新游戏内容。**

### 基本流程

```
1. 构建 Addressables 资源
2. 上传远程资源到 CDN
3. 游戏启动时检查更新
4. 发现有新版本 → 下载 → 应用
5. 加载时优先用远程资源
```

### 检查更新代码

```csharp
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class HotUpdateManager : MonoBehaviour
{
    // 检查并下载更新
    public async void CheckForUpdates()
    {
        // 1. 检查目录更新
        AsyncOperationHandle catalogHandle = Addressables.CheckForCatalogUpdates(false);
        await catalogHandle.Task;

        if (catalogHandle.Status != AsyncOperationStatus.Succeeded)
        {
            Debug.LogError("检查更新失败");
            return;
        }

        var catalogs = catalogHandle.Result as List<string>;
        if (catalogs == null || catalogs.Count == 0)
        {
            Debug.Log("没有新更新");
            return;
        }

        // 2. 更新目录
        AsyncOperationHandle updateHandle = Addressables.UpdateCatalogs(catalogs, false);
        await updateHandle.Task;

        // 3. 获取需要下载的大小
        AsyncOperationHandle<long> sizeHandle = Addressables.GetDownloadSizeAsync("default");
        await sizeHandle.Task;

        long downloadSize = sizeHandle.Result;
        if (downloadSize > 0)
        {
            Debug.Log($"需要下载：{downloadSize / 1024f / 1024f:F2} MB");

            // 4. 下载资源
            AsyncOperationHandle downloadHandle =
                Addressables.DownloadDependenciesAsync("default", false);

            // 监听下载进度
            while (!downloadHandle.IsDone)
            {
                float percent = downloadHandle.PercentComplete;
                Debug.Log($"下载进度：{percent:P0}");
                await System.Threading.Tasks.Task.Delay(100);
            }

            Debug.Log("下载完成！");
        }

        // 5. 释放句柄
        Addressables.Release(catalogHandle);
        Addressables.Release(updateHandle);
        Addressables.Release(sizeHandle);
    }
}
```

---

## 6. CDN 分发

### 什么是 CDN

**CDN（Content Delivery Network）= 内容分发网络**。把资源放到离玩家最近的服务器，下载更快。

### 配置远程路径

`Addressables Groups → 点击组 → Inspector → Build & Load Paths`

```
Build Path:  RemoteBuildPath    （构建到本地服务器目录）
Load Path:   RemoteLoadPath     （运行时从这里下载）

RemoteLoadPath 示例：https://your-game.com/AssetBundles/StandaloneWindows64/
```

### 常用 CDN 服务

| 服务 | 特点 |
|------|------|
| 阿里云 OSS + CDN | 国内速度快，按量计费 |
| 腾讯云 COS + CDN | 腾讯系游戏常用 |
| AWS S3 + CloudFront | 国际分发好 |
| 七牛云 | 免费额度多，中小项目够用 |

---

## 📊 Addressables vs 传统 AssetBundle

| 对比 | 传统 AssetBundle | Addressables |
|------|-----------------|-------------|
| 学习成本 | 高（手动管理依赖） | 低（自动管理） |
| 加载方式 | 手动路径 | 地址名 / AssetReference |
| 依赖处理 | 自己管 | 自动 |
| 热更新 | 自己写流程 | 内置支持 |
| 调试 | 困难 | Groups 界面可视化 |

---

## 🎯 使用建议

1. **小项目**（< 100 个资源）→ Resources 文件夹就够了
2. **中等项目** → 用 Addressables，本地加载
3. **需要热更新** → Addressables + 远程组 + CDN
4. **大型商业项目** → Addressables 是标配

---

## 📚 常见问题

**Q: Addressables 什么时候自动释放资源？**
A: 不会自动释放！必须手动调用 `Addressables.Release()`，否则内存会涨。

**Q: 开发阶段怎么调试？**
A: 用 `Addressables.Use Asset Database` 模式（Play Mode Script），不用每次构建。

**Q: 热更新能更新 C# 代码吗？**
A: 不能！Addressables 只能更新资源（贴图、预制体等）。代码热更需要 ILRuntime 或 HybridCLR。

---

> 💡 **核心原则：** Addressables 让资源管理从"手动地狱"变成"指哪打哪"。先把基础加载搞懂，再考虑热更新和 CDN。
