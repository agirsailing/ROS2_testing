# Running ROS2 in Docker

## Introduction

This guide covers running ROS2 Jazzy (the latest LTS release) in Docker containers and integrating with VSCode or Cursor for development. Using Docker for ROS2 development offers several advantages:

- **No OS changes required**: Run ROS2 on macOS, Windows, or any Linux distribution without native installation
- **Reproducible environments**: Share identical development setups across team members
- **Isolation**: Keep ROS2 dependencies separate from your host system
- **Easy cleanup**: Remove containers without affecting your system

ROS2 Jazzy is based on Ubuntu Noble (24.04) and has long-term support until May 2029.

## Prerequisites

### Docker

**macOS:**
1. Download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. Start Docker Desktop from Applications
3. Verify installation:
   ```bash
   docker run hello-world
   ```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install docker.io
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

### IDE Setup

Install either VSCode or Cursor:

**VSCode:**
- Download from [code.visualstudio.com](https://code.visualstudio.com/)

**Cursor:**
- Download from [cursor.com](https://cursor.com/)

### Dev Containers Extension

Open the Extensions panel (`Cmd+Shift+X` on macOS, `Ctrl+Shift+X` on Linux/Windows) and install:

- **VSCode**: Search for "Dev Containers" (`ms-vscode-remote.remote-containers`)
- **Cursor**: Search for "Dev Containers" (`anysphere.remote-containers`)

### XQuartz (macOS only - for GUI applications)

If you need GUI applications like RViz2:

```bash
brew install --cask xquartz
```

After installation, **restart your computer**, then configure XQuartz:
1. Open XQuartz from Applications
2. Go to **XQuartz → Preferences → Security**
3. Check **"Allow connections from network clients"**

## Quick Start

### Available Docker Images

Docker Hub hosts official ROS2 Jazzy images:

| Image | Description | Size |
|-------|-------------|------|
| `ros:jazzy-ros-core` | Minimal ROS2 core | Smallest |
| `ros:jazzy-ros-base` | Base ROS2 + common dependencies | Medium |
| `osrf/ros:jazzy-desktop` | Desktop tools (RViz2, demos) | Large |
| `ros:jazzy-perception` | Perception libraries (OpenCV, PCL) | Large |

### Pull and Run

```bash
# Pull the desktop image (includes demo packages and RViz2)
docker pull osrf/ros:jazzy-desktop

# Run interactively
docker run -it osrf/ros:jazzy-desktop
```

Once inside the container, verify ROS2 is working:

```bash
# List installed packages
ros2 pkg list

# Get help
ros2 --help
```

## Running ROS2 Nodes

### Single Container with Multiple Nodes

Run the classic talker/listener demo in one container:

```bash
docker run -it osrf/ros:jazzy-desktop

# Inside the container, run listener in background and talker in foreground
ros2 run demo_nodes_cpp listener &
ros2 run demo_nodes_cpp talker
```

You should see the talker publishing messages and the listener receiving them.

### Multiple Containers

Open two terminals and run nodes in separate containers:

**Terminal 1 - Talker:**
```bash
docker run -it --rm osrf/ros:jazzy-desktop ros2 run demo_nodes_cpp talker
```

**Terminal 2 - Listener:**
```bash
docker run -it --rm osrf/ros:jazzy-desktop ros2 run demo_nodes_cpp listener
```

The containers communicate automatically via DDS discovery on the same network.

### Docker Compose

For more complex setups, use Docker Compose. Create a `docker-compose.yml`:

```yaml
version: '3'
services:
  talker:
    image: osrf/ros:jazzy-desktop
    command: ros2 run demo_nodes_cpp talker
    environment:
      - ROS_DOMAIN_ID=42

  listener:
    image: osrf/ros:jazzy-desktop
    command: ros2 run demo_nodes_cpp listener
    depends_on:
      - talker
    environment:
      - ROS_DOMAIN_ID=42
```

Run with:
```bash
docker compose up
```

Stop with `Ctrl+C` or `docker compose down`.

## GUI Support (RViz2, Gazebo, etc.)

### macOS Setup with XQuartz

After installing and configuring XQuartz (see Prerequisites):

**Step 1: Start XQuartz**
```bash
open -a XQuartz
```

**Step 2: Allow network connections (run on your Mac, not in Docker)**
```bash
# Allow connections from localhost and local IP
xhost + 127.0.0.1
xhost + localhost
```

**Step 3: Run container with display forwarding**
```bash
docker run -it \
  -e DISPLAY=host.docker.internal:0 \
  -e QT_X11_NO_MITSHM=1 \
  -e LIBGL_ALWAYS_INDIRECT=1 \
  osrf/ros:jazzy-desktop
```

**Step 4: Test with RViz2 inside the container**
```bash
source /opt/ros/jazzy/setup.bash
rviz2
```

**If you get "could not connect to display" errors:**

1. Verify XQuartz is running: `ps aux | grep XQuartz`
2. Check XQuartz Preferences → Security → "Allow connections from network clients" is checked
3. Restart XQuartz after changing security settings
4. Re-run the `xhost` commands after restarting XQuartz
5. Try using your Mac's IP instead:
   ```bash
   # On your Mac, get your IP
   IP=$(ifconfig en0 | grep inet | awk '$1=="inet" {print $2}')
   xhost + $IP
   
   # Then run container with that IP
   docker run -it \
     -e DISPLAY=$IP:0 \
     -e QT_X11_NO_MITSHM=1 \
     -e LIBGL_ALWAYS_INDIRECT=1 \
     osrf/ros:jazzy-desktop
   ```

### Linux Setup

On Linux, display forwarding is simpler:

```bash
xhost +local:docker

docker run -it \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  osrf/ros:jazzy-desktop
```

### Apple Silicon (M1/M2/M3) Limitations

RViz2 and other 3D applications may have rendering issues on Apple Silicon Macs due to OpenGL emulation through X11. Alternatives:

- **Foxglove Studio**: Web-based visualization that works without X11. Run `ros2 launch foxglove_bridge foxglove_bridge_launch.xml` and connect via browser.
- **PlotJuggler**: For data visualization without 3D rendering
- **rqt tools**: Some rqt plugins work better than full RViz2

## Dev Container Setup for VSCode/Cursor

For serious development, set up a dev container that automatically configures your environment.

### Workspace Structure

Create this folder structure:

```
ros2_ws/
├── .devcontainer/
│   ├── devcontainer.json
│   └── Dockerfile
└── src/
    └── (your ROS2 packages go here)
```

### Create the Dockerfile

Create `.devcontainer/Dockerfile`:

```dockerfile
FROM ros:jazzy

ARG USERNAME=rosdev
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Delete user if it exists (Ubuntu Noble has 'ubuntu' user with UID 1000)
RUN if id -u $USER_UID > /dev/null 2>&1; then userdel $(id -un $USER_UID); fi

# Create the user
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && apt-get update \
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Install development tools
RUN apt-get update && apt-get install -y \
    python3-pip \
    python3-colcon-common-extensions \
    python3-rosdep \
    git \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Initialize rosdep
RUN rosdep init || true

ENV SHELL=/bin/bash

# Set the default user
USER $USERNAME

# Update rosdep as user
RUN rosdep update

# Source ROS2 setup in bashrc
RUN echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc

CMD ["/bin/bash"]
```

### Create devcontainer.json

Create `.devcontainer/devcontainer.json`:

```json
{
  "name": "ROS2 Jazzy Development",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "USERNAME": "rosdev"
    }
  },
  "workspaceFolder": "/home/rosdev/ws",
  "workspaceMount": "source=${localWorkspaceFolder},target=/home/rosdev/ws,type=bind",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools",
        "ms-python.python",
        "ms-iot.vscode-ros",
        "redhat.vscode-xml",
        "redhat.vscode-yaml",
        "eamodio.gitlens"
      ],
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      }
    }
  },
  "containerEnv": {
    "ROS_DOMAIN_ID": "42",
    "ROS_AUTOMATIC_DISCOVERY_RANGE": "LOCALHOST"
  },
  "runArgs": [
    "--network=host"
  ],
  "postCreateCommand": "sudo chown -R $(whoami) /home/rosdev/ws && cd /home/rosdev/ws && rosdep install --from-paths src --ignore-src -y || true"
}
```

### macOS-specific devcontainer.json

For macOS with GUI support, use this version instead:

```json
{
  "name": "ROS2 Jazzy Development",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "USERNAME": "rosdev"
    }
  },
  "workspaceFolder": "/home/rosdev/ws",
  "workspaceMount": "source=${localWorkspaceFolder},target=/home/rosdev/ws,type=bind",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-vscode.cpptools",
        "ms-vscode.cmake-tools",
        "ms-python.python",
        "ms-iot.vscode-ros",
        "redhat.vscode-xml",
        "redhat.vscode-yaml",
        "eamodio.gitlens"
      ],
      "settings": {
        "terminal.integrated.defaultProfile.linux": "bash"
      }
    }
  },
  "containerEnv": {
    "DISPLAY": "host.docker.internal:0",
    "QT_X11_NO_MITSHM": "1",
    "LIBGL_ALWAYS_INDIRECT": "1",
    "ROS_DOMAIN_ID": "42",
    "ROS_AUTOMATIC_DISCOVERY_RANGE": "LOCALHOST"
  },
  "postCreateCommand": "sudo chown -R $(whoami) /home/rosdev/ws && cd /home/rosdev/ws && rosdep install --from-paths src --ignore-src -y || true"
}
```

Note: On macOS, `--network=host` doesn't work the same as Linux. Remove it and use explicit port forwarding if needed.

## Connecting VSCode/Cursor to the Container

### Open in Dev Container

1. Open VSCode or Cursor
2. Open your workspace folder: **File → Open Folder** (`Cmd+K Cmd+O`)
3. Select your `ros2_ws` folder containing `.devcontainer/`
4. When prompted "Folder contains a Dev Container configuration file", click **Reopen in Container**

   Or manually: Open Command Palette (`Cmd+Shift+P`) and run **Dev Containers: Reopen in Container**

5. Wait for the container to build (first time takes a few minutes)

### Verify the Setup

Once connected, open a terminal in VSCode/Cursor (`Ctrl+` ` or **View → Terminal**):

```bash
# Verify ROS2 is available
ros2 --help

# Check environment
echo $ROS_DISTRO
# Should output: jazzy

# List available packages
ros2 pkg list
```

### Test with Demo Nodes

```bash
# In one terminal
ros2 run demo_nodes_cpp talker

# In another terminal (click + in terminal panel)
ros2 run demo_nodes_cpp listener
```

### Build Your Own Package

```bash
# Navigate to src folder
cd ~/ws/src

# Create a new package
ros2 pkg create --build-type ament_python my_package

# Go back to workspace root and build
cd ~/ws
colcon build

# Source the workspace
source install/setup.bash
```

## Tips and Troubleshooting

### ROS2 Nodes Can't Communicate Between Containers

Ensure all containers use the same `ROS_DOMAIN_ID`:
```bash
export ROS_DOMAIN_ID=42
```

### GUI Not Displaying on macOS

**"could not connect to display" or "Could not load Qt platform plugin xcb" errors:**

1. Ensure XQuartz is running: `open -a XQuartz`
2. Check XQuartz Preferences → Security → "Allow connections from network clients" is **checked**
3. **Restart XQuartz** after changing security settings (quit and reopen)
4. Run these commands on your Mac (not in Docker):
   ```bash
   xhost + 127.0.0.1
   xhost + localhost
   ```
5. Verify DISPLAY inside container: `echo $DISPLAY` should show `host.docker.internal:0`
6. If still failing, try with your Mac's IP:
   ```bash
   # On Mac
   export IP=$(ifconfig en0 | grep inet | awk '$1=="inet" {print $2}')
   xhost + $IP
   # Then use -e DISPLAY=$IP:0 when running docker
   ```

### Permission Denied Errors

If you get permission errors on mounted volumes:
```bash
sudo chown -R $(whoami) /home/rosdev/ws
```

### Rebuild the Dev Container

If you change the Dockerfile or devcontainer.json:
- Command Palette → **Dev Containers: Rebuild Container**

### Check Container Logs

If the container fails to start:
- Command Palette → **Dev Containers: Show Container Log**

## References

- [ROS2 Jazzy Documentation](https://docs.ros.org/en/jazzy/)
- [OSRF Docker Images](https://hub.docker.com/r/osrf/ros/tags)
- [VSCode Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers)
- [ROS2 Docker How-To Guide](https://docs.ros.org/en/jazzy/How-To-Guides/Run-2-nodes-in-single-or-separate-docker-containers.html)
