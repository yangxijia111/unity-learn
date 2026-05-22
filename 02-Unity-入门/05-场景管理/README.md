# 场景管理

## 📋 目录

- [SceneManager 概述](#scenemanager-概述)
- [同步加载场景](#同步加载场景)
- [异步加载场景](#异步加载场景)
- [DontDestroyOnLoad](#dontdestroyonload)
- [Build Settings 配置](#build-settings-配置)
- [场景加载管理器](#场景加载管理器)

---

## SceneManager 概述

Unity 的**场景（Scene）**是一个独立的游戏环境，比如：主菜单、关卡1、商店界面。

```csharp
// 需要引用命名空间
using UnityEngine.SceneManagement;
```

每个场景包含自己的 GameObject 集合，切换场景时默认会销毁旧场景的所有对象。

---

## 同步加载场景

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;

public class SyncSceneLoader : MonoBehaviour
{
    // 通过场景名称加载
    public void LoadByName(string sceneName)
    {
        SceneManager.LoadScene(sceneName);
    }

    // 通过 Build Settings 中的索引加载
    public void LoadByIndex(int buildIndex)
    {
        SceneManager.LoadScene(buildIndex);
    }

    // 加载模式：叠加到当前场景（不销毁旧场景）
    public void LoadAdditive(string sceneName)
    {
        SceneManager.LoadScene(sceneName, LoadSceneMode.Additive);
    }

    // 卸载场景（仅 Additive 模式加载的场景）
    public void UnloadScene(string sceneName)
    {
        SceneManager.UnloadSceneAsync(sceneName);
    }
}
```

---

## 异步加载场景

同步加载会卡住游戏画面。异步加载可以显示加载进度条：

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.UI;
using System.Collections;

public class AsyncSceneLoader : MonoBehaviour
{
    [Header("UI")]
    public Slider progressBar;       // 进度条
    public Text progressText;        // 进度百分比文字

    /// <summary>
    /// 异步加载场景并显示进度
    /// </summary>
    public void LoadSceneAsync(string sceneName)
    {
        StartCoroutine(LoadSceneCoroutine(sceneName));
    }

    IEnumerator LoadSceneCoroutine(string sceneName)
    {
        // 开始异步加载
        AsyncOperation asyncLoad = SceneManager.LoadSceneAsync(sceneName);

        // 阻止加载完成后自动切换（用于控制切换时机）
        asyncLoad.allowSceneActivation = false;

        // 更新进度条
        while (!asyncLoad.isDone)
        {
            // asyncLoad.progress 范围是 0 ~ 0.9
            // 0.9 表示加载完成，等待激活
            float progress = Mathf.Clamp01(asyncLoad.progress / 0.9f);

            if (progressBar != null)
                progressBar.value = progress;

            if (progressText != null)
                progressText.text = $"加载中: {(int)(progress * 100)}%";

            // 进度达到 0.9 时，可以切换场景
            if (asyncLoad.progress >= 0.9f)
            {
                if (progressText != null)
                    progressText.text = "按任意键继续...";

                if (Input.anyKeyDown)
                {
                    asyncLoad.allowSceneActivation = true;
                }
            }

            yield return null;
        }
    }
}
```

---

## DontDestroyOnLoad

切换场景时，普通 GameObject 会被销毁。**DontDestroyOnLoad** 让对象跨场景保留：

```csharp
using UnityEngine;

/// <summary>
/// 全局管理器 - 跨场景保留
/// 典型用法：音乐管理器、存档管理器、全局UI
/// </summary>
public class GameManager : MonoBehaviour
{
    // 单例模式（防止重复创建）
    public static GameManager Instance { get; private set; }

    [Header("游戏数据")]
    public int playerScore = 0;
    public string playerName = "Player";

    void Awake()
    {
        // 单例检查
        if (Instance != null && Instance != this)
        {
            Destroy(gameObject); // 销毁重复的
            return;
        }

        Instance = this;
        DontDestroyOnLoad(gameObject); // 跨场景保留

        Debug.Log("GameManager 已初始化，跨场景保留");
    }

    public void AddScore(int amount)
    {
        playerScore += amount;
    }
}

/// <summary>
/// 背景音乐管理器 - 跨场景保留
/// </summary>
public class BGMManager : MonoBehaviour
{
    private static BGMManager instance;

    void Awake()
    {
        if (instance != null)
        {
            Destroy(gameObject);
            return;
        }

        instance = this;
        DontDestroyOnLoad(gameObject);

        // 获取 AudioSource 组件
        AudioSource audio = GetComponent<AudioSource>();
        if (audio != null && !audio.isPlaying)
        {
            audio.Play();
        }
    }
}
```

### DontDestroyOnLoad 的坑

```csharp
// 坑1：重复创建
// 每次加载场景都会执行 Awake()，所以要用单例模式防止重复

// 坑2：DontDestroyOnLoad 的对象不在任何场景里
// 它们在一个特殊的 "DontDestroyOnLoad" 场景中
// SceneManager.GetActiveScene() 不会包含它们

// 坑3：清理问题
// 游戏结束时需要手动销毁，或者在特定条件下销毁
public void Cleanup()
{
    Destroy(gameObject);
    instance = null;
}
```

---

## Build Settings 配置

**Build Settings** 决定哪些场景被打包到游戏里：

```
File → Build Settings（或 Ctrl+Shift+B）

场景列表（按索引排序）：
  0 - MainMenu      ← 游戏启动时加载的第一个场景
  1 - Level1
  2 - Level2
  3 - Shop
  4 - GameOver
```

```csharp
// 在 Build Settings 中：
// 1. 拖入需要的场景文件
// 2. 场景前面的勾 = 是否包含在构建中
// 3. 索引号 = SceneManager.LoadScene(index) 使用的编号
// 4. 顶部选择目标平台（PC/Android/iOS...）
```

---

## 场景加载管理器

一个实用的场景管理器封装：

```csharp
using UnityEngine;
using UnityEngine.SceneManagement;
using System.Collections;
using System;

/// <summary>
/// 场景管理器 - 统一管理场景切换
/// 使用方法：
///   SceneLoader.Instance.LoadScene("Level1");
///   SceneLoader.Instance.LoadSceneWithFade("Level1");
/// </summary>
public class SceneLoader : MonoBehaviour
{
    public static SceneLoader Instance { get; private set; }

    [Header("淡入淡出")]
    public CanvasGroup fadeCanvas;       // 黑色遮罩
    public float fadeDuration = 0.5f;    // 淡入淡出时长

    void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    /// <summary>
    /// 直接加载场景
    /// </summary>
    public void LoadScene(string sceneName)
    {
        SceneManager.LoadScene(sceneName);
    }

    /// <summary>
    /// 带淡入淡出效果的场景切换
    /// </summary>
    public void LoadSceneWithFade(string sceneName)
    {
        StartCoroutine(FadeAndLoad(sceneName));
    }

    IEnumerator FadeAndLoad(string sceneName)
    {
        // 淡出（变黑）
        yield return StartCoroutine(Fade(1));

        // 加载场景
        AsyncOperation asyncLoad = SceneManager.LoadSceneAsync(sceneName);
        yield return asyncLoad;

        // 淡入（变亮）
        yield return StartCoroutine(Fade(0));
    }

    IEnumerator Fade(float targetAlpha)
    {
        if (fadeCanvas == null) yield break;

        float startAlpha = fadeCanvas.alpha;
        float elapsed = 0;

        while (elapsed < fadeDuration)
        {
            elapsed += Time.deltaTime;
            fadeCanvas.alpha = Mathf.Lerp(startAlpha, targetAlpha, elapsed / fadeDuration);
            yield return null;
        }

        fadeCanvas.alpha = targetAlpha;
    }

    /// <summary>
    /// 重新加载当前场景
    /// </summary>
    public void ReloadCurrentScene()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().name);
    }

    /// <summary>
    /// 退出游戏
    /// </summary>
    public void QuitGame()
    {
        #if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
        #else
        Application.Quit();
        #endif
    }
}
```

### 要点总结

| 操作 | 方法 |
|------|------|
| 同步加载 | `SceneManager.LoadScene("场景名")` |
| 异步加载 | `SceneManager.LoadSceneAsync("场景名")` |
| 叠加加载 | `LoadSceneMode.Additive` |
| 跨场景保留 | `DontDestroyOnLoad(gameObject)` |
| 当前场景名 | `SceneManager.GetActiveScene().name` |
| 场景索引 | `SceneManager.GetActiveScene().buildIndex` |

---

> 📌 下一步：学习 [物理系统](../06-物理系统/README.md)
