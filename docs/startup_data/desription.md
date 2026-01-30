This is the **Technical Master Architecture Document**.

It moves beyond business concepts into the raw engineering architecture. It describes the internal mechanics, memory models, data flows, and subsystem interactions of the **xInfer Ecosystem**.

---

# 1. xInfer: The High-Performance Kernel

**xInfer** is a C++20 library designed to execute deep learning models with bare-metal performance. It is not merely a wrapper around other libraries; it is a **Uniform Execution Environment**.

### A. The Hardware Abstraction Layer (HAL)
The core of xInfer is the `BackendInterface`. This is a pure virtual C++ interface that standardizes how models are loaded and executed, regardless of the underlying silicon.

*   **The Problem:** NVIDIA uses `nvinfer1::IExecutionContext`, Intel uses `ov::InferRequest`, and Rockchip uses `rknn_run`. They have totally different memory requirements and function signatures.
*   **The xInfer Solution:** We implement a **Factory Pattern**.
    *   **Input:** A standardized `xinfer::Tensor` (holding raw bytes).
    *   **Process:** The HAL translates this generic tensor into the backend-specific buffer (e.g., `VkBuffer` for Vulkan, `cl_mem` for OpenCL, or `CUdeviceptr` for CUDA).
    *   **Execution:** The backend runs the inference.
    *   **Output:** The HAL maps the results back to a generic `xinfer::Tensor`.

### B. Memory Management: The "Zero-Copy" Engine
Speed is defined by memory bandwidth, not just FLOPs. xInfer implements a strict **Zero-Copy** policy.

1.  **Pinned Memory (Page-Locked):**
    On architectures like NVIDIA Jetson or APUs (integrated graphics), CPU and GPU share physical RAM. xInfer allocates memory using `cudaHostAlloc` or generic `mmap` with specific flags that prevent the OS from paging it out.
    *   *Result:* The GPU can read the input image directly from the CPU's memory address without a `memcpy` operation.
2.  **DMA Buffers (Direct Memory Access):**
    For Linux-based edge devices (Rockchip/NXP), xInfer utilizes `dma_buf`. This allows a camera driver (V4L2) to write a frame to a memory address, and the NPU (Neural Processing Unit) to read from that exact same address.
3.  **Tensor Arena:**
    To prevent memory fragmentation, xInfer pre-allocates a massive contiguous block of RAM at startup (the "Arena"). During inference, intermediate tensors rely on offsets within this arena rather than `malloc`/`free` calls.

### C. The 14 Supported Backends (Detailed Tech Stack)
xInfer natively links against the low-level SDKs of these 14 platforms:

1.  **NVIDIA:** Uses **TensorRT C++ API**. Supports FP16/INT8 dynamic quantization.
2.  **Intel:** Uses **OpenVINO Runtime**. Optimizes for AVX-512 instructions on CPUs.
3.  **AMD (FPGA):** Uses **Vitis AI**. Compiles `.xmodel` files for DPU (Deep Learning Processor Unit) cores.
4.  **Rockchip:** Uses **RKNN Runtime**. Handles RGA (Raster Graphic Acceleration) for pre-processing.
5.  **Qualcomm:** Uses **QNN (Qualcomm Neural Network) SDK**. Targets the Hexagon DSP.
6.  **Apple:** Uses **CoreML** and **Metal Performance Shaders (MPS)** for Mac/iOS.
7.  **Google:** Uses **LibEdgeTPU** for Coral USB/PCIe accelerators.
8.  **MediaTek:** Uses **NeuroPilot SDK** (APU hardware).
9.  **Hailo:** Uses **HailoRT** for high-efficiency dataflow architectures.
10. **Ambarella:** Uses **CVflow** graph compiler.
11. **NXP:** Uses **eIQ** for i.MX8 processors.
12. **Texas Instruments:** Uses **TIDL** for Jacinto processors.
13. **Raspberry Pi:** Uses the **Hailo-8L** integration kit.
14. **RISC-V:** Uses **RVV (RISC-V Vector Extension)** intrinsics for generic acceleration.

---

# 2. xInfer Daemon (`xinferd`): The Resident Agent

**xinferd** is the long-running background service (systemd service) that hosts the xInfer library. It transforms the library into a **Server**.

### A. Process Architecture
`xinferd` is designed to never crash the host, even if a model segfaults.

*   **Main Process:** Handles API requests, authentication, and resource monitoring.
*   **Worker Threads:** A thread pool where each thread manages a specific "Session" (a loaded model).
*   **Isolation:** (Optional) Can run specific unstable experimental backends in a `fork()`ed child process to protect the main daemon.

### B. The Session Manager
This is the state machine of the daemon.
*   **Loading:** When a client requests a model load, the Session Manager assigns a UUID.
*   **Reference Counting:** If multiple applications need "YOLOv8," the Session Manager loads it into VRAM once and shares the handle, saving massive amounts of memory.
*   **Hot-Swapping:** Implements a "Shadow Load" technique.
    *   *Scenario:* Updating a model while it is live.
    *   *Action:* Daemon loads V2.0 into reserve memory. Once ready, it atomically swaps the pointer from V1.0 to V2.0 and then frees V1.0. **Result:** Zero downtime inference updates.

### C. Communication Protocols
1.  **Control Plane (HTTP/REST):**
    *   Uses `httplib`. Endpoints for `/load`, `/unload`, `/status`.
    *   Used by xFabric to configure the device.
2.  **Data Plane (Shared Memory - IPC):**
    *   Used for high-bandwidth video.
    *   **Mechanism:** `xinferd` creates a named shared memory block (`/dev/shm/xinfer_video`). The client writes raw RGB bytes there and sends a signal. `xinferd` processes it in place.

---

# 3. xFabric: The Operating System

**xFabric** is the orchestration layer. It is built using **Qt6 (C++)** for the frontend and **C++20** for the backend logic.

### A. Module 1: xFabric Studio (The IDE)
A desktop application that provides a Visual Development Environment.

*   **Architecture:** Model-View-Controller (MVC).
*   **UI Framework:** Qt Widgets for panels, `QGraphicsScene` for the Node Editor.
*   **Compiler Integration:**
    *   The Studio contains the `xInfer Compiler` binaries.
    *   Users import an ONNX file. The Studio parses the graph, optimizes layers (Fusion), quantizes weights (FP32 -> INT8), and outputs an engine file specific to the selected target (e.g., RK3588).

### B. Module 2: Flow (The Orchestrator)
The Logic Engine that executes pipelines.

*   **Data Structure:** Directed Acyclic Graph (DAG).
*   **Nodes:**
    *   **Source:** Camera (V4L2), RTSP (FFmpeg), File.
    *   **Filter:** Pre-processing (Resize, Normalize - GPU accelerated).
    *   **Infer:** Wraps an `xInfer` Session.
    *   **Logic:** Python Script Node (embedded `pybind11`) for custom "If/Then" logic.
    *   **Sink:** RTMP Stream, WebSocket, MQTT, SQL Database.
*   **Execution:**
    *   Flow uses a **Topological Sort** to determine execution order.
    *   It manages a "Frame Context" pointer that flows through the nodes. Each node modifies the context (e.g., adding Bounding Box metadata) without copying the pixel data.

### C. Module 3: Deployer (Fleet Management)
The system responsible for distributing software to the edge.

*   **Transport Layer:** Built on `libssh2`.
*   **Capabilities:**
    *   **Discovery:** Uses mDNS (Avahi) to find xInfer devices on the local LAN.
    *   **Provisioning:** Can push a "Bootstrap Script" to a fresh Linux install to set up `xinferd` dependencies automatically.
    *   **Atomic Updates:**
        *   The Deployer pushes a `.xfabric` package (Model + Config + License).
        *   It verifies the SHA256 checksum.
        *   It instructs the Daemon to load it in "Staging Mode."
        *   If the Daemon reports "Healthy," it promotes it to "Production."

### D. Module 4: Telemetry (The Watchdog)
Ensures system reliability.

*   **Hardware Monitors:**
    *   Reads `/sys/class/thermal` for CPU temps.
    *   Uses `nvml` (NVIDIA Management Library) for GPU stats.
    *   Reads `/proc/meminfo` for RAM usage.
*   **Drift Detection (The Brain):**
    *   **Statistical Analysis:** xFabric maintains a rolling window of input histograms (e.g., average brightness, color distribution).
    *   **Kolmogorov-Smirnov Test (KS-Test):** It compares the current window against the "Baseline" (training data).
    *   **Alerting:** If the p-value drops below a threshold (meaning the data is statistically different), it triggers a "Drift Alert," signaling that the model is likely failing due to environmental changes.

---

# Summary of Technical Interactions

1.  **User** uses **xFabric Studio** to drag-and-drop a pipeline (Cam -> YOLO -> Screen).
2.  **xFabric Studio** compiles the YOLO model via the **xInfer Compiler** for a specific Target (e.g., Rockchip).
3.  **xFabric Deployer** opens an SSH tunnel to the device and transfers the compiled engine.
4.  **xInfer Daemon** on the device loads the engine into NPU memory using the **HAL**.
5.  **xFabric Flow** (running on the device) grabs video frames, passes pointers to the Daemon via **Shared Memory**, and renders the results.
6.  **Telemetry** watches the whole process and reports latency back to the Studio.