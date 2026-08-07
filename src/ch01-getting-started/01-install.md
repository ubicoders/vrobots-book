# Installing the SDK and the simulator

Get the Rust toolchain, this repository and a running simulator on Windows, Ubuntu or WSL.

```sh
git clone --recurse-submodules https://github.com/ubicoders0/vrobots_sdk
cd vrobots_sdk
cargo build --workspace
```

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Rust | 1.89 or newer | edition 2024; install from <https://rustup.rs/> |
| `vrobots_msgs` submodule | pinned by this repo | ships the generated FlatBuffers Rust |
| The Unity simulator | in Play mode | required by anything that talks to a robot |

You do **not** need `flatc` or `protoc`. The submodule carries the generated code, which is
why the clone above passes `--recurse-submodules`.

If you already cloned without it, the build fails and says so. Repair it with:

```sh
git submodule update --init --recursive
```

## Building the C++ and Python surfaces

Every page in this book shows its example in all three languages. Rust needs nothing beyond
the clone above; the other two need one build step each, and both are optional if you only
read the Rust blocks.

The C++ examples are a CMake project over the header-only wrapper. Building the C ABI crate
generates the C header, so that is the only prerequisite:

```sh
cargo build -p vrobots-sdk-capi --release
cmake -S examples/cpp -B target/cpp-build -DCMAKE_BUILD_TYPE=Release
cmake --build target/cpp-build --config Release
```

That puts one binary per example under `target/cpp-build/`, which is the path the `sh` block
on each page names. On Windows the binaries land in `target\cpp-build\Release\` and carry an
`.exe` suffix, and the DLL is copied beside each one; on Linux the build rpath points at the
cargo target directory, so no `LD_LIBRARY_PATH` is needed.

Python is the `ubicoders-vrsdk` wheel, imported as `vrsdk`. Install it, or build it from
this repository with `maturin develop --release` run in `crates/vrobots-sdk-py/`:

```sh
pip install ubicoders-vrsdk
```

`numpy` is required by the camera examples and `opencv-python` is optional: without it
[Hello image](06-hello-image.md) prints metadata instead of opening a window, and
[Saving a frame](../ch05-cameras/07-saving-frames.md) writes a PPM instead of a PNG. The one
page that needs OpenCV outright is [Showing frames in a window](../ch05-cameras/08-showing-frames.md),
in every language.

## Getting the simulator

Prebuilt simulator packages are at <https://www.ubicoders.com/virtualrobots>. Windows 11
and Ubuntu 22.04 or newer are supported; macOS is not supported yet.

### Windows

Run `virtual_robots.exe`.

### Ubuntu

The build needs `xdg-utils` present and its own executable bit set.

```sh
sudo apt install xdg-utils -y
sudo chmod +x ./virtual_robots.x86_64
./virtual_robots.x86_64
```

Double-clicking `virtual_robots.x86_64` in a file manager works as well.

### WSL

WSL needs a graphics bridge before Unity will render. Install the Mesa and Vulkan packages
and force the D3D12 gallium driver, which routes rendering to the Windows GPU rather than
to the CPU rasteriser.

Save this as `install_wsl_graphics.bash`:

```bash
#!/bin/bash
# 1. Install necessary drivers and diagnostic tools
sudo apt-get update
sudo apt install xdg-utils -y
sudo apt install mesa-utils mesa-vulkan-drivers vulkan-tools -y

# 2. Add GPU bridge variables to .bashrc for persistence
# We use GALLIUM_DRIVER to force the D3D12 bridge (Windows GPU)
# and VK_ICD_FILENAMES to ensure Vulkan doesn't default to the CPU (llvmpipe)
if ! grep -q "GALLIUM_DRIVER=d3d12" ~/.bashrc; then
  echo 'export GALLIUM_DRIVER=d3d12' >> ~/.bashrc
  echo 'export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA' >> ~/.bashrc
fi

# 3. Reload environment
source ~/.bashrc

# 4. Verify the setup
echo "--- Checking OpenGL  ---"
glxinfo -B | grep -E "Device|Accelerated"

echo "--- Checking Vulkan  ---"
vulkaninfo | grep "Vulkan Instance Version"
vkcube
```

Run it, then reload your shell so the exported variables apply:

```sh
bash install_wsl_graphics.bash && source ~/.bashrc
```

Check that Vulkan came up on the GPU. `vkcube` should open a spinning cube window; if it
does not, the simulator will not render either.

```sh
vulkaninfo | grep "Vulkan Instance Version"
vkcube
```

Launch the simulator with Vulkan forced. The `LD_LIBRARY_PATH` edit removes
`/opt/zenoh-c/lib` from the loader path, so the simulator loads its own vendored zenoh
rather than a system copy.

```bash
#!/bin/bash
sudo chmod +x ./virtual_robots.x86_64
export LD_LIBRARY_PATH=$(echo "$LD_LIBRARY_PATH" | sed 's|:/opt/zenoh-c/lib||; s|/opt/zenoh-c/lib:||; s|/opt/zenoh-c/lib||')
nohup ./virtual_robots.x86_64 -force-vulkan > output.log 2>&1 &
```

> **Gotcha.** Camera frames ride iceoryx2 shared memory, so they are same-host only.
> A simulator running under WSL and a client running on Windows are two hosts as far as
> iceoryx2 is concerned: states arrive over zenoh, frames never do. Run both sides in the
> same place when you want images.

## Verifying the install

This prints what the binary you just built actually speaks, and it needs no simulator.

```sh
cargo run -p vrobots-sdk --bin vrobots -- --version
```

The Python wheel installs the same command as a `vrobots` entry point, so `vrobots --version`
prints the same block without a Rust toolchain.

```text
vrobots-sdk 0.1.4
  vrobots_msgs  v2.0.2-31-gac335c0 (schema_version 3)
  flatbuffers   25.12.19
  zenoh         1.9.0
  iceoryx2      0.9.3
  src_id        122
```

The first line is the SDK release. The second is a `git describe` of the `vrobots_msgs`
submodule the FlatBuffers code was generated from, so it moves when the schema does. The
three pins are exact rather than caret ranges: iceoryx2 compares major, minor and patch on
every shared-memory open, and a version one patch off does not error, it silently delivers
nothing. That is the first thing to compare against the simulator build when fields look
like garbage or a camera stream never appears.

**Next:** [First contact: is anything publishing?](02-first-contact.md)

**See also:** [Versions and pins](../ch08-tooling/03-version-and-pins.md), [When nothing happens](08-troubleshooting.md)
