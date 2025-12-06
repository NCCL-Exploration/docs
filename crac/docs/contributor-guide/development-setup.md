# Development Setup

This guide covers setting up a development environment for contributing to CRAC, including debug builds, testing infrastructure, and debugging tools.

## Prerequisites

### System Requirements
- Linux development environment (Ubuntu 20.04+, CentOS 7+, or similar)
- Development tools installed (git, make, gcc/g++, etc.)
- sudo access for system package installation

### Software Requirements

#### CUDA Development Environment
```bash
# Check CUDA version
nvcc --version
# Should be 12.4 or later

# Check NVIDIA driver
nvidia-smi
# Should show driver 550.54.14 or later
```

#### DMTCP Development Setup
```bash
# Clone DMTCP source (if not already available)
git clone https://github.com/dmtcp/dmtcp.git
cd dmtcp

# Build DMTCP in debug mode
./configure
make DEBUG=1

# Set DMTCP_ROOT for development
export DMTCP_ROOT=$PWD
```

#### Development Tools
```bash
# Ubuntu/Debian
sudo apt-get install build-essential gdb valgrind git

# CentOS/RHEL
sudo yum groupinstall "Development Tools"
sudo yum install gdb valgrind git
```

## Repository Setup

### Clone CRAC Repository
```bash
git clone https://github.com/xu-yao0127/crac.git
cd crac
```

### Development Branch Strategy
```bash
# Create a development branch
git checkout -b feature/my-new-feature

# Keep main branch updated
git checkout main
git pull upstream main

# Rebase development branch
git checkout feature/my-new-feature
git rebase main
```

## Build Configuration

### Debug Build
The default Makefile already includes debug flags:

**Makefile**: `crac/Makefile:20-21`
```makefile
override CFLAGS += -g3 -O0 -fPIC -I${DMTCP_INCLUDE} ${CUDA_INCLUDE}
override CXXFLAGS += -g3 -O0 -fPIC ${DMTCP_INCLUDE} ${CUDA_INCLUDE}
```

**Build Command**:
```bash
make clean
make DMTCP_ROOT=$DMTCP_ROOT DEBUG=1
```

### Release Build
For performance testing:
```bash
make clean
make DMTCP_ROOT=$DMTCP_ROOT CFLAGS="-O2 -fPIC" CXXFLAGS="-O2 -fPIC"
```

### Verbose Build
To see all compiler commands:
```bash
make DMTCP_ROOT=$DMTCP_ROOT VERBOSE=1
```

## Development Tools

### GDB Debugging with DMTCP

#### Debugging Plugin Loading
```bash
# Start GDB with DMTCP launch
gdb --args $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter

# In GDB, set breakpoints
(gdb) break dmtcp_event_hook
(gdb) break checkpoint_gpu
(gdb) break restore_gpu

# Run the program
(gdb) run
```

#### Debugging Restart Process
```bash
# Debug restart process
gdb --args $DMTCP_ROOT/bin/dmtcp_restart ckpt_*.dmtcp

# Set breakpoints for restart events
(gdb) break recreate_pipes
(gdb) break restore_gpu
(gdb) run
```

### CUDA Debugging

#### CUDA-MEMCHECK
```bash
# Run with CUDA memory checker
cuda-memcheck $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

#### CUDA-GDB
```bash
# Debug CUDA kernels
cuda-gdb --args $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

### Valgrind Memory Checking
```bash
# Check for memory leaks in the plugin
valgrind --tool=memcheck --leak-check=full \
  $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

## Testing Infrastructure

### Running Tests
```bash
# Build all tests
make tests

# Run individual tests
./test/counter &
./test/counter_mpi &
./test/mpi_cuda &

# Run with DMTCP
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

### Test Automation Script
Create a test script for automated testing:

**test_runner.sh**:
```bash
#!/bin/bash

set -e

DMTCP_ROOT=${DMTCP_ROOT:-/usr/local}
PLUGIN_PATH="./libdmtcp_crac.so"

# Function to run test with checkpoint/restart
run_test() {
    local test_program=$1
    local test_name=$2
    
    echo "Running test: $test_name"
    
    # Start coordinator in background
    $DMTCP_ROOT/bin/dmtcp_coordinator --port 7779 &
    COORD_PID=$!
    sleep 2
    
    # Launch test program
    $DMTCP_ROOT/bin/dmtcp_launch --coord-port 7779 --with-plugin $PLUGIN_PATH ./test/$test_program &
    TEST_PID=$!
    sleep 3
    
    # Take checkpoint
    $DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779
    sleep 2
    
    # Kill test program
    kill $TEST_PID
    wait $TEST_PID 2>/dev/null || true
    
    # Restart from checkpoint
    $DMTCP_ROOT/bin/dmtcp_restart --coord-port 7779 ckpt_*.dmtcp &
    RESTART_PID=$!
    sleep 3
    
    # Take another checkpoint
    $DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779
    sleep 2
    
    # Cleanup
    kill $RESTART_PID
    kill $COORD_PID
    wait 2>/dev/null || true
    
    # Clean checkpoint files
    rm -f ckpt_*.dmtcp dmtcp_restart_script*.sh
    
    echo "Test $test_name: PASSED"
}

# Run all tests
run_test "counter" "Basic CUDA Counter"
run_test "counter_mpi" "MPI + CUDA Counter"
# run_test "mpi_cuda" "MPI + CUDA Vector Add"  # Requires MPI setup

echo "All tests completed!"
```

### Performance Testing
Create performance benchmarks:

**benchmark.sh**:
```bash
#!/bin/bash

DMTCP_ROOT=${DMTCP_ROOT:-/usr/local}
PLUGIN_PATH="./libdmtcp_crac.so"

# Function to measure checkpoint time
measure_checkpoint() {
    local iterations=$1
    local total_time=0
    
    for i in $(seq 1 $iterations); do
        start_time=$(date +%s%N)
        
        # Start coordinator and program
        $DMTCP_ROOT/bin/dmtcp_coordinator --port 7779 >/dev/null 2>&1 &
        COORD_PID=$!
        sleep 1
        
        $DMTCP_ROOT/bin/dmtcp_launch --coord-port 7779 --with-plugin $PLUGIN_PATH ./test/counter >/dev/null 2>&1 &
        TEST_PID=$!
        sleep 2
        
        # Measure checkpoint time
        checkpoint_start=$(date +%s%N)
        $DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779 >/dev/null 2>&1
        checkpoint_end=$(date +%s%N)
        
        checkpoint_time=$((($checkpoint_end - $checkpoint_start) / 1000000)) # milliseconds
        total_time=$(($total_time + $checkpoint_time))
        
        # Cleanup
        kill $TEST_PID $COORD_PID 2>/dev/null || true
        wait 2>/dev/null || true
        rm -f ckpt_*.dmtcp
    done
    
    avg_time=$(($total_time / $iterations))
    echo "Average checkpoint time: ${avg_time}ms"
}

# Run benchmark
measure_checkpoint 5
```

## Code Quality Tools

### Static Analysis
```bash
# Using cppcheck
cppcheck --enable=all --std=c++11 crac.cpp

# Using clang-tidy (if available)
clang-tidy crac.cpp -- -I$DMTCP_ROOT/include -I/usr/local/cuda/include
```

### Code Formatting
```bash
# Using clang-format (create .clang-format file first)
clang-format -i crac.cpp

# Example .clang-format:
# BasedOnStyle: LLVM
# IndentWidth: 2
# ColumnLimit: 80
```

### Memory Leak Detection
```bash
# Build with debug symbols
make clean && make DMTCP_ROOT=$DMTCP_ROOT

# Run with valgrind
valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all \
  $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

## IDE Setup

### VS Code Configuration
**.vscode/settings.json**:
```json
{
    "cmake.configureOnOpen": false,
    "files.associations": {
        "*.cu": "cpp",
        "*.cuh": "cpp"
    },
    "C_Cpp.default.includePath": [
        "${workspaceFolder}/**",
        "${env:DMTCP_ROOT}/include",
        "/usr/local/cuda/include"
    ],
    "C_Cpp.default.cppStandard": "c++11",
    "C_Cpp.default.cStandard": "c11"
}
```

**.vscode/launch.json**:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug CRAC with DMTCP",
            "type": "cppdbg",
            "request": "launch",
            "program": "${env:DMTCP_ROOT}/bin/dmtcp_launch",
            "args": ["--with-plugin", "${workspaceFolder}/libdmtcp_crac.so", "${workspaceFolder}/test/counter"],
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb"
        }
    ]
}
```

## Common Development Workflows

### Adding a New Feature
1. Create feature branch
2. Implement changes
3. Add/update tests
4. Run full test suite
5. Check code quality
6. Submit pull request

### Debugging a Checkpoint Issue
1. Reproduce with debug build
2. Add debug output to relevant functions
3. Use GDB to trace execution
4. Check CUDA state transitions
5. Verify pipe handling
6. Test with simple program first

### Performance Optimization
1. Profile with `perf` or `nvprof`
2. Identify bottlenecks
3. Optimize critical paths
4. Benchmark improvements
5. Verify correctness

## Environment Variables for Development

```bash
# DMTCP debugging
export DMTCP_DEBUG=checkpoint,restart
export DMTCP_LOG_TIMESTAMP=1

# CUDA debugging
export CUDA_LAUNCH_BLOCKING=1
export CUDA_DEVICE_ORDER=PCI_BUS_ID

# CRAC specific (add to code)
export CRAC_DEBUG=1
```

## See Also

- [Code Walkthrough](code-walkthrough.md) - Detailed code analysis
- [Testing Guide](testing.md) - Comprehensive testing information
- [Adding Features](adding-features.md) - Guidelines for extending CRAC
- [Troubleshooting](../troubleshooting.md) - Common development issues