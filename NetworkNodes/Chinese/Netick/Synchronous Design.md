| 特性     | UNet           | Netick         |
| ------ | -------------- | -------------- |
| 输入传输   | 高频RPC          | 结构化压缩输入        |
| 客户端防作弊 | ?              | ?              |
| 事件通信   | 属性同步+OnChanged | 属性同步+OnChanged |
| 带宽优化   | ?              | ?              |

#### 

# 玩家输入

## 其他网络框架

// 移动指令（RPC方式）
[Command]
void CmdMove(Vector2 direction) {
    character.Move(direction * speed); // 直接修改状态
}

// 按钮交互（RPC方式）
[Command]
void CmdInteract() {
    OpenTreasureChest(); // 无验证的危险调用
}

客户端预测不可能**​：RPC直接修改状态，无法回滚 而且RPC滥用

#### Netick的结构化输入

[Networked]
public struct PlayerInput : INetworkInput {
    [Networked] public Vector2 MoveDirection { get; set; }  // 8字节压缩
    public NetworkBool Jump;      // 1位布尔
    public NetworkBool Interact;  // UI按钮事件
}

void OnJumpPressed() => SetInputFlag(ref PlayerInput.Jump);

// 交互按钮
void OnInteractPressed() => SetInputFlag(ref PlayerInput.Interact);

void SetInputFlag<T>(ref T field) where T : unmanaged {
    var input = Sandbox.GetInput<PlayerInput>();
    field = true;
    Sandbox.SetInput(input); 
}

    public override void NetworkFixedUpdate() {
     if (!FetchInput(out PlayerInput input)) return;
    // 客户端预测移动（可回滚）
    character.Move(input.MoveDirection);
    
    // 按钮事件（服务器验证执行）
    if (input.Interact && IsServer) {
        if (CanInteractWith(nearestObject)) { // 距离验证
            nearestObject.Interact();
        }
        input.Interact = false; // 重置标志
    }
    }

你就获得了一个可回滚+客户端预测 的角色控制器了 在有些网络框架较为复杂

- `Sandbox.GetInput/SetInput`：采集输入
- `FetchInput`：在`NetworkFixedUpdate`中执行

| 客户端预测 | 移动 | 直接修改物理状态 |  
| 非预测（仅服务器） | 驾驶载具 | `if (IsServer)` |  
| 预测但非重模拟 | 射击 | `if (!IsResimulating)`|

// 按钮点击触发输入
void OnJumpButtonClicked() {
    var input = Sandbox.GetInput<PlayerInput>();
    input.Jump = true;
    Sandbox.SetInput(input);
}

#### 结构化输入 vs RPC

| 特性          | 传统RPC框架    | Netick结构化输入 |
| ----------- | ---------- | ----------- |
| ​**客户端预测**​ | ❌ 不可能      | ✅ 完整支持      |
| ​**输入压缩**​  | ❌ 每次全量发送   | ✅ 自动差分压缩    |
| ​**反作弊能力**​ | ❌ 客户端可篡改状态 | ✅ 只接收意图     |
| ​**时序一致性**​ | ❌ 乱序到达     | ✅ 严格Tick对齐  |
| ​**带宽效率**​  | ❌ 高频操作浪费   | ✅ 平均节省50%+  |

###### 为什么结构化输入更优越？

​**1时间旅行式回滚**​  
当服务器状态修正时，Netick可回放历史输入重新模拟，实现平滑校正：

public override void NetworkFixedUpdate() {
    if (IsResimulating) {
        // 使用历史输入重新模拟
        FetchInputForTick(resimulateTick); 
    }
}

​**输入源隔离（防作弊核心）​**​  
每个玩家只能控制自己的输入源：

// 服务器分配控制权
void SpawnPlayer(NetworkClient owner) {
    var player = Sandbox.NetworkInstantiate(prefab);
    player.InputSource = owner; // 关键授权
}

- 客户端无法修改他人的输入
- 服务器可随时回收控制权

输入复用机制（`Input Reuse At Low FPS`）在帧率低于服务器 tickrate 时自动复制输入，避免移动卡顿。

Netick 通过 “**属性同步为主、、输入系统为辅、RPC兜底**” 的设计，从根本上解决了传统框架中 RPC 滥用导致的安全漏洞与性能瓶颈
