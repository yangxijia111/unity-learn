# 04 - 输入系统 (Input System)

Unity 有两套输入系统：旧版 Input Manager 和新版 Input System Package。

## 旧版 Input Manager（经典）

内置在 Unity 中，简单直接，适合原型开发和小型项目。

### 基础按键检测

```csharp
using UnityEngine;

public class OldInputExample : MonoBehaviour
{
    void Update()
    {
        // 按键按下瞬间（每按一次只触发一帧）
        if (Input.GetKeyDown(KeyCode.Space))
            Debug.Log("空格按下");

        // 按键持续按住（每帧都触发）
        if (Input.GetKey(KeyCode.W))
            transform.Translate(Vector3.forward * Time.deltaTime * 5f);

        // 按键松开瞬间
        if (Input.GetKeyUp(KeyCode.Space))
            Debug.Log("空格松开");

        // 鼠标按键（0=左键, 1=右键, 2=中键）
        if (Input.GetMouseButtonDown(0))
            Debug.Log("鼠标左键点击");
    }
}
```

### 轴输入（模拟量）

```csharp
public class AxisInputExample : MonoBehaviour
{
    public float moveSpeed = 5f;
    public float rotateSpeed = 120f;

    void Update()
    {
        // 返回 -1 到 1 的平滑值（包含键盘 WASD/方向键 和 手柄摇杆）
        float horizontal = Input.GetAxis("Horizontal"); // 左右
        float vertical = Input.GetAxis("Vertical");     // 上下

        // GetAxisRaw 不做平滑处理，只有 -1, 0, 1
        float rawH = Input.GetAxisRaw("Horizontal");

        // 移动
        Vector3 move = new Vector3(horizontal, 0, vertical);
        transform.Translate(move * moveSpeed * Time.deltaTime, Space.World);

        // 鼠标输入
        float mouseX = Input.GetAxis("Mouse X"); // 鼠标水平移动
        float mouseY = Input.GetAxis("Mouse Y"); // 鼠标垂直移动

        // 旋转视角
        transform.Rotate(Vector3.up, mouseX * rotateSpeed * Time.deltaTime);
    }
}
```

### 轴配置

在 **Edit → Project Settings → Input Manager** 中：
- `Horizontal`：A/D 键 + 左摇杆水平
- `Vertical`：W/S 键 + 左摇杆垂直
- `Fire1`：Ctrl + 手柄 A
- `Jump`：Space + 手柄 X
- `Mouse X/Y`：鼠标移动

## 新版 Input System Package

功能强大，支持**动作映射、设备热切换、组合键**，适合正式项目。

### 安装

**Window → Package Manager → Input System** → Install

安装后在 **Edit → Project Settings → Player → Active Input Handling** 切换为 **Both** 或 **Input System Package (New)**。

### 创建 Input Actions

1. **Project 窗口** → 右键 → Create → Input Actions → 命名（如 `PlayerInputActions`）
2. 双击打开编辑器，创建 Action Map 和 Actions
3. 绑定按键（Keyboard、Gamepad 等）

### 代码使用（PlayerInput 组件方式）

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

public class NewInputExample : MonoBehaviour
{
    private PlayerInputActions inputActions;
    private Vector2 moveInput;

    void Awake()
    {
        inputActions = new PlayerInputActions();
    }

    void OnEnable()
    {
        inputActions.Player.Enable();

        // 订阅事件
        inputActions.Player.Jump.performed += OnJump;
        inputActions.Player.Fire.performed += OnFire;

        // 持续读取（适合移动）
        // inputActions.Player.Move → 在 Update 中读取
    }

    void OnDisable()
    {
        // 取消订阅（重要！）
        inputActions.Player.Jump.performed -= OnJump;
        inputActions.Player.Fire.performed -= OnFire;
        inputActions.Player.Disable();
    }

    void Update()
    {
        // 读取移动输入
        moveInput = inputActions.Player.Move.ReadValue<Vector2>();
        Vector3 move = new Vector3(moveInput.x, 0, moveInput.y);
        transform.Translate(move * 5f * Time.deltaTime);
    }

    void OnJump(InputAction.CallbackContext ctx)
    {
        Debug.Log($"跳跃！按压力度: {ctx.ReadValue<float>()}");
    }

    void OnFire(InputAction.CallbackContext ctx)
    {
        Debug.Log("开火！");
    }
}
```

### 代码使用（SendMessage 方式，更简单）

```csharp
// 需要在 PlayerInput 组件中设置 Behavior = Send Messages
// 方法名必须是 On{ActionName}
public class SimpleInputExample : MonoBehaviour
{
    // Unity 自动调用，方法名 = "On" + Action名
    void OnMove(InputValue value)
    {
        Vector2 input = value.Get<Vector2>();
        Debug.Log($"移动输入: {input}");
    }

    void OnJump(InputValue value)
    {
        Debug.Log("跳跃！");
    }
}
```

## 触摸输入

```csharp
public class TouchInputExample : MonoBehaviour
{
    void Update()
    {
        // 旧版：Input.touches
        for (int i = 0; i < Input.touchCount; i++)
        {
            Touch touch = Input.GetTouch(i);
            Vector2 pos = touch.position;

            switch (touch.phase)
            {
                case TouchPhase.Began:
                    Debug.Log($"触摸开始: {pos}");
                    break;
                case TouchPhase.Moved:
                    Debug.Log($"触摸移动: {pos}");
                    break;
                case TouchPhase.Ended:
                    Debug.Log($"触摸结束: {pos}");
                    break;
            }
        }

        // 新版：Touchscreen.current
        if (Touchscreen.current != null && Touchscreen.current.primaryTouch.press.isPressed)
        {
            Vector2 touchPos = Touchscreen.current.primaryTouch.position.ReadValue();
            // 将屏幕坐标转为世界坐标
            Vector3 worldPos = Camera.main.ScreenToWorldPoint(touchPos);
        }
    }
}
```

## 跨平台输入建议

| 方式 | 适用场景 |
|------|----------|
| 旧 Input Manager | 快速原型、小型项目、仅 PC |
| 新 Input System | 正式项目、多平台、复杂输入需求 |
| `#if UNITY_EDITOR` 条件编译 | 区分编辑器和真机输入 |

## 注意事项

1. 旧版的 `Input.GetKey` 系列在新版启用后**不可用**
2. 新版 Input System 支持同 Action 绑定多个设备（键盘+手柄同时生效）
3. 移动端触摸优先用新版 Input System，兼容性更好
4. 按键配置尽量通过 Input Actions 编辑器管理，**不要硬编码键位**
