# Visualizing Data with Rerun

[Rerun](https://www.rerun.io/) is an open-source visualization tool built for robotics, computer vision, and multimodal data. Instead of juggling separate plotting windows, image viewers, and 3D tools, Rerun lets you stream everything into a single viewer — **images**, **time-series plots**, **3D poses**, point clouds, and more — all synchronized on a shared timeline.

In VRobots, we use Rerun inside a node to visualize robot state as it runs. The workflow is straightforward:

1. **Initialize** a Rerun recording session with `rr.init(...)` and `spawn=True` to launch the viewer automatically.
2. **Define a layout** using Rerun's blueprint API (`rerun.blueprint`) so your panels are arranged the way you want on startup.
3. **Log data** each iteration with `rr.log(...)` — images, scalars, 3D transforms, points, arrows, etc. Rerun handles the rest.

## Installing Rerun

Before running any visualization code, you must install the [Rerun SDK](https://www.rerun.io/). Rerun is a powerful open-source tool for visualizing multimodal data — it handles images, 3D poses, time-series plots, and more, all in a single viewer. We use it extensively to debug and inspect robot state in real time.

Install the **exact version** required by this project:

```bash
pip install rerun-sdk==0.27.3
```

> **Why pin the version?** The Rerun SDK API can change between releases. Using `0.27.3` ensures that the code examples in this chapter (e.g., `rr.Scalars`, `rr.Image`, `rr.Transform3D`) work without modification. If you install a different version, you may encounter breaking API changes.

## Example: Stereo Camera + Pose Visualization

```python
from ubicoders_vrobots_ipc import VRobotNodeBase, vrobot_client_runner, ImageResolution
import rerun as rr
import rerun.blueprint as rrb
import numpy as np

class VRobotNode(VRobotNodeBase):
    def __init__(self, sysId:int = 0):
        super().__init__(sysId)

    # ======================================================
    # Like an Arduio script, setup and update (loop)
    
    # Setup Once Here
    def setup(self):
        self.register_img_subscriber("left", ImageResolution.P720)
        self.register_img_subscriber("right", ImageResolution.P720)

        rr.init("StereoVO", spawn=True)
        rr.log("world", rr.ViewCoordinates.RIGHT_HAND_Y_DOWN, static=True)
        blueprint = rrb.Blueprint(
            rrb.Vertical(
                rrb.Horizontal(
                    rrb.Spatial2DView(
                    origin="/image_left",
                    name="Left Image",
                ),
                rrb.Spatial2DView(
                    origin="/image_right",
                    name="Right Image",
                ),
                ),
                rrb.Horizontal(
                    
                    rrb.Vertical(
                        rrb.TimeSeriesView(origin="/plot/x", name="Position X"),
                        rrb.TimeSeriesView(origin="/plot/y", name="Position Y"),
                        rrb.TimeSeriesView(origin="/plot/z", name="Position Z"),
                    ),
                    rrb.Spatial3DView(
                        origin="/",
                        background=rrb.Background(kind=rrb.BackgroundKind.SolidColor, color=[0, 0, 0]),
                        name="Pose",
                    ),
                    column_shares=[2, 3] # Give more width to the map
                ),
                row_shares=[1, 2] # Give more height to the bottom section
            ),
            collapse_panels=False,
        )
        rr.send_blueprint(blueprint)



    # This is called every iteration.
    def update(self):       
        if self.read_new_states(): 
            ts = self.state.timestamp # unix time in millis
            pos = self.state.linPos
            vel = self.state.linVel
            elapsed_sec:float = (ts - self.first_ts) / 1000.0 # in seconds
            print(f"State t={elapsed_sec:.3f} pos=({pos.x:.2f},{pos.y:.2f},{pos.z:.2f})")
            rr.log("plot/x", rr.Scalars(vel.x))
            rr.log("plot/y", rr.Scalars(vel.y))
            rr.log("plot/z", rr.Scalars(vel.z))
            # plot the pose as a 3D point with orientation for /pose 3d view
            eul = np.array([self.state.euler.x, self.state.euler.y, self.state.euler.z])
            # Euler XYZ to rotation matrix (R = Rz @ Ry @ Rx)
            ex, ey, ez = eul
            cx, sx = np.cos(ex), np.sin(ex)
            cy, sy = np.cos(ey), np.sin(ey)
            cz, sz = np.cos(ez), np.sin(ez)
            rot = np.array([
                [cy*cz, cz*sx*sy - cx*sz, sx*sz + cx*cz*sy],
                [cy*sz, cx*cz + sx*sy*sz, cx*sy*sz - cz*sx],
                [-sy,   cy*sx,            cx*cy            ],
            ])
            rr.log("/pose", rr.Transform3D(
                translation=(pos.x, pos.y, -pos.z),
                mat3x3=rot,
            ))
            rr.log("/pose/point", rr.Points3D(
                positions=[[0, 0, 0]],
                radii=[0.05],
                colors=[[255, 255, 0]],
            ))
            rr.log("/pose/axes", rr.Arrows3D(
                origins=[[0, 0, 0], [0, 0, 0], [0, 0, 0]],
                vectors=[[2, 0, 0], [0, 2, 0], [0, 0, -2]],
                colors=[[255, 0, 0], [0, 255, 0], [0, 0, 255]],
            ))
            

        if self.read_new_image("left"):
            img = self.imgStates["left"].image_data
            elapsed_sec:float = (self.imgStates["left"].ts - self.first_ts) /1000.0 # in seconds
            print(f"Image Left t={elapsed_sec:.3f} size=({img.shape[1]}x{img.shape[0]}x{img.shape[2]})")   
            rgb_img = img[:, :, ::-1] # BGR to RGB
            rr.log("image_left", rr.Image(rgb_img))         

        if self.read_new_image("right"):
            img = self.imgStates["right"].image_data
            elapsed_sec:float = (self.imgStates["right"].ts - self.first_ts) /1000.0 # in seconds
            print(f"Image Right t={elapsed_sec:.3f} size=({img.shape[1]}x{img.shape[0]}x{img.shape[2]})")   
            rgb_img = img[:, :, ::-1] # BGR to RGB
            rr.log("image_right", rr.Image(rgb_img))         
            


    # ======================================================

if __name__ == "__main__":
    vrobot_client_runner([VRobotNode(sysId=0)])



```

## Key Rerun Concepts Used

- **`rr.init("name", spawn=True)`** — starts a recording session and launches the Rerun viewer window.
- **`rr.log(path, data)`** — logs data to a hierarchical path (e.g., `"plot/x"`, `"/pose"`). Rerun organizes everything in a tree, so related data under the same prefix is grouped together.
- **Blueprints** (`rerun.blueprint`) — let you define the viewer layout in code: which panels to show, how to split them, and what data each panel displays. Without a blueprint, Rerun uses auto-layout.
- **`rr.Image`** — logs a 2D image (expects RGB).
- **`rr.Scalars`** — logs a scalar value for time-series plots.
- **`rr.Transform3D`** — sets the position and orientation of an entity in 3D space.
- **`rr.Points3D` / `rr.Arrows3D`** — draws points and arrows in a 3D view, useful for visualizing coordinate axes or landmarks.
