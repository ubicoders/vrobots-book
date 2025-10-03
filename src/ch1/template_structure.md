# Virtual Robot Node Template (Arduino-style)

This template mirrors the **Arduino** programming model—where `setup()` runs **once** and `loop()` runs **every cycle**—but implemented in Python for a virtual robot.

- **`setup()`**: one-time initialization.
- **`update()`**: called each iteration (the “loop”).

> **Timing:** The simulation backend tops out around **50 Hz**, so the loop sleep is set to **0.02 seconds (20 ms)**.



## Core Class: `VRobotNode`

Implements the Arduino-like lifecycle using `VRobotNodeBase`.

```python
from ubicoders_vrobots_ipc.vrobot_node import VRobotNodeBase, vrobot_client_runner

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId: int = 0):
        super().__init__(sysId)

    # ======================================================
    # Arduino-like structure: setup() and update() (loop)

    # Called once at the beginning
    def setup(self):
        pass

    # Called every iteration
    def update(self):
        if self.read_new_states():
            ts = self.state.timestamp  # Unix time in millis
            pos = self.state.linPos
            elapsed_sec = (ts - self.first_ts) / 1000.0  # seconds
            print(f"State t={elapsed_sec:.3f}s pos=({pos.x:.3f},{pos.y:.3f},{pos.z:.3f})")
    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```



## Main Loop: `vrobot_client_runner()`

Accepts a **list** of nodes for multi-robot (fleet/swarm) control.

```python
import time
from typing import Any, List
from ubicoders_vrobots_ipc.vrobot_client import VRobotClient  # adjust import if needed

def vrobot_client_runner(vrobot_nodes: List[Any]):
    vrclient = VRobotClient()
    for node in vrobot_nodes:
        vrclient.add_vrobot_node(node)
    try:
        while True:
            vrclient.update()
            time.sleep(0.02)  # ~50 Hz (0.02 s = 20 ms)
    finally:
        print("Shutting down...")
        vrclient.shutdown()
```

**Why 0.02 s?** The game engine behind the simulator typically won’t exceed **50 Hz** in the best case. Keeping the sleep at `0.02` seconds keeps your node in sync and avoids wasted CPU.



## Example: One-Time Setup

Use `setup()` to create reusable state and register subscriptions only once.

```python
import cv2

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId: int = 0):
        super().__init__(sysId)

    def setup(self):
        # Example goal for a rover (x, y, z)
        self.position_setpoint = [1.0, 2.0, 3.0]

        # Configure a resizable OpenCV window for the left camera feed
        cv2.namedWindow("CamLeft", cv2.WINDOW_NORMAL)
        cv2.resizeWindow("CamLeft", 640, 360)

        # Subscribe once to the left camera stream published by this vrobot
        self.register_img_subscriber("left")

    def update(self):
        if self.read_new_states():
            # Control logic can reference self.position_setpoint here
            pass
```


## Multi-Robot (Fleet) Example

Instantiate one node per robot with distinct `sysId`s. However, this does not instantiate the robot in the sim, to instantiate the robot, checkout service: 
[ch1/hello_service.html](https://ubicoders.github.io/vrobots-book/ch1/hello_service.html) Therefore, it is recommended to run the control script after defining your mission service.

```python
if __name__ == "__main__":
    nodes = [
        VRobotNode(sysId=0),
        VRobotNode(sysId=1),
        VRobotNode(sysId=2),
    ]
    vrobot_client_runner(nodes)
```

## Where do states and images come from?

The vrobot states data come from the topics published through Zenoh and Iceoryx2. Zenoh transport is used for robot states data. Iceoryx2 shared memory is used to publish the image data by the sim. 

```
Listing topics...
  - [i] vr/0/cams/down/720p
  - [i] vr/0/cams/left/720p
  - [i] vr/0/cams/right/720p
  - [z] vr/0/states
```
Simply, the VRobotNode is subscribing the data specific to its sysId as the topics published by the sim follow the naming convention such as "vr/[sysId]/states"


## Key Points

- **Lifecycle:** `setup()` once → `update()` repeatedly.
- **Fleet-ready:** Pass multiple nodes to `vrobot_client_runner`.
- **Timing:** Use **0.02 s** sleep (~**50 Hz**) for stable sim performance.
- **Setup scope:** Put one-time work (targets, subscriptions, UI windows) in `setup()`.
