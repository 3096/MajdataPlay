# MajdataPlay Network Test Port Plan

## Goal

Add a network-facing port so a test client can:

1. Connect to MajdataPlay over the network.
2. Synchronize clocks with the running instance.
3. Send control commands (load, play, pause, stop, reset, etc.).
4. Stream input events to be played inside MajdataPlay as if they came from real hardware.

## Recommended Protocol: WebSocket

The project already embeds and uses `WebSocketSharp-netstandard` for this exact kind of tooling. The existing viewer server is in:

- [Assets/Scripts/Scenes/View/WsServer.cs](Assets/Scripts/Scenes/View/WsServer.cs)

It currently serves a state-only WebSocket at `ws://127.0.0.1:8083/majdata`. We should extend this channel to carry both control commands and scheduled input frames, rather than add a second server stack.

### Why WebSocket

| Option | Verdict |
|---|---|
| **WebSocket over TCP** | Already a dependency, reliable ordered framing, works on IL2CPP / Android, trivial to test from Python/Node/browser. |
| Raw TCP | Must hand-roll length-prefix framing for no real benefit. |
| HTTP/REST (`Doc/proposal.md`) | Fine for one-off uploads (`POST /api/load` with multipart), but no server push, no streaming, and too much overhead for 100+ Hz input. |
| UDP / OSC | Only worthwhile for sub-frame one-way latency over real WiFi. For a test harness, dropped packets produce false test failures. |
| gRPC | HTTP/2 + codegen overhead; overkill for this use case. |

> **Implementation note:** Set `NoDelay = true` on the socket so Nagle's algorithm does not add ~40 ms stalls on tiny input frames.

## Time Synchronization

Do not rely on a single `GET /api/timestamp` to compute clock offset, because it cannot separate network delay from clock offset. Use an NTP-style 4-timestamp exchange repeated several times:

```
Client sends  t1
Server echoes t1 and adds t2, t3
Client records t4
```

$$
\theta = \frac{(t_2 - t_1) + (t_3 - t_4)}{2} \qquad
\delta = (t_4 - t1) - (t_3 - t_2)
$$

- $\theta$: clock offset to add to local time to get server time.
- $\delta`: round-trip delay.

Run ~20 probes and keep the sample with the minimum $\delta$ as the best offset.

### Important Scheduling Rule

The client should send input events **with a future server timestamp** (30–50 ms ahead on loopback). The server buffers them and only dispatches each event at its scheduled time. This makes jitter inside the lookahead window irrelevant.

The authoritative clock to expose over the wire is `MajTimeline.UnscaledTime`, because that is the value the existing input pipeline stamps onto `InputDeviceReport`.

## Input Injection Point

The input pipeline already supports synthetic injection:

- [Assets/Scripts/IO/InputManager/InputManager.cs](Assets/Scripts/IO/InputManager/InputManager.cs) uses two `ConcurrentQueue<InputDeviceReport>` buffers:
  - `_buttonRingInputBuffer`
  - `_touchPanelInputBuffer`
- [Assets/Scripts/IO/Base/InputDeviceReport.cs](Assets/Scripts/IO/Base/InputDeviceReport.cs) is a small struct: `{ Index, State, Timestamp }`.
- [Assets/Scripts/IO/InputManager/RawDeviceHandle/InputManager.DummyInput.cs](Assets/Scripts/IO/InputManager/RawDeviceHandle/InputManager.DummyInput.cs) already demonstrates enqueuing fake reports.

However, the dequeue paths (e.g. [InputManager.DequeueButton.cs](Assets/Scripts/IO/InputManager/InputManager.DequeueButton.cs)) drain the queue every frame and **ignore** `report.Timestamp`. Therefore the scheduling logic must sit **upstream**: keep remote events in a sorted pending list and enqueue them into the `ConcurrentQueue` exactly when `MajTimeline.UnscaledTime` reaches the event's due time.

## Proposed Wire Format

### Control Plane (text JSON)

Reuse the existing `MajWsRequestBase` / `MajWsResponseBase` envelope used by `MajdataWsService`.

#### Time sync request

```json
{
  "requestType": "TimeSync",
  "requestData": { "t1": 1234567890123 }
}
```

#### Time sync response

```json
{
  "responseType": "TimeSync",
  "responseData": {
    "t1": 1234567890123,
    "t2": 1234567890130,
    "t3": 1234567890131
  }
}
```

#### Play command (scheduled)

```json
{
  "requestType": "Play",
  "requestData": {
    "startAt": 0,
    "speed": 1.0,
    "atTicks": 987654321000
  }
}
```

`atTicks` is the server `MajTimeline.UnscaledTime` at which playback should begin. The server waits until that time before starting.

### Input Plane (binary frames)

Use binary WebSocket frames for the high-frequency input stream to avoid JSON parse overhead.

```
Byte   Field
0      kind = 0x01 (input batch)
1      count (number of events in this frame)
2…     events, 12 bytes each

Event layout (little-endian):
  Offset  Size  Type    Meaning
  0       1     u8      device: 0 = buttonRing, 1 = touchPanel
  1       1     u8      index: button 0-11 or sensor 0-33
  2       1     u8      state: 0 = Off, 1 = On
  3       1     —       reserved / padding
  4       8     i64     dueTicks (server MajTimeline.UnscaledTime ticks)
```

A 64-event burst is well under 1 KB.

## State Diagram

The existing viewer states from `Doc/proposal.md` remain valid:

```
                                      ^-maidata-v----> [Error] -maidata--v
                                      |v--------<------------------------<
[Idle] -load-> [Loaded] -maidata-> [Ready] -play-> [Playing] -pause-> [Paused] -resume -> |
                                      ^                ^     -stop------>|                |
                                      <----------------|-----------------v                |
                                                       ^----------------------------------<
```

## Suggested Next Steps

1. Add a configurable listen address/port for the WebSocket server (currently hard-coded to `127.0.0.1:8083`).
2. Implement the NTP-style `TimeSync` request/response pair in `MajdataWsService`.
3. Add a `RemoteInputScheduler` class that:
   - holds incoming binary input events in a sorted buffer,
   - dispatches them into `_buttonRingInputBuffer` / `_touchPanelInputBuffer` when due,
   - drops events whose due time has already passed.
4. Extend `MajdataWsService.OnMessage` to parse binary input batches and forward them to the scheduler.
5. Expose a small Python test client that performs clock sync, sends a `Play` command, and replays a recorded input file.

## Files to Touch

- [Assets/Scripts/Scenes/View/WsServer.cs](Assets/Scripts/Scenes/View/WsServer.cs) — extend the WebSocket service.
- [Assets/Scripts/IO/InputManager/InputManager.cs](Assets/Scripts/IO/InputManager/InputManager.cs) or a new `RemoteInputScheduler.cs` — scheduled remote input source.
- [Assets/Scripts/IO/Base/InputDeviceReport.cs](Assets/Scripts/IO/Base/InputDeviceReport.cs) — already the right shape; may not need changes.
- Settings schema / `ProjectSettings` — if the port should be user-configurable.
