# Franka Panda Research Robot V4 - Drake Integration

Drake-based forward/inverse kinematics and motion planning for Franka Emika Panda (FER) robot with support for both simulation and hardware deployment.

## Prerequisites

- **OS:** Ubuntu 24.04 (Noble Numbat)
- **RT Kernel** 
- **Docker Desktop:** For building libfranka 0.8.0
- **Drake:** Built from source using Bazel
- **Hardware:** Franka Emika Panda (FER) with firmware v4.x

**Note:** These are the steps that I followed and may not be the most optimal method.

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

### 3. Install Drake from Source
Follow the documentation provided here:
https://drake.mit.edu/bazel.html

Set environment variables(TBV):
**These commands may or may not be required — needs verification**

```bash
echo 'export DRAKE_INSTALL_DIR=$HOME/drake' >> ~/.bashrc
echo 'export PATH=$DRAKE_INSTALL_DIR/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$DRAKE_INSTALL_DIR/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export PYTHONPATH=~/drake/install/lib/python3.12/site-packages:$PYTHONPATH' >> ~/.bashrc
source ~/.bashrc
```

### 4. Clone and Build franka_drake
```bash
mkdir -p ~/ws/franka
cd ~/ws/franka
git clone https://github.com/KhachDavid/franka_drake.git
cd franka_drake
```

**Important:** Edit `CMakeLists.txt` to add manual Franka setup:
```bash
nano CMakeLists.txt
```
Find the line:
```cmake
find_package(Franka REQUIRED)
```

Replace with:
```cmake
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
```

Build simulation version:
```bash
mkdir build && cd build
cmake .. -DCMAKE_PREFIX_PATH="$HOME/drake/install"
make -j$(nproc)
```

### 5. Install libfranka
```bash
cd ~
git clone https://github.com/frankaemika/libfranka.git
cd libfranka
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

### 6. Build libfranka 0.8.0 for Hardware (Docker)
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

### 7. Build franka_drake for Hardware
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

### 8. Set Library Path
```bash
echo 'export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

## Usage

### Running Simulation

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

Built-in libfranka examples:
```bash
cd ~/libfranka/build/examples
./generate_elbow_motion 127.0.0.1
./generate_consecutive_motions 127.0.0.1
```

Custom Drake-based examples:
```bash
cd ~/ws/franka/franka_drake/build
# Forward Kinematics: <robot_ip> <j0> <j1> <j2> <j3> <j4> <j5> <j6> <speed>
./bin/panda-fk-simple 127.0.0.1 0 -45 0 -135 0 90 45 0.5

# Inverse Kinematics: <robot_ip> <x> <y> <z> <speed>
./bin/panda-ik-simple 127.0.0.1 0.5 0.0 0.5 0.3
```

### Running on Hardware

**Prerequisites:**
- Robot powered on and connected via Ethernet
- Network configured (robot at 172.16.0.2)
- Brakes released in Franka Desk
- User stop button released (white light)
- FCI mode activated

Built-in libfranka examples:
```bash
export LD_LIBRARY_PATH=~/libfranka-hw-bin:$LD_LIBRARY_PATH
cd ~/libfranka-hw-bin/examples
./generate_elbow_motion 172.16.0.2
./generate_consecutive_motions 172.16.0.2
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
