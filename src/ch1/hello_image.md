# hello-states

## Intro
This script assumes the simulation is running.

## hello_image

hello_image.py

## additional pip package

This example uses opencv.
```
pip install opencv-python
```


```py
from ubicoders_vrobots_ipc import VRobotNodeBase, vrobot_client_runner, ImageResolution
import cv2

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId:int = 0):
        super().__init__(sysId)

    # ======================================================
    # Like an Arduio script, setup and update (loop)
    
    # Setup Once Here
    def setup(self):
        cv2.namedWindow("CamLeft", cv2.WINDOW_NORMAL)    # make resizable window        
        self.register_img_subscriber("left", ImageResolution.P720)

    # This is called every iteration.
    def update(self):       
        if self.read_new_states(): 
            ts = self.state.timestamp # unix time in millis
            pos = self.state.linPos
            elapsed_sec:float = (ts - self.first_ts) / 1000.0 # in seconds
            print(f"State t={elapsed_sec:.3f} pos=({pos.x:.2f},{pos.y:.2f},{pos.z:.2f})")

        if self.read_new_image("left"):
            img = self.imgStates["left"].image_data
            elapsed_sec:float = (self.imgStates["left"].ts - self.first_ts) /1000.0 # in seconds
            print(f"Image Left t={elapsed_sec:.3f} size=({img.shape[1]}x{img.shape[0]}x{img.shape[2]})")            
            cv2.imshow("CamLeft", img)
            cv2.waitKey(1)

    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])


```
