# Replication System

## 1. Definition of core concepts

### 1. Networking

Refers to a technical system that enables real-time data exchange and state synchronization between multiple independent devices through network connections, allowing distributed users to interact in a shared virtual environment. In Unity, it specifically refers to a technical solution for implementing multiplayer online games.

Network workflow

### 2. Network workflow

Refers to the complete processing flow of data from the sender to the receiver in the network, including **connection management, data encapsulation, transmission, parsing** and other steps. Whether it is early low-level development or modern high-level frameworks, the core goal is always: **reliable, efficient, and secure data transmission**.

### 2. Replication System

In multiplayer online games, the replication system (Replication System) is the core mechanism to ensure that all clients are synchronized with the server. It is responsible for efficiently and reliably propagating the state of game objects (such as player position, health, item status, etc.) to all relevant clients.

Consistency: The game world state seen by all clients is consistent with the authoritative version of the server.

Low latency: Minimize the time from player operation to visual feedback (usually required to be <150ms).

Bandwidth efficiency: Transmit necessary data under limited network conditions, such as "Apex Legends" only uses 20-60KB/s bandwidth.

Fault tolerance: Deal with network problems such as packet loss and jitter to prevent synchronization breaks.

+---------------------+
| Game logic layer | // Generate state change events
+---------------------+
|
+---------------------+
| Replication management layer | // Determine synchronization objects, frequency, and methods
+---------------------+
|
+---------------------+
| Network transport layer | // Data serialization, compression, and transmission
+---------------------+

## Synchronization method

State synchronization (State Synchronization
Principle: The server periodically sends the complete or incremental state of the game object to the client.

Implementation:

Snapshot synchronization:
The server sends the complete world state at a fixed frequency (such as 30Hz).

Example: Quake III uses 10Hz snapshots, and the client smoothly transitions through interpolation.

Delta compression:
Only send the difference data from the previous snapshot, saving 30-70% bandwidth.

Optimization: Use bitmask to mark the changed field, such as Unreal Engine's RepNotify.

## Remote call

Remote procedure call (RPC)
Function: Synchronize instantaneous events (such as shooting, skill release).

Type:

Reliable RPC: Ensure arrival, used for critical events (such as damage calculation).

Unreliable RPC: Allow loss, used for high-frequency non-critical events (such as footsteps).

## Prediction and correction

Prediction and correction (Prediction & Reconciliation)
Client-side prediction (Client-side Prediction):
Respond to local operations immediately without waiting for server confirmation.
Implementation: Maintain an operation history buffer with a typical depth of 5-10 frames.

Server Reconciliation:
When the server state arrives, roll back and replay the predicted frame.

Dynamic priority:
Allocate bandwidth based on object importance, for example:

Player character: priority 10

Distant NPC: priority 2

Static scene object: priority 0 (no update)

Relevance filtering:
Only synchronize objects visible/interactive to the client, such as in PlayerUnknown's Battlegrounds:

Player synchronization position within 1km

Synchronize detailed actions within 500m

Synchronize bullet trajectories within 100m

## 1. High-Level Networking Replication System Introduction

**Replication system** is a set of global strategies for efficiently and reliably synchronizing game states in a distributed environment. Its core goal is not to "simply let players see the same picture", but to:

1. **Anti-packet loss**: Maintain a smooth experience at a packet loss rate of 10%~30%.
2. **Low bandwidth**: Reduce the amount of data through compression and incremental updates.
3. **Low latency**: Prioritize synchronization of critical data to reduce the impact of transmission delay.
4. **State convergence**: Even if packet loss occurs, the client can eventually converge to the correct state.

**Common misunderstandings**:

- **"Reliable transmission = replication system"**: Direct reliance on TCP or reliable UDP (such as ENET) will lead to the following problems:
- Retransmission of outdated data (such as obsolete intermediate states).
- Cumulative confirmation mechanism causes "head-of-line blocking".
- Unable to optimize for game data types (such as position > special effect priority).

## 2. Bandwidth issues

Take UNet FishNet Netcode **Mirror** as an example. Its synchronization logic directly calls three modes of underlying transmission:

1. **Unreliable transmission** (Unreliable): No guarantee of arrival, suitable for non-critical data (such as particle special effects).
2. **Reliable Ordered** (Reliable Ordered): Guaranteed to arrive in order, but retransmission of old data blocks the channel.
3. **Reliable Unordered**: Guaranteed arrival but not order, suitable for independent events (such as scoring).

**Root of the problem**:

- **Stateless snapshot management**: Each time a variable is synchronized, the framework directly calls reliable transmission to send the full amount of data instead of incremental updates.

- **Head of line blocking effect**: If an old data packet is lost, all subsequent data must wait for retransmission, even if they are outdated.

- **Bandwidth waste**: Frequently send intermediate states (such as the position of each frame on the character's movement path) instead of the final target position.

**Example comparison**:

The character moves from A to B

The Mirror process is as follows

Send A→A1→A2→B

All intermediate positions only send the final position of B

Netick UE Photon

Only send the final position of B

Directly send the latest B, ignoring old data

## 1. Delta Snapshots

- The server maintains a complete game state snapshot (Snapshot).

- Each time you synchronize, calculate the difference (Delta) between the current snapshot and the snapshot that the client has confirmed.
- **Only the difference part** is sent, and a sequence number (Sequence ID) is attached to identify the snapshot version.

Even if there is continuous packet loss, the client will eventually receive a complete snapshot

Summary: In the Delta snapshot, we encode a complete snapshot of the entire network game state and compare it with the state that the client has, so we automatically fight against packet loss because we always send everything that has not been confirmed.

## **2. Eventual Consistency**

**Principle**:

- Each network variable (such as player health) is synchronized independently, and updates are sent when marked as "dirty".

- If no confirmation is received, the latest value** (non-historical value) is retransmitted in the next cycle.
- Temporary inconsistency is allowed, but eventually all clients converge to the same state.

**Advantages**:

- **Low overhead**: No need to maintain global snapshots, suitable for scenarios with a large number of dynamic objects.
- **Resilient transmission**: Packet loss is tolerated for non-critical data (such as environmental effects).

Summary: In eventual consistency, we send changes once or twice for network data, and only resend when loss is confirmed. This is worse than Delta snapshots during loss, but it is still better than simply sending things with the reliable ordered option because we only resend the latest state of network variables, not all intermediate values. .

The first approach is used by CSGO, Overwatch, Quake, and many AAA games.

The second is used by Unreal Engine Network and Halo.

## Source:

**《CS:GO Network Architecture Explained》** (Valve Developer Blog)

```javascript
https://developer.valvesoftware.com/wiki/Source_Multiplayer_Networking
```

**Overwatch Network Synchronization GDC Speech**

```javascript
https://www.youtube.com/watch?v=W3aieHjyNvw
```

```javascript
https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-0-documentation?application_version=5.0
```