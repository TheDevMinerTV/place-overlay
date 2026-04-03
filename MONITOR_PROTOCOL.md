# tyles-monitor WebSocket Protocol Specification

## Overview

`tyles-monitor` exposes a WebSocket server that provides realtime completion status for artwork targets on the tyles.place collaborative pixel canvas. Clients connect to receive the current state of all tracked artworks and receive incremental updates as pixels change on the canvas.

## Connection

- **Transport:** WebSocket (RFC 6455)
- **Default endpoint:** `ws://<host>:9400`
- **Subprotocol:** None
- **Message encoding:** JSON over WebSocket text frames

## Message Flow

```
Client                              Server
  |                                    |
  |──── WS handshake ────────────────>|
  |<─── WS handshake response ───────|
  |                                    |
  |<─── "full" message ──────────────|  (sent immediately after connect)
  |                                    |
  |<─── "update" messages ───────────|  (ongoing, debounced ~250ms)
  |<─── "full" messages ─────────────|  (after periodic resync, every ~5min)
  |                                    |
  |──── Ping ────────────────────────>|
  |<─── Pong ────────────────────────|
  |                                    |
  |──── Close ───────────────────────>|
  |<─── Close ───────────────────────|
```

The server is send-only for application data. Clients do not send application messages; only WebSocket control frames (Ping, Close) are handled.

## Message Types

All messages are JSON objects with a `type` field that discriminates the message kind.

### `full` — Complete Status

Sent:
- Immediately when a client connects
- After a periodic canvas resync (~every 5 minutes)
- When the client falls behind on the broadcast channel (lag recovery)

Contains the full state of every tracked artwork.

```json
{
  "type": "full",
  "artworks": [
    {
      "name": "hohenzollern.png",
      "x": 420,
      "y": 6,
      "width": 92,
      "height": 50,
      "total_pixels": 3500,
      "correct_pixels": 3200,
      "completion": 91.43,
      "eta_seconds": 842.5
    }
  ]
}
```

#### Fields: `artworks[]`

| Field            | Type            | Description                                                                 |
|------------------|-----------------|-----------------------------------------------------------------------------|
| `name`           | string          | Artwork identifier (matches the `name` field in `target_config.toml`)       |
| `x`              | integer         | X coordinate of the artwork's top-left corner on the canvas (after offsets) |
| `y`              | integer         | Y coordinate of the artwork's top-left corner on the canvas (after offsets) |
| `width`          | integer         | Width of the artwork source image in pixels                                 |
| `height`         | integer         | Height of the artwork source image in pixels                                |
| `total_pixels`   | integer         | Number of non-transparent target pixels in this artwork                     |
| `correct_pixels` | integer         | Number of canvas pixels currently matching the target                       |
| `completion`     | number          | Completion percentage, rounded to 2 decimal places (0.00 – 100.00)         |
| `eta_seconds`    | number \| absent | Estimated seconds until 100% completion, rounded to 1 decimal place. Absent if not enough data or no progress (see below). `0.0` if already complete. |

Notes:
- `total_pixels` may be less than `width * height` because transparent pixels in the source PNG are excluded.
- `x` and `y` include the global `add-x` / `add-y` offsets from the config. They may be negative if the artwork extends beyond the canvas origin.
- Artworks with `disabled: true` in the config are excluded entirely.
- An artwork with zero target pixels reports `completion: 100.0`.
- `eta_seconds` is computed from a rolling 5-minute window of completion samples. It is **absent** (not `null`) when: there is less than 1 second of sample data, or the artwork is making no forward progress (stable or regressing). Clients should treat a missing `eta_seconds` as "unknown".

### `update` — Incremental Update

Sent when pixel changes on the canvas affect one or more artwork completion counts. Updates are debounced at ~250ms intervals — multiple pixel changes within that window are batched into a single message.

Only artworks whose `correct_pixels` count actually changed are included.

```json
{
  "type": "update",
  "artworks": [
    {
      "name": "hohenzollern.png",
      "correct_pixels": 3201,
      "completion": 91.46,
      "eta_seconds": 812.3
    }
  ]
}
```

#### Fields: `artworks[]`

| Field            | Type            | Description                                                         |
|------------------|-----------------|---------------------------------------------------------------------|
| `name`           | string          | Artwork identifier (same as in `full` messages)                     |
| `correct_pixels` | integer         | Updated count of matching pixels                                    |
| `completion`     | number          | Updated completion percentage, rounded to 2 decimal places          |
| `eta_seconds`    | number \| absent | Estimated seconds until 100% (same rules as in `full`)             |

Notes:
- `update` messages do not include `x`, `y`, `width`, `height`, or `total_pixels`. These values do not change between `full` messages. Clients should merge updates into their local state by matching on `name`.
- After a config reload (every ~5 minutes), the server sends a `full` message instead of an `update`, since artwork definitions may have changed (added, removed, repositioned).

## Client Implementation Guide

### Maintaining State

1. On receiving a `full` message: replace the entire local artwork list.
2. On receiving an `update` message: for each entry, find the matching artwork by `name` and update `correct_pixels` and `completion`.
3. If an `update` references a `name` not in the client's local list, ignore it (a `full` message will follow shortly).

### Reconnection

The server does not implement heartbeats. Clients should:
- Respond to server Ping frames (most WebSocket libraries do this automatically).
- Implement reconnection with backoff on unexpected disconnects.
- Expect a `full` message immediately after each reconnect.

### Example: JavaScript

```javascript
const ws = new WebSocket("ws://localhost:9400");
let artworks = new Map();

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  if (msg.type === "full") {
    artworks.clear();
    for (const a of msg.artworks) {
      artworks.set(a.name, a);
    }
  } else if (msg.type === "update") {
    for (const a of msg.artworks) {
      const existing = artworks.get(a.name);
      if (existing) {
        existing.correct_pixels = a.correct_pixels;
        existing.completion = a.completion;
        existing.eta_seconds = a.eta_seconds; // may be undefined
      }
    }
  }
};
```

## Data Semantics

### Color Matching

A pixel is considered "correct" when its RGB value on the canvas matches the expected RGB value from **any** artwork covering that position. Where artworks overlap, a pixel satisfies all of them if it matches any of their expected colors. Both canvas and artwork pixels are quantized to the nearest palette color using OKLab perceptual distance (matching pixelmaschine's quantization).

### Completion Calculation

```
completion = round(correct_pixels / total_pixels * 100, 2)
```

If `total_pixels` is 0 (artwork has no non-transparent pixels), `completion` is `100.0`.

### ETA Calculation

ETA is computed from a rolling 5-minute window of `(timestamp, correct_pixels)` samples:

```
rate = (newest_correct - oldest_correct) / elapsed_seconds
eta  = remaining_pixels / rate
```

The field is **absent** (not included in the JSON object) when:
- Less than 1 second of sample history exists (e.g., immediately after startup)
- The artwork is not making forward progress (`rate <= 0` — stable or regressing)

When the artwork is already 100% complete, `eta_seconds` is `0.0`.

### Update Frequency

- **Pixel updates:** Debounced at ~250ms. During active canvas editing, clients can expect roughly 4 messages/second at most.
- **Full resyncs:** Every `--reload-interval` seconds (default 300). These correct any accumulated drift and reflect config changes (new artworks, moved artworks, palette changes).
