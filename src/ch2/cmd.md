# Publishing Command Data in VRobotNode

Publishing command (cmd) data to control a robot is enabled through predefined publishers in **`VRobotNodeBase`**, which use **FlatBuffers** for structured messaging.

In the `"hello-control"` example, the robot receives commands because the node builds and publishes messages in the expected format.

## Example: Sending Multirotor Commands

```python
from ubicoders_vrobots_ipc.vrobot_node import VRobotNodeBase, vrobot_client_runner
import time
from typing import List

...

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId: int = 0):
        super().__init__(sysId)
   ...
    def update(self):       
        ...
        cool_control_result = [1600, 1600, 1600, 1600]
        self.update_cmd_multirotor(cool_control_result)

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```

## Command Message Pipeline

The publishing flow follows three steps:  
**(1) Build command message → (2) Serialize with FlatBuffers → (3) Publish over Zenoh.**

```python
...

    def update_cmd_multirotor(self, pwm: List[int]):
        self.cmdMsgT: CommandMsgT = CommandMsgT()
        self.cmdMsgT.timestamp = time.time() * 1e3
        self.cmdMsgT.cmdId = VROBOTS_CMDS.SET_PWM
        self.cmdMsgT.sysId = self.sysId
        self.cmdMsgT.intArr = [int(pwm[i]) for i in range(4)]
        self.build_and_publish_cmd()

    def build_and_publish_cmd(self):
        builder = flatbuffers.Builder(512)
        os = self.cmdMsgT.Pack(builder)
        builder.Finish(os, bytes("CMD0", "utf-8"))
        payload = builder.Output()
        self.publish_cmd(payload)

    def publish_cmd(self, cmd_data: bytes):
        self.zenoh_node.publish(f"vr/{self.sysId}/cmd", cmd_data)

...
```

## Key Points

- **Correct cmdId is essential:**  
  `self.cmdMsgT.cmdId = VROBOTS_CMDS.SET_PWM` tells the simulator that this is a **multirotor PWM command**.  
  If the `cmdId` is wrong, the simulator ignores the input.

- **System-specific expectations:**  
  Each robot type (`multirotor`, `rover`, etc.) expects different command formats. Even if `sysId` is correct, commands must match the expected type.

- **Timestamping:**  
  The message includes a Unix timestamp in **milliseconds** to synchronize actions.

- **Publishing channel:**  
  Commands are published to:  
  ```
  vr/[sysId]/cmd
  ```
