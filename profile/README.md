# Skynet

**Modular Autonomous Driving System for Lane Keeping Assistance**

A distributed, real-time autonomous driving system supporting both simulation (CARLA) and real hardware (Jetracer), featuring modular architecture with independent components for detection, decision-making, hardware integration, and monitoring.

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        ADS-Skynet Ecosystem                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐                                 ┌──────────────┐    │
│  │   CARLA     │                                 │    Viewer    │    │
│  │  Simulator  │                                 │   (Web UI)   │    │
│  └──────┬──────┘                                 └──────▲───────┘    │
│         │                                               │            │
│         │ SHM/ZMQ                            WebSocket  │            │
│         ▼                                    (8080/8081)│            │
│  ┌─────────────┐          ┌──────────────┐             │             │
│  │ Simulation  │◄────────►│     LKAS     │◄────────────┘             │
│  │  (Client)   │  SHM     │  Detection + │      ZMQ                  │
│  │             │          │   Decision   │    (5557-59)              │
│  └─────────────┘          └──────┬───────┘                           │
│                                  │ SHM/ZMQ                           │
│  ┌─────────────┐                 │                                   │
│  │   Vehicle   │◄────────────────┘                                   │
│  │ (Jetracer)  │        ZMQ (5560-62)                                │
│  │  Hardware   │                                                     │
│  └─────────────┘                                                     │
│                                                                      │
│  All modules depend on skynet-common (shared types & communication)  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Repositories

### [Lkas](https://github.com/ADS-Skynet/Lkas) - Core LKAS Module
**Lane Keeping Assist System - Detection & Decision**

Real-time computer vision pipeline for lane detection and vehicle steering control with pluggable algorithms.

- **Detection Methods:**
  - `cv`: Classical OpenCV (Canny edge detection + Hough Transform) — 5-15ms latency
  - `dl`: Deep Learning (PyTorch-based segmentation) — 15-30ms latency
  - Configurable ROI (Region of Interest) masking
  - Temporal smoothing for stability

- **Decision Control:**
  - `pd`: Proportional-Derivative controller (fast, stateless)
  - `pid`: Proportional-Integral-Derivative controller (zero steady-state error)
  - `mpc`: Model Predictive Control (future optimization)
  - Lane departure detection (CENTERED / DRIFT / DEPARTURE)
  - Adaptive throttle policy with automatic speed reduction
  - Emergency braking when no lanes detected

- **Communication:**
  - Shared memory IPC for ultra-low latency (~0.01ms)
  - ZMQ broker for distributed messaging
  - Live parameter tuning (detection + decision)
  - Configurable broadcasting

```bash
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast  # OpenCV detection
lkas --method dl --broadcast  # Deep learning detection
```

---

### [Viewer](https://github.com/ADS-Skynet/Web-viewer) - Web Monitoring
**Web-based Remote Viewer with WebSocket Streaming**

Real-time monitoring and control interface for the autonomous driving system with optimized binary streaming. Modular architecture with dedicated overlay, WebSocket server, HTTP server, and message handler components.

- **Live Monitoring:**
  - WebSocket-based camera feed streaming (binary, no base64 overhead)
  - Lane detection visualization with HUD overlay
  - Vehicle state telemetry dashboard
  - Real-time FPS and latency metrics

- **Remote Control:**
  - Pause/Resume/Respawn actions (keyboard: `Space` / `R`)
  - Live parameter tuning (Canny, Hough, PID gains)
  - Detection method switching (cv/dl)
  - Throttle policy adjustment

- **Performance:**
  - ~50-100ms total latency (acquisition → browser)
  - Up to 60 FPS WebSocket broadcast
  - JPEG quality: 75 (configurable)
  - Rendering offloaded to viewer machine (vehicle CPU stays free)

```bash
pip install git+https://github.com/ADS-Skynet/Web-viewer.git
viewer --port 8080  # HTTP server at 8080, WebSocket at 8081
# Open browser: http://localhost:8080
```

---

### [Common](https://github.com/ADS-Skynet/Common) - Shared Infrastructure
**Platform-Independent Shared Components**

Lightweight shared library providing common types, communication utilities, configuration management, and visualization for all modules.

- **Shared Types:**
  - Lane data models (`Lane`, `LaneDepartureStatus`, `LaneMetrics`)
  - Detection data (`DetectionData` with lane coordinates)
  - Vehicle state (`VehicleState` telemetry)

- **Communication:**
  - ZMQ pub-sub utilities (`ViewerSubscriber`, `ActionPublisher`, `ParameterPublisher`)
  - Unified message protocols for inter-process communication

- **Configuration:**
  - `ConfigManager` — unified YAML + dataclasses config loader
  - Centralized ZMQ ports, shared memory names, camera settings, and control limits
  - Single source of truth for all modules

- **Visualization:**
  - `LKASVisualizer` for rendering lane overlays and HUD
  - Consistent visual style across all modules

- **Key Features:**
  - Minimal dependencies (no CARLA, PyTorch, or heavy frameworks)
  - M1 Mac compatible
  - Works on restricted platforms (Jetracer, embedded systems)

```bash
pip install git+https://github.com/ADS-Skynet/Common.git
# Used as dependency by other modules
```

---

### [Vehicle](https://github.com/ADS-Skynet/Vehicle-jetracer) - Hardware Integration
**Real Jetracer Vehicle Support**

Hardware loop for running the autonomous driving system on actual Jetracer vehicles.

- **Hardware Support:**
  - Jetracer motor control integration
  - Camera capture via V4L2 video device (`/dev/video4` by default)
  - Configurable camera device path

- **Communication:**
  - ZMQ integration with LKAS broker
  - Vehicle state publishing at configurable Hz
  - Parameter subscription for live tuning
  - Action subscription (pause/resume/respawn)
  - Shared memory channels for ultra-low latency frame and control transfer

- **Testing Utilities:**
  - Motor test script (`test_motor.py`)
  - Camera test script (`test_camera.py`)

```bash
pip install git+https://github.com/ADS-Skynet/Vehicle-jetracer.git
vehicle --device /dev/video4 --publish-state-hz 10
vehicle --device /dev/video4 --keepalive  # Keep camera alive while paused
```

---

## Quick Start

### Prerequisites

- Python 3.10+
- CARLA Simulator 0.9.16 (for simulation mode)
- CUDA-capable GPU (optional, for deep learning)

### Simulation Setup (CARLA)

```bash
# Terminal 1: Start CARLA Simulator
./CarlaUE4.sh

# Terminal 2: Start LKAS (Detection + Decision)
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast  # or --method dl for deep learning

# Terminal 3: Start Simulation (CARLA Client)
simulation --broadcast

# Terminal 4: Start Web Viewer (Optional)
pip install git+https://github.com/ADS-Skynet/Web-viewer.git
viewer --port 8080

# Open browser: http://localhost:8080
```

### Real Vehicle Setup (Jetracer)

```bash
# On Jetracer device: Start vehicle hardware loop
pip install git+https://github.com/ADS-Skynet/Vehicle-jetracer.git
vehicle --device /dev/video4 --publish-state-hz 10

# On same or different machine: Start LKAS
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast

# On monitoring laptop: Start web viewer
pip install git+https://github.com/ADS-Skynet/Web-viewer.git
viewer --port 8080
# Open browser: http://localhost:8080
```

### Development Setup

For local development with all modules:

```bash
# Clone all repositories
git clone git@github.com:ADS-Skynet/Common.git
git clone git@github.com:ADS-Skynet/Lkas.git
git clone git@github.com:ADS-Skynet/Vehicle-jetracer.git
git clone git@github.com:ADS-Skynet/Web-viewer.git

# Install in editable mode (order matters - common first!)
pip install -e ./Common
pip install -e ./Lkas
pip install -e ./Vehicle-jetracer
pip install -e ./Web-viewer
```

---

## Key Features

- **Modular Architecture** — Each component runs independently with clean interfaces
- **Ultra-Low Latency** — Shared memory IPC (~0.01ms) for real-time performance
- **Pluggable Algorithms** — Switch between CV/DL detection and PD/PID/MPC controllers
- **Live Tuning** — Adjust detection and decision parameters without restart
- **Real Hardware Support** — Run on actual Jetracer vehicles with full telemetry
- **WebSocket Streaming** — Optimized binary streaming (no base64 overhead)
- **Remote Monitoring** — Web-based viewer accessible from any device
- **Distributed Deployment** — Run components on different machines via ZMQ
- **Platform Independent** — Works on Linux, macOS (including M1), and embedded systems
- **Unified Config** — Single `config.yaml` (managed by `skynet-common`) shared by all modules

---

## Performance

| Metric | OpenCV Method (`cv`) | Deep Learning (`dl`) |
|--------|---------------------|---------------------|
| Detection Latency | 5-15ms | 15-30ms |
| Decision Latency | <2ms | <2ms |
| IPC Latency (SHM) | ~0.01ms | ~0.01ms |
| End-to-End Latency | 10-25ms | 30-50ms |
| Target FPS | 30+ | 30+ |
| GPU Usage | None | ~2GB VRAM |
| Viewer Latency | +50-100ms | +50-100ms |

---

## Communication

### ZMQ Ports

| Port | Direction | Purpose |
|------|-----------|---------|
| 5557 | LKAS → Viewer | Data broadcast (frames, detections, telemetry) |
| 5558 | Viewer → LKAS | Action commands (pause/resume/respawn) |
| 5559 | Viewer → LKAS | Parameter updates (detection + decision tuning) |
| 5560 | LKAS → Servers | Parameter forwarding to detection/decision servers |
| 5561 | LKAS → Vehicle | Action forwarding to simulation/vehicle |
| 5562 | Vehicle → LKAS | Vehicle status broadcasting |

### Web Viewer

| Port | Protocol | Purpose |
|------|----------|---------|
| 8080 | HTTP | Web interface (HTML/CSS/JS) |
| 8081 | WebSocket | Binary frame streaming |

### Shared Memory Channels

- **ImageChannel** — Ultra-low latency frame transfer (~0.01ms)
- **DetectionChannel** — Lane detection results
- **ControlChannel** — Steering commands

---

## Contributing

We welcome contributions! Each repository has its own contribution guidelines. Please:

1. Fork the relevant repository
2. Create a feature branch
3. Follow the existing code style (Black, isort, flake8)
4. Submit a pull request

---

## Team

**Skynet Team** — SEA:ME Autonomous Driving Project

---

## License

See individual repository LICENSE files.

---

**Built with:** Python, OpenCV, PyTorch, CARLA, ZeroMQ, Flask
