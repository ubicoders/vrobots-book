# Two cameras at once

Independent streams, independent freshness, and pairing a frame with a state snapshot.

```sh
cargo run -p vrobots-examples --bin ex16_two_cameras
./target/cpp-build/ex16_two_cameras
python examples/python/ex16_two_cameras.py
```

A stereo pair, or a forward camera and a downward one. Each `mount_camera` call adds to
this handle's list and returns its own `CameraStream`, and each stream owns its own reader
thread, its own sequence numbers and its own freshness.

## Mounting both

Both cameras mount at the same resolution, because resolution is one knob for the whole
robot. Asking the second one for a different resolution is refused with
`VrError::InvalidArgument` rather than restarting the first stream under a new name behind
your back. Format is per camera; only resolution is shared.

From `examples/rust/src/bin/ex16_two_cameras.rs`:

```rust
let left = robot.mount_camera(LEFT, RESOLUTION, FORMAT)?;
let right = robot.mount_camera(RIGHT, RESOLUTION, FORMAT)?;
println!("left : {}", left.service_name());
println!("right: {}", right.service_name());
println!("mounted by this handle: {:?}", robot.mounted_cameras());
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex16_two_cameras.cpp</code>)</summary>

```cpp
        vrsdk::CameraStream left = robot.mount_camera(LEFT, RESOLUTION, FORMAT);
        vrsdk::CameraStream right = robot.mount_camera(RIGHT, RESOLUTION, FORMAT);
        std::printf("left : %s\n", left.service_name().c_str());
        std::printf("right: %s\n", right.service_name().c_str());
        std::printf("mounted by this handle:");
        for (const std::array<std::string, 3>& spec : robot.mounted_cameras()) {
            std::printf(" %s(%s_%s)", spec[0].c_str(), spec[1].c_str(), spec[2].c_str());
        }
        std::printf("\n");
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex16_two_cameras.py</code>)</summary>

```python
    left = mr.mount_camera(LEFT, RESOLUTION, FORMAT)
    right = mr.mount_camera(RIGHT, RESOLUTION, FORMAT)
    print(f"left : {left.service_name}")
    print(f"right: {right.service_name}")
    print(f"mounted by this handle: {[str(c) for c in mr.mounted_cameras()]}")
```

</details>

Only the printing of the mounted list differs. `service_name` is a property in Python and a
method in the other two, and C++ has no `CameraSpec` type, so `mounted_cameras()` there hands
back a `std::array<std::string, 3>` of name, resolution and format that the loop formats
itself.

Two service names and the handle's own list, in mount order:

```text
left : vrobots/1/i/cam/left/720p_rgb8
right: vrobots/1/i/cam/right/720p_rgb8
mounted by this handle: [CameraSpec { name: "left", resolution: P720, format: Rgb8 }, CameraSpec { name: "right", resolution: P720, format: Rgb8 }]
```

Each call sends only its own camera, so mounting `right` cannot disturb `left`, and neither
disturbs the scene's `front_left` and `front_right`, which keep streaming throughout. Four
cameras are on the robot by the end of setup, two of them yours.

## Two consumers in one loop

There is no combined "wait for both", by design: the cameras are separate iceoryx2 services
and they render on their own schedules. Two `fresh()` calls in one loop are genuinely
independent, and neither can consume the other's frame.

```rust
while n_left < FRAMES || n_right < FRAMES {
    // Two consumers, each draining its own stream. Neither call can consume
    // the other's frame.
    if let Some(f) = left.fresh() {
        n_left += 1;
        last_left_ns = f.t_ns;
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex16_two_cameras.cpp</code>)</summary>

```cpp
        while (n_left < FRAMES || n_right < FRAMES) {
            // Two consumers, each draining its own stream. Neither call can
            // consume the other's frame.
            if (const std::optional<vrsdk::Frame> f = left.fresh()) {
                ++n_left;
                last_left_ns = f->t_ns();
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex16_two_cameras.py</code>)</summary>

```python
    while n_left < FRAMES or n_right < FRAMES:
        # Two consumers, each draining its own stream. Neither call can consume
        # the other's frame.
        f = left.read()
        if f is not None:
            n_left += 1
            last_left_ns = f.t_ns
```

</details>

Rust and C++ take the frame out of an `Option`, so the `if` both tests and binds. Python calls
`left.read()`, which returns the frame or `None`, and this is the per-stream read that gives
each `CameraStream` its own freshness; `t_ns` is a method in C++ and a field in the other two.

Neither branch prints on every pass: the example reports every twentieth frame per camera.
The loop is paced with `robot.rate(HZ)` at 100 Hz against streams that render at about 60
fps, so most iterations find one stream fresh and the other not.

## Skew between the two

The only honest way to relate two frames is to subtract their capture stamps. Both are on
the same clock, so the difference is a real interval.

```rust
let skew_ms = if last_left_ns == 0 {
    f64::NAN
} else {
    (f.t_ns - last_left_ns) as f64 / 1e6
};
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex16_two_cameras.cpp</code>)</summary>

```cpp
                    const double skew_ms =
                        last_left_ns == 0 ? 0.0
                                          : static_cast<double>(f->t_ns() - last_left_ns) / 1e6;
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex16_two_cameras.py</code>)</summary>

```python
                skew_ms = (
                    float("nan") if last_left_ns == 0 else (f.t_ns - last_left_ns) / 1e6
                )
```

</details>

The subtraction is the same in all three. Only the "no left frame yet" value differs: Rust and
Python report `NaN`, C++ reports `0.0`, so a C++ reader cannot tell that first row from a
genuinely zero skew.

Printed every twentieth frame, alongside the sequence numbers each stream keeps for itself:

```text
L frame 1: seq=8 t=2.104
R frame 1: seq=8 t=2.104  skew_vs_last_left=+0.0 ms
L frame 21: seq=28 t=2.437
R frame 21: seq=28 t=2.437  skew_vs_last_left=+0.2 ms
```

<!-- VERIFY: the seq, t and skew magnitudes above are illustrative and have not been measured against a live run of ex16. -->

Sequence numbers are per stream and start from that stream's own beginning, so `left` and
`right` agreeing on a number means nothing. Compare `t_ns`, never `seq`.

## Pairing a frame with a state

The same rule applies between a camera and the state stream, and it is the more common
case. **The SDK never pairs them for you.** Frames arrive at the render rate, states at 25
Hz, and no frame belongs to any state. What they share is the clock: `Frame::t_ns` and
`State::t_ns` are both nanoseconds on the simulator's unix clock, and `Frame::elapsed` and
`State::elapsed` count from the same epoch, the robot's first state sample.

So fusion code subtracts. From `examples/rust/src/bin/ex03_hello_image.rs`, which reports
the age of each frame against the state snapshot taken in the same iteration:

```rust
let s = robot.states();

// Images are a separate stream with their own timestamps -- never assume
// they match the state's. Compare t_ns explicitly when fusing.
if let Some(frame) = cam.fresh() {
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex03_hello_image.cpp</code>)</summary>

```cpp
            const vrsdk::State s = robot.states();

            // A value only if new since the last read.
            if (auto frame = cam.fresh()) {
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex03_hello_image.py</code>)</summary>

```python
        s = mr.states

        # Images are a separate stream with their own timestamps -- never assume
        # they match the state's. Compare t_ns explicitly when fusing.
        if cam.fresh:
            frame = cam.frame  # metadata for the image we are about to read
```

</details>

Python splits what the other two do in one move: `mr.states` and `cam.fresh` are properties,
and the frame comes from `cam.frame` after the freshness test rather than out of the test
itself.

`(s.t_ns - frame.t_ns) as f64 / 1e6` is that age in milliseconds. For anything that needs a
state at the instant of capture rather than the newest one, keep a short ring of recent
snapshots and pick the one whose `t_ns` is closest to the frame's.

## Cleaning up

Each unmount removes exactly the name it is given, so the order does not matter and the
second call cannot undo the first.

```rust
robot.unmount_camera(LEFT)?;
robot.unmount_camera(RIGHT)?;
println!(
    "unmounted both; mounted list is now {:?}",
    robot.mounted_cameras()
);
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex16_two_cameras.cpp</code>)</summary>

```cpp
        robot.unmount_camera(LEFT);
        robot.unmount_camera(RIGHT);
        std::printf("unmounted both; mounted list is now %zu entries\n",
                    robot.mounted_cameras().size());
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex16_two_cameras.py</code>)</summary>

```python
    mr.unmount_camera(LEFT)
    mr.unmount_camera(RIGHT)
    print(f"unmounted both; mounted list is now {mr.mounted_cameras()}")
```

</details>

The two unmounts are identical. C++ prints the size of the list rather than the list, because
its entries are `std::array<std::string, 3>` with no formatting of their own.

The handle's list is empty afterwards, and the scene's own cameras are untouched:

```text
unmounted both; mounted list is now []
```

**Next:** [Saving a frame](07-saving-frames.md)

**See also:** [Freshness](04-freshness.md), [Formats and resolution](02-formats-and-resolution.md), [More than one robot](../ch08-tooling/06-multi-robot.md)
