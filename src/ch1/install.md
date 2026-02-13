# ⚙️ Installation

## 💻 Supported Operating Systems

- ✅ Windows 11
- ✅ Ubuntu 22.04+
- ⬜ macOS (not supported yet)

---

## 🖥️ Recommended Hardware

- **RAM**: 32 GB
- **GPU**: NVIDIA RTX Series (recommended for best performance)

---

## 📦 Download

## Prebuilt packages are available at: [https://www.ubicoders.com/virtualrobots](https://www.ubicoders.com/virtualrobots)

## Setup - Windows

Just run "virtual_robots.exe"

## Setup - Ubuntu

```
sudo apt install xdg-utils -y
sudo chmod +x ./virtual_robots.x86_64
./  .x86_64
or
# run "virtual_robots.x86_64" by double clicking it.
```

## Setup - WSL

Install Vulkan (WSL graphics bridge)

install_wsl_graphics.bash
```bash
#!/bin/bash
# 1. Install necessary drivers and diagnostic tools
sudo apt update
sudo apt install -y xdg-utils mesa-vulkan-drivers vulkan-tools mesa-utils

# 2. Add GPU bridge variables to .bashrc for persistence
# We use GALLIUM_DRIVER to force the D3D12 bridge (Windows GPU)
# and VK_ICD_FILENAMES to ensure Vulkan doesn't default to the CPU (llvmpipe)
if ! grep -q "GALLIUM_DRIVER=d3d12" ~/.bashrc; then
  echo 'export GALLIUM_DRIVER=d3d12' >> ~/.bashrc
  echo 'export MESA_D3D12_DEFAULT_ADAPTER_NAME=NVIDIA' >> ~/.bashrc
  echo 'export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/d3d12_icd.x86_64.json' >> ~/.bashrc
fi

# 3. Reload environment
source ~/.bashrc

# 4. Verify the setup
echo "--- Checking OpenGL  ---"
glxinfo -B | grep -E "Device|Accelerated"

echo "--- Checking Vulkan  ---"
vulkaninfo | grep "Selected GPU"


```

Check Vulkan

```
vulkaninfo | grep "Vulkan Instance Version"
vkcube
```

Run

run_vrobots.bash

```bash
#!/bin/bash
# Ensure the binary is executable
chmod +x ./virtual_robots.x86_64

# Verification check: Stop if we are still on CPU (llvmpipe)
RENDERER=$(glxinfo -B | grep "Device" | grep -o "llvmpipe")
if [ "$RENDERER" == "llvmpipe" ]; then
    echo "ERROR: GPU not detected. Still using CPU (llvmpipe). Run Script 1 first."
    exit 1
fi

echo "GPU Detected: Launching simulation with Vulkan..."

# Run the simulation in the background
# -force-vulkan: Tells Unity to use the Vulkan API
# VK_ICD_FILENAMES: Double-check the Vulkan path for this specific process
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/d3d12_icd.x86_64.json

nohup ./virtual_robots.x86_64 -force-vulkan > output.log 2>&1 &

echo "Simulation started. Check progress with: tail -f output.log"
```

## 🐍 Python Client (all platforms)

Install the Python IPC client:

```bash
pip install ubicoders-vrobots-ipc
```

with ROS2 system python:

```bash
sudo python3 -m pip install ubicoders-vrobots-ipc ubicoders-g2opy rerun-sdk==0.22.0  --break-system-packages
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
```
