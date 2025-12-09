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
│         ▼                                     (8080-81) │            │
│  ┌─────────────┐          ┌──────────────┐             │             │
│  │ Simulation  │◄────────►│     LKAS     │◄────────────┘             │
│  │  (Client)   │  ZMQ     │  Detection + │      ZMQ                  │
│  │             │  5560-63 │   Decision   │    (5557-59)              │
│  └─────────────┘          └──────┬───────┘                           │
│                                  │ SHM/ZMQ                           │
│  ┌─────────────┐                 │                                   │
│  │   Vehicle   │◄────────────────┘                                   │
│  │ (Jetracer)  │           ZMQ (5560-63)                             │
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
  - `cv`: Classical OpenCV (Canny edge detection + Hough Transform) - 5-15ms latency
  - `dl`: Deep Learning (PyTorch-based segmentation) - 15-30ms latency
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
  - Configurable broadcasting (15-60 FPS)

```bash
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast  # OpenCV detection
lkas --method dl --broadcast  # Deep learning detection
```

---

### [Carla-client](https://github.com/ADS-Skynet/Carla-client) - Simulation Module
**CARLA Simulator Integration**

Orchestrates the connection between CARLA simulator and the LKAS module.

- **Vehicle Management:**
  - Spawn and control vehicles
  - Camera sensor setup
  - Apply steering/throttle commands
- **Data Pipeline:**
  - Frame capture and preprocessing
  - Shared memory publishing
  - Real-time data broadcasting
- **Visualization:**
  - HUD overlay
  - Lane visualization
  - Performance metrics

```bash
pip install git+https://github.com/ADS-Skynet/Carla-client.git
simulation --broadcast
```

---

### [Viewer](https://github.com/ADS-Skynet/Web-viewer) - Web Monitoring
**Web-based Remote Viewer with WebSocket Streaming**

Real-time monitoring and control interface for the autonomous driving system with optimized binary streaming.

- **Live Monitoring:**
  - WebSocket-based camera feed streaming (binary, no base64 overhead)
  - Lane detection visualization with HUD overlay
  - Vehicle state telemetry dashboard
  - Real-time FPS and latency metrics

- **Remote Control:**
  - Pause/Resume/Respawn actions
  - Live parameter tuning (Canny, Hough, PID gains)
  - Detection method switching (cv/dl)
  - Throttle policy adjustment

- **Performance:**
  - ~50-100ms total latency (acquisition → browser)
  - 30 FPS max for bandwidth optimization
  - JPEG quality: 75 (configurable)
  - Rendering offloaded to laptop (vehicle CPU stays free)

```bash
pip install git+https://github.com/ADS-Skynet/Web-viewer.git
viewer --port 8080  # HTTP server at 8080, WebSocket at 8081
# Open browser: http://localhost:8080
```

---

### [Common](https://github.com/ADS-Skynet/Common) - Shared Infrastructure
**Platform-Independent Shared Components**

Lightweight shared library providing common types, communication utilities, and visualization for all modules.

- **Shared Types:**
  - Lane data models (Lane, LaneDepartureStatus, LaneMetrics)
  - Detection data (DetectionData with lane coordinates)
  - Vehicle state (VehicleState telemetry)

- **Communication:**
  - ZMQ pub-sub utilities (ViewerSubscriber, ActionPublisher, ParameterPublisher)
  - Unified message protocols for inter-process communication

- **Visualization:**
  - LKASVisualizer for rendering lane overlays and HUD
  - Consistent visual style across all modules

- **Configuration:**
  - Unified ConfigManager for YAML + dataclasses
  - Shared configuration schemas

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
  - Camera capture via V4L2 video device
  - Configurable camera device path

- **Communication:**
  - ZMQ integration with LKAS broker
  - State publishing (telemetry) at configurable Hz
  - Parameter subscription for live tuning
  - Action subscription (pause/resume/respawn)

- **Testing Utilities:**
  - Motor test script
  - Camera test script
  - Hardware diagnostics

```bash
pip install git+https://github.com/ADS-Skynet/Vehicle-jetracer.git
vehicle --device /dev/video4 --publish-state-hz 10
```

---

## Quick Start

### Prerequisites

- Python 3.10+
- CARLA Simulator 0.9.16
- CUDA-capable GPU (optional, for deep learning)

### Simulation Setup (CARLA)

```bash
# Terminal 1: Start CARLA Simulator
./CarlaUE4.sh

# Terminal 2: Start LKAS (Detection + Decision)
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast  # or --method dl for deep learning

# Terminal 3: Start Simulation (CARLA Client)
pip install git+https://github.com/ADS-Skynet/Carla-client.git
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
git clone git@github.com:ADS-Skynet/Carla-client.git
git clone git@github.com:ADS-Skynet/Vehicle-jetracer.git
git clone git@github.com:ADS-Skynet/Web-viewer.git

# Install in editable mode (order matters - common first!)
pip install -e ./Common
pip install -e ./Lkas
pip install -e ./Carla-client
pip install -e ./Vehicle-jetracer
pip install -e ./Web-viewer
```

---

## Key Features

- **Modular Architecture** - Each component runs independently with clean interfaces
- **Ultra-Low Latency** - Shared memory IPC (~0.01ms) for real-time performance
- **Pluggable Algorithms** - Switch between CV/DL detection and PD/PID/MPC controllers
- **Live Tuning** - Adjust detection and decision parameters without restart
- **Real Hardware Support** - Run on actual Jetracer vehicles with full telemetry
- **WebSocket Streaming** - Optimized binary streaming (no base64 overhead)
- **Remote Monitoring** - Web-based viewer accessible from any device
- **Distributed Deployment** - Run components on different machines via ZMQ
- **Platform Independent** - Works on Linux, macOS (including M1), and embedded systems

---

## Performance

| Metric | OpenCV Method (`cv`) | Deep Learning (`dl`) |
|--------|---------------------|---------------------|
| Detection Latency | 5-15ms | 15-30ms |
| Decision Latency | <2ms | <2ms |
| IPC Latency (SHM) | ~0.01ms | ~0.01ms |
| End-to-End Latency | 10-25ms | 30-50ms |
| Target FPS | 60+ | 30+ |
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
| 5560 | Simulation/Vehicle → LKAS | Frame input (camera feed) |
| 5562 | Vehicle → LKAS | Vehicle status (telemetry) |
| 5563 | LKAS → Simulation/Vehicle | Steering output (control commands) |

### Web Viewer

| Port | Protocol | Purpose |
|------|----------|---------|
| 8080 | HTTP | Web interface (HTML/CSS/JS) |
| 8081 | WebSocket | Binary frame streaming |

### Shared Memory Channels

- **ImageChannel** - Ultra-low latency frame transfer (~0.01ms)
- **DetectionChannel** - Lane detection results
- **ControlChannel** - Steering commands

---

## Contributing

We welcome contributions! Each repository has its own contribution guidelines. Please:

1. Fork the relevant repository
2. Create a feature branch
3. Follow the existing code style (Black, isort, flake8)
4. Submit a pull request

---

## Team

**Skynet Team** - SEA:ME Autonomous Driving Project

---

## License

See individual repository LICENSE files.

---

**Built with:** Python, OpenCV, PyTorch, CARLA, ZeroMQ, Flask
