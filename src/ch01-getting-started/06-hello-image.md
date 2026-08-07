# Hello image

Mount a camera, read frames as they arrive, and unmount it before you exit.

```sh
cargo run -p vrobots-examples --bin ex03_hello_image
./target/cpp-build/ex03_hello_image
python examples/python/ex03_hello_image.py
```

## Mounting a camera

`mount_camera` creates a camera of your choosing on the robot and subscribes to its
iceoryx2 stream in a single call. The name, resolution and format are constants, as they
are in every example.

From `examples/rust/src/bin/ex03_hello_image.rs`:

```rust
const SYS_ID: u32 = 1; // the multirotor in the test scene
const CAMERA: &str = "left";
const RESOLUTION: &str = "720p";
const FORMAT: &str = "rgb8";
const FRAMES: u64 = 120; // then unmount and exit
const HZ: f64 = 100.0;

fn main() -> Result<(), VrError> {
    // ===== setup =====
    vrobots_sdk::init_logging("info");
    let robot = VirtualRobot::connect(RobotType::Multirotor, Some(SYS_ID))?;

    // mount_camera CREATES the camera on the robot (srv/cameras) and subscribes
    // to its iox2 stream. Use open_camera(...) to subscribe to an existing one
    // without mutating the sim.
    let cam = robot.mount_camera(CAMERA, RESOLUTION, FORMAT)?;
    println!("camera stream: {}", cam.service_name());

    let mut seen = 0u64;
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex03_hello_image.cpp</code>)</summary>

```cpp
constexpr std::uint32_t SYS_ID = 1;  // the multirotor in the test scene
constexpr const char* CAMERA = "left";
constexpr const char* RESOLUTION = "720p";
constexpr const char* FORMAT = "rgb8";
constexpr std::uint64_t FRAMES = 120;  // then unmount and exit
constexpr double HZ = 100.0;

int main() {
    try {
        // ===== setup =====
        vrsdk::check_version();
        vrsdk::VirtualRobot robot(vrsdk::RobotType::Multirotor, SYS_ID);
        robot.connect();

        // mount_camera CREATES the camera on the robot (srv/cameras) and
        // subscribes to its iox2 stream. Use open_camera(...) to subscribe to an
        // existing one without mutating the sim.
        vrsdk::CameraStream cam = robot.mount_camera(CAMERA, RESOLUTION, FORMAT);
        std::printf("camera stream: %s\n", cam.service_name().c_str());

        std::uint64_t seen = 0;
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex03_hello_image.py</code>)</summary>

```python
SYS_ID = 1  # the multirotor in the test scene
CAMERA = "left"
RESOLUTION = "720p"
FORMAT = "rgb8"
FRAMES = 300  # then unmount and exit
HZ = 100


def main() -> None:
    # ===== setup =====
    vrsdk.init_logging("info")
    mr = VirtualRobot(RobotType.MULTIROTOR, sys_id=SYS_ID)
    mr.connect()

    # mount_camera CREATES the camera on the robot (srv/cameras) and subscribes
    # to its iox2 stream. Use open_camera(...) to subscribe to an existing one
    # without mutating the sim.
    cam = mr.mount_camera(CAMERA, RESOLUTION, FORMAT)
    print(f"camera stream: {cam.service_name}")

    seen = 0
```

</details>

The printed name is the iceoryx2 service, and it matches a row of `vrobots topic list`
character for character:

```text
camera stream: vrobots/1/i/cam/left/720p_rgb8
```

Mounting **adds** one camera. The `front_left` and `front_right` the test scene ships keep
streaming right through the run, and cameras other clients mounted are untouched.
`open_camera` is the read-only alternative that attaches to an existing stream without
mutating the simulator at all.

## The loop, and what fresh() means

Images and states are two independent streams. The image half of the loop runs once per
frame while the state half runs every iteration, and the code says which by branching on
`fresh()`.

```rust
    // ===== loop =====
    while seen < FRAMES {
        let s = robot.states();

        // Images are a separate stream with their own timestamps -- never assume
        // they match the state's. Compare t_ns explicitly when fusing.
        if let Some(frame) = cam.fresh() {
            // Some only if new since the last read
            seen += 1;
            println!(
                "Image {} t={:.3} size=({}x{}) seq={} lag_vs_state={:.1} ms",
                frame.camera_name,
                frame.elapsed,
                frame.width,
                frame.height,
                frame.seq,
                (s.t_ns - frame.t_ns) as f64 / 1e6
            );
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex03_hello_image.cpp</code>)</summary>

```cpp
        // ===== loop =====
        while (seen < FRAMES) {
            const vrsdk::State s = robot.states();

            // A value only if new since the last read.
            if (auto frame = cam.fresh()) {
                ++seen;
                std::printf("Image %s t=%.3f size=(%ux%u) seq=%llu lag_vs_state=%.1f ms\n",
                            frame->camera_name.c_str(), frame->elapsed(), frame->width(),
                            frame->height(),
                            static_cast<unsigned long long>(frame->seq()),
                            static_cast<double>(s.t_ns - frame->t_ns()) / 1e6);
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex03_hello_image.py</code>)</summary>

```python
    # ===== loop =====
    while seen < FRAMES:
        s = mr.states

        # Images are a separate stream with their own timestamps -- never assume
        # they match the state's. Compare t_ns explicitly when fusing.
        if cam.fresh:
            frame = cam.frame  # metadata for the image we are about to read
            img = cam.image  # numpy (h, w, c) uint8, top-down, RGB(A)
            seen += 1

            print(
                f"Image {frame.camera_name} t={frame.elapsed:.3f} "
                f"size=({frame.width}x{frame.height}) seq={frame.seq} "
                f"lag_vs_state={(s.t_ns - frame.t_ns) / 1e6:.1f} ms"
            )
```

</details>

Rust and C++ ask and receive in one move, so the frame arrives inside an `Option` that the
`if` unwraps. Python splits it: `cam.fresh` is a boolean property and `cam.image` is the
numpy array, and reading the image is what consumes the freshness.

```text
State t=0.410 (no new frame)
Image left t=0.412 size=(1280x720) seq=17 lag_vs_state=12.4 ms
      sky-ness (B-R) top=+98 bottom=-25 (top-down: sky above ground), fov_y=61.9 deg
State t=0.420 (no new frame)
```

`fresh()` returns `Some` only when a frame has arrived since the last call, and it hands
each frame out exactly once. That makes it the right read for work that must not run twice
on one image. Its counterpart `latest()` returns the current frame regardless and does not
consume freshness.

The `lag_vs_state` figure is why the two streams are never paired by the SDK. Frames
arrive at the render rate and states at 25 Hz, so no frame belongs to any state. Both
`t_ns` values are on the same clock, so subtracting them is meaningful, and doing that
subtraction explicitly is what fusion looks like here.

> **Gotcha.** Camera frames ride iceoryx2 shared memory, so they are **same-host only**.
> zenoh will happily reach a simulator on another machine and deliver states, and not one
> frame will follow. There is no error: `mount_camera` times out after
> `ConnectOptions::camera_timeout`, five seconds by default.

Rows in `frame.data` are row-major and **top-down**: row 0 is the top of the picture. The
wire is bottom-up, following Unity's render order, and the SDK flips while copying. The
sky-ness figures are the check on that, measuring blue minus red on the first and last
rows: outdoors the top row is sky and the bottom is ground. Channel order is the
renderer's own RGB, never swapped, so OpenCV users convert to BGR at the call site.

## Unmounting before you exit

The loop is bounded by `FRAMES` rather than running forever, and that is the whole reason:
Ctrl-C would skip the cleanup and leave the camera mounted on a robot that outlives the
process.

```rust
    // Leave the sim as we found it: this removes `left` by name, so the cameras
    // the scene mounted are exactly as they were before the run.
    robot.unmount_camera(CAMERA)?;
    let stats = cam.stats();
    println!(
        "unmounted {CAMERA}; {seen} frame(s), received={} decode_errors={} seq_gaps={}",
        stats.received, stats.decode_errors, stats.seq_gaps
    );
    Ok(())
}
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex03_hello_image.cpp</code>)</summary>

```cpp
        // Leave the sim as we found it: this removes "left" by name, so the
        // cameras the scene mounted are exactly as they were before the run.
        robot.unmount_camera(CAMERA);
        const vrsdk_camera_stats_t st = cam.stats();
        std::printf("unmounted %s; %llu frame(s), received=%llu decode_errors=%llu seq_gaps=%llu\n",
                    CAMERA, static_cast<unsigned long long>(seen),
                    static_cast<unsigned long long>(st.received),
                    static_cast<unsigned long long>(st.decode_errors),
                    static_cast<unsigned long long>(st.seq_gaps));
        return 0;
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex03_hello_image.py</code>)</summary>

```python
    # Leave the sim as we found it: this removes `left` by name, so the cameras
    # the scene mounted are exactly as they were before the run.
    mr.unmount_camera(CAMERA)
    st = cam.stats
    print(
        f"unmounted {CAMERA}; {seen} frame(s), received={st.received} "
        f"decode_errors={st.decode_errors} seq_gaps={st.seq_gaps}"
    )
```

</details>

```text
unmounted left; 120 frame(s), received=120 decode_errors=0 seq_gaps=0
```

That count is `FRAMES`, so the Python run prints 300 rather than 120: it opens a window and
wants more of them. `unmount_camera` removes exactly the name it is given and nothing else,
which is what makes "leave the simulator as you found it" literally true here. Dropping the `CameraStream`
would stop the reader thread but would **not** unmount the camera in the simulator.

**Next:** [Hello service](07-hello-service.md)

**See also:** [Mount, open and unmount](../ch05-cameras/01-mount-open-unmount.md), [Freshness](../ch05-cameras/04-freshness.md), [Inside a frame](../ch05-cameras/03-frames.md), [Showing frames in a window](../ch05-cameras/08-showing-frames.md)
