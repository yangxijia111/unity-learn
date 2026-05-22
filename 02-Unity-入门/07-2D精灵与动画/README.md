# 2D 精灵与动画

## 📋 目录

- [Sprite 精灵基础](#sprite-精灵基础)
- [SpriteRenderer](#spriterenderer)
- [Sprite Atlas 图集](#sprite-atlas-图集)
- [Animator Controller](#animator-controller)
- [Animation Clip](#animation-clip)
- [状态机与过渡](#状态机与过渡)
- [完整示例](#完整示例)

---

## Sprite 精灵基础

**Sprite（精灵）** 是 2D 游戏中的图片单位，本质是一张绑定到场景中的 2D 图像：

```
导入图片设置：
1. 将图片拖入 Project 面板
2. Inspector 中设置 Texture Type → Sprite (2D and UI)
3. Sprite Mode：
   - Single：单张图片
   - Multiple：图集（切割多张小图）
4. Pixels Per Unit：1 单位 = 多少像素（默认 100）
5. Filter Mode：
   - Point：像素风格（保留锯齿）
   - Bilinear：平滑过渡
   - Trilinear：更平滑（3D 用）
```

### Sprite 多图切割

```
对于 Sprite Sheet（角色动画表）：
1. Sprite Mode 设为 Multiple
2. 点击 "Sprite Editor"
3. Slice → Grid by Cell Size（网格切割）
4. 输入每帧大小（如 32x32）
5. Apply 保存
```

---

## SpriteRenderer

**SpriteRenderer** 是显示 2D 精灵的渲染器（类似 3D 的 MeshRenderer）：

```csharp
using UnityEngine;

public class SpriteRendererControl : MonoBehaviour
{
    private SpriteRenderer sr;

    void Start()
    {
        sr = GetComponent<SpriteRenderer>();

        // 基本属性
        sr.sprite = null;                    // 设置显示的精灵
        sr.color = Color.white;              // 颜色叠加
        sr.flipX = false;                    // 水平翻转
        sr.flipY = false;                    // 垂直翻转
        sr.sortingLayerName = "Default";     // 排序层
        sr.sortingOrder = 0;                 // 排序顺序（数字大的在前）
        sr.material = null;                  // 材质
    }

    // 翻转精灵（角色朝向）
    public void FlipSprite(float horizontalInput)
    {
        if (horizontalInput > 0)
            sr.flipX = false;  // 面朝右
        else if (horizontalInput < 0)
            sr.flipX = true;   // 面朝左
    }

    // 改变颜色（受伤闪白）
    public void FlashWhite()
    {
        StartCoroutine(FlashRoutine());
    }

    System.Collections.IEnumerator FlashRoutine()
    {
        sr.color = Color.white;
        yield return new WaitForSeconds(0.1f);
        sr.color = Color.red;
        yield return new WaitForSeconds(0.1f);
        sr.color = Color.white;
    }

    // 淡入淡出
    public void FadeOut(float duration)
    {
        StartCoroutine(FadeRoutine(duration));
    }

    System.Collections.IEnumerator FadeRoutine(float duration)
    {
        float elapsed = 0;
        Color startColor = sr.color;

        while (elapsed < duration)
        {
            elapsed += Time.deltaTime;
            float alpha = Mathf.Lerp(1, 0, elapsed / duration);
            sr.color = new Color(startColor.r, startColor.g, startColor.b, alpha);
            yield return null;
        }
    }
}
```

### 排序层设置

```
Edit → Project Settings → Tags and Layers → Sorting Layers

排序层（从上到下渲染，上面的在后）：
  - Background   （背景）
  - Default       （默认）
  - Player        （玩家）
  - Foreground    （前景）
  - UI            （UI）

同一层内，Sorting Order 越大越靠前显示
```

---

## Sprite Atlas 图集

**Sprite Atlas（精灵图集）** 将多张小图打包成一张大图，减少 Draw Call，提升性能：

```
创建步骤：
1. Project 面板 → 右键 → Create → 2D → Sprite Atlas
2. 把需要打包的 Sprite 文件夹拖到 Objects for Packing 列表
3. 打包后 Unity 自动合并

打包设置：
- Include in Build：构建时打包（默认开启）
- Allow Rotation：允许旋转以节省空间
- Tight Packing：紧密打包（更省空间）
```

```csharp
// 运行时加载图集中的 Sprite
using UnityEngine;
using UnityEngine.U2D;

public class AtlasLoader : MonoBehaviour
{
    public SpriteAtlas atlas;   // 在 Inspector 中拖入图集

    void Start()
    {
        // 从图集中获取指定名称的 Sprite
        Sprite mySprite = atlas.GetSprite("sprite_name");
        GetComponent<SpriteRenderer>().sprite = mySprite;
    }
}
```

---

## Animator Controller

**Animator Controller（动画控制器）** 是状态机，管理动画之间的切换逻辑：

```
创建步骤：
1. Project 面板 → 右键 → Create → Animator Controller
2. 双击打开 Animator 窗口
3. 添加动画状态（Animation Clip）
4. 设置过渡条件（Parameters）
5. 将 Animator Controller 拖到 GameObject 的 Animator 组件

Animator 窗口组成：
- Layers：动画层（基础层 + 覆盖层）
- Parameters：控制动画切换的参数
- States：动画状态（橙色=默认状态）
- Transitions：状态之间的过渡线
```

```csharp
using UnityEngine;

public class AnimatorControl : MonoBehaviour
{
    private Animator animator;

    void Start()
    {
        animator = GetComponent<Animator>();
    }

    void Update()
    {
        // 设置参数值来触发动画切换

        // Float 参数（速度）
        float speed = Input.GetAxis("Vertical");
        animator.SetFloat("Speed", speed);

        // Bool 参数（是否奔跑）
        bool isRunning = Input.GetKey(KeyCode.LeftShift);
        animator.SetBool("IsRunning", isRunning);

        // Trigger 参数（一次性触发，如跳跃、攻击）
        if (Input.GetKeyDown(KeyCode.Space))
        {
            animator.SetTrigger("Jump");
        }

        // Int 参数（状态编号等）
        animator.SetInteger("WeaponType", 1);
    }

    // 动画事件回调（在 Animation Clip 中设置）
    public void OnAttackHit()
    {
        Debug.Log("攻击命中！");
    }

    // 检查当前动画状态
    void CheckAnimationState()
    {
        AnimatorStateInfo stateInfo = animator.GetCurrentAnimatorStateInfo(0);

        // 检查是否在某个动画状态
        if (stateInfo.IsName("Player_Run"))
        {
            Debug.Log("正在跑步");
        }

        // 获取动画播放进度（0~1）
        float normalizedTime = stateInfo.normalizedTime;
    }
}
```

---

## Animation Clip

**Animation Clip（动画片段）** 包含具体的动画数据（关键帧）：

```
创建步骤：
1. 选中 GameObject
2. Window → Animation → Animation（Ctrl+6）
3. 点击 "Create" 创建新 Animation Clip
4. 点击红色录制按钮开始录制
5. 在时间线上设置关键帧
6. 再次点击录制按钮停止

Animation 窗口功能：
- 时间线：0:00 到动画结束
- 添加属性：可以动画化任何组件的属性
- 曲线模式：调整关键帧之间的插值
- 预览：拖动时间线预览动画
```

```csharp
// 代码中控制 Animation Clip（使用 Animation 组件，较老的方式）
using UnityEngine;

public class AnimationClipControl : MonoBehaviour
{
    private Animation anim;

    void Start()
    {
        anim = GetComponent<Animation>();
    }

    void Update()
    {
        // 播放指定动画
        if (Input.GetKeyDown(KeyCode.Alpha1))
        {
            anim.Play("Attack");
        }

        // 交叉淡入（平滑过渡到新动画）
        if (Input.GetKeyDown(KeyCode.Alpha2))
        {
            anim.CrossFade("Run", 0.3f);
        }
    }
}
```

### 动画事件

在 Animation Clip 中可以添加事件，在特定帧调用脚本方法：

```
Animation 窗口 → 时间线上右键 → Add Animation Event
- Function：要调用的方法名（如 "OnAttackHit"）
- Int/Float/String/Obj：传递的参数
```

---

## 状态机与过渡

Animator 的核心是**状态机**，通过条件判断切换动画：

```
状态图示例：

  [Idle] ──Speed>0.1──→ [Run]
    ↑                      │
    │                     Speed<0.1
    │                      ↓
    └──Speed<0.1────── [Walk]

  [AnyState] ──Jump触发──→ [Jump]
    ↑                         │
    └───Jump结束─────────────┘
```

```
过渡设置（Transition）：
- Has Exit Time：是否有退出时间（动画播完才切换）
- Exit Time：退出时间点（0~1）
- Transition Duration：过渡时长
- Conditions：切换条件列表

过渡条件类型：
- Float: Greater / Less
- Bool: True / False
- Trigger: (无条件)
- Int: Equals / Not Equals / Greater / Less
```

```csharp
// 状态机最佳实践：

// 1. 使用 Sub-State Machine 管理复杂动画组
//    比如把所有攻击动画放在 "Attacks" 子状态机中

// 2. 使用 Any State 处理全局转换
//    从任何状态都可以切换到 "Death" 或 "Hurt"

// 3. 保持状态机简洁
//    太多状态和过渡线会让维护变得困难
//    考虑用 Blend Tree 合并相似动画

// 4. Blend Tree（混合树）
//    根据参数值混合多个动画
//    常用于：跑步方向混合（前后左右）
//    创建：右键 → Create State → From New Blend Tree
```

---

## 完整示例

一个完整的 2D 角色动画控制器：

```csharp
using UnityEngine;

/// <summary>
/// 2D 角色动画控制器
/// 搭配 Animator Controller 使用
/// </summary>
[RequireComponent(typeof(Animator))]
[RequireComponent(typeof(Rigidbody2D))]
[RequireComponent(typeof(SpriteRenderer))]
public class CharacterAnimator2D : MonoBehaviour
{
    // Animator 参数名（要和 Animator Controller 中设置的一致）
    private static readonly int SpeedParam = Animator.StringToHash("Speed");
    private static readonly int IsGroundedParam = Animator.StringToHash("IsGrounded");
    private static readonly int JumpTriggerParam = Animator.StringToHash("Jump");
    private static readonly int AttackTriggerParam = Animator.StringToHash("Attack");
    private static readonly int HurtTriggerParam = Animator.StringToHash("Hurt");
    private static readonly int DieTriggerParam = Animator.StringToHash("Die");

    private Animator animator;
    private Rigidbody2D rb;
    private SpriteRenderer sr;

    [Header("地面检测")]
    public Transform groundCheck;
    public float groundCheckRadius = 0.2f;
    public LayerMask groundLayer;

    private bool isGrounded;
    private bool isDead;

    void Start()
    {
        animator = GetComponent<Animator>();
        rb = GetComponent<Rigidbody2D>();
        sr = GetComponent<SpriteRenderer>();
    }

    void Update()
    {
        if (isDead) return;

        // 地面检测
        isGrounded = Physics2D.OverlapCircle(
            groundCheck.position,
            groundCheckRadius,
            groundLayer
        );

        // 获取水平输入
        float moveX = Input.GetAxisRaw("Horizontal");

        // 更新动画参数
        animator.SetFloat(SpeedParam, Mathf.Abs(moveX));
        animator.SetBool(IsGroundedParam, isGrounded);

        // 角色朝向翻转
        if (moveX != 0)
        {
            sr.flipX = moveX < 0;
        }

        // 跳跃
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            animator.SetTrigger(JumpTriggerParam);
            rb.velocity = new Vector2(rb.velocity.x, 10f);
        }

        // 攻击
        if (Input.GetMouseButtonDown(0))
        {
            animator.SetTrigger(AttackTriggerParam);
        }
    }

    // === 动画事件回调 ===

    // 攻击命中帧
    public void OnAttackHitFrame()
    {
        Debug.Log("攻击命中帧");
        // 检测敌人并造成伤害
    }

    // 攻击结束
    public void OnAttackEnd()
    {
        Debug.Log("攻击动画结束");
    }

    // 脚步声
    public void OnFootstep()
    {
        // 播放脚步音效
    }

    // === 公共方法 ===

    public void PlayHurt()
    {
        if (!isDead)
            animator.SetTrigger(HurtTriggerParam);
    }

    public void PlayDeath()
    {
        isDead = true;
        animator.SetTrigger(DieTriggerParam);
    }

    // 调试可视化
    void OnDrawGizmos()
    {
        if (groundCheck != null)
        {
            Gizmos.color = isGrounded ? Color.green : Color.red;
            Gizmos.DrawWireSphere(groundCheck.position, groundCheckRadius);
        }
    }
}
```

### 要点总结

| 概念 | 说明 |
|------|------|
| Sprite | 2D 图片单位 |
| SpriteRenderer | 渲染精灵的组件 |
| Sprite Atlas | 打包多图减少 Draw Call |
| Animator Controller | 动画状态机 |
| Animation Clip | 具体的动画数据 |
| Parameters | 控制动画切换的参数 |
| Transition | 状态之间的过渡规则 |
| Blend Tree | 根据参数混合多个动画 |
| Animation Event | 在动画帧调用脚本方法 |

---

> 📌 下一步：学习 [UI 系统（UGUI）](../08-UI系统（UGUI）/README.md)
