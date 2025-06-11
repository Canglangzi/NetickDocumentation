# Preface:

I used Mirror, switched to fishnet, and then switched to Nettick. The reason I used them was that I needed to make an authoritative FPS server to prevent cheating. I gave up mirror because it had no prediction (I don’t know how it is used now). Fishnet had prediction, but the prediction function was not perfect at that time, and the examples could not run. Now I switched to netick and I can implement prediction with 200 lines of code. I will describe the network connection workflow in Unity from beginning to end. The reason for not choosing photon is of course the expensive charging policy and the lack of freedom.

Some people say that multiplayer games are difficult for independent games, but in the hight network framework, it can actually become very simple.

The core of multiplayer games is that each computer runs an independent local simulation, and then transmits key data to each other through the network. The so-called server is essentially a computer running a specific logic.

## 1. Your computer is your world:

- Whether you use Unity or Unreal, when your character runs, jumps, and shoots in the game world, **all this happens first on your own computer**. The game engine maintains the state of the entire game world in your local memory (at least the part within your field of view).

- Physics calculation, animation playback, input response, special effects rendering... All these actions that "feel" instantaneous are simulated in real time by your local machine. The engine will not ask the server "Where should I stand now?" every frame.

## 2 Network: Data Porter

- The role played by the network layer (whether it is Unity's Netcode/UNet/Mirror or Unreal's Replication/RPC) is a diligent courier.

- **Responsibilities of the server (another computer):**

- **Running authoritative simulation:** It runs the "only truth" version of the game. It determines the rules (such as whether this skill hits? Can this door be opened?), processes the core logic, and calculates the final result.

- **Collection and distribution:** Collect all operation instructions sent by the client (player computer) (such as "I pressed the W key", "I clicked the left mouse button"), and then calculates the **changes in the world state** (such as the new position of player A, the reduction of player B's health, and the bomb exploded) based on the results of the authoritative simulation.

- **Dispatching data packets:** Package this "authoritative world state snapshot" or "state change increment" and send it to all relevant clients through the "courier" (network).

- ## Three Responsibilities of your computer (client):

- **Sending operations:** Package your local operations (key presses, mouse movements, skill release requests) and send them to the server. **Note: The moment you press the key, the character may have moved locally (prediction), but this requires final confirmation from the server. **

- **Receive updates:** Receive the "express package" (network data packet) sent by the server, which contains the world state confirmed by the server.

- **Apply updates:** **The most critical step! ** Parse the received data packet and use the data in it to **update your local game world state**. For example:

- Apply the new position and rotation of other players sent by the server to their corresponding local models.

- Update your own health sent by the server (may be different from the local prediction and need to be corrected).

- Generate an explosion effect at the location specified by the server.

- **Local prediction and interpolation:** In order to make you feel that the game is smooth (even if there is network delay), the client will:

- **Prediction:** When you press the movement key, without waiting for the server to reply, move your character locally first. If the position replied by the server is different from your prediction, then **position correction** is performed (which may appear as the character "teleporting" for a short time).

- **Interpolation:** For other moving objects (such as other players, NPCs), it will not jump directly to the latest position sent by the server, but will make a smooth transition between two known positions to avoid "jittering".

## **Simple process example (general):**

1. **Player A (Client):** Press the "W" key to move forward.

2. **Player A local:** Immediately move the character according to the input (prediction).

3. **Player A client:** Send a `Command/MoveRequest` (RPC) to the server: "Player A requests to move forward".

4. **Server:** Receive the request and perform authoritative verification (is it legal? Are there any obstacles?). Calculate the new position of player A.

5. **Server:** Update the position of player A in its authoritative world state.

6. **Server:** Package the new position information of player A and send it to all clients (including player A himself) via "express delivery".

7. **Player B (Client):** Receive the server data packet and parse the new position of player A.

8. **Player B local:** **Apply data! ** Move player A's game object to the new position (may smoothly interpolate the transition).

9. **Player A (Client):** Also receives the update of its own position sent by the server. Compare the local predicted position:

- If consistent: everything is fine.

- If inconsistent (for example, the server detects that it hits a wall): **Apply server data! ** Correct its own character position** to the position sent by the server (may cause a small "fallback").
- **The core idea is indeed simple:** Local simulation + network data transmission and reception + application data = the basis of multiplayer games. The server is the computer that runs the core logic.

- **The implementation details are full of challenges:** "Simple" does not mean "easy". How to compress data efficiently? How to deal with network delay (Lag) and jitter (Jitter)? How to design fair prediction and compensation (Reconciliation)? How to prevent cheating? How to synchronize a large number of dynamic objects? How to handle disconnection and reconnection? How to design network topology (dedicated server, listening server, P2P)? These are the real difficulties and the key to distinguishing good and bad network implementations.

- **The role of the engine:** The network framework provided by Unity and Unreal is to solve these "devil's details" and provide a relatively reliable and efficient communication mechanism (such as RPC, state synchronization, object generation/destruction synchronization, prediction and interpolation tools, etc.), so that developers do not have to start from the bottom of the Socket communication to build wheels, and can focus more on the game logic itself.

Application level

# Unity vs Unreal: workflow implementation comparison

| Features | Unity (common solution: Netcode for GameObjects / Mirror) | Unreal Engine (built-in network framework) |
| -------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **Core model** | More flexible. You can choose authoritative server (`ServerRpc`/`ClientRpc`), and also support peer-to-peer (P2P). Commonly used `Server` authoritative model. | **Mandatory authoritative server model**. The server is the only source of truth, and the client can only replicate state/call RPC. |
| **State synchronization** | **Requires explicit definition of synchronization variables:** Use `[NetworkVariable]` (Netcode) or `[SyncVar]` (Mirror) to mark variables that need to be automatically synchronized. Modifications take effect on the server and are automatically synchronized to the client. | **Automatic replication:** Use the `Replicated` attribute to mark variables. When variables are changed on the server, the engine automatically synchronizes to related clients. **More granular control**. |
| **Remote call** | **RPC (Remote Procedure Call):**<br>`ServerRpc`: Client -> Server call. <br>`ClientRpc`: Server -> Client call. | **RPC (Remote Procedure Call):**<br>`Server` function: Client -> Server call. <br>`Client` or `NetMulticast` function: Server -> Client (single/multiple) call. |
| **Object generation** | **Network generation required:** Use `NetworkManager.Spawn` (Netcode) or `NetworkServer.Spawn` (Mirror) to generate on the server and automatically synchronize to the client. | **Network generation required:** Use `SpawnActor` and related network parameters to generate on the server and automatically copy to the client. |
| **Prediction core** | **Requires manual implementation or library dependency:** Netcode provides basic network Transform components. Complex predictions (such as shooting judgment) need to be handled by yourself or use third-party solutions. | **Built-in powerful prediction support: ** `Ability System` (GAS) has good support for skill prediction. `CharacterMovementComponent` has built-in movement prediction. |
| **Network topology** | Flexible support for Dedicated Server, Listen Server, P2P. | Mainly optimize Dedicated Server and Listen Server. P2P support is weak. |