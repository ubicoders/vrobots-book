# hello-rtg

## Real Time Graph (RTG)

run realtime graph

```bash
rtg-sub -c 1
```

plot 3 axis

``` bash
rtg-sub -c 3
```

options

```
rtg-sub --help
```

```
options:
  -h, --help            show this help message and exit
  -c CHANNELS, --channels CHANNELS
                        Number of subplots (channels) [default: 3]
  -f FPS, --fps FPS     Qt redraw rate [default: 50]
  -b BUFFER_SECS, --buffer-secs BUFFER_SECS
                        Time window length in seconds [default: 6.0]
  -t TOPIC, --topic TOPIC
                        Zenoh topic to subscribe to [default: vr/0/rtg]
  --background BACKGROUND
                        Plot background color [default: 'k']
```

## how to plot

The main script needs to publish the realtime values to plot. Keep the terminal running the rtg-sub. Open a new terminal and run hello_states.py after adding 3 lines.


*updated "hello_states.py"*

``` py
from ubicoders_vrobots_ipc.vrobot_node import VRobotNodeBase, vrobot_client_runner
#===================================
# add this line
from ubicoders_vrobots_ipc.rtg_pub import RTGPub 
#===================================

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId:int = 0):
        super().__init__(sysId)

    def setup(self):
        #===================================
        # add this line
        self.rtg = RTGPub(topic_name=f"vr/{self.sysId}/rtg")
        #===================================

    def update(self):       
        if self.read_new_states(): 
            ts = self.state.timestamp
            pos = self.state.linPos
            elapsed_sec = (ts - self.first_ts) / 1000.0
            print(f"State t={elapsed_sec} pos=({pos.x:.2f},{pos.y:.2f},{pos.z:.2f})")
            #===================================
            # add this line
            self.rtg.publish(elapsed_sec, [pos.x, pos.y, pos.z])
            #===================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])
```