```markdown
```graph TD
 A[网络编程范式] --> B[High-Level Networking]
 A --> C[Low-Level Networking]
 B --> D[游戏引擎集成层]
 C --> E[操作系统协议层]
```

1. 早期网络工作流（手动时代）**
- **核心特点**：开发者需手动处理所有网络细节，类似“从零造轮子”。

- **典型流程**：
1. **建立连接**：手动编写Socket代码（TCP/UDP），处理三次握手、端口绑定。

2. **数据序列化**：将对象转为二进制（如C++的memcpy），需自行处理字节序。

3. **发送数据包**：通过send()/recv()收发数据，处理分包、粘包问题。

4. **状态同步**：手动实现心跳包、超时重传、丢包补偿。

5. **反作弊**：服务器需逐行验证客户端数据合法性。
- **代码示例（C++ Socket）**：

```cpp
// 服务端监听
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
bind(server_fd, (struct sockaddr*)&address, sizeof(address));
listen(server_fd, 3);

// 客户端连接
int client_socket = accept(server_fd, (struct sockaddr*)&client_addr, &addr_len);
send(client_socket, "Hello Client", strlen("Hello Client"), 0);
```

**2. 现代网络工作流（框架时代）**

- **核心特点**：使用高层框架封装底层细节，开发者聚焦业务逻辑。
- **典型流程**：
1. **声明网络对象**：通过框架标记需同步的变量（如[Networked]）。
2. **自动序列化**：框架自动将对象转为二进制，处理字节序和压缩。
3. **预测与插值**：内置客户端预测、服务器回滚、平滑插值功能。
4. **事件驱动**：通过RPC（如RPC）处理玩家交互。
5. **安全托管**：框架自动过滤非法数据，提供基础反作弊支持。

```cpp
// 定义一个同步玩家状态的网络对象
public class Player : NetworkBehaviour
{
    [Networked] public Vector3 Position { get; set; }
    [Networked] public int Health { get; set; } = 100;
}

// 服务器权威伤害计算
public void ApplyDamage(int damage)
{
    if (IsServer)
    {
        Health -= damage;
    }
}
```

**1. 底层发包框架（Low-Level Networking）**

- **定位**：直接操作网络协议栈，关注**字节流传输**。

- **典型技术**：

- **协议层**：TCP/UDP、WebSocket。

- **工具库**：C/C++的Socket API、C#的System.Net.Sockets。

- **适用场景**：

- 需要极致性能（如高频交易系统）。

- 自定义协议需求（如物联网设备通信）。

- **优缺点**：

- ✅ 完全控制传输细节，灵活性高。

- ❌ 开发成本高，需处理粘包、心跳、重传等复杂问题。

**2. 高层网络框架（High-Level Networking）**

- **定位**：抽象网络细节，提供**业务逻辑层API**。

- **典型框架**：

- 网络框架：Unity Netcode、Unreal Online Subsystem、**Netick**。

- 通用框架：gRPC、SignalR、Socket.IO。

- **适用场景**：

- 快速开发多人游戏或实时应用。

- 无需深入网络协议细节的项目。

- **优缺点**：

- ✅ 开箱即用，内置同步、预测、插值功能。

- ❌ 灵活性受限
1. **连接阶段**：
- 客户端向服务器发起连接请求（IP:Port 或 Matchmaking 服务）。

- 服务器验证凭证（Token/SteamID），建立会话。
1. **数据同步阶段**：
- **状态同步**：服务器定期广播游戏世界状态（如玩家位置、NPC血量）。

- **输入同步**：客户端发送操作指令（如移动、攻击），服务器计算并广播结果。
1. **预测与修正**：
- 客户端本地预测移动（立即显示效果），服务器验证后修正偏差。
1. **断开处理**：
- 心跳检测断线，重连时同步最新状态。

```markdown
```cs

// Unity Netcode 高层API示例
public class PlayerController : NetworkBehaviour 
{
    NetworkVariable<Vector3> Position = new(); // 自动同步变量 服务器->客户端 不同网络框架可能可以选择服务器同步不同的客户端

    [ServerRpc] // 客户端→服务器调用
    void MoveServerRpc(Vector3 dir) {
        Position.Value += dir * 5f; // 仅服务器可修改
    }

    [ClientRpc] // 服务器→客户端广播
    void PlayVFXClientRpc() {
        Instantiate(explosionVFX); 
    }
}
```

#### **Unreal引擎方案**

```markdown
```cpp
\####

// Unreal 复制系统
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutProps) const
{
 Super::GetLifetimeReplicatedProps(OutProps);
 DOREPLIFETIME(AMyCharacter, Health); // 声明同步变量
}

// RPC示例
UFUNCTION(Server, Reliable) // 服务端RPC
void ServerFireProjectile(FVector Location);

UFUNCTION(NetMulticast, Unreliable) // 多播RPC
void MulticastPlayImpactEffect();
```

graph TB
    A[游戏逻辑层] -->|调用| B[高层网络API]
    B -->|性能瓶颈处| C[底层Socket通道]
    C --> D[自定义二进制协议]

graph TD
    A[项目需求] --> B{需要深度网络控制?}
    B -->|是| C[底层+高层混合架构]
    B -->|否| D[纯高层框架]
    C --> E{性能敏感型?}
    E -->|是| F[关键路径用裸Socket]
    E -->|否| G[扩展引擎网络模块]
    D --> H{快速原型?}
    H -->|是| I[Unity Netcode/Unreal Replication]
    H -->|否| J[Mirror/Fish-Networking]

总结: 早期网络工作流 他们会手动写每个数据包 在中国多数游戏开发厂商都使用这种方法和使用帧同步 而在 Hight Level Network 中  他会自动序列化 自动在IL（编译）阶段生成一些代码 让你快速实现多人游戏
