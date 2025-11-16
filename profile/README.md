# Skynet

**Modular Autonomous Driving System for Lane Keeping Assistance**

A distributed, real-time autonomous driving system built for the CARLA simulator, featuring modular architecture with independent components for detection, simulation, and monitoring.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      ADS-Skynet Ecosystem                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌──────────────┐         ┌──────────────┐                   │
│    │    CARLA     │         │    Viewer    │                   │
│    │  Simulator   │         │  (Monitor)   │                   │
│    └──────┬───────┘         └──────▲───────┘                   │
│           │                        │                           │
│           ▼                        │ ZMQ                       │
│    ┌──────────────┐         ┌──────┴───────┐                   │
│    │  Simulation  │◄───────►│     LKAS     │                   │
│    │   (Client)   │   SHM   │  (Detection  │                   │
│    │              │         │  & Decision) │                   │
│    └──────────────┘         └──────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Repositories

### [Lkas](https://github.com/ADS-Skynet/Lkas) - Core LKAS Module
**Lane Keeping Assist System - Detection & Decision**

Real-time computer vision pipeline for lane detection and vehicle steering control.

- **Detection Methods:**
  - Classical OpenCV (Canny + Hough Transform)
  - Deep Learning (YOLO-based segmentation)
- **Decision Control:**
  - PD controller for steering
  - Lane departure detection
  - Adaptive throttle policy
- **Communication:**
  - ZMQ broker for distributed messaging
  - Shared memory for ultra-low latency IPC
  - Live parameter tuning support

```bash
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast
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

### [Viewer](https://github.com/ADS-Skynet/Viewer) - Web Monitoring
**Web-based Remote Viewer**

Real-time monitoring and control interface for the autonomous driving system.

- **Live Monitoring:**
  - Camera feed streaming
  - Lane detection visualization
  - Vehicle state dashboard
- **Remote Control:**
  - Pause/Resume/Respawn
  - Parameter adjustment
  - Detection method switching
- **Data Analysis:**
  - Latency metrics
  - Detection confidence
  - Control performance

```bash
pip install git+https://github.com/ADS-Skynet/Viewer.git
viewer --port 8080
```

---

## Quick Start

### Prerequisites

- Python 3.10+
- CARLA Simulator 0.9.16
- CUDA-capable GPU (optional, for deep learning)

### Full System Setup

```bash
# Terminal 1: Start CARLA Simulator
./CarlaUE4.sh

# Terminal 2: Start LKAS (Detection + Decision)
pip install git+https://github.com/ADS-Skynet/Lkas.git
lkas --method cv --broadcast

# Terminal 3: Start Simulation (CARLA Client)
pip install git+https://github.com/ADS-Skynet/Carla-client.git
simulation --broadcast

# Terminal 4: Start Web Viewer (Optional)
pip install git+https://github.com/ADS-Skynet/Viewer.git
viewer --port 8080

# Open browser: http://localhost:8080
```

### Development Setup

For local development with all modules:

```bash
# Clone all repositories
git clone git@github.com:ADS-Skynet/Lkas.git
git clone git@github.com:ADS-Skynet/Carla-client.git
git clone git@github.com:ADS-Skynet/Viewer.git

# Install in editable mode
pip install -e ./Lkas
pip install -e ./Carla-client
pip install -e ./Viewer
```

---

## Key Features

- **Modular Architecture** - Each component runs independently
- **Low Latency** - Shared memory IPC for real-time performance (<50ms end-to-end)
- **Flexible Detection** - Switch between CV and DL methods on-the-fly
- **Live Tuning** - Adjust parameters without system restart
- **Remote Monitoring** - Web-based viewer accessible from any device
- **Distributed Deployment** - Run components on different machines

---

## Performance

| Metric | OpenCV Method | YOLO Method |
|--------|--------------|-------------|
| Detection Latency | 5-15ms | 20-40ms |
| End-to-End Latency | 10-25ms | 30-50ms |
| Target FPS | 60+ | 30+ |
| GPU Usage | None | ~2GB VRAM |

---

## Communication

| Port | Purpose |
|------|---------|
| 5557 | Data broadcast (→ Viewer) |
| 5558 | Action commands (← Viewer) |
| 5559 | Parameter updates (← Viewer) |
| 5560 | Frame input (← Simulation) |
| 5563 | Steering output (→ Simulation) |

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
