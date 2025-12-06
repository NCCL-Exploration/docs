# Troubleshooting

This guide covers common errors, issues, and their solutions when using CRAC.

## Installation Issues

### CUDA Not Found

**Error**: `cuda.h: No such file or directory`

**Causes**:
- CUDA not installed
- CUDA headers not in standard location
- Incorrect CUDA version

**Solutions**:
```bash
# Check CUDA installation
nvcc --version
# Should show CUDA 12.4 or later

# Find CUDA headers
find /usr -name "cuda.h" 2>/dev/null
find /usr/local -name "cuda.h" 2>/dev/null

# Update Makefile if CUDA is in non-standard location
make CUDA_INCLUDE=-I/path/to/cuda/include DMTCP_ROOT=$DMTCP_ROOT
```

### DMTCP Headers Missing

**Error**: `dmtcp.h: No such file or directory`

**Causes**:
- DMTCP_ROOT not set correctly
- DMTCP not built/installed

**Solutions**:
```bash
# Set DMTCP_ROOT correctly
export DMTCP_ROOT=/path/to/dmtcp

# Verify DMTCP headers exist
ls $DMTCP_ROOT/include/dmtcp.h

# Build DMTCP if needed
cd /path/to/dmtcp
./configure
make
```

### Linking Errors

**Error**: `undefined reference to 'cuCheckpointProcessLock'`

**Causes**:
- CUDA version doesn't support checkpoint APIs
- libcuda.so not linked

**Solutions**:
```bash
# Check CUDA version
nvcc --version
# Must be 12.4 or later

# Check libcuda.so
ldconfig -p | grep libcuda

# Update Makefile to link CUDA libraries
make LDFLAGS="-lcuda -ldl" DMTCP_ROOT=$DMTCP_ROOT
```

## Runtime Issues

### Plugin Loading Failed

**Error**: `Failed to load plugin: libdmtcp_crac.so`

**Causes**:
- Plugin not found
- Missing dependencies
- Permission issues

**Solutions**:
```bash
# Check plugin exists and is readable
ls -la libdmtcp_crac.so

# Check dependencies
ldd libdmtcp_crac.so

# Use absolute path
dmtcp_launch --with-plugin /full/path/to/libdmtcp_crac.so ./test/counter
```

### CUDA Checkpoint APIs Not Available

**Error**: `CUDA_ERROR_NOT_SUPPORTED` from checkpoint APIs

**Causes**:
- CUDA driver version too old
- GPU doesn't support checkpointing
- CUDA runtime version mismatch

**Solutions**:
```bash
# Check driver version
nvidia-smi
# Must be 550.54.14 or later

# Check GPU support
cuda-checkpoint --help 2>/dev/null
# Should show help if supported

# Update NVIDIA driver
sudo apt-get install nvidia-driver-550
# or download from NVIDIA website
```

### Permission Denied on GPU Devices

**Error**: `Permission denied accessing /dev/nvidia*`

**Causes**:
- User not in appropriate groups
- Incorrect device permissions

**Solutions**:
```bash
# Add user to video group
sudo usermod -a -G video $USER
sudo usermod -a -G render $USER

# Log out and log back in
# Or use sudo for testing

# Check device permissions
ls -l /dev/nvidia*
# Should show group access
```

## Checkpoint Issues

### Checkpoint Hangs

**Symptom**: Process hangs during checkpoint, never completes

**Causes**:
- GPU work not completing
- Deadlock in application
- Insufficient memory

**Solutions**:
```bash
# Synchronize CUDA before checkpoint
cudaDeviceSynchronize();

# Check GPU utilization
nvidia-smi
# Look for high GPU utilization

# Increase timeout
dmtcp_coordinator --checkpoint-timeout 60

# Check system memory
free -h
# Ensure sufficient RAM for checkpoint
```

### Checkpoint Fails with Memory Error

**Error**: `CUDA_ERROR_OUT_OF_MEMORY` during checkpoint

**Causes**:
- Insufficient host memory for GPU state copy
- Memory fragmentation

**Solutions**:
```bash
# Check available memory
free -h

# Reduce GPU memory usage
# Allocate smaller buffers
# Free unused allocations

# Increase swap space
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### Checkpoint Fails with State Error

**Error**: `CUDA_ERROR_INVALID_STATE` from checkpoint APIs

**Causes**:
- GPU in unexpected state
- Previous checkpoint failed
- CUDA context issues

**Solutions**:
```bash
# Reset CUDA context
cudaDeviceReset();
cudaFree(0); // Force context creation

# Check current state
CUprocessState state;
cuCheckpointProcessGetState(getpid(), &state);
printf("Current state: %d\n", state);

# Restart application if in bad state
```

## Restart Issues

### Restart Fails to Find Checkpoint Files

**Error**: `Cannot open checkpoint file: ckpt_*.dmtcp`

**Causes**:
- Checkpoint files missing
- Wrong file permissions
- Incorrect working directory

**Solutions**:
```bash
# Check checkpoint files exist
ls -la ckpt_*.dmtcp

# Check file permissions
chmod 644 ckpt_*.dmtcp

# Use correct working directory
cd /path/to/checkpoint/files
dmtcp_restart ckpt_*.dmtcp
```

### Restart Fails with GPU Error

**Error**: `CUDA_ERROR_UNKNOWN` during GPU restore

**Causes**:
- GPU hardware changed
- Driver version mismatch
- Corrupted checkpoint data

**Solutions**:
```bash
# Verify same GPU hardware
nvidia-smi
# Compare with checkpoint time

# Check driver version consistency
nvidia-smi | grep "Driver Version"

# Try fresh checkpoint if data corrupted
rm ckpt_*.dmtcp
# Restart application and create new checkpoint
```

### Restart Fails with Pipe Recreation Error

**Error**: `Failed to recreate pipes` or assertion failure

**Causes**:
- Too many pipes for buffer size
- Pipe inode conflicts
- File descriptor conflicts

**Solutions**:
```bash
# Increase pipe buffer size in crac.cpp
#define MAX_PIPE_FDS 2048  // Increase from 1024

# Check for pipe leaks
lsof -p <pid> | grep pipe

# Reduce application pipe usage
# Close unused pipes
```

## Performance Issues

### Slow Checkpoint Times

**Symptom**: Checkpoint taking minutes instead of seconds

**Causes**:
- Large GPU memory allocations
- Slow storage for checkpoint files
- Network storage latency

**Solutions**:
```bash
# Check GPU memory usage
nvidia-smi

# Use local SSD for checkpoints
export DMTCP_CHECKPOINT_DIR=/tmp

# Reduce checkpoint size
# Free unused GPU memory
cudaFree(large_buffers);

# Profile checkpoint time
time dmtcp_command --checkpoint
```

### High Memory Usage

**Symptom**: System runs out of memory during checkpoint

**Causes**:
- GPU memory larger than available RAM
- Memory leaks in application
- Multiple processes checkpointing simultaneously

**Solutions**:
```bash
# Monitor memory usage
watch -n 1 'free -h && echo "---" && nvidia-smi'

# Limit concurrent checkpoints
export DMTCP_CHECKPOINT_INTERVAL=30

# Use swap space
sudo swapon -a

# Reduce GPU memory usage
# Use smaller data structures
```

## Debugging Techniques

### Enable Debug Output

**CRAC Debug**:
```cpp
// Add to crac.cpp for debugging
#define CRAC_DEBUG 1

#if CRAC_DEBUG
printf("DEBUG: %s:%d - message\n", __FILE__, __LINE__);
#endif
```

**DMTCP Debug**:
```bash
export DMTCP_DEBUG=checkpoint,restart
export DMTCP_LOG_TIMESTAMP=1

# Run with debug output
dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

**CUDA Debug**:
```bash
export CUDA_LAUNCH_BLOCKING=1
export CUDA_DEBUG=1

# Use cuda-memcheck
cuda-memcheck dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

### Use GDB with DMTCP

```bash
# Debug checkpoint process
gdb --args dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter

# In GDB
(gdb) break checkpoint_gpu
(gdb) break restore_gpu
(gdb) run

# Debug restart process
gdb --args dmtcp_restart ckpt_*.dmtcp
```

### Check System Logs

```bash
# Check kernel logs
dmesg | grep -i cuda
dmesg | grep -i dmtcp

# Check system logs
journalctl -u systemd-journald | grep -i cuda
tail -f /var/log/syslog
```

## Common Application-Specific Issues

### MPI Applications

**Issue**: MPI processes hang during checkpoint

**Solutions**:
```bash
# Use MPI-aware DMTCP
export DMTCP_MPI_SUPPORT=1

# Check MPI integration
mpirun -np 2 dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter_mpi

# Ensure all processes checkpoint simultaneously
dmtcp_command --checkpoint --all-processes
```

### Multi-GPU Applications

**Issue**: Checkpoint fails with multiple GPUs

**Current Limitation**: CRAC supports single GPU per process

**Workaround**:
```bash
# Use single GPU per process
export CUDA_VISIBLE_DEVICES=0
# or 1, 2, 3 for different processes

# Launch separate processes
dmtcp_launch --with-plugin ./libdmtcp_crac.so ./app_gpu0 &
dmtcp_launch --with-plugin ./libdmtcp_crac.so ./app_gpu1 &
```

### Applications with UVM

**Issue**: Checkpoint fails with Unified Virtual Memory

**Current Limitation**: UVM not supported by CUDA checkpoint APIs

**Workaround**:
```bash
# Disable UVM
export CUDA_DISABLE_UVM=1

# Use explicit memory management
cudaMalloc(&d_ptr, size);
cudaMemcpy(d_ptr, h_ptr, size, cudaMemcpyHostToDevice);
# Instead of UVM allocation
```

## Getting Help

### Collect Debug Information

When reporting issues, collect this information:

```bash
# System information
uname -a
lscpu
free -h

# CUDA information
nvidia-smi
nvcc --version
cat /usr/local/cuda/include/cuda.h | grep CUDA_VERSION

# DMTCP information
dmtcp_launch --version
echo $DMTCP_ROOT

# CRAC information
ls -la libdmtcp_crac.so
ldd libdmtcp_crac.so

# Error reproduction
# Steps to reproduce
# Full error messages
# Debug logs if available
```

### Known Limitations

- **UVM Support**: Not supported by current CUDA checkpoint APIs
- **Multi-GPU**: Single GPU per process only
- **IPC Memory**: Inter-process GPU memory not supported
- **Dynamic Parallelism**: Limited support
- **CUDA 12.4+**: Requires minimum CUDA 12.4

### Bug Reports

Report issues at: https://github.com/xu-yao0127/crac/issues

Include:
- System information (above)
- Complete error messages
- Steps to reproduce
- Expected vs actual behavior

## See Also

- [Installation Guide](../getting-started/installation.md) - Installation troubleshooting
- [Development Setup](../contributor-guide/development-setup.md) - Debugging setup
- [Testing Guide](../contributor-guide/testing.md) - Test-specific issues
- [Architecture Overview](../architecture/overview.md) - Understanding system behavior
- [Glossary](../reference/glossary.md) - Terminology reference