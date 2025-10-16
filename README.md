# Remote OBS Controller & Camera Streaming System

![License](https://img.shields.io/badge/license-AGPLv3-blue.svg)
![Python](https://img.shields.io/badge/python-3.8+-blue.svg)
![Version](https://img.shields.io/badge/version-2.5.0b-green.svg)

## Project Overview

This repository contains two complementary systems designed for live sports broadcasting automation, specifically optimized for paintball tournament livestreaming. The project consists of the Remote OBS Controller (ROC) for automated scene management and a camera streaming system for RTMP-to-V4L2 integration.

### Remote OBS Controller (ROC)

The ROC is an autonomous scene management system for Open Broadcaster Software (OBS) that handles real-time transitions based on live scoreboard data. Deployed as a systemd service within a Linux LXC container, it enables hands-free operation during live events, allowing operators to focus on commentary and field camera work.

### Camera Streaming System

The camera streaming system manages multiple IP camera feeds, converting RTMP streams to V4L2 loopback devices for seamless integration with OBS Studio. The system includes intelligent stream selection, automatic failover, and robust error recovery.

## Table of Contents

- [Features](#features)
- [System Requirements](#system-requirements)
- [Installation](#installation)
  - [ROC Setup](#roc-setup)
  - [Camera Streaming Setup](#camera-streaming-setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [Technical Architecture](#technical-architecture)
- [Development](#development)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Features

### Remote OBS Controller

- **Autonomous Scene Management**: Automatically switches between Interview, Break, Game, Breakout, and Default scenes based on real-time game state
- **Persistent Browser Sessions**: Maintains single Selenium session for efficient scoreboard monitoring with dynamic data validation
- **Intelligent State Detection**: Uses timer-based heuristics to detect game starts, breaks, and pauses with sub-second accuracy
- **Ultra-Fast Polling**: 0.1-second polling intervals during critical moments ensure instant transitions
- **Pause Detection**: Prevents unwanted scene changes during game stalls by tracking consecutive identical timer values
- **Robust Network Verification**: Validates connectivity to scoreboards and OBS WebSocket before operation
- **WebSocket Keep-Alive**: Maintains persistent OBS connection with automatic reconnection and ping handling
- **Manual Override**: Supports operator intervention via pause file for manual scene control
- **Mid-Match Startup**: Correctly handles startup during ongoing matches (post-power surge scenarios)

### Camera Streaming System

- **Multi-Camera Support**: Handles up to 16 IP cameras simultaneously
- **Intelligent Stream Selection**: Automatically selects optimal stream quality (main, ext, sub) based on resolution, FPS, and frame duplication metrics
- **Automatic Failover**: Falls back to lower-quality streams when primary streams fail
- **Frame Rate Stabilization**: Enforces source FPS to prevent overshooting and duplicate frames
- **Real-Time Monitoring**: Continuously monitors FFmpeg processes with automatic restart on failure
- **V4L2 Integration**: Seamless integration with V4L2 loopback devices for OBS Studio compatibility
- **Comprehensive Logging**: Per-camera logs with centralized error tracking for debugging
- **DKMS Management**: Automatic handling of V4L2 loopback module updates and cleanup

## System Requirements

### Hardware
- CPU: Multi-core processor (4+ cores recommended for multiple camera streams)
- RAM: 8GB minimum (16GB recommended for 16 cameras)
- Network: Gigabit Ethernet for stable RTMP streaming
- Storage: 20GB free space for logs and temporary files

### Software
- Operating System: Ubuntu 20.04 LTS or later (22.04 recommended)
- Python: 3.8 or higher
- Chromium and ChromeDriver (for ROC)
- FFmpeg: 4.2 or higher (for camera streaming)
- V4L2 Loopback: Modified version from [aab18011/v4l2loopback](https://github.com/aab18011/v4l2loopback)

### Network Requirements
- Stable connectivity to IP cameras on port 1935 (RTMP)
- Access to scoreboard web interfaces (typically ports 5000 or 8080)
- Internet connectivity for dependency installation (8.8.8.8)
- OBS Studio host accessible via WebSocket (default port 4455)

## Installation

### ROC Setup

1. **Install System Dependencies**:
   ```bash
   sudo apt update
   sudo apt install -y python3-venv python3-pip chromium chromedriver
   ```

2. **Clone Repository**:
   ```bash
   git clone https://github.com/aab18011/OPS-Live-Controller.git
   cd OPS-Live-Controller
   ```

3. **Create Configuration Directory**:
   ```bash
   sudo mkdir -p /etc/roc
   sudo cp config.json.example /etc/roc/config.json
   sudo chmod 644 /etc/roc/config.json
   ```

4. **Configure ROC**:
   Edit `/etc/roc/config.json` with your environment settings:
   ```json
   {
       "obs": {
           "host": "192.168.1.100",
           "port": 4455,
           "password": "your_obs_password"
       },
       "scoreboards": {
           "field1": "192.168.1.222:5000",
           "field2": "192.168.1.251:5000"
       },
       "bracket_file": "/root/orss/bracket.ods",
       "chrome_binary": "/usr/bin/chromium",
       "chrome_driver": "/usr/bin/chromedriver",
       "default_scene": "Default Scene",
       "break_scene": "Break Scene",
       "game_scene": "Game Scene",
       "breakout_scene": "Breakout Scene",
       "interview_scene": "Interview Scene",
       "venv_path": "/opt/roc-venv",
       "field_number": 1,
       "polling_interval": 0.1
   }
   ```

5. **Install ROC as Systemd Service**:
   ```bash
   sudo cp roc-controller.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable roc-controller
   sudo systemctl start roc-controller
   ```

6. **Verify Installation**:
   ```bash
   sudo systemctl status roc-controller
   tail -f /var/log/roc-controller.log
   ```

### Camera Streaming Setup

1. **Install Dependencies**:
   ```bash
   sudo apt update
   sudo apt install -y python3 python3-pip ffmpeg git dkms
   ```

2. **Install V4L2 Loopback Module**:
   Follow instructions at [aab18011/v4l2loopback](https://github.com/aab18011/v4l2loopback):
   ```bash
   cd /home/user/Documents
   git clone https://github.com/aab18011/v4l2loopback.git
   cd v4l2loopback
   sudo make && sudo make install
   sudo depmod -a
   ```

3. **Deploy Setup Script**:
   ```bash
   sudo cp setup_v4l2loopback.sh /usr/local/bin/
   sudo chmod +x /usr/local/bin/setup_v4l2loopback.sh
   ```

4. **Configure Cameras**:
   Create `/etc/roc/cameras.json`:
   ```json
   [
       {
           "ip": "192.168.1.110",
           "user": "admin",
           "password": "camera_password"
       },
       {
           "ip": "192.168.1.142",
           "user": "admin",
           "password": "camera_password"
       }
   ]
   ```

5. **Deploy Camera Streamer**:
   ```bash
   sudo cp camera_streamer.py /usr/local/bin/
   sudo chmod +x /usr/local/bin/camera_streamer.py
   sudo mkdir -p /var/log/cameras
   ```

6. **Install as Systemd Service**:
   ```bash
   sudo cp camera-streamer.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable camera-streamer
   sudo systemctl start camera-streamer
   ```

7. **Setup Log Rotation**:
   Create `/etc/logrotate.d/camera_streamer`:
   ```
   /var/log/cameras/*.log /var/log/camera_streamer.log /var/log/ffmpeg_errors.log {
       daily
       rotate 7
       compress
       delaycompress
       missingok
       notifempty
       create 644 root root
   }
   ```

8. **Verify Camera Streams**:
   ```bash
   ls /dev/video*
   ffplay -f v4l2 -i /dev/video0
   tail -f /var/log/camera_streamer.log
   ```

## Configuration

### ROC Configuration

The ROC configuration file at `/etc/roc/config.json` defines operational parameters:

- **OBS Settings**: WebSocket connection details for scene control
- **Scoreboards**: URLs for field scoreboard web interfaces
- **Scene Names**: Customizable OBS scene names for different game states
- **Polling Interval**: Set to 0.1 seconds for ultra-fast response during critical transitions
- **Virtual Environment**: Path for persistent Python environment isolation
- **Field Number**: Designates which field scoreboard to monitor (1 or 2)

The bracket file (`bracket.ods`) is optional. If missing, the system will log a warning and continue operation without bracket data.

### Camera Streaming Configuration

Camera configuration at `/etc/roc/cameras.json` defines camera parameters:

```json
[
    {
        "ip": "192.168.1.110",
        "user": "admin",
        "password": "camera_password"
    }
]
```

Each camera must specify IP address and password. Username defaults to "admin" if not provided. The system supports up to 16 cameras mapped to `/dev/video0` through `/dev/video15`.

## Usage

### ROC Operation

1. **Start the Service**:
   ```bash
   sudo systemctl start roc-controller
   ```

2. **Monitor Real-Time Logs**:
   ```bash
   tail -f /var/log/roc-controller.log
   ```

3. **Manual Pause Control**:
   - Pause automation:
     ```bash
     touch /tmp/roc-pause
     ```
   - Resume automation:
     ```bash
     rm /tmp/roc-pause
     ```

4. **Stop the Service**:
   ```bash
   sudo systemctl stop roc-controller
   ```

The ROC monitors the configured scoreboard URL, parsing team names and game timers to trigger appropriate scene transitions. Scene switching logic follows:

- **Intermission**: Displays Interview Scene when no active game or break timer
- **Break**: Switches to Break Scene when break timer is counting down
- **Game Start**: Triggers Breakout Scene (7 seconds), then Default Scene (30 seconds), followed by Game Scene
- **During Game**: Alternates between Game Scene and Default Scene every 40 seconds unless game is paused

### Camera Streaming Operation

1. **Start Camera Streaming**:
   ```bash
   sudo systemctl start camera-streamer
   ```

2. **Monitor Stream Status**:
   ```bash
   tail -f /var/log/camera_streamer.log
   ```

3. **View Individual Camera Logs**:
   ```bash
   tail -f /var/log/cameras/camera0.log
   ```

4. **Check Stream Quality**:
   ```bash
   grep "Selected" /var/log/camera_streamer.log
   ```

5. **Test V4L2 Devices**:
   ```bash
   ffplay -f v4l2 -i /dev/video0
   v4l2-ctl --list-devices
   ```

The system automatically selects the best available stream for each camera based on quality scoring (resolution × FPS with penalty for duplicate frames). If a stream fails, it automatically falls back to lower-quality streams and retries.

### OBS Studio Integration

1. **Install OBS Studio**:
   ```bash
   flatpak install flathub com.obsproject.Studio
   ```

2. **Grant Permissions**:
   ```bash
   flatpak override com.obsproject.Studio --device=all
   flatpak override com.obsproject.Studio --filesystem=host
   ```

3. **Add Camera Sources**:
   - Add V4L2 Video Capture sources for each `/dev/videoX` device
   - Set resolution and FPS based on logs from camera streamer
   - Disable buffering in OBS for minimal latency

4. **Configure WebSocket**:
   - Enable OBS WebSocket in Tools > WebSocket Server Settings
   - Set password to match ROC configuration
   - Ensure port 4455 is accessible from ROC container

## Technical Architecture

### ROC Architecture

The ROC uses a class-based architecture centered on the `ROCController` class, leveraging Python's `asyncio` for non-blocking operations critical to real-time performance.

**Key Components**:

- **Scoreboard Parsing**: Selenium maintains a persistent browser session to the scoreboard web interface. The parser prioritizes JavaScript `scoreboardState` object access with BeautifulSoup DOM parsing as fallback. `WebDriverWait` ensures valid data before parsing to prevent premature capture of placeholder values.

- **Game State Detection**: Timer-based heuristics infer game states by analyzing break and game timer changes. New games are detected via significant time jumps (>60 seconds), common start times (5, 10, or 12 minutes), or break timer reaching zero with active game timer. Pause detection tracks three consecutive identical timer readings to prevent scene changes during stalls.

- **OBS Integration**: WebSocket communication handles scene switching with authentication, keep-alive pings (10-second intervals), and scene switch caching to prevent redundant operations. Task cancellation prevents overlapping breakout sequences during rapid state changes.

- **Virtual Environment Management**: Automatic creation and verification of persistent virtual environment at `/opt/roc-venv` with marker file (`.roc-venv`) for state tracking across reboots.

**Design Rationale**:

The asynchronous architecture enables ultra-fast polling (0.1 seconds) during critical moments without blocking execution. Persistent browser sessions reduce overhead compared to page reloads, while scene switch caching and task management optimize WebSocket traffic. Headless Chrome with container-optimized flags (`--no-sandbox`, `--disable-gpu`) ensures compatibility with LXC environments.

### Camera Streaming Architecture

The camera streaming system centers on the `CameraStreamer` class, managing FFmpeg processes for RTMP-to-V4L2 conversion.

**Key Components**:

- **Stream Testing**: Tests all available stream types (main, ext, sub) for each camera, extracting resolution, FPS, and duplicate frame counts from FFmpeg output. Quality scoring (`width × height × fps × (1 - dup_count/1000)`) prioritizes high-resolution, high-framerate streams with minimal duplication.

- **FFmpeg Optimization**: Commands include `-re` for real-time reading, `-rtmp_live live` for proper RTMP handling, `-vf fps=fps=<source_fps>` to enforce frame rates, and `-vsync 1` for variable frame rate syncing. These parameters stabilize output and reduce duplicate frames.

- **Process Management**: Monitors FFmpeg processes continuously, implementing exponential backoff retry logic (5 to 30 seconds) with automatic fallback to lower-quality streams after 12 retry attempts (4 cycles through stream types).

- **V4L2 Integration**: The setup script (`setup_v4l2loopback.sh`) manages DKMS module installation, cleaning up broken entries and removing old versions before installing the current version. It loads the module with 16 devices using exclusive capabilities mode.

**FFmpeg Command Structure**:

```bash
ffmpeg -re -rtmp_live live -i rtmp://<ip>/bcs/channel0_<type>.bcs?channel=0&stream=<num>&user=<user>&password=<pass> -vf fps=fps=<fps> -vsync 1 -r <fps> -f v4l2 /dev/video<i>
```

This structure ensures stable, real-time streaming with minimal latency and frame duplication.

## Development

### Project Structure

```
.
├── main.py                      # ROC main script
├── camera_streamer.py           # Camera streaming script
├── setup_v4l2loopback.sh        # V4L2 module setup script
├── roc-controller.service       # ROC systemd service
├── camera-streamer.service      # Camera systemd service
├── README.md                    # This file
├── CHANGELOG.md                 # Version history
└── /etc/roc/
    ├── config.json              # ROC configuration
    └── cameras.json             # Camera configuration
```

### Contributing

Contributions are welcome. To contribute:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Commit changes with descriptive messages (`git commit -m 'Add feature X'`)
4. Push to your fork (`git push origin feature/your-feature`)
5. Open a pull request with detailed description of changes

Code should follow PEP 8 style guidelines. Include tests for new features and update documentation accordingly.

### Testing

The systems have been validated for:

- **ROC**: Mid-match startup, game state transitions, pause detection, network reliability under 50-750ms latency, scene switching precision
- **Camera Streaming**: Multi-camera stream selection, automatic failover, FPS stabilization, frame duplication reduction, V4L2 device integration

To test locally, configure test scoreboard and camera infrastructure, start services, and monitor logs for expected behavior.

### Logging

Both systems maintain comprehensive logs:

- **ROC**: `/var/log/roc-controller.log` for operational events
- **Camera Streamer**: `/var/log/camera_streamer.log` for system events, `/var/log/cameras/camera<i>.log` for per-camera FFmpeg output, `/var/log/ffmpeg_errors.log` for centralized error tracking

Third-party library verbosity is suppressed (Selenium, urllib3, ChromeDriver) to maintain log clarity. Log rotation is recommended via `logrotate` to manage disk usage.

## Troubleshooting

### ROC Issues

**Problem**: Service fails to start or exits immediately  
**Solution**: Check logs for specific errors. Common issues include missing configuration file, invalid JSON syntax, or network connectivity problems. Verify scoreboard URLs are accessible and OBS WebSocket is enabled.

**Problem**: Stuck on Interview Scene  
**Solution**: Verify scoreboard is loading valid team data (not placeholders like "abcd" or "efghi"). Check browser session is established correctly. Increase `WebDriverWait` timeout if network latency is high.

**Problem**: Scene switches lag behind game state  
**Solution**: Verify polling interval is set to 0.1 seconds. Check OBS WebSocket connection stability. Ensure system resources are adequate for Selenium and WebSocket operations.

**Problem**: Virtual environment creation fails  
**Solution**: Verify Python 3.8+ is installed. Check `/opt/roc-venv` directory permissions. Manually create venv and install dependencies if automatic creation fails.

### Camera Streaming Issues

**Problem**: Stream failures with "End of file" errors  
**Solution**: Verify camera connectivity (`ping <ip>` and `nc -zv <ip> 1935`). Test RTMP stream directly with `ffplay`. Check camera firmware is up-to-date. Increase `TEST_TIMEOUT` if network latency is high.

**Problem**: FPS overshooting (e.g., 13 fps for 12 fps stream)  
**Solution**: Verify FFmpeg commands include `-vf fps=fps=<source_fps>` and `-vsync 1`. Check logs for actual source FPS and adjust if mismatched.

**Problem**: Excessive duplicate frames  
**Solution**: Confirm `-vsync 1` and `-r <source_fps>` are applied. Check quality scores in logs for high `dup=X` values. Consider camera settings if duplicates persist across all streams.

**Problem**: V4L2 devices not found  
**Solution**: Run `/usr/local/bin/setup_v4l2loopback.sh` manually. Verify V4L2 loopback module is loaded (`lsmod | grep v4l2loopback`). Check for DKMS errors in setup script output.

**Problem**: Camera processing incomplete  
**Solution**: Review `/var/log/camera_streamer.log` for errors. Ensure all cameras in configuration are reachable. Check system resources are adequate for number of cameras.

### Log Analysis

Monitor logs for patterns:
- **ROC**: Look for state transitions, WebSocket errors, authentication failures, or parsing issues
- **Camera Streamer**: Check for stream test results, FFmpeg exit codes, connectivity failures, or quality degradation

Use `grep` to filter logs:
```bash
grep -i "error" /var/log/roc-controller.log
grep "Selected" /var/log/camera_streamer.log
grep "dup=" /var/log/cameras/camera0.log
```

## License

This project is licensed under the Affero General Public License v3.0 (AGPLv3). See LICENSE file for complete terms.

## Acknowledgments

**Author**: Aidan A. Bradley  
**Organization**: OPS (Original Paintball Series)  
**Purpose**: Live tournament broadcasting automation

The system has evolved over 2.5 years through iterative development, surviving a data crash that required significant rewriting. The core controller logic (main.py) survived intact, while supporting systems were rebuilt with improved architecture and robustness.

Special thanks to the paintball community for feedback and testing during live events.
