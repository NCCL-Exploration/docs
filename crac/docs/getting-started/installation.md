# Installation

This guide covers the prerequisites and step-by-step installation instructions for CRAC.

## Prerequisites

### System Requirements
- Linux operating system (tested on Ubuntu 20.04+ and CentOS 7+)
- x86_64 architecture

### Software Requirements

#### CUDA Toolkit
- **CUDA 12.4 or later** - Required for native checkpoint/restart APIs
- NVIDIA driver 550.54.14 or later
- CUDA Driver API libraries (`libcuda.so`)

#### DMTCP
- **DMTCP 3.0.0 or later** - Distributed MultiThreaded Checkpointing tool
- Source installation recommended for plugin development

#### Build Tools
- GCC 7.5+ or compatible C++ compiler
- GNU Make 3.8+
- NVCC (NVIDIA CUDA Compiler) - for building test programs

#### Development Libraries
- Standard C/C++ libraries
- Dynamic linking library (`libdl.so`)
- POSIX threads library (`libpthread.so`)

## Verifying Prerequisites

### Check CUDA Version
```bash
nvcc --version
# Should show CUDA 12.4 or later

nvidia-smi
# Should show driver version 550.54.14 or later
```

### Check CUDA Checkpoint Support
```bash
# Test if CUDA checkpoint APIs are available
cat > test_cuda_checkpoint.c << 'EOF'
#include <cuda.h>
#include <stdio.h>

int main() {
    CUresult ret = cuInit(0);
    if (ret != CUDA_SUCCESS) {
        printf("CUDA initialization failed\n");
        return 1;
    }
    
    // Try to get checkpoint process state
    CUprocessState state;
    ret = cuCheckpointProcessGetState(getpid(), &state);
    if (ret == CUDA_SUCCESS) {
        printf("CUDA checkpoint APIs are available\n");
        return 0;
    } else {
        printf("CUDA checkpoint APIs not available: %d\n", ret);
        return 1;
    }
}
EOF

nvcc -o test_cuda_checkpoint test_cuda_checkpoint.c -lcuda
./test_cuda_checkpoint
```

### Check DMTCP Installation
```bash
# If DMTCP is installed system-wide
dmtcp_launch --version

# Or if built from source
/path/to/dmtcp/bin/dmtcp_launch --version
```

## Building CRAC

### 1. Clone the Repository
```bash
git clone https://github.com/xu-yao0127/crac.git
cd crac
```

### 2. Set DMTCP_ROOT Environment Variable
```bash
# If DMTCP is installed system-wide
export DMTCP_ROOT=/usr/local

# If DMTCP is built from source
export DMTCP_ROOT=/path/to/dmtcp

# Add to your shell profile for persistence
echo 'export DMTCP_ROOT=/path/to/dmtcp' >> ~/.bashrc
```

### 3. Build the Plugin
```bash
make DMTCP_ROOT=$DMTCP_ROOT
```

This will:
- Compile `crac.cpp` into `crac.o`
- Link into `libdmtcp_crac.so` shared library
- Build test programs in the `test/` directory

### 4. Verify Build Output
```bash
ls -la libdmtcp_crac.so
# Should show the plugin library

ls -la test/
# Should show compiled test programs: counter, counter_mpi, mpi_cuda
```

## Build Targets

The Makefile provides several targets:

### Default Build
```bash
make DMTCP_ROOT=$DMTCP_ROOT
# Equivalent to: make default
```
Builds the plugin library and all test programs.

### Plugin Only
```bash
make DMTCP_ROOT=$DMTCP_ROOT libdmtcp_crac.so
```
Builds only the plugin library without tests.

### Tests Only
```bash
make DMTCP_ROOT=$DMTCP_ROOT tests
```
Builds only the test programs.

### Clean Build
```bash
make clean
# Removes object files and library

make distclean
# Removes all build artifacts including checkpoint files
```

### Distribution Package
```bash
make dist
# Creates a tar.gz package of the source
```

## Environment Setup

### Library Paths
Add the CUDA and DMTCP libraries to your system path:

```bash
# CUDA libraries
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

# DMTCP libraries (if built from source)
export LD_LIBRARY_PATH=$DMTCP_ROOT/lib:$LD_LIBRARY_PATH

# Add to shell profile for persistence
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
echo "export LD_LIBRARY_PATH=\$DMTCP_ROOT/lib:\$LD_LIBRARY_PATH" >> ~/.bashrc
```

### Plugin Path
Make note of the plugin path for later use:
```bash
export CRAC_PLUGIN_PATH=$PWD/libdmtcp_crac.so
echo "export CRAC_PLUGIN_PATH=\$PWD/libdmtcp_crac.so" >> ~/.bashrc
```

## Verification Steps

### 1. Test Plugin Loading
```bash
# Check if the plugin can be loaded by DMTCP
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin $PWD/libdmtcp_crac.so --help
# Should show DMTCP help without errors
```

### 2. Run Basic Test
```bash
# Launch a simple CUDA program with CRAC
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin $PWD/libdmtcp_crac.so ./test/counter
```

The program should start running and incrementing a counter. You can then test checkpointing:

```bash
# In another terminal, trigger a checkpoint
$DMTCP_ROOT/bin/dmtcp_command --checkpoint
```

### 3. Check System Logs
```bash
# Look for any CUDA or DMTCP errors
dmesg | grep -i cuda
dmesg | grep -i dmtcp
```

## Common Installation Issues

### CUDA Not Found
```bash
# Error: cuda.h: No such file or directory
# Solution: Install CUDA development package
sudo apt-get install cuda-libraries-dev-12-4  # Ubuntu/Debian
# or
sudo yum install cuda-devel-12-4  # CentOS/RHEL
```

### DMTCP Headers Missing
```bash
# Error: dmtcp.h: No such file or directory
# Solution: Set DMTCP_ROOT correctly
export DMTCP_ROOT=/path/to/dmtcp/source
```

### Linking Errors
```bash
# Error: undefined reference to `cuCheckpointProcessLock'
# Solution: Ensure CUDA 12.4+ is installed and libcuda.so is accessible
ldconfig -p | grep libcuda
```

### Permission Issues
```bash
# Error: Permission denied accessing /dev/nvidia*
# Solution: Add user to appropriate groups
sudo usermod -a -G video,render $USER
# Log out and log back in
```

## Next Steps

After successful installation:

1. Read the [Quick Start Guide](quick-start.md) for your first checkpoint/restart test
2. Review the [Architecture Overview](../architecture/overview.md) to understand how CRAC works
3. Check the [Testing Guide](../contributor-guide/testing.md) for running the test suite

## See Also

- [Quick Start](quick-start.md) - First steps with CRAC
- [Development Setup](../contributor-guide/development-setup.md) - Setting up for development
- [Troubleshooting](../troubleshooting.md) - Common issues and solutions