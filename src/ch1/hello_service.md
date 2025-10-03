# hello-service

## intro
One can reuqest a serivce to the sim. For instance, the script below is to create a new virtual robot in the sim.

hello_service.py
```py
from ubicoders_vrobots_ipc import req_srv_mission
from ubicoders_vrobots_msgs.M100_mission_generated import Vec3MsgT, MissionMsgT, VRSceneObjectT


if __name__ == "__main__":
    pos = Vec3MsgT()
    pos.x, pos.y, pos.z = 53.6, 180.5, -0.3

    vrobot = VRSceneObjectT()
    vrobot.objectType = "multirotor"
    vrobot.position = pos

    mission = MissionMsgT()
    mission.vrobots =  [vrobot]   
    mission.newMission = False

    req_srv_mission(mission)
    
```