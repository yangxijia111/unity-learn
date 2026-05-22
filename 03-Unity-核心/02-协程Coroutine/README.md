# 02 - 协程 Coroutine

协程是 Unity 中处理**异步和延时逻辑**的核心机制，不需要多线程就能实现分帧执行。

## 基本概念

协程本质上是一个可以**暂停和恢复**执行的方法，通过 `IEnumerator` 和 `yield return` 实现。

## 基础用法

### 定义协程

```csharp
using UnityEngine;
using System.Collections;

public class CoroutineBasics : MonoBehaviour
{
    // 协程必须返回 IEnumerator
    IEnumerator MyCoroutine()
    {
        Debug.Log("协程开始");
        
        // 等待 2 秒
        yield return new WaitForSeconds(2f);
        
        Debug.Log("2秒后执行");
        
        // 等待下一帧
        yield return null;
        
        Debug.Log("又过了一帧");
    }
}
```

### 启动和停止

```csharp
public class CoroutineControl : MonoBehaviour
{
    private Coroutine myCoroutine;

    void Start()
    {
        // 方式1：直接启动（无法中途停止）
        StartCoroutine(MyCoroutine());

        // 方式2：保存引用（可以中途停止）
        myCoroutine = StartCoroutine(MyCoroutine());
    }

    void Update()
    {
        // 按下 Esc 停止协程
        if (Input.GetKeyDown(KeyCode.Escape) && myCoroutine != null)
        {
            StopCoroutine(myCoroutine);
            Debug.Log("协程已停止");
        }

        // 停止该脚本上所有协程
        if (Input.GetKeyDown(KeyCode.Q))
        {
            StopAllCoroutines();
        }
    }

    IEnumerator MyCoroutine()
    {
        while (true) // 无限循环
        {
            Debug.Log("循环中...");
            yield return new WaitForSeconds(1f);
        }
    }
}
```

## yield return 常用指令

| 指令 | 作用 |
|------|------|
| `yield return null` | 等待下一帧 |
| `yield return new WaitForSeconds(2f)` | 等待指定秒数（受 timeScale 影响） |
| `yield return new WaitForSecondsRealtime(2f)` | 等待指定秒数（不受 timeScale 影响） |
| `yield return new WaitForEndOfFrame()` | 等到本帧渲染结束后 |
| `yield return new WaitForFixedUpdate()` | 等到下一个 FixedUpdate |
| `yield return new WaitUntil(() => condition)` | 等待条件为 true |
| `yield return new WaitWhile(() => condition)` | 等待条件为 false |
| `yield return StartCoroutine(Other())` | 等待另一个协程完成 |

## 实战示例

### 延迟执行

```csharp
public class DelayExample : MonoBehaviour
{
    void Start()
    {
        // 3 秒后销毁自身
        StartCoroutine(DelayedDestroy(gameObject, 3f));
    }

    IEnumerator DelayedDestroy(GameObject target, float delay)
    {
        yield return new WaitForSeconds(delay);
        Destroy(target);
    }
}
```

### 倒计时

```csharp
public class Countdown : MonoBehaviour
{
    void Start()
    {
        StartCoroutine(CountdownRoutine(10));
    }

    IEnumerator CountdownRoutine(int seconds)
    {
        for (int i = seconds; i > 0; i--)
        {
            Debug.Log($"倒计时: {i}");
            yield return new WaitForSeconds(1f);
        }
        Debug.Log("时间到！");
    }
}
```

### 渐变动画（颜色/透明度）

```csharp
public class FadeExample : MonoBehaviour
{
    public SpriteRenderer spriteRenderer;
    public float fadeDuration = 2f;

    void Start()
    {
        StartCoroutine(FadeOut());
    }

    IEnumerator FadeOut()
    {
        Color color = spriteRenderer.color;
        float elapsed = 0f;

        while (elapsed < fadeDuration)
        {
            elapsed += Time.deltaTime;
            color.a = Mathf.Lerp(1f, 0f, elapsed / fadeDuration);
            spriteRenderer.color = color;
            yield return null; // 每帧更新
        }

        // 确保最终值
        color.a = 0f;
        spriteRenderer.color = color;
    }
}
```

### 轮询等待条件

```csharp
public class WaitUntilExample : MonoBehaviour
{
    public GameObject target;

    void Start()
    {
        StartCoroutine(WaitForTargetActive());
    }

    IEnumerator WaitForTargetActive()
    {
        Debug.Log("等待目标激活...");

        // 等待 target 被激活
        yield return new WaitUntil(() => target.activeInHierarchy);

        Debug.Log("目标已激活！");
    }
}
```

## 注意事项

1. **协程不是多线程** —— 仍在主线程执行，只是分帧，不会提高性能
2. **对象销毁后协程会停止** —— 如果协程挂在已销毁的对象上，会自动终止
3. **timeScale = 0 时**，`WaitForSeconds` 会一直等待，改用 `WaitForSecondsRealtime`
4. **避免在协程里做耗时运算**，会卡主线程
5. **嵌套协程** 可以用 `yield return StartCoroutine()` 实现顺序执行
6. 协程内不能用 `try-catch` 包裹 `yield return` 语句
