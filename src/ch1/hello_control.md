# hello-control


hello_control.py

```py
from ubicoders_vrobots_ipc import VRobotNodeBase, vrobot_client_runner

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId:int = 0):
        super().__init__(sysId)

    # ======================================================
    # Like an Arduio script, setup and update (loop)
    
    # Setup Once Here
    def setup(self):
        pass

    # This is called every iteratoin.
    def update(self):       
        if self.read_new_states(): 
            ts = self.state.timestamp # unix time in millis
            pos = self.state.linPos
            elapsed_sec = (ts - self.first_ts) / 1000.0 # in seconds
            print(f"State t={elapsed_sec} pos=({pos.x:.3f},{pos.y:.2f},{pos.z:.2f})")

            # Do some COOL control here and publish
            cool_control_result = [1501, 1501, 1501, 1501]
            self.update_cmd_multirotor(cool_control_result)
 
    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```