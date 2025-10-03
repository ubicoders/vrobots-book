# Subscribing to Custom Data

In addition to publishing, you can **subscribe to any arbitrary topic** with a custom node.  
This works independently of the VRobot simulator and lets you process or monitor custom data streams.

## Example: Custom Subscriber Node

```python
from ubicoders_vrobots_ipc.node_zenoh import ZenohNode
from ubicoders_vrobots_ipc.vrobot_node import vrobot_client_runner

class MyOwnVRNode():
    def __init__(self):
        self.setup()

    # Listener callback function for incoming data
    def listener_callback(self, sample):
        topic: str = sample.key_expr
        payload: bytes = sample.payload.to_bytes()
        print(f"Received data: {payload.decode('utf-8')}")

    # Ensure clean shutdown (avoid freezing terminal on Ctrl+C)
    def shutdown(self):
        self.zenoh_node.shutdown()

    # Follow the convention of VRobotNodeBase - setup and update
    def setup(self):
        # Use a negative sysId to avoid conflicts with vrobot nodes
        self.zenoh_node = ZenohNode(-123)

        # Subscribe to a custom topic
        self.zenoh_node.create_subscriber("my/own/topic", self.listener_callback)

    def update(self):
        # Main loop – perform custom logic if needed
        pass

if __name__ == "__main__":
    vrobot_client_runner([MyOwnVRNode()])
```

## Key Points

- **Custom subscription:** You can subscribe to any topic (e.g., `"my/own/topic"`).  
- **Callback-driven:** Define a `listener_callback` function to handle received messages.  
- **Clean exit:** Implement `shutdown()` to gracefully close the Zenoh node and prevent terminal freezes.  
- **Independent of simulator:** Works even if no simulation is running, making it suitable for standalone message handling.
