```markdown
graph TD
A[Network Programming Paradigm] --> B[High-Level Networking]
A --> C[Low-Level Networking]
B --> D[Game Engine Integration Layer]
C --> E[Operating System Protocol Layer]
```

1. Early Network Workflow (Manual Era)**
- **Core Features**: Developers need to manually handle all network details, similar to "reinventing the wheel from scratch".

- **Typical Process**:
1. **Establish a connection**: Manually write Socket code (TCP/UDP), handle three-way handshakes, and port binding.

2. **Data Serialization**: Convert objects to binary (such as C++ memcpy), and handle byte order by yourself.

3. **Send Data Packet**: Send and receive data through send()/recv(), and handle packet segmentation and sticky packet problems.

4. **State Synchronization**: Manually implement heartbeat packets, timeout retransmission, and packet loss compensation.

5. **Anti-cheating**: The server needs to verify the legitimacy of the client data line by line.
- **Code example (C++ Socket)**:

```cpp
// Server listening
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
bind(server_fd, (struct sockaddr*)&address, sizeof(address));
listen(server_fd, 3);

// Client connection
int client_socket = accept(server_fd, (struct sockaddr*)&client_addr, &addr_len);
send(client_socket, "Hello Client", strlen("Hello Client"), 0);
```

**2. Modern network workflow (framework era)**

- **Core features**: Use high-level frameworks to encapsulate underlying details, allowing developers to focus on business logic.
- **Typical process**:

1. **Declare network objects**: Mark variables to be synchronized through the framework (such as [Networked]).

2. **Automatic serialization**: The framework automatically converts objects to binary, handles byte order and compression.

3. **Prediction and interpolation**: Built-in client prediction, server rollback, and smooth interpolation functions.

4. **Event-driven**: Handle player interactions through RPC (such as RPC).

5. **Secure hosting**: The framework automatically filters illegal data and provides basic anti-cheating support.

```cpp
// Define a network object to synchronize player status
public class Player : NetworkBehaviour
{
[Networked] public Vector3 Position { get; set; }
[Networked] public int Health { get; set; } = 100;
}

// Server authoritative damage calculation
public void ApplyDamage(int damage)
{
if (IsServer)
{
Health -= damage;
}
}
```

**1. Low-Level Networking Framework**

- **Positioning**: Directly operate the network protocol stack, focusing on **byte stream transmission**.

- **Typical technologies**:

- **Protocol layer**: TCP/UDP, WebSocket.

- **Tool library**: Socket API of C/C++, System.Net.Sockets of C#.

- **Applicable scenarios**:

- Extreme performance is required (such as high-frequency trading systems).

- Custom protocol requirements (such as IoT device communication).

- **Advantages and Disadvantages**:

- ✅ Full control over transmission details, high flexibility.

- ❌ High development cost, complex issues such as sticky packets, heartbeats, and retransmissions need to be handled.

**2. High-Level Networking Framework**

- **Positioning**: Abstract network details and provide **Business Logic Layer API**.

- **Typical Framework**:

- Network Framework: Unity Netcode, Unreal Online Subsystem, **Netick**.

- General Framework: gRPC, SignalR, Socket.IO.

- **Applicable Scenarios**:

- Rapid development of multiplayer games or real-time applications.

- Projects that do not require in-depth network protocol details.

- **Advantages and Disadvantages**:

- ✅ Ready to use out of the box, with built-in synchronization, prediction, and interpolation functions.

- ❌ Limited flexibility
1. **Connection Phase**:

- The client initiates a connection request to the server (IP:Port or Matchmaking service).

- The server verifies the credentials (Token/SteamID) and establishes a session.
1. **Data synchronization phase**:
- **State synchronization**: The server broadcasts the game world status (such as player position, NPC health) regularly.

- **Input synchronization**: The client sends operation instructions (such as movement, attack), and the server calculates and broadcasts the results.
1. **Prediction and correction**:
- The client predicts movement locally (immediately displays the effect), and the server corrects the deviation after verification.
1. **Disconnection processing**:
- Heartbeat detection disconnection, synchronize the latest status when reconnecting.

```markdown
```cs

// Unity Netcode high-level API example
public class PlayerController : NetworkBehaviour
{
NetworkVariable<Vector3> Position = new(); // Automatically synchronize variables Server->Client Different network frameworks may choose to synchronize different clients with the server

[ServerRpc] // Client→Server call
void MoveServerRpc(Vector3 dir) {
Position.Value += dir * 5f; // Only server can modify
}

[ClientRpc] // Server→Client broadcast
void PlayVFXClientRpc() {
Instantiate(explosionVFX);
}
}
```

#### **Unreal engine solution**

```markdown
```cpp
\####

// Unreal replication system
void AMyCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutProps) const
{
Super::GetLifetimeReplicatedProps(OutProps);
DOREPLIFETIME(AMyCharacter, Health); // Declare synchronization variables
}

// RPC example
UFUNCTION(Server, Reliable) // Server-side RPC
void ServerFireProjectile(FVector Location);

UFUNCTION(NetMulticast, Unreliable) // Multicast RPC
void MulticastPlayImpactEffect();
```
```markdown
graph TB
A[Game logic layer] -->|Call| B[High-level network API]
B -->|Performance bottleneck| C[Bottom-level Socket channel]
C --> D[Custom binary protocol]
```
```markdown
graph TD
A[Project requirements] --> B{Deep network control required?}
B -->|Yes| C[Bottom-level + high-level hybrid architecture]
B -->|No| D[Pure high-level framework]
C --> E{Performance sensitive?}
E -->|Yes| F[Bare Socket for critical path]
E -->|No| G[Extended Engine Network Module]
D --> H{Rapid Prototyping?}
H -->|Yes| I[Unity Netcode/Unreal Replication]
H -->|No| J[Mirror/Fish-Networking]
```
Summary: In the early network workflow, they would manually write each data packet. Most game developers in China use this method and use frame synchronization. In Hight Level Network, it will automatically serialize and automatically generate some code in the IL (compilation) stage, allowing you to quickly implement multiplayer games.