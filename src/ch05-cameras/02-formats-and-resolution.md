# Formats and resolution

The resolutions and pixel formats on offer, and the one setting that is robot-wide.

```sh
cargo run -p vrobots-examples --bin ex15_camera_mono
./target/cpp-build/ex15_camera_mono
python examples/python/ex15_camera_mono.py
```

## Resolution

| Variant | Accepted strings | Width x height |
|---|---|---|
| `Resolution::P360` | `"360p"`, `"360"` | 640 x 360 |
| `Resolution::P720` | `"720p"`, `"720"` | 1280 x 720, the simulator's default |
| `Resolution::P1080` | `"1080p"`, `"1080"` | 1920 x 1080 |

Bare heights are accepted because the service field is a number while the stream segment is
a `p`-suffixed string. Widths are the 16:9 partners of the height: `width() == height * 16 / 9`.

## Pixel format

| Variant | Accepted string | Bytes per pixel | Notes |
|---|---|---|---|
| `PixelFormat::Mono8` | `"mono8"` | 1 | 8-bit greyscale, the integer BT.601 luma of the render |
| `PixelFormat::Rgb8` | `"rgb8"` | 3 | R, G, B in that order, not BGR |
| `PixelFormat::Rgba8` | `"rgba8"` | 4 | R, G, B, A; the renderer's native readback and the cheapest for the simulator to publish |

Parsing is case-insensitive. Both strings are baked into the iceoryx2 service name, so a
format change is an unmount and remount rather than a field write, and every consumer has
to resubscribe under the new name.

## What a frame costs

Resolution times format is the whole story, since the pixels are tightly packed and there
is no compression anywhere on the path.

| Stream | Bytes per frame |
|---|---|
| 720p rgba8, the scene default | 1280 x 720 x 4 = 3 686 400 |
| 720p rgb8, `ex03_hello_image` | 1280 x 720 x 3 = 2 764 800 |
| 360p mono8, `ex15_camera_mono` | 640 x 360 x 1 = 230 400 |

`CameraSpec::data_size()` returns that number for any combination. Sixteen times less data
per frame is worth having for anything that needs only luminance and geometry: optical
flow, fiducials, horizon detection. The stream rides shared memory, so the saving is memory
bandwidth rather than network, but at 60 fps it is 200 MB/s against 13 MB/s.

From `examples/rust/src/bin/ex15_camera_mono.rs`, the two strings are the entire
difference from `ex03_hello_image`:

```rust
const CAMERA: &str = "mono";
const RESOLUTION: &str = "360p"; // 360p | 720p | 1080p -- robot-wide
const FORMAT: &str = "mono8"; // mono8 | rgb8 | rgba8 -- per camera
```

<details>
<summary>The same in C++ (<code>examples/cpp/ex15_camera_mono.cpp</code>)</summary>

```cpp
constexpr const char* CAMERA = "mono";
constexpr const char* RESOLUTION = "360p";  // 360p | 720p | 1080p -- robot-wide
constexpr const char* FORMAT = "mono8";     // mono8 | rgb8 | rgba8 -- per camera
```

</details>

<details>
<summary>The same in Python (<code>examples/python/ex15_camera_mono.py</code>)</summary>

```python
CAMERA = "mono"
RESOLUTION = "360p"  # 360p | 720p | 1080p -- robot-wide
FORMAT = "mono8"  # mono8 | rgb8 | rgba8 -- per camera
```

</details>

The three strings are the stream identity on the wire, so they are literal strings in every
surface: there is no enum for either, and a typo in one of them is a stream that does not
exist rather than a compile error.

The stream name carries both, and `mounted_cameras()` reports the spec back:

```text
camera stream: vrobots/1/i/cam/mono/360p_mono8
mounted by this handle: [CameraSpec { name: "mono", resolution: P360, format: Mono8 }]
```

## Resolution is robot-wide

**Format is per camera. Resolution is one knob for the whole robot**, shared by every
camera on it, and it is the only part of a mount request that is not confined to the camera
being named. Changing it restarts every stream on that robot under a new service name.

Three consequences, in the order people meet them:

1. Mounting a second camera at a different resolution from one **this handle** already
   holds is refused client-side with `VrError::InvalidArgument`, before anything reaches
   the wire. Mount them all at one resolution, or unmount the others first.
2. The guard can only see this handle's cameras. Mounting `mono` at 360p does not unmount
   the scene's `front_left` and `front_right`, but it does move the resolution knob, so
   they come back at 360p under new names: `vrobots/1/i/cam/front_left/360p_rgba8`.
   Whatever was reading the 720p name is reading a service that no longer exists.
3. Unmounting does not put the knob back. `unmount_camera` removes one camera; resolution
   is a separate setting, and nothing in the SDK restores it. The robot streams at 360p
   until something sets it again or the simulator restarts.

The refusal names the cameras in the way:

```text
invalid_argument: camera "wide" asks for 720p but "mono" is already mounted at 360p -- resolution is one setting for the WHOLE robot, and changing it restarts every stream under a new name. Mount them all at the same resolution, or unmount the others first
```

> **Gotcha.** If `ex13_open_camera` times out on `front_left` at 720p, an earlier run of
> `ex15_camera_mono` moved the robot to 360p. Open `front_left` at `"360p"` instead, or
> restart the simulator.

## What is checked before anything is sent

`CameraSpec::parse` validates all three strings at the call site, because every one of them
fails later as a stream that never arrives rather than as an error.

| Input | Rule | On failure |
|---|---|---|
| `name` | non-empty, and only letters, digits, `_` and `-` | `VrError::InvalidArgument`, naming the offending character |
| `resolution` | one of the accepted strings above | `VrError::InvalidArgument`, naming the accepted set |
| `format` | one of the accepted strings above | `VrError::InvalidArgument`, naming the accepted set |

The name has to survive being pasted into an iceoryx2 service name. The simulator does not
check it and does not error on a bad one: it mounts a camera whose service can never be
opened.

**Next:** [Inside a frame](03-frames.md)

**See also:** [Mount, open and unmount](01-mount-open-unmount.md), [Two cameras at once](06-two-cameras.md), [Appendix C: Error reference](../appendix-c-errors.md)
