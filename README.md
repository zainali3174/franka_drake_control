# Franka Panda Research Robot V4 - Drake Integration

Drake-based forward/inverse kinematics and motion planning for Franka Emika Panda (FER) robot with support for both simulation and hardware deployment.

## Prerequisites

- **OS:** Ubuntu 24.04 (Noble Numbat)
- **RT Kernel** 
- **Docker Desktop:** For building libfranka 0.8.0
- **Drake:** Built from source using Bazel
- **Ros:** ROS 2 Jazzy
- **Hardware:** Franka Emika Panda (FER) with firmware v4.x

**Note:** These are the steps that I followed and may not be the most optimal method. This project assumes a standard development environment with common robotics/system libraries already installed.
Additional dependencies may be required depending on your setup. Please install any missing packages reported during compilation.

## Current System Architecture
```
┌─────────────────────────────────────────────────────────┐
│                    HOST SYSTEM (Ubuntu 24.04)           │
│                                                         │
│  ┌──────────────────────┐      ┌────────────────────┐   │
│  │   SIMULATION TRACK   │      │   HARDWARE TRACK   │   │
│  │                      │      │                    │   │
│  │  libfranka 0.10.x+   │      │  libfranka 0.8.0   │   │
│  │  (FCI Protocol v5)   │      │  (FCI Protocol v4) │   │
│  │                      │      │                    │   │
│  │  Target: 127.0.0.1   │      │  Target: 172.16.0.2│   │
│  └──────────┬───────────┘      └─────────┬──────────┘   │
│             │                            │              │
│             ▼                            ▼              │
│  ┌──────────────────────┐      ┌────────────────────┐   │
│  │  Drake FCI Sim       │      │  Real Robot FCI    │   │
│  │  (franka_drake)      │      │  Control Box       │   │
│  └──────────────────────┘      └────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Installation Steps

### 1. Install RT Kernel
Follow the documentation provided here:
https://frankarobotics.github.io/docs/libfranka/docs/real_time_kernel.html

After installation, reboot your computer:
- GRUB menu -> Advanced options for Ubuntu -> Select kernel with '-rt' (e.g., 6.8.0-rt8)

Verify RT kernel:
```bash
uname -r
# Should show something like: 6.8.0-rt8
```

### 2. Install Docker Desktop
Follow the documentation provided here:
https://docs.docker.com/desktop/setup/install/linux/ubuntu/

**Note:** Make sure docker is running before running any docker command.

### 3. Install Drake

**Note:** These instructions cover Drake built from source using Bazel. If you want to avoid hours of compilation, a pre-built binary alternative is provided below (currently unverified - TBV).

#### Option A: Drake Built from Source (Recommended - Verified)

Follow the documentation provided here:
https://drake.mit.edu/bazel.html

Set environment variables for source build:
```bash
echo 'export DRAKE_INSTALL_DIR=$HOME/drake/install' >> ~/.bashrc
echo 'export CMAKE_PREFIX_PATH=$HOME/drake/install:$CMAKE_PREFIX_PATH' >> ~/.bashrc
echo 'export PATH=$DRAKE_INSTALL_DIR/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$DRAKE_INSTALL_DIR/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export PYTHONPATH=$HOME/drake/install/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
source ~/.bashrc
```

#### Option B: Drake Pre-built Binary (Alternative - TBV)

```bash
cd ~
wget https://github.com/RobotLocomotion/drake/releases/download/v1.40.0/drake-1.40.0-noble.tar.gz
mkdir -p ~/drake
tar -xvzf drake-1.40.0-noble.tar.gz -C drake --strip-components=1
```

Set environment variables for pre-built binary:
```bash
echo 'export DRAKE_INSTALL_DIR=$HOME/drake' >> ~/.bashrc
echo 'export CMAKE_PREFIX_PATH=$HOME/drake:$CMAKE_PREFIX_PATH' >> ~/.bashrc
echo 'export PATH=$DRAKE_INSTALL_DIR/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$DRAKE_INSTALL_DIR/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export PYTHONPATH=$HOME/drake/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
echo 'source /opt/ros/jazzy/setup.bash' >> ~/.bashrc
source ~/.bashrc
```


### 4. Installing ROS 2 Jazzy 
```bash

sudo apt install software-properties-common curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
# Install ROS
sudo apt update
sudo apt install ros-jazzy-desktop python3-colcon-common-extensions
```
### 5. Create Workspace and Clone Panda Description

```bash
# Create your workspace 
mkdir -p ~/ws/franka/src
cd ~/ws/franka/src

# Get the panda description
git clone https://github.com/m-elwin/franka_description
cd franka_description
git checkout panda
```

### 6. Install libfranka (Used for Simulation)

```bash
cd ~
git clone https://github.com/KhachDavid/libfranka.git
cd libfranka
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
sudo make install
```

### 7. Clone franka_drake Driver

```bash
cd ~/ws/franka
git clone https://github.com/KhachDavid/franka_drake.git
cd franka_drake
```

**Important:** Edit `CMakeLists.txt` to add manual Franka setup:
```bash
nano CMakeLists.txt
```
Find the lins:
```cmake
find_package(drake REQUIRED)
find_package(Poco REQUIRED COMPONENTS Foundation Net)

```

 with:
```cmake
find_package(drake REQUIRED)
# Manual Franka setup for hardware compatibility
if(DEFINED ENV{FRANKA_HW_BUILD})
    add_library(Franka::Franka SHARED IMPORTED)
    set_target_properties(Franka::Franka PROPERTIES
        IMPORTED_LOCATION "$ENV{HOME}/libfranka-hw-bin/libfranka.so.0.8.0"
        IMPORTED_SONAME "libfranka.so.0.8"
        INTERFACE_INCLUDE_DIRECTORIES "$ENV{HOME}/libfranka-hw-bin/include"
    )
else()
    find_package(Franka REQUIRED)
endif()
find_package(Poco REQUIRED COMPONENTS Foundation Net)
```

### 8. Generate URDF for Drake

Build franka_description package:
```bash
cd ~/ws/franka
colcon build --packages-select franka_description
source install/setup.bash
```

Generate URDF:
```bash
cd src/franka_description
git branch # you should be on panda branch
xacro robots/fer/fer.urdf.xacro hand:=true > fer_drake.urdf
```

**Important:** The URDF uses .stl meshes, which cause scale/unit problems in Drake. To avoid errors, collision meshes need to be commented out.

Create a Python script to comment out STL collisions:
```bash
cd ~/ws/franka/src/franka_description
cat > comment_stl_collisions.py << 'EOF'
import re

with open('fer_drake.urdf', 'r') as f:
    content = f.read()

lines = content.split('\n')
result = []
in_collision = False
collision_buffer = []
has_stl = False

for line in lines:
    if '<collision' in line:
        in_collision = True
        collision_buffer = [line]
        has_stl = False
    elif in_collision:
        collision_buffer.append(line)
        if '.stl' in line:
            has_stl = True
        if '</collision>' in line:
            if has_stl:
                result.append('    <!-- Temporarily disabled for Drake')
                result.extend(collision_buffer)
                result.append('    -->')
            else:
                result.extend(collision_buffer)
            in_collision = False
            collision_buffer = []
    else:
        result.append(line)

with open('fer_drake.urdf', 'w') as f:
    f.write('\n'.join(result))

print("Done! STL collisions commented out.")
EOF
```

Run the script:
```bash
python3 comment_stl_collisions.py
```

### 9. Build franka_drake

```bash
cd ~/ws/franka/franka_drake
mkdir -p build && cd build
cmake ..
make -j$(nproc)
```
**Note:** Now you can test your simulations.

### 10. Build libfranka 0.8.0 for Hardware (Docker)
```bash
cd ~
git clone https://github.com/frankaemika/libfranka.git libfranka-hw
cd libfranka-hw
git checkout 0.8.0
git submodule update --init --recursive
```

Create Dockerfile:
```bash
cat > Dockerfile.libfranka080 << 'EOF'
FROM ubuntu:18.04

RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    git \
    libpoco-dev \
    libeigen3-dev

WORKDIR /opt/libfranka
COPY . .

RUN git submodule update --init --recursive && \
    mkdir -p build && cd build && \
    cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTS=OFF .. && \
    cmake --build . -- -j4

CMD ["/bin/bash"]
EOF
```

Build Docker image:
```bash
docker build --no-cache -f Dockerfile.libfranka080 -t libfranka:0.8.0 .
```

Extract binaries and dependencies:
```bash
mkdir -p ~/libfranka-hw-bin

# Extract build artifacts
docker run --rm -v ~/libfranka-hw-bin:/output libfranka:0.8.0 \
    bash -c "cp -r /opt/libfranka/build/* /output/"

# Extract Poco libraries
docker run --rm -v ~/libfranka-hw-bin:/output libfranka:0.8.0 \
    bash -c "cp /usr/lib/libPoco*.so* /output/"

# Extract headers
docker run --rm -v ~/libfranka-hw-bin:/output libfranka:0.8.0 \
    bash -c "cp -r /opt/libfranka/include /output/"
```

Verify version:
```bash
ls ~/libfranka-hw-bin/libfranka.so*
# Should show: libfranka.so.0.8 and libfranka.so.0.8.0
```

### 11. Build franka_drake for Hardware
```bash
cd ~/ws/franka/franka_drake
mkdir build-hw && cd build-hw

export FRANKA_HW_BUILD=1
cmake .. -DCMAKE_PREFIX_PATH="$HOME/drake/install"
make -j$(nproc)
```

Verify correct linking:
```bash
ldd ./bin/panda-fk-simple | grep libfranka
# Should show: libfranka.so.0.8 => /home/<user>/libfranka-hw-bin/libfranka.so.0.8
```

### 13. Set Library Path
```bash
echo 'export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

## Usage

### Running a libfranka built-in example in simulation

**Terminal 1:** Start Drake simulation server
```bash
cd ~/ws/franka/franka_drake/build
./bin/franka-fci-sim-embed-example
```

**Browser:** Open Meshcat visualization
```
http://localhost:7000/
```

**Terminal 2:** Run examples
```bash
cd ~/libfranka/build/examples
./generate_consecutive_motions 127.0.0.1
```

### Running a libfranka built-in example on hardware

**Prerequisites:**
- Robot powered on and connected via Ethernet
- Network configured (robot at 172.16.0.2)
- Brakes released in Franka Desk
- User stop button released (white light)
- FCI mode activated

**Terminal 1:** Run examples
```bash
export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH
cd ~/libfranka-hw-bin/examples
./generate_elbow_motion 172.16.0.2
```

## Creating Your Own Examples

### File Structure
```bash
cd ~/ws/franka/franka_drake/examples
touch my_example.cpp
nano my_example.cpp
# Write your code here
```

### Update CMakeLists.txt
```bash
nano ~/ws/franka/franka_drake/examples/CMakeLists.txt
```

Add at the end:
```cmake
# My custom example
add_executable(my_example my_example.cpp)
target_link_libraries(my_example franka_drake_core Franka::Franka)
set_target_properties(my_example PROPERTIES
    OUTPUT_NAME "my-example"
)
install(TARGETS my_example
    RUNTIME DESTINATION bin
)
```

### Build for Simulation
```bash
cd ~/ws/franka/franka_drake/build
cmake ..
make my_example -j$(nproc)
./bin/my-example 127.0.0.1
```

### Build for Hardware
```bash
cd ~/ws/franka/franka_drake/build-hw
export FRANKA_HW_BUILD=1
cmake .. -DCMAKE_PREFIX_PATH="$HOME/drake/install"
make my_example -j$(nproc)
export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH
./bin/my-example 172.16.0.2
```

Custom Drake-based examples:
```bash
cd ~/ws/franka/franka_drake/build
# Forward Kinematics: <robot_ip> <j0> <j1> <j2> <j3> <j4> <j5> <j6> <speed>
./bin/panda-fk-simple 127.0.0.1 0 -45 0 -135 0 90 45 0.5

# Inverse Kinematics: <robot_ip> <x> <y> <z> <speed>
./bin/panda-ik-simple 127.0.0.1 0.5 0.0 0.5 0.3
```

Custom Drake-based examples:
```bash
cd ~/ws/franka/franka_drake/build-hw
export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH

# Forward Kinematics
./bin/panda-fk-simple 172.16.0.2 0 -45 0 -135 0 90 45 0.5

# Inverse Kinematics
./bin/panda-ik-simple 172.16.0.2 0.5 0.0 0.5 0.3
```

### Robot Connection Refused

**Checklist:**
- [ ] Robot powered on
- [ ] Network cable connected
- [ ] Correct IP (172.16.0.2)
- [ ] Robot unlocked in Franka Desk
- [ ] FCI mode activated
- [ ] No other program controlling robot



## Key Differences: Simulation vs Hardware

| Aspect | Simulation | Hardware |
|--------|-----------|----------|
| **Build Directory** | `build/` | `build-hw/` |
| **libfranka Version** | Modern (0.10.x+) | 0.8.0 |
| **libfranka Location** | System `/usr/local/` | `~/libfranka-hw-bin/` |
| **Target IP** | 127.0.0.1 | 172.16.0.2 |
| **RT Kernel** | Not needed | Required |
| **LD_LIBRARY_PATH** | Not needed | Must set to `~/libfranka-hw-bin` |

## Known Limitations

- **Gripper mimic warning:** Safe to ignore - only affects internal Drake modeling, not actual gripper control
- **Docker for building only:** Cannot run libfranka in Docker due to RT requirements
- **Two libfranka versions:** Necessary due to robot firmware FCI protocol version mismatch
- **Process isolation:** Drake FK computation uses fork() to avoid memory conflicts with libfranka 0.8.0

## Resources

- [Drake Documentation](https://drake.mit.edu)
- [Franka Control Interface (FCI) Docs](https://frankaemika.github.io/docs/)
- [libfranka Compatibility Table](https://frankaemika.github.io/docs/compatibility.html)
- [franka_drake Original Repo](https://github.com/KhachDavid/franka_drake)

## License

MIT License - See LICENSE file for details


## Acknowledgments

- Drake team at MIT
- Franka Emika for libfranka
- franka_drake original developers

---

**For questions or issues, please open a GitHub issue.**
