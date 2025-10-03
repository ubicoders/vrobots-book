# Subscribing robot data

## Reading the states


```py

from ubicoders_vrobots_ipc.vrobot_node import VRobotNodeBase, vrobot_client_runner

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
            ts = self.state.timestamp
            pos = self.state.linPos
            elapsed_sec = (ts - self.first_ts) / 1000.0
            print(f"State t={elapsed_sec} pos=({pos.x:.2f},{pos.y:.2f},{pos.z:.2f})")

    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```

Once state data is availabe in the update() loop, one can read the state values listed below.

For instance, state.linPos is type of Vec3() which is made of Vec3, which again has x, y, z as memeber variables. Therefore to read x, y, z position, one can simply access by

```py
x = self.state.linPos.x
y = self.state.linPos.y
z = self.state.linPos.z
```

## available states

```py
name = string
sysId = int
timestamp = float
linAcc = Vec3()
linVel = Vec3()
linPos = Vec3()
altitude = float
angAcc = Vec3()
angVel = Vec3()
euler = Vec3()
eulerDot = Vec3()
quaternion = Vec4()
pwm = []
actuators = []
force = Vec3()
torque = Vec3()
accelerometer = Vec3()
gyroscope = Vec3()
magnetometer = Vec3()
mass = float
cg = Vec3()
moi3x1 = Vec3()
extraProps = []
collisions = []
```

```
class Vec4(Vec4MsgT):
    def __init__(self,):
        self.x = 0
        self.y = 0
        self.z = 0
        self.w = 0

class Vec3(Vec3MsgT):
    def __init__(self):
        self.x = 0
        self.y = 0
        self.z = 0
```




