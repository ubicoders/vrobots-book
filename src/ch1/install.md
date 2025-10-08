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
Prebuilt packages are available at: [https://www.ubicoders.com/virtualrobots](https://www.ubicoders.com/virtualrobots)
---


## Setup - Windows
Just run "virtual_robots.exe"

## Setup - Ubuntu
```
sudo apt install xdg-utils -y
sudo chmod +x ./vrobots.x86_64
./vrobots.x86_64
or
# run "vrobots.x86_64" by double clicking it.
```

## Setup - WSL

Install Vulkan (WSL graphics bridge)
```
sudo apt update
sudo apt install xdg-utils -y
sudo apt install mesa-vulkan-drivers vulkan-tools -y
vulkaninfo | grep "Vulkan Instance Version"
vkcube
```

Check Vulkan
```
vulkaninfo | grep "Vulkan Instance Version"
vkcube
```

```
sudo chmod +x ./vrobots.x86_64
```

Run
```
nohup ./vrobots.x86_64 -force-vulkan > output.log 2>&1 &
```



## 🐍 Python Client (all platforms)
Install the Python IPC client:

```bash
pip install ubicoders-vrobots-ipc
```

