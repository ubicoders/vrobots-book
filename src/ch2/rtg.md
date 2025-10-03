# Publishing RTG Data

You can publish **real-time graph (RTG)** data from a node, and visualize it with the separate **`rtg-sub`** Python app.  
For `rtg-sub` to parse and plot correctly, publish samples in the expected tuple format:

```
(ts_in_seconds, [var1, var2, ...])
```

- **`ts_in_seconds`**: floating-point seconds (not milliseconds).
- **Variables**: a **list** of numeric values (one or more series).

## RTG Publisher in a Node

```python
...
import time
from ubicoders_vrobots_ipc.rtg_pub import RTGPub
...

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId: int = 0):
        super().__init__(sysId)
        ...
        # Initialize RTG publisher (choose a topic/name that rtg-sub subscribes to)
        self.rtg = RTGPub(topic=f"vr/{self.sysId}/rtg/pose")

    def setup(self):
        ...
        self.first_ts = time.time()  # seconds

    def update(self):
        ...
        if self.read_new_states():
            ts = self.state.timestamp            # ms
            pos = self.state.linPos
            elapsed_sec = (ts - self.first_ts * 1e3) / 1000.0  # convert -> seconds

            # Publish RTG sample as (t_seconds, [values...])
            self.rtg.publish(elapsed_sec, [pos.x, pos.y, pos.z])
        ...
```

## `rtg-sub` Subscriber App (Concept)

```python
...
# Inside the rtg-sub app callback/loop
# t: float (seconds), values: List[float]
window.add_sample(float(t), values)
...
```

## Tips

- Ensure time is in **seconds** before publishing (convert from ms if needed).
- Keep the variables in a **list**, even for a single series: `[value]`.
- Use stable, descriptive topics (e.g., `vr/<sysId>/rtg/<signal>`).
