# Publishing Custom Data

Beyond simulation control, you can publish **any arbitrary data** on any topic name.  
This can be completely **independent of the VRobot simulator** and is useful for custom messaging, debugging, or integrating with other apps.

## Example: Custom Node

```python
from ubicoders_vrobots_ipc.node_zenoh import ZenohNode
from ubicoders_vrobots_ipc.vrobot_node import vrobot_client_runner

class MyOwnVRNode():
    def __init__(self):
        self.setup()

    # Following the convention of VRobotNodeBase: setup + update
    def setup(self):
        # Use a negative sysId to avoid conflicts with real vrobot nodes
        self.zenoh_node = ZenohNode(-123)
        
        # Create a publisher for a custom topic
        self.zenoh_node.create_publisher("my/own/topic")

    def update(self):
        # Publish arbitrary payload (raw bytes)
        self.zenoh_node.publish("my/own/topic", b"hello from my own node")

if __name__ == "__main__":
    vrobot_client_runner([MyOwnVRNode()])
```

## Verifying with Topic Listing

Run `list-topics` in another terminal:

```
Listing topics...
...
  - [z] my/own/topic
```

## Key Points

- **Unique sysId:** Use a negative (or otherwise unused) sysId for standalone publishers to avoid conflicts with simulation nodes.  
- **Custom topics:** You can name topics however you like (e.g., `"my/own/topic"`).  
- **Independent publishing:** This mechanism works outside the VRobot simulator, making it suitable for general data pipelines or external monitoring.
