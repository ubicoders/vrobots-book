# Subscribing to Images in VRobotNode

To process camera feeds from the simulator, a `VRobotNode` must explicitly register image subscribers.  
This is done using:

```python
self.register_img_subscriber("left")
```

## Topic Naming Convention

The VRobots simulator publishes image data in the format:

```
vr/[sysId]/cams/[side]/720p
```

- **`sysId`** → Robot system ID (e.g., `0`, `1`, `2`).
- **`side`** → Camera identifier (e.g., `left`, `right`, `down`).

Example published topics:

```
[i] vr/0/cams/down/720p
[i] vr/0/cams/left/720p
[i] vr/0/cams/right/720p
[z] vr/0/states
```

Each robot may have a different set of cameras. For instance, the `"multirotor"` currently provides three: **`left`**, **`right`**, and **`down`**.  
When registering subscribers, you must use the correct **camera side name**.



## Example: Image Subscriber with OpenCV

```python
from ubicoders_vrobots_ipc.vrobot_node import VRobotNodeBase, vrobot_client_runner
import cv2

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId: int = 0):
        super().__init__(sysId)

    # ======================================================
    # Arduino-like lifecycle: setup() and update()

    # One-time setup
    def setup(self):
        # OpenCV window setup
        cv2.namedWindow("CamLeft", cv2.WINDOW_NORMAL)  # resizable window
        cv2.resizeWindow("CamLeft", 640, 360)

        # Subscribe to left camera stream
        self.register_img_subscriber("left")

    # Called every iteration
    def update(self):
        #  State updates 
        if self.read_new_states():
            ts = self.state.timestamp  # Unix time (ms)
            pos = self.state.linPos
            elapsed_sec: float = (ts - self.first_ts) / 1000.0
            print(f"State t={elapsed_sec:.3f} pos=({pos.x:.2f},{pos.y:.2f},{pos.z:.2f})")

        #  Image updates 
        if self.read_new_image("left"):
            img = self.imgStates["left"].image_data
            elapsed_sec: float = (self.imgStates["left"].ts - self.first_ts) / 1000.0
            print(f"Image Left t={elapsed_sec:.3f} size=({img.shape[1]}x{img.shape[0]}x{img.shape[2]})")

            # Display live feed
            cv2.imshow("CamLeft", img)
            cv2.waitKey(1)

    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```



## Key Points

- **Subscription naming:** Match the correct camera `side` (`left`, `right`, `down`).
- **One-time registration:** Call `register_img_subscriber()` in `setup()`.
- **Live display:** Use OpenCV (`imshow`, `waitKey`) to show images in real-time.
- **Integration:** Image reading (`read_new_image()`) and state reading (`read_new_states()`) can run in parallel each iteration.
