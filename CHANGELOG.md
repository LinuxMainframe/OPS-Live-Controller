# Changelog

All notable changes to the Remote OBS Controller and Camera Streaming System are documented in this file. The format is based on Keep a Changelog, and this project adheres to Semantic Versioning.

## [Unreleased]

### Planned
- Multi-field support with concurrent monitoring
- Advanced bracket integration with automatic team lookup
- Web dashboard for system monitoring and control
- Enhanced error recovery with intelligent retry strategies
- C library support for improved performance and portability:
  - **libwsv5**: C port of OBS WebSocket v5 protocol
    - Already constructed with approximately 75% of full protocol implemented
    - Provides low-level WebSocket communication for OBS scene control
  - **libpicc**: C port of FFmpeg-to-V4L2 loopback device handler
    - Extended functionality to support any IP PoE camera output connected to persistent streaming endpoint
    - Automatic stream quality and resolution detection with intelligent selection algorithm
    - Handles multi-camera stream management with minimal overhead
  - **libwspar**: C port of web server parser for scoreboard monitoring
    - Generic webpage parsing library adaptable to various scoreboard systems
    - Two-component architecture: HTML parser object and search/monitor agent
    - Parser opens and processes any webpage, agent searches for changes and monitors specific elements
  - **libsslog**: C port of scene switching logic engine
    - Not yet constructed, planned implementation for custom event-driven scene transitions
    - JSON-based configuration files for defining custom switching behaviors
    - Drop-in behavior system allowing users to create and load saved event patterns
    - Modular design for different event types beyond paintball tournaments

---

## Remote OBS Controller

### [2.5.0b] - 2025-08-21

#### Added
- Ultra-fast polling implementation with dynamic 0.1-second intervals during critical moments (break timer nearing zero, game starts) for instant scene transitions
- Game pause detection by tracking three consecutive identical timer values (approximately 0.3 seconds) to prevent unwanted scene changes during game stalls
- Task cancellation mechanism for breakout sequence to avoid overlapping transitions during rapid game state changes
- Scene switch caching system to prevent redundant OBS WebSocket scene switches and optimize performance
- Enhanced breakout sequence with precise timing: 7 seconds Breakout Scene, 30 seconds Default Scene, then Game Scene with asynchronous task handling for non-blocking transitions
- Strengthened OBS keep-alive mechanism with robust ping handling and automatic reconnection logic for reliable WebSocket connectivity
- Further suppression of verbose third-party logging (selenium, urllib3) to focus on critical ROC events, with ChromeDriver logs redirected to /dev/null

#### Changed
- Refactored entire codebase into class-based structure with `ROCController` class for improved organization, maintainability, and state encapsulation
- Enhanced configuration handling with default fallback and key merging to gracefully handle missing configuration fields
- Improved game start detection logic to identify new games via significant time jumps (>60 seconds), common game start times (5, 10, 12 minutes), or break timer reaching zero with active game timer
- Optimized scoreboard parsing to prioritize JavaScript `scoreboardState` access with DOM fallback only when necessary, reducing parsing overhead
- Removed sleep delays during critical transitions (break-to-game) for maximum responsiveness to state changes
- Added support for mid-match startup scenarios (post-power surge recovery) ensuring correct scene selection based on current game state at script initialization

#### Fixed
- Mid-match startup behavior now correctly switches to appropriate scene when script starts during ongoing match
- Eliminated excessive Selenium HTML dumps in logs by setting appropriate logging levels and suppressing ChromeDriver output

### [2.4.0] - 2025-08-20

#### Added
- SQLite database integration (`/var/roc/roc.db`) for persistent error logging (`error_logs`), match data tracking (`match_logs`), and team image storage (`teams`) as BLOBs (later removed in v2.2.8)
- Automated setup script (`setup.py`) to handle dependency installation, virtual environment creation, configuration file setup, and systemd service configuration
- Systemd service configuration (`roc-controller.service`) for reliable startup with automatic restart capability via `systemctl`
- Comprehensive network connectivity checks for WAN (8.8.8.8) and LAN (scoreboard IPs: 192.168.1.222:5000 for Field 1, 192.168.1.251:5000 for Field 2) with errors logged to file and database
- Persistent page loading mechanism in Selenium to maintain single browser session, polling for updates to key DOM elements (`main-game-team1-name`, `main-game-team2-name`, `break-timer`, `game-timer`) without constant page reloading
- Team image storage functionality in database (placeholder implementation, not fully utilized, removed in v2.2.8)

#### Changed
- Updated Field 1 scoreboard IP from 192.168.1.200:5000 to 192.168.1.222:5000, maintaining Field 2 at 192.168.1.251:5000
- Removed one-second stability check for data validation, now accepting first valid data (non-placeholder team names, valid timer formats) to prevent premature exits on timer changes
- Made bracket file (`bracket.ods`) optional, logging warning instead of error if missing to allow continued operation without tournament schedule data
- Refined camera switching logic: Breakout Scene (7 seconds), Default Scene (30 seconds), then Game Scene for new games, with 40-second alternation between Game Scene and Default Scene during active play
- Enhanced logging with detailed state transition tracking and comprehensive error reporting to `/var/log/roc-controller.log` and database

#### Fixed
- Resolved `FileNotFoundError` for missing `/root/orss/bracket.ods` by implementing empty match list fallback
- Fixed premature parsing of placeholder team names ("abcd", "efghi") by implementing `WebDriverWait` to ensure valid `scoreboardState` or DOM data before processing
- Corrected persistent Interview Scene issue caused by invalid scoreboard data or incorrect state detection, improving transition logic

### [2.2.8] - 2025-07-15

#### Removed
- SQLite database functionality (`/var/roc/roc.db`) including error logging, match logging, and team image storage due to lack of utilization and unnecessary complexity

#### Changed
- Refined virtual environment existence checks for improved handling of pre-existing `venv` at `/opt/roc-venv`
- Streamlined logging to focus exclusively on file-based output (`/var/log/roc-controller.log`), removing database logging dependencies

### [2.2.5] - 2025-06-20

#### Added
- Robust virtual environment verification using `.roc-venv` marker file to ensure persistence across system reboots
- Reliable Python-based dependency installation within virtual environment, replacing earlier less reliable bash-based approach

#### Fixed
- Resolved dependency installation issues by implementing native Python `pip` package installation for required libraries (pandas, obs-websocket-py, selenium, beautifulsoup4, websockets, odfpy)

### [2.1.12] - 2025-05-10

#### Added
- Comprehensive logging system with detailed debugging information including timestamps, state transitions, and error context for improved traceability
- Modernized systemd service configuration using current best practices with proper `WorkingDirectory`, `User`, and `Restart` directives for enhanced reliability

#### Changed
- Standardized log format across all modules for better readability and consistency, facilitating debugging and system monitoring

### [2.1.9] - 2025-04-25

#### Fixed
- Further refined OBS WebSocket connection handling with improved reconnection logic and error recovery to reduce communication overhead and increase stability

### [2.1.5] - 2025-04-10

#### Changed
- Enhanced OBS WebSocket session persistence to reduce connection drops and optimize communication efficiency

#### Fixed
- Addressed excessive WebSocket traffic by streamlining authentication and request handling, resulting in more stable OBS connection

### [2.1.0] - 2025-03-15

#### Added
- Consolidated architecture with single-file structure (`main.py`), merging previously separate Python modules for simplified maintenance and deployment

#### Changed
- Integrated all functionality (scoreboard parsing, OBS control, dependency management) into unified `main.py` script, eliminating need for bash script orchestration
- Migrated dependency management to Python-based approach (partially successful, fully resolved in v2.2.5)

#### Removed
- Deprecated multiple Python module files and bash script orchestration from v2.0.0, reducing system complexity

### [2.0.0] - 2025-02-01

#### Added
- Initial production release of Remote OBS Controller designed specifically for paintball tournament livestreaming automation
- Selenium-based scoreboard monitoring and data parsing from Field 1 (192.168.1.200:5000) and Field 2 (192.168.1.251:5000) scoreboards
- OBS WebSocket integration for scene control with authentication and automatic scene switching
- Tournament bracket parsing from `bracket.ods` files using pandas and odfpy libraries
- Network connectivity verification for both LAN and WAN connections
- Manual pause/resume functionality via `/tmp/roc-pause` file for operator control
- Initial camera switching logic implementing Interview Scene, Break Scene, Game Scene, Breakout Scene, and Default Scene transitions
- Persistent virtual environment at `/opt/roc-venv` with bash script-based dependency management
- Systemd service integration for deployment in Linux LXC container environment
- Bash script orchestration layer for managing multiple Python modules and dependencies

#### Known Issues
- Script failure with `FileNotFoundError` when `bracket.ods` file is missing
- Premature parsing capturing placeholder team names ("abcd", "efghi") due to incomplete scoreboard loading
- One-second stability check causing unexpected exits on large timer changes
- Excessive HTML output from Selenium cluttering log files
- Frequent stalling on Interview Scene due to invalid data or state detection errors
- Bash script-based dependency management proving less reliable than desired

### [1.x.x] - 2023-2024 (Estimated)

#### Development Phase
- Initial prototyping and proof-of-concept development
- Multiple iteration cycles on scoreboard parsing logic
- Early OBS WebSocket integration experiments
- Testing with live tournament events for validation
- Data crash event requiring system rebuild (main.py survived intact)

---

## Camera Streaming System

### [1.1.0] - 2025-08-21

#### Added
- FPS stabilization mechanism with `-vf fps=fps=<source_fps>` and `-vsync 1` in FFmpeg commands to enforce source frame rates and reduce duplicate frame generation
- Real-time streaming optimization with `-re` and `-rtmp_live live` flags for proper RTMP input handling and live streaming mode
- Camera status summary table logging all camera configurations (IP, stream type, resolution, FPS, status) after initial setup completion
- Duplicate frame tracking in stream testing, parsing and logging `dup=X` counts from FFmpeg output
- Log rotation recommendation using `logrotate` for automated log file management

#### Changed
- Increased stream test timeout from 5 to 15 seconds (`TEST_TIMEOUT`) to improve stream detection reliability on high-latency networks
- Increased retry logic from default to 12 attempts (`MAX_RETRIES`) representing 4 complete cycles through available stream types for robust error recovery
- Modified quality scoring algorithm to penalize duplicate frames using formula: `width * height * fps * (1 - dup_count / 1000)`
- Enhanced error log monitoring with updated `grep` pattern to include "end of file" errors for more comprehensive error capture

#### Fixed
- Resolved FPS overshooting issues (e.g., 13 fps output for 12 fps source stream) through enforced frame rate control with `-vf fps` and `-r <source_fps>` parameters
- Reduced excessive frame duplication (e.g., `dup=137` for camera2) by implementing variable frame rate syncing with `-vsync 1`
- Fixed incomplete camera processing ensuring all configured cameras (including previously skipped cameras at 192.168.1.233, 192.168.1.68) are processed with status logged

### [1.0.0] - 2025-08-01

#### Added
- Initial production implementation of camera streaming system with RTMP-to-V4L2 integration
- Multi-camera support for up to 16 IP cameras with automatic V4L2 loopback device mapping
- Intelligent stream testing and selection logic evaluating main, ext, and sub streams based on quality metrics
- Comprehensive logging system with per-camera logs in `/var/log/cameras/` and centralized error log at `/var/log/ffmpeg_errors.log`
- Systemd service integration for headless operation with automatic restart capability
- Camera connectivity testing via socket connections to RTMP port 1935 before stream initialization
- Automatic stream fallback mechanism for failed primary streams
- Signal handling for graceful shutdown on SIGINT and SIGTERM

#### Known Issues
- Stream failures for some cameras (e.g., 192.168.1.9, 192.168.1.174) due to "End of file" and "Input/output error" events
- FPS instability with output rates exceeding source rates (e.g., 13 fps for 12 fps source)
- Excessive duplicate frame generation (e.g., `dup=137` for camera2) affecting stream quality
- Incomplete processing for some cameras (e.g., 192.168.1.233, 192.168.1.68) during initial setup phase

### [0.x.x] - 2024-2025 (Estimated)

#### Development Phase
- Initial FFmpeg integration experiments with RTMP sources
- V4L2 loopback device testing and configuration optimization
- Stream quality evaluation algorithm development
- Multi-camera handling and process management implementation
- Error recovery logic iteration and refinement

---

## V4L2 Loopback Setup Script

### [1.0.0] - 2025-08-21

#### Added
- Automated DKMS module management for v4l2loopback with version tracking via Git commits
- Intelligent update detection comparing local and remote Git commits to determine if module rebuild is necessary
- Broken DKMS entry cleanup removing entries with missing `dkms.conf` files
- Old version removal logic cleaning up outdated DKMS installations before new version deployment
- Current version force removal ensuring clean installation state
- Module loading with 16 devices configured in exclusive capabilities mode
- Video device numbering from 0-15 with labeled card names ('Cam0' through 'Cam15')
- User context preservation for Git operations to maintain proper file permissions

#### Technical Details
- Version determination using `git describe --always --dirty` for accurate commit-based versioning
- Kernel version detection for DKMS installation targeting current running kernel
- Module parameters: `devices=16`, `exclusive_caps=1`, `video_nr=0-15`, custom card labels

---

## System Integration

### [2.5.0] - 2025-08-21

#### Infrastructure Improvements
- Complete system integration between ROC and camera streaming with unified configuration approach
- Coordinated logging strategy across all system components for centralized monitoring
- Systemd service orchestration ensuring proper startup order and dependency management
- Shared network validation framework used by both ROC and camera streaming components

#### Documentation
- Comprehensive unified README.md covering both systems with detailed installation procedures
- Complete CHANGELOG.md tracking version history across all components
- Technical architecture documentation explaining design decisions and implementation details
- Troubleshooting guide covering common issues across both systems

---

## Historical Notes

### Data Recovery Event (Estimated 2023-2024)

A significant data loss event occurred during the development phase, requiring substantial system reconstruction. The main ROC controller logic (`main.py`) survived intact, preserving core functionality including scoreboard parsing, state detection, and OBS integration. Supporting systems including the camera streaming infrastructure, configuration management, and deployment scripts required complete rewriting.

This event led to improved architectural decisions:
- Stronger separation of concerns between components
- More robust error handling and recovery mechanisms
- Enhanced logging for debugging and system monitoring
- Better documentation practices for system replication

The rebuilt systems incorporated lessons learned from the original implementation, resulting in more maintainable, reliable, and performant code.

### Version Numbering

Version numbers follow Semantic Versioning (MAJOR.MINOR.PATCH):
- **MAJOR**: Incompatible API changes or complete system rewrites
- **MINOR**: New functionality added in backward-compatible manner
- **PATCH**: Backward-compatible bug fixes
- **Beta suffix (b)**: Pre-release versions under active testing

The ROC and camera streaming system maintain independent version numbers reflecting their separate development cycles, with coordinated releases for major infrastructure updates.

---

## Migration Guide

### Upgrading from v2.4.0 to v2.5.0b

1. **Configuration Changes**:
   - Add `polling_interval: 0.1` to `/etc/roc/config.json` if not present
   - Verify all scene names match OBS configuration
   - No changes required for camera streaming configuration

2. **Database Removal** (if upgrading from v2.4.0):
   - Database functionality removed in v2.2.8, no migration necessary
   - Remove `/var/roc/roc.db` if it exists
   - Update any external scripts relying on database

3. **Service Updates**:
   ```bash
   sudo systemctl stop roc-controller
   sudo systemctl stop camera-streamer
   sudo cp main.py /usr/local/bin/
   sudo cp camera_streamer.py /usr/local/bin/
   sudo systemctl start roc-controller
   sudo systemctl start camera-streamer
   ```

4. **Verification**:
   ```bash
   tail -f /var/log/roc-controller.log
   tail -f /var/log/camera_streamer.log
   ```

### Upgrading Camera Streaming from v1.0.0 to v1.1.0

1. **Update Configuration**:
   - No configuration changes required
   - Existing `/etc/roc/cameras.json` compatible with v1.1.0

2. **Update Scripts**:
   ```bash
   sudo systemctl stop camera-streamer
   sudo cp camera_streamer.py /usr/local/bin/
   sudo cp setup_v4l2loopback.sh /usr/local/bin/
   sudo systemctl start camera-streamer
   ```

3. **Monitor Improvements**:
   ```bash
   # Check for improved FPS stability
   grep "Selected" /var/log/camera_streamer.log
   
   # Verify reduced duplicate frames
   grep "dup=" /var/log/cameras/camera*.log
   ```

---

## License

This project is licensed under the Affero General Public License v3.0 (AGPLv3).

Key points:
- Source code must be made available to users of the software over a network
- Modifications must be released under AGPLv3
- Commercial use is permitted with source disclosure
- No warranty provided

See LICENSE file for complete terms and conditions.

---

## Acknowledgments

**Primary Developer**: Aidan A. Bradley  
**Organization**: Original Paintball Series (OPS)  
**Project Duration**: 2.5+ years (2022-2025)

Special recognition:
- OPS staff and volunteers for field testing and feedback
- Paintball community for supporting live streaming initiatives
- Open source contributors to underlying technologies (FFmpeg, Selenium, OBS)

This project represents significant iteration and refinement through real-world deployment at live tournament events. The system has processed thousands of games and managed hundreds of camera streams, continuously improving through operational experience.

---

**Last Updated**: October 15, 2025  
**Document Version**: 2.5.0  
**Maintainer**: Aidan A. Bradley
