# Quick Start

This guide walks you through your first checkpoint/restart experience with CRAC using a simple CUDA program.

## Prerequisites

Before starting, ensure you have:

- Completed the [Installation Guide](installation.md)
- CUDA 12.4+ and NVIDIA driver 550+ installed
- DMTCP built and available
- CRAC plugin compiled (`libdmtcp_crac.so`)

## Step 1: Start DMTCP Coordinator

The DMTCP coordinator manages checkpoint/restart operations. Start it in a dedicated terminal:

```bash
# Start coordinator on default port (7779)
$DMTCP_ROOT/bin/dmtcp_coordinator

# Or specify a custom port
$DMTCP_ROOT/bin/dmtcp_coordinator --port 7779
```

You should see output like:
```
dmtcp_coordinator starting...
Type '?' for help.
```

## Step 2: Launch a CUDA Program with CRAC

In a second terminal, launch the simple counter test program:

```bash
# Navigate to CRAC directory
cd /path/to/crac

# Launch with CRAC plugin
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter
```

Expected output:
```
0...
1...
2...
3...
...
```

The program is now running under DMTCP control with CRAC managing CUDA state.

## Step 3: Take a Checkpoint

In the coordinator terminal (first terminal), trigger a checkpoint:

```bash
# Type 'c' and press Enter, or use the command
c
```

You should see checkpoint progress:
```
(1): Checkpointing...
(1): Checkpointing GPU
finished checkpoining GPU
(1): Checkpoint complete.
```

In the program terminal, you'll see the program pause briefly during checkpointing, then resume:
```
15...
16...
17...
```

## Step 4: Verify Checkpoint Files

Check that checkpoint files were created:

```bash
ls -la ckpt_*.dmtcp
# Should show files like: ckpt_12345_abcdef.dmtcp
```

## Step 5: Kill and Restart the Program

### Kill the Running Program
In the program terminal, press `Ctrl+C` to stop the running process.

### Restart from Checkpoint
```bash
# Restart using the checkpoint files
$DMTCP_ROOT/bin/dmtcp_restart ckpt_*.dmtcp
```

Expected output:
```
Restarting...
16...
17...
18...
```

Notice that the counter continues from where it left off (16, 17, 18...), proving that the CUDA state was successfully checkpointed and restored.

## Step 6: Try MPI + CUDA (Optional)

If you have MPI installed, try the MPI+CUDA test:

```bash
# Launch with 2 processes
mpirun -np 2 $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter_mpi
```

Take a checkpoint as before, and you'll see both processes continue correctly after restart.

## Understanding What Happened

### During Checkpoint
1. **DMCP Coordinator** sends checkpoint signal to all processes
2. **CRAC Plugin** receives `DMTCP_EVENT_PRESUSPEND` event
3. **CUDA Lock**: `cuCheckpointProcessLock()` blocks new GPU work
4. **GPU Checkpoint**: `cuCheckpointProcessCheckpoint()` copies GPU memory to host
5. **State Save**: GPU process state becomes `CU_PROCESS_STATE_CHECKPOINTED`
6. **DMCP** saves process memory including GPU state

### During Restart
1. **DMCP** restores process memory from checkpoint files
2. **CRAC Plugin** receives `DMTCP_EVENT_RESTART` event
3. **GPU Restore**: `cuCheckpointProcessRestore()` restores GPU memory
4. **GPU Unlock**: `cuCheckpointProcessUnlock()` resumes GPU operations
5. **State Recovery**: GPU process state returns to `CU_PROCESS_STATE_RUNNING`

## Common First-Run Issues

### "CUDA checkpoint APIs not available"
```bash
# Check CUDA version
nvcc --version
# Must be 12.4 or later

# Check driver version
nvidia-smi
# Must be 550.54.14 or later
```

### "Permission denied accessing /dev/nvidia*"
```bash
# Add user to video group
sudo usermod -a -G video $USER
# Log out and log back in
```

### "Plugin failed to load"
```bash
# Check plugin path
ls -la ./libdmtcp_crac.so
# Should exist and be readable

# Check dependencies
ldd ./libdmtcp_crac.so
# Should show libcuda.so and libdl.so
```

### "Checkpoint failed"
```bash
# Check coordinator is running
$DMTCP_ROOT/bin/dmtcp_command --status

# Check logs in coordinator terminal
# Look for error messages
```

## Advanced Usage

### Automatic Checkpointing
Launch with automatic checkpointing every 10 seconds:
```bash
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so --interval 10 ./test/counter
```

### Custom Coordinator Port
```bash
# Terminal 1: Start coordinator on port 8888
$DMTCP_ROOT/bin/dmtcp_coordinator --port 8888

# Terminal 2: Launch program with custom port
$DMTCP_ROOT/bin/dmtcp_launch --coord-port 8888 --with-plugin ./libdmtcp_crac.so ./test/counter
```

### Checkpoint on Demand
```bash
# In another terminal, trigger checkpoint programmatically
$DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779
```

## Next Steps

Now that you've successfully used CRAC:

1. **Learn the Architecture**: Read the [Architecture Overview](../architecture/overview.md) to understand how CRAC works internally
2. **Run the Test Suite**: See the [Testing Guide](../contributor-guide/testing.md) for comprehensive testing
3. **Try Your Own Programs**: Apply CRAC to your CUDA applications
4. **Explore Development**: Read the [Development Setup](../contributor-guide/development-setup.md) if you want to contribute

## See Also

- [Installation Guide](installation.md) - If you haven't installed CRAC yet
- [Architecture Overview](../architecture/overview.md) - Understanding how CRAC works
- [Troubleshooting](../troubleshooting.md) - Solutions to common problems
- [Testing Guide](../contributor-guide/testing.md) - Running the test suite