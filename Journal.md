# Project Journal

Running log of progress, problems, and solutions.
Written as I go — not polished, just honest.

---

## Session 1 — Toolchain Setup & First Flash

**Date:** 24 May 2026

**Goal:** Get Zephyr toolchain working on WSL2 and flash 
hello_world to ESP32-S3 DevKitC-1.

**Environment:**
- Host: Windows 11 + WSL2
- Ubuntu 26.04 LTS (Resolute)
- Zephyr: v4.4.99
- West: v1.5.0
- Zephyr SDK: v1.0.1 (minimal)
- Board: ESP32-S3 DevKitC-1 (esp32s3_devkitc/esp32s3/procpu)

---

### What I did

1. Installed WSL2 with Ubuntu 26.04 on Windows 11
2. Installed West via pip
3. Ran `west init` and `west update` (~20 minutes, ~3GB download)
4. Installed Zephyr Python requirements
5. Installed Zephyr SDK and Xtensa ESP32-S3 toolchain
6. Fetched Espressif HAL blobs
7. Built hello_world successfully
8. Flashed to ESP32-S3 from Windows PowerShell

---

### Problems and fixes

**Problem 1: pip refused to install West**

Ubuntu 26.04 enforces PEP 668 — no system-wide pip installs.

**Fix:**
```bash
pip3 install --user west --break-system-packages
```

---

**Problem 2: Zephyr SDK version mismatch**

Downloaded SDK v0.17.0 but Zephyr 4.4.99 requires SDK v1.0.1.

**Fix:**
```bash
cat ~/zephyrproject/zephyr/SDK_VERSION
# Output: 1.0.1
```
Downloaded correct version:
```bash
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v1.0.1/zephyr-sdk-1.0.1_linux-x86_64_minimal.tar.xz
tar xvf zephyr-sdk-1.0.1_linux-x86_64_minimal.tar.xz
cd zephyr-sdk-1.0.1
./setup.sh
./setup.sh -t xtensa-espressif_esp32s3_zephyr-elf
```

Note: Use the **minimal** SDK variant — the full SDK URL returns 404 
for v1.0.1.

---

**Problem 3: esptool not found during build**

**Fix:**
```bash
# Create venv first (required on Ubuntu 26.04)
python3 -m venv ~/.venv/zephyr
source ~/.venv/zephyr/bin/activate
pip install west
pip install -r ~/zephyrproject/zephyr/scripts/requirements.txt
west packages pip --install
```

Add to ~/.bashrc so venv activates automatically:
```bash
echo 'source ~/.venv/zephyr/bin/activate' >> ~/.bashrc
```

---

**Problem 4: libusb missing during pip install**

**Fix:**
```bash
sudo apt install -y libusb-1.0-0-dev libhidapi-dev pkg-config
```

---

**Problem 5: WSL2 flashing fails with I/O error**

usbipd USB passthrough is unstable during the flash operation — 
connection drops mid-flash.

**Fix: Flash from Windows PowerShell instead of WSL2.**

Build in WSL2, flash from Windows. Clean separation.

Install esptool on Windows:
```powershell
pip install esptool
```

Flash command (run from PowerShell):
```powershell
python -m esptool --port COM3 --baud 460800 --before default-reset --after hard-reset write-flash -u --flash-mode dio --flash-freq 80m --flash-size 8MB 0x0 "\\wsl.localhost\Ubuntu\home\edip\zephyrproject\zephyr\build\zephyr\zephyr.bin"
```

This accesses the WSL2 build output directly via the 
`\\wsl.localhost\` UNC path — no need to copy files.

---

**Problem 6: dialout group not applied in WSL2**

Even after `sudo usermod -a -G dialout $USER`, the group 
wasn't active. `newgrp dialout` workaround needed, or use 
the Windows PowerShell flash approach above which bypasses 
this entirely.

---

### Key workflow for this project

**Build (in WSL2):**
```bash
source ~/.venv/zephyr/bin/activate
cd ~/zephyrproject/zephyr
west build -p -b esp32s3_devkitc/esp32s3/procpu samples/hello_world
```

**Flash (in Windows PowerShell):**
```powershell
python -m esptool --port COM3 --baud 460800 --before default-reset --after hard-reset write-flash -u --flash-mode dio --flash-freq 80m --flash-size 8MB 0x0 "\\wsl.localhost\Ubuntu\home\edip\zephyrproject\zephyr\build\zephyr\zephyr.bin"
```

**USB reattach after Windows restart (PowerShell as Admin):**
```powershell
usbipd attach --wsl --busid 1-2
```

Note: Run `usbipd detach --busid 1-2` before opening serial 
monitor — COM3 can only be held by one application at a time.

---

### Build output confirmed

---

### Result

✅ Toolchain working  
✅ hello_world builds on ESP32-S3  
✅ hello_world flashed to board  
✅ Serial monitor — COM3 conflict with usbipd, resolving next session

---
