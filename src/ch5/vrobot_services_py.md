# python scripts



## 1. Reset a vrobot

req_srv_reset(sysId:int)

```py
from ubicoders_vrobots_ipc import req_srv_reset

if __name__ == "__main__":
    req_srv_reset(sysId=0)
```

## 2. Reset all vorobots
req_srv_reset_all()
```py
from ubicoders_vrobots_ipc import req_srv_reset_all

if __name__ == "__main__":
    req_srv_reset_all()
```

## 3. Set property of a vrobot

```py
from ubicoders_vrobots_ipc import req_srv_mission, req_srv_physical_property
from ubicoders_vrobots_msgs.S007_srv_vrobotphysicalpropertymsg_generated import Vec3MsgT, SrvVRobotPhysicalPropertyMsgT
import time

if __name__ == "__main__":
    prop = SrvVRobotPhysicalPropertyMsgT()
    prop.timestamp = time.time() * 1e3
    prop.setMass = 3
    prop.setMoi3x1 = [1, 1, 1]
    req_srv_physical_property(prop)
```
## 4. Mission of the stage.

create a new vrobot
```py
from ubicoders_vrobots_ipc import req_srv_mission
from ubicoders_vrobots_msgs.M100_mission_generated import Vec3MsgT, MissionMsgT, VRSceneObjectT
import time

if __name__ == "__main__":
    pos = Vec3MsgT()
    pos.x, pos.y, pos.z = 53.6, 180.5, -0.3

    vrobot = VRSceneObjectT()
    vrobot.objectType = "multirotor" # "heli", "car", "omrover"
    vrobot.position = pos

    mission = MissionMsgT()
    mission.vrobots =  [vrobot]   
    mission.newMission = False # True = delete all vrobots currently instantiated.

    req_srv_mission(mission)
```

mission service message defnition in flatbuffers.

```bash
include "vectors.fbs";

table VRSceneObject{
    object_type:string;
    name:string;
    scale:float;

    // position, rotation, etc
    position:Vec3Msg;
    eul:Vec3Msg;
    angvel:Vec3Msg;
    linvel:Vec3Msg;
}

table Quest{
    id:uint;
    name:string;
    type:string; // collision, move nvr object, move vrobot, etc

    target:VRSceneObject;
    is_completed:bool=false;
}


table MissionMsg{
    id:uint;
    name:string;
    timestamp:double;

    // new mission or continue from previous
    new_mission:bool=false;

    // scene name
    main_scene:string;
    other_scenes:[string];

    // main camera position, rotation
    main_camera_position:Vec3Msg;
    main_camera_eul:Vec3Msg;

    clear_nvr_objects:bool=false;
    nvr_objects:[VRSceneObject];
    clear_vrobots:bool=false;
    vrobots:[VRSceneObject];
    quests:[Quest];    
}

root_type MissionMsg;
file_identifier "M100";
```

More docs coming soon.