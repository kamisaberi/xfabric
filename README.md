# xFabric: The Edge AI Operating System

## This Project Tagged as In Future Project. Right Now development Progress is Stalled 


[![License](https://img.shields.io/badge/License-Proprietary%20%2F%20Commercial-blue.svg)](LICENSE)
[![Core](https://img.shields.io/badge/Powered%20By-xInfer-orange)](https://github.com/kamisaberi/xinfer)
[![Status](https://img.shields.io/badge/Status-Alpha-yellow)]()

**xFabric** is the orchestration, management, and visualization platform built for the **xInfer** runtime.

While **xInfer** provides the raw, high-performance C++ execution engine across 15+ hardware targets, **xFabric** provides the **Application Layer** needed to build, deploy, and monitor scalable AI products in the real world.

Think of **xInfer** as the Kernel (Linux), and **xFabric** as the OS (Android).

---

## 🏗️ Architecture

xFabric sits on top of the xInfer Core, extending it with five Enterprise-grade modules:

| Module | Directory | Purpose |
| :--- | :--- | :--- |
| **📡 Serving** | `src/serving` | Turns C++ models into **REST Microservices**. |
| **🌊 Flow** | `src/flow` | **Low-Code** pipeline orchestrator using JSON configuration. |
| **🩺 Telemetry** | `src/telemetry` | **Observability**, drift detection, and hardware monitoring. |
| **🚀 Deployer** | `src/deployer` | **MLOps** fleet management via SSH/SCP. |
| **🖥️ Studio** | `ui/xinfer_studio` | A professional **Qt6 GUI** for visualization and control. |

---

## 🌟 Key Features

### 1. Zero-Code Pipelines (Flow)
Stop writing C++ glue code. Define your entire vision pipeline in JSON. Switch from a USB Camera to an RTSP stream or change the AI model without recompiling.

```json
{
  "nodes": [
    { "id": "cam", "type": "CameraSource", "params": { "id": "0" } },
    { "id": "ai",  "type": "ObjectDetector", "params": { "model": "yolo.engine" } },
    { "id": "out", "type": "NetworkSink", "params": { "protocol": "UDP" } }
  ]
}
```

### 2. Instant Microservices (Serving)
Expose your high-performance C++ models to the web. xFabric includes a lightweight, multi-threaded HTTP server.

*   **Endpoint:** `POST /v1/models/yolov8:predict`
*   **Format:** JSON-in, JSON-out.
*   **Performance:** Zero-copy internal tensor handling.

### 3. Production Observability (Telemetry)
Don't fly blind. xFabric monitors the health of your edge devices.
*   **Hardware:** CPU/GPU Temp, RAM Usage, NPU Utilization.
*   **Data Drift:** Statistical analysis (Z-Score) of live input data to detect when retraining is needed.
*   **Integration:** Exports metrics to Prometheus/Grafana or local logs.

### 4. xInfer Studio (GUI)
A full IDE for Edge AI.
*   **Visualize:** See bounding boxes, segmentation masks, and pose skeletons in real-time.
*   **Manage:** Connect to remote devices (Jetson, Rockchip, FPGA).
*   **Compile:** Drag-and-drop ONNX files to compile them for specific hardware targets.

---

## 🛠️ Build & Install

**xFabric** automatically fetches the **xInfer** core engine during the build process.

### Prerequisites
*   **C++20 Compiler**
*   **CMake 3.20+**
*   **Qt 6.x** (Required for the Studio GUI)
*   **libssh2** (For the Deployer module)
*   **OpenCV 4.x**

### Building
```bash
git clone https://github.com/kamisaberi/xfabric.git
cd xfabric
mkdir build && cd build

# Configure (xInfer will be downloaded automatically via FetchContent)
cmake .. 

# Build all modules
make -j$(nproc)
```

---

## 🚀 Usage

### Running the GUI (Studio)
```bash
./ui/xinfer_studio/xinfer_studio
```

### Running the Model Server
```bash
# Serves all models found in ./my_models on port 8080
./src/serving/xinfer-server ./my_models 8080
```

### Running a Flow Pipeline
```bash
# Executes a pipeline defined in JSON
./src/flow/xinfer-flow configs/security_camera.json
```

---

## 💼 Commercial & Licensing

**xInfer (The Engine)** is available under the **MIT License**.
**xFabric (The Platform)** is available under a **Commercial/Dual License**.

*   **Community Edition:** Includes Flow and Serving.
*   **Enterprise Edition:** Includes xInfer Studio, Telemetry, and the Fleet Deployer.

For enterprise support and licensing inquiries for **Aegis Sky** or **Blackbox SIEM** integrations, please contact the maintainers.

---

## 🤝 Relationship with xInfer

xFabric depends on `xInfer` for all heavy lifting.
*   **xInfer** handles the **Math** (CUDA, AVX, NPU drivers).
*   **xFabric** handles the **Business Logic** (Networking, Config, UI).

If you only need to run a model inside a tight loop on a microcontroller, use **xInfer**.
If you are building a product, a server, or a system, use **xFabric**.
